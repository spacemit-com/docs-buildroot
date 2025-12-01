---
sidebar_position: 2
---

# Buildroot 2.0 Release Notes [End of Life]

Buildroot 2.0 will reach end of maintenance on July 31, 2025. We recommend using version 2.2 or later. If you have any questions, please contact us.

## v2.0.4 release note

Release date: 2024-12-11

### Major Updates

- Fixed the issue of illegal instruction with `openssl speed -evp chacha20`
- Fixed errors in certain scenarios with `rwsem/spinlock`
- Optimized PCIe enumeration time in u-boot
- Improved kernel boot speed
- Optimized PCIe memory range definition to enhance NVMe SSD read/write performance
- Updated GPU driver to v24.2
- Added support for GRUB

## v2.0.2 release note

Release date: 2024-11-11

### Major Updates

- u-boot supports non-AI extension cores as boot cores
- Supports Hynix LPDDR4x DRAM chips
- Supports DDR bandwidth statistics tool
- Supports es8323 audio codec chip
- Supports rt-linux patch
- Supports AMD graphics cards
- Fixed the issue where the system would freeze due to excessive interrupts from rtl8852bs after shutting down CPUs 1 to 7
- Fixed the issue where the PWM on gpio74 pin was not working properly
- Fixed the issue where Node.js would occasionally encounter illegal instructions
- Disabled the LOCKDEP option to avoid affecting system performance

## v2.1 release note

Release date: 2024-10-28

### Major Updates

- Fix warning messages after enabling kernel LOCKDEP options
- [MUSE Book] Fix the issue of abnormal wake-up from suspend

## v2.0 release note

Release date: 2024-10-22

### Major Updates

- Fix u-boot keyboard compatibility issues
- Fix the issue of no bootlogo when SPINOR boots eMMC
- Add NFS boot support to u-boot
- Fix several kernel Oops issues
- Temporarily fix kernel Hung issues in certain scenarios

## v2.0rc7 release note

Release date: 2024-9-30

### Major Updates

- Fix the issue of card mass production failure
- Fix the memory out-of-bounds access issue in the rtl8852bs driver
- Fix the error in setting gpio in the rf driver
- Fix the bootlogo flickering issue
- Update sdio delay parameters to improve sdio stability
- Add ADC functionality for the P1 chip

## v2.0rc6 release note

Release date: 2024-9-10

### Major Updates

- [MUSE Book] Optimize screen wakeup speed

## v2.0rc5 release note

Release date: 2024-9-2

### Major Updates

- Update rcpu firmware

## v2.0rc4 release note

Release date: 2024-8-29

### Major Updates

- Support u-boot PWM control of LCD backlight
- Support u-boot fuel gauge framework
- Support MIPI DSI display jd9365dah3
- Support Type-C chip husb239
- Fix u-boot EFI free memory issue
- Fix u-boot low probability LCD screen flickering issue
- Fix I2C abnormality during shutdown
- Optimization of kernel module loading speed

## v2.0rc3 release note

Release date：2024-8-10

### Features

#### Main components

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.6.36
- buildroot 2023.02.9
- onnxruntime 1.15.1
- img-gpu-powervr 23.2: GPU DDK
- mesa3d 22.3.5
- QT 5.15 (with GPU enabled)
- k1x-vpu-firmware: Video Process Unit firmware
- k1x-vpu-test: Video Process Unit test program
- k1x-jpu: JPEG Process Unit API
- k1x-cam: CMOS Sensor and ISP API
- mpp: Media Process Platform
- FFmpeg 4.4.4 (with Hardware Accelerated)
- GStreamer 1.22.8 (with Hardware Accelerated)
- v2d-test: 2D Unit test program
- factorytest: factory test app

#### Main drivers

**System**

- clk
- pinctrl
- timer
- watchdog
- RTC
- DMA
- msgbox
- spinlock
- TCM

**Interface**

- USB 2.0/3.0
- PCIe 2.1
- UART
- I2C
- SPI
- PWM
- CAN

**Storage**

- MMC (sdcard/eMMC/SDIO)
- SPI NOR
- NVMe SSD

**Network**

- GMAC
- WiFi
- BT

**Display**

- DPU
- GPU
- MIPI DSI
- HDMI 1.4

**Media**

- VPU
- JPU
- V2D
- V4L2
- CMOS Sensor
- ISP
- I2S
- HDMI Audio

**Power**

- cpufreq
- thermal
- PMIC

**Crypto**

- AES
