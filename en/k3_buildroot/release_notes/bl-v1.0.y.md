---
sidebar_position: 1
---

# Buildroot 1.0

## v1.0.5 Release Notes

Release Date: 2026-07-23

### Major Updates

- Fixed the NVMe timeout issue.
- Fixed the black screen and unavailable mouse and keyboard after resuming from suspend.
- Fixed the issue where TX delay did not take effect during SD/SDIO TX tuning.
- Fixed the load access fault issue when `remoteproc` obtains `get_loaded_rsc_table`.
- Fixed an issue where abnormal VPU messages could cause the decoder to hang.
- Fixed the cluster power-off failure when the rvtrace clock is enabled by default.
- Added support for PXE booting through the second boot device specified by TLV.
- Added support for RVA23 extension feature descriptions in DTS.
- Added support for using the SoC RTC as the clock source for RT-Linux.
- Optimized asynchronous PCIe resume to reduce resume time.
- Optimized the suspend flow by switching to a hardware-aware external interrupt mechanism.
- Optimized the I2C, GMAC, QSPI, and eSPI pinctrl drive strength on the PICO board.
- Optimized cpuidle to power off cluster3 synchronously when enabled.
- Optimized perf rvtrace by using an ELF-based method instead of `/dev/mem` access.

## V1.0.0 Release Notes

Release Date: 2026-04-30

### Highlights

#### Core Components

- OpenSBI 1.6
- U-Boot 2022.10
- Linux 6.18
- buildroot 2025.02.6
- img-gpu-powervr 24.2: GPU DDK
- mesa3d 24.04.1
- k3x-vpu-firmware: Video Process Unit firmware
- k3x-vpu-test: Video Process Unit test program
- k3x-cam: CSI Unit test program
- mpp: Media Process Platform
- FFmpeg 7.1.1 (with Hardware Accelerated)
- GStreamer 1.27.2 (with Hardware Accelerated)
- v2d-test: 2D Unit test program
- factorytest: factory test app

#### Major Drivers

**System drivers**

- clk
- pinctrl
- timer
- watchdog
- RTC
- DMA
- msgbox

**Interface drivers**

- USB 2.0/3.0
- PCIe 3.0
- UART
- I2C
- SPI
- PWM
- CAN

**Storage drivers**

- MMC (SD card/eMMC/SDIO)
- QSPI
- UFS

**Network drivers**

- GMAC
- WiFi
- BT

**Display drivers**

- DPU
- GPU
- MIPI DSI
- eDP/DP

**Multimedia drivers**

- VPU
- V2D
- V4L2
- CMOS sensor
- I2S

**Power management**

- cpufreq
- thermal
- PMIC

### Known Issues

- Suspend-to-RAM support is not yet fully stable.
