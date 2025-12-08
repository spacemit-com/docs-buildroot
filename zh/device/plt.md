---
sidebar_position: 9
---

# 产测工具

本文档介绍产测工具。

## 简介

产测工具是一个用于测试板卡或整机硬件接口连通性的系统，基于 Buildroot 裁剪，集成 factorytest 应用。该系统通常运行在 SD 卡上。

## 编译

```shell
$ cd /path/to/buildroot-sdk
$ make envconfig           
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_defconfig
  2. spacemit_k1_minimal_defconfig
  3. spacemit_k1_plt_defconfig


your choice (1-3): 

```

选择`3`，然后回车即开始编译。

编译完成，可以看到以下输出：

```shell
Images successfully packed into /path/to/buildroot-sdk/output/k1_plt/images/buildroot-k1_plt.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/path/to/buildroot-sdk/output/k1_plt/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/path/to/buildroot-sdk/output/k1_plt/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/path/to/work/buildroot-sdk/output/k1_plt/images"" (stderr):
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
Successfully generated at /path/to/buildroot-sdk/output/k1_plt/images/buildroot-k1_plt-sdcard.img
```

其中：
- `buildroot-k1_plt.zip` 适用于 Titan Flasher，或者解压后用 fastboot 刷机。
- `buildroot-k1_plt-sdcard.img` 为 SD 卡固件，解压后可以用 `dd` 命令或者 [balenaEtcher](https://etcher.balena.io/) 写入sdcard。

固件默认用户名：`root`，密码：`bianbu`。

## 定制

### 添加板型

已支持板型：

- deb1

产测工具的系统通常不直接使用板型在 U-Boot 和内核的 DTS，而是基于它定制。

添加新板型的步骤如下：

1. 将 U-Boot 的 DTS 拷贝到 `buildroot-ext/board/spacemit/k1/plt_dts/u-boot`，例如k1-x_deb2.dts

2. 根据需要对 DTS 文件进行定制。

3. 将内核的 DTS 拷贝到 `buildroot-ext/board/spacemit/k1/plt_dts/kernel`，例如k1-x_deb2.dts

4. 修改 DTS 文件中的 DTSI 路径，确保路径正确。例如：

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

5. 根据需要修改其他相关配置。

6. 修改 `output/k1_plt/.config`，添加新板型的 DTS。例如：

   ```diff
   -BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1"
   -BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts"
   -BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts"
   +BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1 k1-x_deb2"
   +BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb2.dts"
   +BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb2.dts"
   ```

7. 重新编译 U-Boot、内核和固件：

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

8. 保存配置

   ```shell
   make savedefconfig
   ```

### 定制 rootfs

通过 `buildroot-ext/board/spacemit/k1/plt_overlay` 定制 rootfs。该目录下的文件会在制作镜像前拷贝到 `output/k1_plt/target` 目录。
