---
sidebar_position: 14
---

# FAQs

## Components

### What components are needed to create a Linux distribution?

To create a Linux distribution, at a minimum, the following components need to be integrated for the system to run to the command line.

- [opensbi](https://gitee.com/spacemit-buildroot/opensbi)
- [uboot-2022.10](https://gitee.com/spacemit-buildroot/uboot-2022.10)
- [linux-6.1](https://gitee.com/spacemit-buildroot/linux-6.1) or [linux-6.6](https://gitee.com/spacemit-buildroot/linux-6.6)
- esos.elf

Among them, `esos.elf` is the firmware for the RCPU (Real-Time CPU), responsible for initializing some hardware modules and forwarding HDMI Audio interrupts. It is dependent on the Linux kernel and the system will not boot without it. It is released in the [buildroot-ext](https://gitee.com/spacemit-buildroot/buildroot-ext) repository, located at `board/spacemit/k1/target_overlay/lib/firmware/esos.elf`, and needs to be installed in the initramfs `/lib/firmware` directory.

To support GPU, the following components need to be integrated:

- [img-gpu-powervr](https://gitee.com/spacemit-buildroot/img-gpu-powervr)
- [mesa3d](https://gitee.com/spacemit-buildroot/mesa3d)

To support video hardware acceleration, the following components need to be integrated:

- [k1x-vpu-firmware](https://gitee.com/spacemit-buildroot/k1x-vpu-firmware)
- [mpp](https://gitee.com/spacemit-buildroot/mpp)
- FFmpeg
- GStreamer

The patches for FFmpeg and GStreamer are in the [buildroot](https://gitee.com/spacemit-buildroot/buildroot) repository, located at `package/ffmpeg` and `package/gstreamer1` respectively.
