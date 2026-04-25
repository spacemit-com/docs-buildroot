---
sidebar_position: 9
---

# Product Line Tool

This document describes the product line tool.

## Introduction

The production test tool is a system used to verify hardware interface connectivity on boards or complete devices. Based on a customized Buildroot distribution, it integrates the factorytest application. The system typically runs from an SD card.

```
$ cd /path/to/buildroot-sdk
$ make envconfig
Available configs in buildroot-sdk/buildroot-ext/configs:
  1. spacemit_k3_ci_defconfig
  2. spacemit_k3_defconfig
  3. spacemit_k3_fpga_defconfig
  4. spacemit_k3_plt_defconfig

Your choice (1-4):

```

Select option `4` and press Enter to start compiling.

After the build is complete, you will see output similar to the following:

```
Images successfully packed into /k3_plt/images/Buildroot-k3_plt.zip


Generating sdcard image...................................
INFO: cmd: "mkdir -p "/k3_plt/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/k3_plt/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/k3_plt/images"" (stderr):
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'env' from 'env.bin' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'bootinfo' from 'factory/bootinfo_block.bin' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'fsbl' from 'factory/FSBL.bin' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'esos' from 'esos.itb' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'opensbi' from 'fw_dynamic.itb' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'uboot' from 'u-boot.itb' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'bootfs' (in MBR) from 'bootfs.img' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition 'rootfs' (in MBR) from 'rootfs.ext4' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition '[MBR]' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition '[GPT header]' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition '[GPT array]' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): adding primary partition '[GPT backup]' ...
INFO: hdimage(Buildroot-k3_plt-sdcard.img): writing GPT
INFO: hdimage(Buildroot-k3_plt-sdcard.img): writing protective MBR
INFO: hdimage(Buildroot-k3_plt-sdcard.img): writing MBR
INFO: cmd: "rm -rf "/k3_plt/build/genimage.tmp/"" (stderr):
Successfully generated at /k3_plt/images/Buildroot-k3_plt-sdcard.img

```

Where:
- `Buildroot-k3_plt.zip`: For Titan Flasher, or extract it and flash it with fastboot.
- `Buildroot-k3_plt-sdcard.img`: SD card firmware. After extraction, it can be written to an SD card using the `dd` command or [balenaEtcher](https://etcher.balena.io/).

Default credentials:
- Username: `root`
- Password: `bianbu`

## Customization

### Adding New Board Support

Currently, the product line tool supports the K3 deb1 and com260 boards. To support a new board, follow these steps:

1. Add the U-Boot configuration and kernel device tree for the board.

2. The source code for the product line tool is located in the `buildroot-sdk/package-src/factorytest` directory. Modify the source code for the target board.

3. Recompile U-Boot, the kernel, and the firmware package:

```shell
make uboot-rebuild
make linux-rebuild
make
```

### Customizing rootfs

You can customize the rootfs through `buildroot-ext/board/spacemit/k3/plt_overlay`. Files in this directory are copied to the `output/k3_plt/target` directory before the image is generated.
