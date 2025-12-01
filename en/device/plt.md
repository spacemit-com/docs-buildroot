---
sidebar_position: 9
---

# Product Line Tool

This document describes product line tool.

## Introduction

The production test tool is a dedicated system for verifying hardware interface connectivity on PCBs or complete devices. Based on a customized Buildroot distribution, it integrates the factorytest application. The system typically runs from an SD card.

## Compilation

```shell
$ cd /path/to/buildroot-sdk
$ make envconfig
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_defconfig
  2. spacemit_k1_minimal_defconfig
  3. spacemit_k1_plt_defconfig


your choice (1-3):

```

Select option `3` and press Enter to start compiling.

Upon successful compilation, the system generates:

```shell
Images successfully packed into /path/to/bianbu-linux/output/k1_plt/images/bianbu-linux-k1_plt.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/path/to/bianbu-linux/output/k1_plt/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/path/to/bianbu-linux/output/k1_plt/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/path/to/work/bianbu-linux/output/k1_plt/images"" (stderr):
INFO: hdimage(sdcard.img): adding partition 'bootinfo' from 'factory/bootinfo_sd.bin' ...
INFO: hdimage(sdcard.img): adding partition 'fsbl' (in MBR) from 'factory/FSBL.bin' ...
INFO: hdimage(sdcard.img): adding partition 'env' (in MBR) from 'env.bin' ...
INFO: hdimage(sdcard.img): adding partition 'opensbi' (in MBR) from 'fw_dynamic.itb' ...
INFO: hdimage(sdcard.img): adding partition 'uboot' (in MBR) from 'u-boot.itb' ...
INFO: hdimage(sdcard.img): adding partition 'bootfs' (in MBR) from 'bootfs.img' ...
INFO: hdimage(sdcard.img): adding partition 'rootfs' (in MBR) from 'rootfs.ext4' ...
INFO: hdimage(sdcard.img): adding partition '[MBR]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT header]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT array]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT backup]' ...
INFO: hdimage(sdcard.img): writing GPT
INFO: hdimage(sdcard.img): writing protective MBR
INFO: hdimage(sdcard.img): writing MBR
Successfully generated at /path/to/bianbu-linux/output/k1_plt/images/bianbu-linux-k1_plt-sdcard.img
```

Output files:  
- `bianbu-linux-k1_plt.zip`: For Titan Flasher or fastboot flashing  
- `bianbu-linux-k1_plt-sdcard.img`: SD card image (flash using `dd` or [balenaEtcher](https://etcher.balena.io/))  

Default credentials:  
- Username: `root`  
- Password: `bianbu`  

## Customization

### Adding New Board Support

Currently supported boards:  
- deb1

The production system uses customized device trees rather than direct copies from U-Boot/kernel DTS.

Steps to add a new board:

1. Copy U-Boot DTS to:  
   `buildroot-ext/board/spacemit/k1/plt_dts/u-boot/` (e.g., `k1-x_deb2.dts`)  

2. Customize the DTS as needed  

3. Copy kernel DTS to:  
   `buildroot-ext/board/spacemit/k1/plt_dts/kernel/` (e.g., `k1-x_deb2.dts`)  

4. Update include paths in DTS files. Example:

   ```diff
   -#include "k1-x.dtsi"
   -#include "k1-x_pinctrl.dtsi"
   -#include "lcd/lcd_gx09inx101_mipi.dtsi"
   -#include "k1-x-hdmi.dtsi"
   -#include "k1-x-lcd.dtsi"
   -#include "k1-x-camera-sdk.dtsi"
   +#include "spacemit/k1-x.dtsi"
   +#include "spacemit/k1-x_pinctrl.dtsi"
   +#include "spacemit/lcd/lcd_gx09inx101_mipi.dtsi"
   +#include "spacemit/k1-x-hdmi.dtsi"
   +#include "spacemit/k1-x-lcd.dtsi"
   +#include "spacemit/k1-x-camera-sdk.dtsi"
   ```

5. Update other configurations

6. Modify `output/k1_plt/.config` to include new DTS for the new board type:

   ```diff
   -BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1"
   -BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts"
   -BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts"
   +BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1 k1-x_deb2"
   +BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb2.dts"
   +BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb2.dts"
   ```

7. Recompile U-Boot, Kernel, and firmware

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

8. Save configuration

   ```shell
   make savedefconfig
   ```

### Customizing rootfs

By customizing rootfs with `buildroot-ext/board/spacemit/k1/plt_overlay`，the files in this directory will be copied to the `output/k1_plt/target` directory before the image is made. 
