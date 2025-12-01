---
sidebar_position: 3
---

# Buildroot 2.2 Release Notes

## v2.2.7 release note

Release date: 2025-8-13

Compared to 2.2.6, 2.2.7 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for AMD graphics cards
- Added support for sound card playback-only configuration
- Added support IME extensions for use by user programs
- Fixed the issue of abnormal use of mutex_unlock in rtl8852bs

### Build updates

- Build dependencies have been packaged into a container. Builds run in the container by default. To build on the host, set the environment variable `export DIRECT_BUILD=1`. (Note: when switching between container and host builds you must clean the output directory.)
- Added a set of commands to ease developing multiple solutions simultaneously; run `make help` for details.

## v2.2.6 release note

Release date: 2025-7-17

Compared to 2.2.4, 2.2.6 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for small memory solution
- Added support for spi nand boot solution
- Added support for the MUSE-Pi-Pro board-level LED mode as heartbeat mode
- Added support for remoteproc VIRTIO_F_ACCESS_PLATFORM functionality
- Fixed the issue of display freezing during hibernation wake-up aging
- Fixed the maximum execution jitter delay of emac
- Fixed the i2s system clock divider parameters to reduce sysclk jitter
- Fixed the issue where multiple sensors of the camera could not be powered on simultaneously
- Fixed the issue where some husb239 adapters were unable to negotiate 12V
- Fixed the warning of uninitialized lock in rtl8852bs rg_interface and the oops issue during stanby test

## v2.2.4 release note

Release date: 2025-6-25

Compared to 2.2.2, 2.2.4 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for MMC to adjust tx delaycode according to cpufreq to improve data transmission stability
- Added support for WOL wake-up function of the RTL8211F network interface card
- Added support for es8316/es8375 codec
- Added support for i2s dsp_a/b format
- Added support for MOTORCOMM PHY driver
- Fixed the issue that the rapid change of UART CTS state fails to generate an interrupt
- Fixed the retransmission issue caused by the probabilistic failure of i2c transmission
- Fixed the system panic issue caused by global out-of-bounds / slab out-of-bounds memory access in rtl8852bs
- Fixed the configuration length of GPU IO space
- Fixed the issue of random MAC addresses in some scenarios

## v2.2.2 release note

Release date: 2025-5-23

Compared to 2.1.1, 2.2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Fixed the issue of the remote login interface crashes during remote access

## v2.2.1 release note

Release date: 2025-5-15

Compared to 2.2, 2.2.1 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for Realtek network phy status detection function
- Added support for drm yuv444 format
- Added support for standard v4l2 via shared dmabuf
- Added support for spi high frequency clock phase calibration
- Fixed the out-of-bounds issue caused by v2d dmabuf memory exceeding 4GB
- Fixed the issue that the baud rate setting fails in some scenarios of UART
- Fixed the issue of chromium crashing on the bianbu cloud platform
- Fixed the issue that rtl8852bs wifi iperf scenario runs away
- Fixed the abnormal data transmission problem during the aic8800 wifi firmware download process

## v2.2 release note

Release date: 2025-4-23

Compared to 2.1, 2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for printing esos version function
- Added support for HIDRAW function
- Fixed the issue of hub initialization timing errors when usb asynchronous stanby
- Fixed the issue of occasional errors in USB flash
- Fixed the issue of spi0/spi1 dma transfer erros

## v2.2rc4 release note

Release date: 2025-4-11

Compared to 2.1, 2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for GPU DVFS
- Added support for RTL8125 and RTL8168 modules
- Added support for USB network sharing device function
- Added support for configure ddr priority on USB2.0
- Fixed the issue of dtb data exception
- Fixed the issue of uart driver drop frame
- Fixed low-probability display issues in system reboot scenarios
- Fixed display errors caused by frequent HDMI hot-swapping in some scenarios
- Fixed low-probability EMAC errors and the last CPU could not enter suspend during suspend/resume
- Fixed compatibility issues during sdio clock switching
