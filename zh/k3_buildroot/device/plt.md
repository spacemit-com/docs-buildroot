---
sidebar_position: 9
---

# 产测工具

本文档介绍产测工具。

## 简介

产测工具是一个用于测试板卡或整机硬件接口连通性的系统，基于 Buildroot 裁剪，集成 factorytest 应用。该系统通常运行在 SD 卡上。

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

选择`4`，然后回车即开始编译。

编译完成，可以看到以下输出：

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

其中：
- `Buildroot-k3_plt.zip` 适用于 Titan Flasher，或者解压后用 fastboot 刷机。
- `Buildroot-k3_plt-sdcard.img` 为 SD 卡固件，解压后可以用 `dd` 命令或者 [balenaEtcher](https://etcher.balena.io/) 写入sdcard。

固件默认用户名：`root`，密码：`bianbu`。

## 定制

### 添加板型

目前产测工具已支持k3 deb1和com260板型，如果要支持新的板型步骤如下：

1. 增加板型Uboot和内核设备树。

2. 产测工具源码位于`buildroot-sdk/package-src/factorytest`目录，针对目标板型修改源码。

3. 重新编译U-Boot、内核和打包固件：

```shell
make uboot-rebuild
make linux-rebuild
make
```

### 定制 rootfs

通过 `buildroot-ext/board/spacemit/k3/plt_overlay` 定制 rootfs。该目录下的文件会在制作镜像前拷贝到 `output/k3_plt/target` 目录。
