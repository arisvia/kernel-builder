# Android 16 on Linux 4.19.325 (polaris / SDM845) Kernel Bring-up Handoff

## Final Verified Status (2026-10-22)
- **Kernel Version**: `Linux version 4.19.325-BakaSU-polaris (kernel@build) #1 SMP PREEMPT Thu Oct 22 01:06:54 UTC 2026` (Clean, NO trailing '+' sign!).
- **Bootloader**: `sys.boot_completed=1`, `ro.boot.bootreason=bootloader`.
- **Root**: `BakaSU` (KSU subsystem) fully operational (`uid=0(root) gid=0(root) context=u:r:ksu:s0`).
- **Power & Charging**:
  - `POWER_SUPPLY_PROP_CHARGING_ENABLED` exposed in `drivers/power/supply/qcom/qpnp-smb2.c` and mapped to `chg_disable_votable`.
  - Fixes LineageHealth HAL / Android 16 ChargingControl from falling back to `input_suspend` and cutting VBUS input power.
  - VBUS stays online continuously (`USB powered: true`), lockscreen fast-charging animations and status bar lightning bolt indicators 100% operational.
  - `CONFIG_QPNP_QNOVO5=y` enabled in `polaris_defconfig`.
- **BPF Subsystem**: 100% operational for Android 16 (API 36). All maps and progs in tethering apex, `netd.o` (`BPF_MAP_TYPE_SK_STORAGE`, `BPF_MAP_TYPE_RINGBUF`, `PTR_TO_MEM`), and `filterPowerSupplyEvents.o` load cleanly. `bpfloader` exits with `status 0`.
- **Sensors / SLPI**: Subsystem PIL mailbox channel probe deferral resolved via fallback in `qmp_edge_init()`. SLPI firmware (`slpi.mdt`, `slpi.bXX`) loads at boot. 36/36 hardware sensors active and reporting.
- **Congestion Control & Qdisc**: `CONFIG_TCP_CONG_ADVANCED=y`, `CONFIG_TCP_CONG_BBR=y`, `CONFIG_DEFAULT_TCP_CONG="bbr"`. `CONFIG_NET_SCH_FQ=y` enabled (`CONFIG_DEFAULT_NET_SCH="fq"`).
- **I/O Scheduler**: `CONFIG_DEFAULT_DEADLINE=y`, `CONFIG_DEFAULT_IOSCHED="deadline"`, `CONFIG_MQ_IOSCHED_DEADLINE=y`. Legacy `CONFIG_IOSCHED_CFQ` completely disabled to prevent Android vendor init from forcing CFQ on UFS storage.
- **Defconfig Hygiene**: Cleaned 7 undefined ghost symbols (`CONFIG_TOUCHSCREEN_COMMON`, `CONFIG_INPUT_FINGERPRINT`, `CONFIG_MFD_SPK_ID`, `CONFIG_LEDS_QPNP_HAPTICS`, `CONFIG_QPNP_COINCELL`, `CONFIG_MSM_SPCOM_LEGACY`, `CONFIG_DRM_SDE_RSC`).
- **Device Tree**: Preserved all 21 compiled SDM845 board DTBs in `Image.gz-dtb` for universal retail hardware revision and reference board compatibility.
- **CI / Build System**: ThinLTO cache (`~/.thinlto-cache`) via `ld-wrapper`, ccache 5G. Build time ~8.5 min. Dual output: fastboot standalone `boot.img` + AnyKernel3 recovery zip.
