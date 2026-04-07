---
sidebar_position: 2
---

# 方案管理

本文档介绍SDK如何管理方案（solution），包括方案的配置文件，如何定制方案和如何添加新方案等。

## 方案的配置文件

以默认方案K3为例，通常包含以下配置文件：

```shell
buildroot-ext/board/spacemit/k3/partition_*.json
buildroot-ext/board/spacemit/k3/env_k3.txt
buildroot-ext/board/spacemit/k3/bianbu.bmp
buildroot-ext/board/spacemit/k3/dracut.conf
buildroot-ext/board/spacemit/k3/target_overlay
buildroot-ext/configs/spacemit_k3_defconfig
bsp-src/opensbi/platform/generic/configs/k3_defconfig
bsp-src/uboot-2022.10/configs/k3_defconfig
bsp-src/linux-6.18/arch/riscv/configs/k3_bianbu_defconfig
```

**buildroot-ext/board/spacemit/k3/partition_*.json**

分区配置文件：

- `partition_8M.json`：for 8MB flash
- `partition_64M.json`：for 64MB flash
- `partition_universal.json`：for high capacity flash，例如sdcard、SSD、UFS

**buildroot-ext/board/spacemit/k3/env_k3.txt**

u-boot环境变量，可定制启动参数。

**buildroot-ext/board/spacemit/k3/bianbu.bmp**

u-boot启动logo：

- 格式：BMP
- 分辨率：小于或等于屏幕分辨率
- 位深度：32

**buildroot-ext/board/spacemit/k3/dracut.conf**

Dracut的配置文件，Dracut是一个制作initramfs镜像的工具。

**buildroot-ext/board/spacemit/k3/target_overlay**

顾名思义，该目录是对target目录的override。

**buildroot-ext/board/spacemit/k3/prepare_img.sh**

K3方案的镜像打包脚本，buildroot编译的最后一个步骤会调用该脚本。

**buildroot-ext/configs/spacemit_k3_defconfig**

Buildroot的配置文件。

**bsp-src/opensbi/platform/generic/configs/k3_defconfig**

opensbi的配置文件。

**bsp-src/uboot-2022.10/configs/k3_defconfig**

u-boot的配置文件。

**bsp-src/linux-6.18/arch/riscv/configs/k3_bianbu_defconfig**

kernel的配置文件。

## 默认分区配置

尽量不要修改`bootfs`前面的分区配置，避免引导代码中有hard code，修改之后无法正常引导。

### sdcard分区配置

完全按`partition_universal.json`文件`partitions`数组配置。

分区配置如下：

```
----------------------------------------------------------------------------------------------
|gpt    |   env   |  bootinfo  | fsbl | esos | opensbi | uboot |    bootfs    |    rootfs    |
----------------------------------------------------------------------------------------------
^       ^         ^            ^      ^      ^         ^       ^              ^              ^
|       |         |            |      |      |         |       |              |              |
0K      640K      1M          1536K   4M     7M        8M      12M            268M        Capacity
```

### SPINOR + SSD/UFS分区配置

SSD或者UFS方案，需要用SPINOR引导，bootfs之前的分区都存储在SPINOR，按`partition_8M.json`文件`partitions`数组配置。

SPINOR分区配置如下：

```
-----------------------------------------------------
|bootinfo   | fsbl | env |  esos  | opensbi | uboot |
-----------------------------------------------------
^           ^      ^     ^        ^         ^       ^
|           |      |     |        |         |       |
0K          128K   640K  704K     1728K     2112K  Capacity
```

bootfs和rootfs存储在SSD/UFS，按`partition_universal.json`文件`partitions`数组配置。

```
----------------------------------------------------------------------------------------------
|gpt    |   env   |  bootinfo  | fsbl | esos | opensbi | uboot |    bootfs    |    rootfs    |
----------------------------------------------------------------------------------------------
^       ^         ^            ^      ^      ^         ^       ^              ^              ^
|       |         |            |      |      |         |       |              |              |
0K      640K      1M          1536K   4M     7M        8M      12M            268M        Capacity
```

## 定制方案

以下内容说明如何修改uboot环境变量、分区信息、根文件系统。

### 修改uboot环境变量

K3方案可以修改`buildroot-ext/board/spacemit/k3/env_k3.txt`文件来定制uboot环境变量。

### 定制分区信息

K3方案可以修改`buildroot-ext/board/spacemit/k3/partition_universal.json`文件来定制分区，可根据实际需求修改：

```
{
  "version": "1.0",
  "format": "gpt",
  "partitions": [
    {
      "name": "env",
      "comment": "uboot env data partition, MUST not change its offset and size",
      "hidden": true,
      "offset": "640K",
      "size": "64K",
      "image": "env.bin"
    },
    {
      "name": "bootinfo",
      "comment": "contain address and size info of fsbl, MUST not change its position",
      "hidden": true,
      "offset": "1M",
      "size": "128K",
      "image": "factory/bootinfo_block.bin"
    },
    {
      "name": "fsbl",
      "comment": "its offset was designated by bootinfo",
      "hidden": true,
      "offset": "1536K",
      "size": "512K",
      "image": "factory/FSBL.bin"
    },
    {
     "name": "esos",
      "hidden": true,
      "offset": "4M",
      "size": "3M",
      "image": "esos.itb"
    },
    {
      "name": "opensbi",
      "hidden": true,
      "offset": "7M",
      "size": "1M",
      "image": "fw_dynamic.itb"
    },
    {
      "name": "uboot",
      "hidden": true,
      "offset": "8M",
      "size": "4M",
      "image": "u-boot.itb"
    },
    {
      "name": "bootfs",
      "offset": "12M",
      "size": "256M",
      "image": "bootfs.img",
      "compress": "gzip-5"
    },
    {
        "name": "rootfs",
        "offset": "268M",
        "size": "-",
        "image": "rootfs.ext4",
        "compress": "gzip-5"
    }
  ]
}

```
- **name**：表示分区名字，必须。已有分区名字不要修改，某些地方有引用这些名字。

- **hidden**：表示分区隐藏，不会出现在分区表中，也就是常说的/dev/目录下看不见节点，默认不隐藏，可选。

- **size**：表示分区大小，单位：K，M，G，最后一个分区可选。

- **offset**：表示相对0地址偏移，单位：K，M，G，可选。

- **image**：表示分区对应的镜像，必须且不能为空。

- **compress**：表示镜像的压缩方式。

一般会修改size、镜像名称，**增加分区建议在uboot分区之后增加自己的自定义分区**。增加/删除分区/隐藏分区/都会导致/dev/目录下对应的节点顺序变化，dd烧写分区时请注意。


### 定制根文件系统

K3方案定制根文件系统可通过修改`buildroot-ext/board/spacemit/k3/target_overlay`来实现，buildroot编译最终的所有输出位于`output/xxx/target`目录，并把它打包成rootfs分区。有时我们想方便的往rootfs添加点配置文件或者预编译的二进制文件，又不想麻烦写mk文件，则可以通过如下方式：

1. 直接把文件放在output/xxx/target，重新make打包，但make clean之后会失效。

2. 把文件、目录放在target_overlay目录，目录结构按`output/xxx/target/`下面的子目录结构组织，buildroot打包target目录前会把target_overlay拷贝过去覆盖`output/xxx/target/`。

## 添加新方案

1. 在`buildroot-ext/board/spacemit/`新建方案目录，一般命名方式：芯片名称-板子名称/。

2. 根据需要修改分区表`partitions_universal.json`。

3. 根据需要修改uboot env.txt。

4. 在`buildroot-ext/configs`复制一个新的defconfig文件，命名跟方案名一样。

5. make envconfig时选择该defconfig，make menuconfig配置并保存。一般需要修改如下内容，建议直接编辑defconfig文件：

  - 指向新的内核配置、uboot配置、opensbi配置。

  - 指向新的打包脚本prepare_image.sh。

6. 编译。
