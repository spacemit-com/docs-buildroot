---
sidebar_position: 12
---

# FAQs

## Components

### What components are needed to create a Linux distribution?
To create a Linux distribution, at a minimum, the following components need to be integrated for the system to boot to the command line.

- [opensbi](https://git.spacemit.com/buildroot-k3/opensbi)
- [uboot-2022.10](https://git.spacemit.com/buildroot-k3/uboot)
- [linux-6.18](https://git.spacemit.com/buildroot-k3/linux-6.18)
- esos

Among them, `esos` is the firmware for the RCPU (Real-Time CPU), responsible for initializing some hardware modules and forwarding HDMI audio interrupts. It depends on the Linux system, and the system will not boot without it. It is released in the [esos](https://git.spacemit.com/buildroot-k3/esos) and [esos-lite](https://git.spacemit.com/buildroot-k3/esos-lite) repositories. Compiling `esos` generates the `rt24_os0_rcpu.elf` and `rt24_os1_rcpu.elf` files, which need to be installed in the `/lib/firmware` directory of the initramfs.

To support GPU, the following components need to be integrated:

- [img-gpu-powervr](https://git.spacemit.com/buildroot-k3/img-gpu-powervr)
- [mesa](https://git.spacemit.com/buildroot-k3/mesa)

To support video hardware acceleration, the following components need to be integrated:

- [k3x-vpu-firmware](https://git.spacemit.com/buildroot-k3/k3x-vpu-firmware)
- [mpp](https://git.spacemit.com/buildroot-k3/mpp)
- FFmpeg
- GStreamer

The patches for FFmpeg and GStreamer are in the [buildroot](https://git.spacemit.com/buildroot-k3/buildroot) repository, located in `package/ffmpeg` and `package/gstreamer1`, respectively.
