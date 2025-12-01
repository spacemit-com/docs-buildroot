---
sidebar_position: 1
---

# Buildroot 1.0 Release Notes [End of Life]

## Buildroot 1.0 End of Life Announcement
 Buildroot V1.0, which was built on linux-6.1.Please note that Buildroot V1.0 is no longer maintained and has reached its End of Life (EOL).
 
Buildroot 1.0 End of Life (EOL) Time: 2024/12/31

Please refer to the document for the Buildroot V2.0 upgrade

## v1.0.15 release note

Release data: 2024-9-7

### Major Updates

- Update aic8800 firmware path

## v1.0.14 release note

Release data: 2024-8-31

### Major Updates

- Support u-boot PWM control of LCD backlight
- Support u-boot fuel gauge framework
- Support MIPI DSI display jd9365dah3
- Support Type-C chip husb239
- Fix u-boot EFI free memory issue
- Fix u-boot low probability LCD screen flickering issue
- Fix SPINOR partitions create failed issue
- Update rcpu firmware
- Optimization of kernel module loading speed

## v1.0.13 release note

Release data: 2024-8-16

### Major Updates

- Supports using `inxi` to get the CPU temperature
- Fixes SSD compatibility issues in u-boot
- Fixes an error that occurs when more than 16 bootarg parameters are used in u-boot env
- Fixes an crash issue when CAN transmission at 8M baud rate
- [MUSE Book] Fixes power key event  error when LID switching frequently
- [MUSE Book] Supports ADB

## v1.0.12 release note

Release data: 2024-8-2

### Major Updates

- Support aic8800 WiFi module

## v1.0.11 release note

Release data: 2024-8-1

### Major Updates

- Support EEPROM programming with part number
- Support power on/off for rtl8852be on suspend
- Update pin-up and pull-down states
- Fix PWM clock configuration error under u-boot
- Fix GMAC Phy failing to enter low-power mode on suspend
- Fix abnormal behavior of WiFi and GMAC functions when CPU runs at `1.8GHz@1.16v`
- Fix Samsung 970pro 1T SSD being recognized as 1 lane
- [MUSE Book] Fix the issue where battery level shows only 99% after fully charged
- [MUSE Book] Improve wake-up speed to within 2 seconds after suspend

## v1.0.9 release note

Release data: 2024-7-20

### Major Updates

- Fixes an issue where MIPI DSI screen JD9365DA displays nothing
- [MUSE Book] Resolves a low-probability issue of no display during boot or flashing
- [MUSE Book] Fixes a low-probability issue of flickering logo during boot
- [MUSE Book] Fix the issue where battery level shows only 99% even after fully charged
- [MUSE Book] Fixes a low-probability issue of immediate wakeup after suspend and abnormal (high) power consumption during suspend

## v1.0.8 release note

Release data: 2024-7-16

### Major Updates

- Update the RTL8852BU driver and firmware to fix bluetooth related issues.
- Resolve the issue of no sound output on select HDMI displays.
- Address the issue of abnormal sound after waking up from suspend mode in certain scenarios.
- [MUSE Card Support] Enable compatibility with MUSE Card.

## v1.0.7 release note

Release data: 2024-7-11

### Major Updates

- Supports more CPU frequency scaling policies
- Enables WiFi/USB wake-up capabilities
- Allows checking chip names, version numbers, and more
- Fixes an issue with incorrect PWM frequency configuration
- Resolves an issue where some 16GB DDR modules were inaccessible
- Fixes a low-probability error in headphone detection
- Addresses a low-probability issue of no sound after waking up from sleep
- Resolves an extremely rare kernel panic issue with rtl8852be
- [BPI-F3] Fixes an issue where touchpac was not functioning
- [MUSE Book] Resolves an issue where u-boot could not be updated online
- [MUSE Book] Fixes a low-probability issue of flickering boot logo

## v1.0.5 release note

Release data: 2024-6-28

### Major Updates

- u-boot supports CPU `1.6GHz@1.05v`
- Supports LPDDR4 type DDR particles
- Supports ota7290b MIP screen
- Supports es7210/es8516 audio codec
- Fixes audio playback noise issues
- Fixes HDMI EDID acquisition defects, supporting complete 256 byte EDID
- Fixes Crypto non-16 byte alignment processing errors
- Fixes rtl8852be sleep and wakeup anomalies
- Fixes u-boot SPINOR low probability erase and write failure issues
- Fixes u-boot Sandisk 128GB eMMC HS400 mode read-write errors
- Fixes u-boot NVMe SSD read-write issues under certain conditions
- [MUSE Box] Supports DP plug-in and pull-out dynamic detection
- [MUSE Book] Fixes probabilistic screen glitches during startup and wakeup
- [MUSE Book] Fixes probabilistic inability to wake up the screen from sleep
- [MUSE Book] Fixes camera high-resolution lag issues
- [MUSE Book] Optimizes eDP screen initialization speed
- [MUSE Book] Optimizes eDP screen backlight brightness curve

## v1.0.3 release note

Release date: 2024-6-19

### Major Updates

- Support for codec chip es8316 on Linux 6.1
- u-boot supporting SPI drivers
- Fixed the issue of not waking up after long-term sleep
- Fixed the low-probability issue of sound card creation failure
- Fixed the low-probability issue of i2c communication timeout
- Fixed the issue of pcie 2 lane not adaptively supporting 1 lane
- Fixed the low-probability CRC error issue of SDIO WiFi under low temperature
- [BPI-F3] Supporting fan cooling function
- [MUSE Pi] Supporting MIPI-CSI Sensor
- [MUSE Box] Supporting sound card headset and Mic detection function
- [MUSE Box] Supporting DP display function (Note: Not supporting hot swapping)
- [MUSE Book] Supporting logo display during u-boot phase
- [MUSE Book] Updating production testing tools
- [MUSE Book] Fixed the issue of not being able to obtain the display EDID

## v1.0 release note

Release date: 2024-5-30

### Features

#### Main components

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.1
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

### Know issue

- Suspend to ram is not ready
