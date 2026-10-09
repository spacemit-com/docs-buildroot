---
sidebar_position: 1
---

# Buildroot 1.0

## v1.0.9 Release Notes

Release Date: 2026-09-18

### Major Updates

- Added support for K3 secure firmware (OP-TEE) build
- Added support for ROS-related HID sensor configuration
- Added support for Xbox controllers
- Added support for optional LPDDR5 and LPDDR4X compilation
- Fixed C3 warm-up to avoid CPU clock FC timeout issue
- Fixed potential Load address misaligned exception in ccu_mix.c
- Adjusted debian packaging method for opensbi/esos/u-boot
- Optimized bootloader startup speed

## v1.0.8 Release Notes

Release Date: 2026-09-16

### Major Updates

- Added support for FM25LQ128I3 spi-nor flash
- Added support for imx219 camera spacemit,dt-filter
- Added support for esos debian package compilation
- Fixed K3 Pico-itx fan control stability
- Fixed USB hub port reset with added TRSTRCY recovery delay
- Fixed SPL stage P1 non-volatile register value always being 0xf0
- Fixed PCIe K3 deinit function

## v1.0.7 Release Notes

Release Date: 2026-08-26

### Major Updates

- Added support for RISC-V64 hardware cryptographic acceleration
- Added support for reading K3 chip ID
- Fixed PCIe link training failure during wake-up recovery
- Fixed MIPI DSI panel not switching to DSI function (default routed to DP/eDP causing display anomalies)
- Fixed duplicate EDID mode removal issue
- Fixed UFS spacemit mixed read/write serialization
- Fixed fastboot oem_read truncation issue at offsets >4GB
- Fixed DDR reg_base mask boundary validation
- Fixed USB hub port reset recovery delay (added TRSTRCY)
- Optimized K3 OPP table to use efuse calibration values for thermal temperature reference

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
