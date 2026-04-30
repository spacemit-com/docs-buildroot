---
sidebar_position: 1
---

# Buildroot 1.0

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
- k3x-cam: CSI Unit test progrom
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
