# Xiaomi Mi Mix 2S (polaris) - Qualcomm SDM845 Linux 4.19 Kernel

> **Target Platform**: Android 16 (API 36 / QPR2 / LineageOS 22+ / Axion OS 2.8)  
> **Kernel Version**: `4.19.325-BakaSU-polaris`  
> **Toolchain**: AOSP Clang 20.0.0 (`clang-r547379`) + Pure `LLVM=1` + ThinLTO  
> **Status**: Production Ready / Fully Verified on Hardware

---

## 1. 核心架构演进与关键问题解决方案

### 1.1 eBPF 子系统全面升级（解决 Android 16 `bpfloader` 闪退）
Android 16 将 eBPF 运行要求提升到与 5.10+ 对齐的水平，原生 4.19 会在 `netd.o` 加载时失败。
- **BPF_MAP_TYPE_SK_STORAGE (Type 24)**:
  - 移植 `include/net/bpf_sk_storage.h` 与 `net/core/bpf_sk_storage.c`，使用 `map->pages` 和 `bpf_map_precharge_memlock` 进行内存计费。
  - 在 `include/net/sock.h` 中通过 `ANDROID_KABI_USE(2, ...)` 注入 `sk_bpf_storage` 指针，确保 struct 尺寸完全不破坏 KABI。
  - 在 `net/core/sock.c` 的 `__sk_destruct` 中调用 `bpf_sk_storage_free`，并在 `sk_clone_lock` 中显式置空防止子套接字共享父存储。
  - 注册 `BPF_FUNC_sk_storage_get` (107) 与 `BPF_FUNC_sk_storage_delete` (108)。
- **BPF_MAP_TYPE_RINGBUF (Type 27)**:
  - 移植双重映射页（Double-mapped Data Pages）环形缓冲区 `kernel/bpf/ringbuf.c`，实现 `map_mmap` 与 `map_poll`。
  - 注册 Ringbuf 核心辅助函数（130-134）：`ringbuf_output`, `ringbuf_reserve`, `ringbuf_submit`, `ringbuf_discard`, `ringbuf_query`。
- **Verifier (验证器) PTR_TO_MEM 语义支持**:
  - `ringbuf_reserve` 返回具有安全边界的指针。在 `verifier.c` 和 `bpf_verifier.h` 中回移植 `PTR_TO_MEM` / `PTR_TO_MEM_OR_NULL` 寄存器状态与 `mem_size` 边界校验，解决程序通过 R0 写入 Sample 数据被误判为非指针访问的问题。
- **cgroup sockopt & release 支持**:
  - 支持 `BPF_PROG_TYPE_CGROUP_SOCKOPT` (Type 25)、`BPF_CGROUP_INET_SOCK_RELEASE`，并在 `sock_filter_func_proto` 中注册 `bpf_get_socket_cookie`。

---

### 1.2 传感器与 SLPI 通信（解决 QMP AOP 邮箱探测死锁）
- **现象**: 引导后 `sscrpcd` 与 `android.hardware.sensors@1.0-service` 崩溃，所有 36 个传感器离线，PIL 驱动持续报 `subsys-pil-tz ... probe deferral (-517)`。
- **根因**: `drivers/mailbox/msm_qmp.c` 中，设备树 `qcom,qmp-aop` 节点未指定 `mboxes` 属性，导致 `mbox_request_channel()` 返回 `-ENOENT (-2)`。原代码仅检查了 `-ENODEV (-19)`，导致 probe 失败退出而未进入 IPC 寄存器回退路径。
- **修复**:
  ```c
  if (PTR_ERR(mdev->mbox_chan) != -ENODEV && PTR_ERR(mdev->mbox_chan) != -ENOENT)
  ```
  放行 `-ENOENT` 顺利进入 `qmp_parse_ipc()`，SLPI 固件（`slpi.mdt`, `slpi.bXX`）正常加载，36/36 硬件传感器全绿。

---

### 1.3 充电控制与 VBUS 电源路径（解决 Android 16 充电无动画与无反馈）
- **现象**: 插上充电器有物理电流，但系统状态始终显示 `Discharging`，锁屏无全屏动画，状态栏无闪电图标，`healthd` 报 `st=3 chg=`。
- **根因**:
  - LineageOS / Axion OS 的 `vendor.lineage.health-service.default` 内置充电保护控制（`ChargingControl`，夜间限制 80%）。
  - 该服务优先探测 `/sys/class/power_supply/battery/charging_enabled`，如果不存在则回退至 `/sys/class/power_supply/battery/input_suspend`。
  - 内核 `qpnp-smb2.c` 未暴露 `charging_enabled`，导致服务回退锁死 `input_suspend=1`，物理 VBUS 供电路径被掐断（强制 0mA），触发 `USB powered: false`。
- **修复**:
  - 在 `drivers/power/supply/qcom/qpnp-smb2.c` 的 `smb2_batt_props` 中注册 `POWER_SUPPLY_PROP_CHARGING_ENABLED`。
  - Getter 映射到 `!get_effective_result(chg->chg_disable_votable)`。
  - Setter 映射到 `vote(chg->chg_disable_votable, USER_VOTER, !val->intval, 0)`。
  - 标记该属性在 `smb2_batt_prop_is_writeable` 中可写。
  - 开启 `CONFIG_QPNP_QNOVO5=y`。
  - 结果：LineageHealth 正确选中 `charging_enabled`，仅控制流入电池的电流，VBUS 始终在线（`USB powered: true`），全屏充电动画与闪电指示 100% 恢复。

---

## 2. 现代性能与调度调优

| 优化维度 | 关键配置 / 补丁 | 收益说明 |
|---|---|---|
| **网络调度** | `CONFIG_TCP_CONG_BBR=y`<br>`CONFIG_NET_SCH_FQ=y`<br>`CONFIG_DEFAULT_NET_SCH="fq"` | 默认全局启用 BBR + FQ 发包调步，消除高延迟网络下的丢包与重传，提升网络吞吐。 |
| **闪存 I/O** | `CONFIG_DEFAULT_DEADLINE=y`<br>`# CONFIG_IOSCHED_CFQ is not set` | 彻底移除针对机械硬盘的 CFQ 调度器，防止 Android vendor init 强制写入 cfq，确保 UFS 闪存以最低延迟运行在 `deadline`。 |
| **内存页回收** | `CONFIG_LRU_GEN=y`<br>`CONFIG_LRU_GEN_ENABLED=y` | 移植 Google MGLRU 多代页回收算法（运行态 `0x0003`），精准区分热页与冷页，极大减少高内存压力下的 UI 掉帧与后台被杀。 |
| **内存压缩** | `CONFIG_CRYPTO_LZ4=y`<br>`CONFIG_CRYPTO_ZSTD=y` | 内置高速 `lz4` 与高压缩率 `zstd` 算法，实测 2GB ZRAM 获得高达 3.53:1 压缩比。 |
| **Root 驱动** | `CONFIG_KSU=y`<br>`CONFIG_KSU_MANUAL_HOOK=y` | 内置最新 BakaSU（KernelSU）4.2+ 驱动，完整支持 Android 16 超级用户权限。 |

---

## 3. GitHub Actions CI 极速构建工作流

- **瘦身与构建缓存**:
  - ThinLTO: 封装 `ld.lld` 注入 `--thinlto-cache-dir=$HOME/.thinlto-cache`，配合 actions 缓存将链接瓶颈降低 60%。
  - ccache: 设置 `CCACHE_COMPILERCHECK=none` 与 `CCACHE_NOHASHDIR=true`，整机全量构建从 21 分钟缩短至 **8 分钟 30 秒**。
- **纯净版本号**:
  - 内核源码根目录创建空 `.scmversion`，并在 make 参数中传递 `LOCALVERSION=""`，彻底消灭内核版本末尾恼人的 `+` 号。
- **双产物直出**:
  - 流水线不仅生成 Recovery 专用的 `AnyKernel3` 卡刷包，还会从 `BASE_BOOT_URL` 下载原厂 boot.img 并通过 `magiskboot` 在 Runner 上直接压合出免解包的 Fastboot 线刷镜像 `BakaSU-polaris-boot-*.img`。

---

## 4. 关键验证指令手册

```bash
# 1. 验证内核版本与干净后缀
adb shell "uname -r"
# 预期: 4.19.325-BakaSU-polaris (无 + 号)

# 2. 验证 BPF 运行时状态
adb shell "getprop init.svc.bpfloader; getprop bpf.progs_loaded"
# 预期: stopped \n 1

# 3. 验证充电控制与 VBUS 状态
adb shell "su -c 'cat /sys/class/power_supply/battery/charging_enabled; cat /sys/class/power_supply/battery/input_suspend; dumpsys battery | grep powered'"
# 预期: 1 (或 0 如果处于保护中), input_suspend 恒为 0, USB powered: true

# 4. 验证硬件传感器
adb shell "dumpsys sensorservice | head -10"
# 预期: Total 36 h/w sensors, 36 running 0 disabled clients

# 5. 验证 I/O 调度器与网络 FQ
adb shell "cat /sys/block/sda/queue/scheduler; cat /proc/sys/net/core/default_qdisc"
# 预期: noop [deadline], fq
```
