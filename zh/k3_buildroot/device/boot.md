---
sidebar_position: 3
---

# 启动开发指南

## 概述

### 编写目的

本指南主要介绍 SpacemiT 的刷机/启动流程、自定义配置方法，以及 U-Boot 和 OpenSBI 相关的驱动开发与调试技巧，旨在帮助开发者快速掌握设备的启动开发与二次开发流程。

### 适用范围

本文档适用于SpacemiT的K3系列SOC。

### 相关人员

- 刷机/启动开发工程师
- 内核开发工程师

### 文档结构

1. **刷机启动流程与配置**：介绍设备启动机制及自定义方法。
2. **U-Boot/OpenSBI 开发调试**：涵盖编译配置、驱动开发与调试技巧。
3. **常见问题处理**：提供开发过程中常见问题的解决方案。

## 刷机启动

本章节介绍刷机和启动相关的配置以及实现原理，并提供自定义刷机、启动的方式。
> **注意:** 刷机与启动是相互关联的，如果需要对刷机做自定义配置，启动可能也需要做相关的调整，反之亦然。

### 固件布局

K3 系列 SOC 常见的固件布局有以下三种。以下是布局特点：

![alt text](static/image_structure.png)

1. **eMMC/SD卡/UFS**
    - 通过GPT table来索引各个分区。
    - eMMC只使用user分区。
    - bootinfo地址固定在0x100000或0x110000 byte偏移处。
	- FSBL通过bootinfo索引找到其位置

2. **SPI-NOR + UFS/SSD/eMMC**
    - bootinfo地址固定在0或0x10000 byte偏移处。
	- FSBL通过bootinfo索引找到其位置
    - 固件部分在SPI-NOR上，部分在第二级启动介质上。
	- 第二级启动介质可能存在多个，启动优先级可配置
    - 第二级启动介质通过GPT table来索引各个分区。

3. **SPI-NAND**
    - bootinfo地址固定在0或0x10000 byte偏移处。
	- FSBL通过bootinfo索引找到其位置
	- 可将全部数据存放于spinand，也可增加第二级启动介质支持

### 刷机流程和配置

以下内容包含的**刷机**、**烧写**为同一个概念。**卡量产**本质上也是刷机，只是数据来源于SD卡。而卡量产和卡启动为两个不同的概念，**卡启动**是将镜像烧写到SD卡里面，从SD卡里面启动系统。

#### 刷机流程

刷机流程是通过 Fastboot 协议将 PC 端的镜像传输到设备，并通过 MMC/MTD/SCSI/NVMe 等接口写入到对应的存储介质。完整的刷机流程包括进入 U-Boot Fastboot 模式、镜像传输、存储介质设备检测、创建 GPT 表、写入数据到存储介质等步骤。

##### U-Boot Fastboot 模式

本平台的刷机通讯协议基于 Fastboot 协议，并进行了自定义扩展。所有实际的刷机行为都在 U-Boot Fastboot 模式下进行。进入 U-Boot Fastboot 模式后，通过 PC 端执行 `fastboot devices` 可以检测到设备（需要配置 PC 的 Fastboot 环境）。

```sh
~$ fastboot devices
?         Android Fastboot
```

以下介绍进入 U-Boot Fastboot 模式的三种方法：

1. **通过按键组合进入**：按下板子上的 FEL 键和 RESET 键进入 BROM-Fastboot 模式，然后在上位机（PC）执行以下命令，使板子进入 U-Boot 刷机交互界面。BROM-Fastboot 仅用于加载启动 U-Boot Fastboot。

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null
fastboot stage u-boot.itb
fastboot continue
```

2. **通过 ADB 命令进入**：对于已经启动到操作系统的设备，可以通过上位机（PC）运行以下命令

```sh
adb reboot bootloader
```

使板子进入 U-Boot 刷机交互界面。（某些固件可能移除了 ADB 功能，因此该方法不通用。）

3. **通过串口进入**：在板子启动时，通过串口长按 `s` 键进入 U-Boot Shell，然后在串口执行以下命令进入 U-Boot 刷机交互界面：

```sh
fastboot 0
```

以下介绍不同存储介质组合的刷机命令。BROM 可以根据 boot pin 切换不同的启动介质（NOR/NAND/eMMC/UFS），这取决于硬件设计，具体请参考硬件参考设计。

##### eMMC 刷机

- **eMMC 刷机流程**
  对于 eMMC 的刷机流程如下，前面部分是从brom-fastboot启动到uboot-fastboot模式，下面其他的介质也一样

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep to wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null
fastboot stage u-boot.itb
fastboot continue

fastboot flash gpt partition_universal.json
fastboot flash bootinfo factory/bootinfo_block.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash esos esos.itb
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb
fastboot flash ESP esp.img
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4

```

- **eMMC 分区表配置**
分区表保存在 `buildroot-ext/board/spacemit/k3/partition_universal.json`。其中，bootinfo 用于保存启动介质与FSBL地址、长度等相关信息。

```sh
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
      "image": "bootfs.vfat",
      "compress": "gzip-5"
    },
    {
      "name": "rootfs",
      "size": "-"
    }
  ]
}

```

##### NOR+BLK 设备刷机

- **NOR+BLK 刷机流程**
K3 支持 NOR+SSD/eMMC/UFS 等 BLK 设备的组合刷机启动，且为自适应方式。

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null
fastboot stage u-boot.itb
fastboot continue

#刷spi nor
fastboot flash mtd partition_4M.json
fastboot flash bootinfo factory/bootinfo_spinor.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash esos esos.itb
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

#刷block设备
#刷机工具会根据实际的分区表烧写镜像，所以对于nor+blk,blk设备的fsbl/uboot/等分区并没有实际作用
fastboot flash gpt partition_universal.json
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

- **NOR+BLK 分区表配置**
  - NOR 分区表位于 `buildroot-ext/board/spacemit/k3/partition_4M.json`。对于 NAND/NOR 设备，分区表以 `partition_xM.json` 命名，并需根据实际 Flash 容量重命名，否则会导致刷机时找不到对应的分区表。
    - NOR/NAND 的分区表会向最小容量兼容，例如 NOR 的介质容量为 8MB，刷机包中只有 `partition_4M.json`，则会匹配到 `partition_4M.json` 的分区表。
  - 对于分区表配置
    - MTD 存储介质（NAND/NOR Flash）以 `partition_4M.json` 等容量形式表示。
    - BLK 设备（包括 eMMC/SD/SSD 等）以 `partition_universal.json` 命名。

- **NOR Flash 分区表修改**：
  1. 分区起始地址和 size 默认以 64KB 对齐（对应 erase size 为 64KB）。
  2. 如果需要将起始地址和 size 更改为 4KB 对齐，则需开启 U-Boot 的编译配置 `CONFIG_SPI_FLASH_USE_4K_SECTORS`。

```sh
//buildroot-ext/board/spacemit/k3/partition_4M.json
{
  "version": "1.0",
  "format": "mtd",
  "partitions": [
    {
      "name": "bootinfo",
      "comment": "contain address and size info of fsbl, MUST not change its position",
      "offset": "0",
      "size": "128K",
      "image": "factory/bootinfo_spinor.bin"
    },
    {
      "name": "fsbl",
      "comment": "its offset was designated by bootinfo",
      "offset": "128K",
      "size": "512K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "comment": "uboot env data partition, MUST not change its offset and size",
      "offset": "640K",
      "size": "64K",
      "image": "env.bin"
    },
    {
      "name": "esos",
      "offset": "704K",
      "size": "1M",
      "image": "esos.itb"
    },
    {
      "name": "opensbi",
      "offset": "1728K",
      "size": "384K",
      "image": "fw_dynamic.itb"
    },
    {
      "name": "uboot",
      "offset": "2112K",
      "size": "3M",
      "image": "u-boot.itb"
    }
  ]
}
```

- **BLK设备 分区表说明**
BLK 设备（SSD, eMMC, UFS, SDCard 等）统一使用 `partition_universal.json`分区表。


##### NAND 刷机启动

- **NAND 刷机流程**
K3 支持使用 NAND 进行刷机启动。但由于 **NAND 和 NOR 共用同一个 SPI 接口**，两者只能选其一，系统默认使用 NOR 启动。
如需启用 NAND 启动，请参考刷机启动章节中的 NAND 配置说明，完成相关配置。

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null
fastboot stage u-boot.itb
fastboot continue

fastboot flash mtd partition_64M.json
fastboot flash bootinfo factory/bootinfo_spinand.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash esos esos.itb
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

fastboot flash user-bootfs bootfs.img
fastboot flash user-rootfs rootfs.img
```

- **UBIFS 文件系统镜像制作说明**
NAND 上的文件系统依赖于 UBIFS，需提前生成 `bootfs.img` 和 `rootfs.img`，制作方法如下：

```sh
#制作bootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/bootfs.img -d input_bootfs/

#制作rootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/rootfs.img -d input_rootfs/

#input_bootfs和input_rootfs应该存放bootfs和rootfs中的文件
#例如对于bootfs,应该放入如下文件Image.itb、env_k3.txt、bianbu.bmp

#不同的nand需要修改对应的参数，以下是一些参数说明，具体可查看mkfs.ubifs的用法
-m 2048：设置最小输入/输出单元大小为 2048 字节（2 KiB），要与NAND Flash 的页大小相匹配。
-e 124KiB：设置逻辑擦除块的大小为 124 KiB（千字节），这应小于实际的物理擦除块的大小，以留出空间用于 UBI 的管理结构。
-c 8124：设置最大逻辑擦除块的数量为 8124，这个数字限制了文件系统的最大大小，基于逻辑擦除块的大小和数量计算。
-x zlib：设置压缩类型为 zlib，这将在文件系统上对数据进行 zlib 压缩。
-o output/ubifs.img：指定输出文件名，将生成的 UBIFS 文件系统镜像保存为 output/ubifs.img。
-d input/：指定源目录为 input/，mkfs.ubifs 将会把这个目录下的文件和文件结构创建成 UBIFS 文件系统镜像。
```

- **NAND + BLK 组合启动**
本平台也支持 NAND + BLK 设备组合启动方式，相关刷机流程可参考 **NOR+BLK 设备刷机** 章节。

- **NAND 分区表配置**
以下示例为适用于 256MB 容量 NAND 的分区表：

```sh
//buildroot-ext/board/spacemit/k3/partition_64M.json
{
  "version": "1.0",
  "format": "mtd",
  "partitions": [
    {
      "name": "bootinfo",
      "comment": "contain address and size info of fsbl, MUST not change its position",
      "offset": "0",
      "size": "128K",
      "image": "factory/bootinfo_spinand.bin"
    },
    {
      "name": "fsbl",
      "comment": "its offset was designated by bootinfo",
      "offset": "128K",
      "size": "512K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "comment": "uboot env data partition, MUST not change its offset and size",
      "offset": "640K",
      "size": "64K",
      "image": "env.bin"
    },
    {
      "name": "esos",
      "offset": "704K",
      "size": "1M",
      "image": "esos.itb"
    },
    {
      "name": "opensbi",
      "offset": "1728K",
      "size": "384K",
      "image": "fw_dynamic.itb"
    },
    {
      "name": "uboot",
      "offset": "2112K",
      "size": "3008K",
      "image": "u-boot.itb"
    },
    {
      "name": "bootfs",
      "offset": "5M",
      "size": "30M",
      "image": "bootfs.vfat"
    }
  ]
}
```

##### 卡启动

卡启动是指将系统镜像写入到 **SD 卡** 中，当设备插入该 SD 卡后，**上电启动时会优先从卡中加载系统**。

K3 平台支持卡启动，并且在启动时会**优先尝试从 SD 卡启动**，如果启动失败，系统将退回到使用 **pin select** 选中的其他存储介质启动。

>**注：** 本平台 **不支持通过 fastboot 协议** 将镜像烧写到 SD 卡。

制作卡启动镜像需使用 **TitanFlash 工具** 或 `dd` 命令完成。

**使用刷机工具制作卡启动镜像：**

1. 将 TF 卡插入读卡器，并连接到电脑的 USB 接口。
2. 打开电脑端的 **TitanFlash 刷机工具**（安装方式请参考 [TitanFlash 安装](https://spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md)）。
3. 点击顶部菜单栏中的 **研发工具 → 卡启动**。
4. 点击对应的 **选择SD卡** 和 **选择刷机包**。
![alt text](static/flash_tool_1.png)
5. 点击 **执行**，开始烧写镜像。
6. 烧写成功后，将 TF 卡插入设备，上电后设备即可实现卡启动。

##### 卡量产

卡量产是指：将刷机相关的镜像文件写入 **SD 卡** 中，当设备上电启动后，**首先从 SD 卡启动**，并检测到处于量产模式时，**将卡内镜像文件按分区表配置写入到 pin select 选中的存储介质**（如 eMMC、NOR、NAND、UFS 等）。

K3 平台支持卡量产烧录。通过 **TitanFlash 工具** 可将 SD 卡制作成量产卡。将量产卡插入设备并上电后，系统会自动将镜像烧录到目标存储介质。

**卡量产制作步骤：**

1. 将 TF 卡插入读卡器，并接入电脑 USB 接口。
2. 打开 TitanFlash 工具，点击 **量产工具 → 制作量产卡**。
3. 选择需要烧录的刷机包，点击 **执行** 开始制作。
![alt text](static/flash_tool_2.png)
4. 制作完成后，将 SD 卡插入设备。
5. 上电后设备会自动进入卡烧录流程，并将镜像写入目标存储介质。
6. 烧录完成后，**务必拔出 SD 卡**，以避免再次上电重复烧录。

#### 刷机工具

本章节简要介绍刷机工具的使用方式和相关配套内容。

##### 刷机工具使用

对于镜像的烧写，平台支持以下两种方式：

- **TitanFlash 刷机工具**：
  面向一般开发者，适用于完整刷机包的烧写，界面友好、功能完整。
  工具的安装与使用说明详见官方文档：[TitanFlash 使用手册](https://spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md)

- **Fastboot 工具**：
  面向具备一定开发能力的用户，适合进行单个分区的镜像烧写。
  **注意：** 若分区镜像烧写错误，可能导致系统启动异常，请谨慎操作。
  相关烧写流程详见本文的 **刷机流程** 章节，fastboot 环境安装可参考以下链接：

  - [参考链接 1](https://www.jb51.net/article/271550.htm)
  - [参考链接 2](https://blog.csdn.net/qq_34459334/article/details/140128714)

> 固件镜像的生成方式，请参考文档：[下载和编译](https://sdk.spacemit.com/source)

##### 刷机介质选择

- 硬件平台支持通过 **Boot Download Sel Switch 拨码开关** 切换启动介质。
  具体使用方法可参考 MUSE Pi 用户指南中的 [Boot Download Sel & JTAG Sel](https://spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_pi/pi_user_guide.md) 章节。

- 其他平台或方案，请参考对应的硬件用户使用手册。

- 对于不同的启动介质，刷机方式已做成**自动适配**。

### 启动配置

本章节介绍 eMMC、SD、NOR、NAND、UFS 启动的配置，以及自定义启动配置相关的注意事项。刷机启动需要配合正确的分区表等配置以及正确的启动配置。

#### 启动流程

本章节介绍平台芯片的启动流程，并且提供定制化启动的修改方。
如下框图，启动流程主要包括以下阶段：
**brom -> fsbl -> opensbi -> uboot -> kernel**
其中，`bootinfo` 提供 fsbl 所在偏移、大小等信息。

![alt text](static/boot_proc.png)

根据硬件的 **Boot Pin Select 配置**，系统会从 SD 卡、eMMC、NOR、NAND、UFS 等介质中加载下一级启动镜像。
**不同的启动介质，整体启动流程与上图一致。**
Boot Pin Select 的详细介绍请参考 **刷机工具使用** 章节中的 **刷机介质选择** 。

在 K3 平台上，设备上电后将：

1. **优先尝试从 SD 卡启动**；
2. 若 SD 卡启动失败，则根据 Boot Pin Select 配置，依次尝试从 eMMC、NAND、NOR 或 UFS 加载 bootloader

以下小节将根据启动顺序，分别介绍bootinfo、fsbl、opensbi、uboot 和 kernel 等启动配置

##### BROM (Boot ROM)

brom 是预置于 SoC 的启动代码，出厂即无法改变。芯片上电或复位后自动运行。它根据 Boot Pin 判断启动介质 (如 SD、eMMC、NOR、NAND、UFS)，并从中加载 `bootinfo` 和 `fsbl`。
> Boot Pin 设置详见 **刷机工具使用** 章节中的 **刷机介质选择** 。

##### bootinfo

brom 启动后会从对应存储介质的物理 **固定地址**（不同启动介质bootinfo位置参见下表）偏移读取 bootinfo 信息。bootinfo 包含 fsbl、fsbl1 的位置、大小 等信息，bootinfo支持备份。

|启动介质|bootinfo 0 偏移|bootinfo 1 偏移|
|:-------:|:--------:|:--------:|
|SDCard|0x100000|0x110000|
|eMMC|0x100000|0x110000|
|UFS|0x100000|0x110000|
|SPI-NOR|0|0x10000|
|SPI-NAND|0|0x10000|

其中，bootinfo 描述文件在如下目录，以 `.json` 格式描述

```sh
uboot-2022.10$ ls board/spacemit/k3/configs/
bootinfo_block.json  bootinfo_spinand.json  bootinfo_spinor.json
```

JSON 文件示例说明如下，记录了各种存储介质的默认 fsbl/fsbl1 位置。fsbl1 为备份。如果需要修改偏移地址，可以修改 `bootinfo_*json` 文件，然后重新编译 uboot。

fsbl 的位置尽量保持原厂设置。如果需要自定义，需要考虑不同存储介质的特性，例如 NAND 需要按 sector 对齐等。K3 fsbl 存储在 user 区域。

```sh
block(eMMC/SD/UFS): spl0=0x180000, spl1=0x200000
nor:  spl0=0x20000, spl1=0xA0000
nand: spl0=0x20000, spl1=0xA0000
```

编译uboot时会在其根目录下生成对应的 `*.bin` 文件等 bootinfo 镜像文件，生成方式可参考 `uboot-2022.10/board/spacemit/k3/config.mk`

```sh
uboot-2022.10$ ls -l
bootinfo_block.bin
bootinfo_spinand.bin
bootinfo_spinor.bin
```

##### FSBL (First Stage Boot Loader)

brom 获取 bootinfo 后，会从指定的 offset 加载 `fsbl` 。

`fsbl` 文件是由 4K 的头信息与 `u-boot-spl.bin` 组合而成，头信息包含 uboot-spl 的一些必要信息，如 CRC 校验值等。

fsbl 启动后，会先 **初始化 DDR**，然后从分区表 **加载 esos, opensbi 和 uboot 到内存的指定位置**，再 **运行 esos, opensbi**，接着 **opensbi 会启动 uboot**。(fsbl 如何初始化 DDR 的内容不在本文档的描述范围内)

##### esos, opensbi 和 uboot 加载启动

K3 平台在 FSBL 之后，SPL 分两个阶段加载镜像：

1. **opensbi 加载**：由 SPL 标准加载框架（`spl_mmc_load` / `spl_spi_load_image` / `spl_ufs_load_image`）负责，opensbi 是 SPL 的”主镜像”，加载完成后 SPL 跳转到 `spl_invoke_opensbi()`。
2. **esos + uboot 加载**：在跳转 opensbi 之前，由 `board_load_extra_fits()`（`board/spacemit/k3/spl_extra_fit.c`）额外加载，esos 先于 uboot 加载。

**esos**（Embedded SoC OS）是 K3 特有的 SoC 固件分区。分区布局（NOR 为例）：

```
spi-flash: 128K@0(bootinfo), 512K@128K(fsbl), 64K@640K(env),
           1M@704K(esos), 384K@1728K(opensbi), -@2112K(uboot)
```

**opensbi 加载策略**（按优先级）：

1. **分区名加载**（默认）：SPL 扫描分区表，查找名为 `opensbi` 的分区（MTD 设备通过 env `opensbi_partition` 指定，默认 `opensbi`）。
2. **绝对偏移加载**：通过 env 变量 `opensbi_offset` 指定字节偏移（默认 `0x700000`）。

**esos / uboot 加载策略**（按优先级）：

1. **分区名加载**（默认）：通过 env 变量 `extra_esos_partition=esos`、`extra_uboot_partition=uboot` 指定分区名，SPL 扫描分区表找到对应分区后加载 FIT 镜像。
2. **绝对偏移加载**：通过 env 变量 `esos_offset`、`uboot_offset` 指定字节偏移。
3. **文件系统加载**（MMC/UFS，需开启 `CONFIG_SYS_BOOTLOADER_FS_PARTITION_NAME`）：从指定文件系统分区读取 `esos.itb` 和 `u-boot.itb` 文件，路径由 `esos_itb_path`、`uboot_itb_path` 控制，可通过 env `bootloader_from_fs=0` 关闭。

**opensbi 跳转流程**（`common/spl/spl_opensbi.c`）：

SPL 加载完 opensbi 后调用 `spl_invoke_opensbi()`，该函数：
1. 调用 `board_load_extra_fits()` 加载 esos 和 uboot，获取 uboot 入口地址；
2. 构造 `fw_dynamic_info` 结构体，将 uboot 入口地址（`next_addr`）传给 opensbi；
3. 跳转到 opensbi 入口，opensbi 完成初始化后按 `next_addr` 启动 U-Boot。

默认 env 配置（`include/configs/k3.h`）：

```c
“opensbi_offset=0x700000\0”
“extra_esos_partition=esos\0”
“extra_uboot_partition=uboot\0”
“esos_offset=0x400000\0”
“uboot_offset=0x800000\0”
“esos_itb_path=esos.itb\0”
“uboot_itb_path=u-boot.itb\0”
```

内存加载地址由 defconfig 指定：

```
CONFIG_SPL_OPENSBI_LOAD_ADDR=0x100000000   # opensbi 加载地址
CONFIG_SYS_TEXT_BASE=0x102000000           # uboot 加载地址
```

###### eMMC / SD 启动

eMMC 与 SD 都使用 **MMC 驱动**，根据 boot pin select 选择设备（SD=MMC0，eMMC=MMC2）。

K3 平台默认先尝试从 SD 启动，失败后再尝试其他介质（eMMC/NOR/NAND/UFS）。

**opensbi 加载**：`spl_mmc_load()` 先从 `opensbi_offset`（env，默认 `0x700000`）或名为 `opensbi` 的分区加载 opensbi FIT 镜像。

**esos / uboot 加载**：opensbi 加载完成后，`spl_invoke_opensbi()` 调用 `board_load_extra_fits()` 按分区名（`esos`/`uboot`）或偏移加载。

SPL DTS 需开启 eMMC/SD 节点：

```sh
//uboot-2022.10/arch/riscv/dts/k3_spl.dts
        /* SD */
		sdhci0: sdh@d4280000 {
			u-boot,dm-spl;
			compatible = “spacemit,k3-sdhci”;
			bus-width = <4>;
			broken-cd;
			no-1-8-v;
			cap-sd-highspeed;
			sdh-phy-module = <0>;
			status = “okay”;
		};

        /* eMMC */
		sdhci2: sdh@d4281000 {
			u-boot,dm-spl;
			compatible = “spacemit,k3-sdhci”;
			bus-width = <8>;
			non-removable;
			mmc-hs400-1_8v;
			mmc-hs400-enhanced-strobe;
			sdh-phy-module = <1>;
			status = “okay”;
		};
```

**加载方式（按优先级）：**

1. **分区名方式**（默认）

   SPL 扫描 GPT 分区表，按 `extra_esos_partition`（默认 `esos`）和 `extra_uboot_partition`（默认 `uboot`）找到对应分区起始 LBA，加载 FIT 镜像。

   加载顺序：先加载 `esos`，再加载 `uboot`（含 opensbi entry point）。

2. **绝对偏移方式**

   设置 env 变量 `esos_offset` 和 `uboot_offset`（字节偏移，需对齐到块大小），SPL 直接按偏移读取 FIT 镜像。

   ```sh
   # 示例：通过 U-Boot Shell 临时设置
   => setenv esos_offset 0x400000
   => setenv uboot_offset 0x800000
   => saveenv
   ```

###### NOR 启动

K3 平台支持通过 NOR 介质启动，常见的启动组合包括：

- NOR (u-boot-spl / esos / uboot / opensbi) + SSD (bootfs / rootfs)
- NOR (u-boot-spl / esos / uboot / opensbi) + eMMC (bootfs / rootfs)
- NOR (u-boot-spl / esos / uboot / opensbi) + UFS (bootfs / rootfs)


如果同时存在多个block设备，可通过在代码(board/spacemit/k3/k3.c)中设置优先级；如下代码所示，系统默认会**优先尝试从 UFS 启动**。

```c
static struct k3_nor_boot_target k3_nor_boot_prio[] = {
#ifdef CONFIG_SCSI
	{ K3_NOR_BOOT_TARGET_SCSI, "scsi", "ufs_devnum",
	  K3_NOR_UFS_DEVNUM_DEFAULT },
#endif
#ifdef CONFIG_NVME
	{ K3_NOR_BOOT_TARGET_NVME, "nvme", "ssd_devnum",
	  K3_NOR_SSD_DEVNUM_DEFAULT },
#endif
#ifdef CONFIG_MMC
	{ K3_NOR_BOOT_TARGET_MMC, "mmc", "emmc_devnum",
	  K3_NOR_EMMC_DEVNUM_DEFAULT },
#endif
#if defined(CONFIG_USB) && defined(CONFIG_USB_STORAGE)
	{ K3_NOR_BOOT_TARGET_UDISK, "usb", "usb_devnum",
	  K3_NOR_USB_DEVNUM_DEFAULT },
#endif
};
```

或者在uboot dts中添加如下节点，设置最高优先级。

```c
{
        nor-boot-priority-helper {
                highest-priority = "ssd";
                // one of ufs/scsi, ssd/nvme, mmc ,usb
        };
};

```

对于spi-nor的配置，在 SPL DTS 配置如下：

```c
//uboot-2022.10/arch/riscv/dts/k3_spl.dts
		qspi: spi@d420c000 {
			u-boot,dm-spl;
			compatible = "spacemit,k3-qspi";
			#address-cells = <1>;
			#size-cells = <0>;
			pinctrl-names = "default";
			pinctrl-0 = <&pinctrl_qspi>;
			reg = <0x0 0xd420c000 0x0 0x1000>,
			      <0x0 0xb8000000 0x0 0x4100000>;
			reg-names = "qspi-base", "qspi-mmap";
			qspi-sfa1ad = <0x4000000>;
			qspi-sfa2ad = <0x100000>;
			qspi-sfb1ad = <0x100000>;
			qspi-sfb2ad = <0x100000>;
			qspi-pmuap-reg = <0xd4282860>;
			qspi-mpmu-acgr-reg = <0xd4051024>;
			qspi-ahbread = <0>;
			qspi-freq = <26500000>;
			qspi-id = <4>;
			clocks = <&ccu CLK_QSPI>,
			          <&ccu CLK_QSPI_BUS>;
			clock-names = "qspi_clk", "qspi_bus_clk";
			resets = <&reset RESET_QSPI>,
				<&reset RESET_QSPI_BUS>;
			reset-names = "qspi_reset", "qspi_bus_reset";
			spi-max-frequency = <81300000>;
			status = "okay";

			flash@0 {
				u-boot,dm-spl;
				compatible = "jedec,spi-nor";
				reg = <0>;
				spi-max-frequency = <81300000>;
				m25p,fast-read;
				broken-flash-reset;
				status = "okay";
			};
		};
```

启动配置步骤（SPL 加载 NOR）：

**默认已启用以下配置，如需确认可通过 menuconfig 检查。**

1. **SPL 基本功能开启**
   执行 `make uboot_menuconfig`，选择 `SPL configuration options`
    ![alt text](static/spl-config_6.png)

2. **启用以下选项：**

   - `Support MTD drivers`
   - `Support SPI DM drivers in SPL`
   - `Support SPI drivers`
   - `Support SPI flash drivers`
   - `Support for SPI flash MTD drivers in SPL`
   - `Support loading from mtd device`
   - 设置 `Partition name to use to load U-Boot from`，该值需与分区表中的实际名称一致。

    ![alt text](static/spl-config_7.png)

3. **启用 BLK 设备支持**（如 SSD/eMMC）
   进入 `Device Drivers -> Fastboot support`，勾选：

   - `Support blk device`
   - SSD 对应 NVMe，eMMC 对应 MMC

    ![alt text](static/spl-config_8.png)

4. **启用 MTD 环境变量（env）支持**

   SPL 启动后需从 env 获取 MTD 分区信息：

   - 进入 `Environment` 配置页
   - 启用 SPI env 加载
   - 设置 env 的偏移地址，需与分区表一致（例如：`0xA0000`）

    ![alt text](static/spl-config_9.png)

    ![alt text](static/spl-config_10.png)

5. **适配 SPI Flash 驱动（根据硬件）**
   若默认未包含目标芯片的驱动：

   - 执行 `make uboot_menuconfig`
   - 进入 `Device Drivers -> MTD Support -> SPI Flash Support`
   - 根据硬件的 SPI Flash 厂商，勾选选择对应的驱动程序

   ![alt text](static/spl-config_11.png)

   如驱动列表中没有你的 Flash 型号，可以在代码上手动直接添加。`flash_name` 可以自定义，一般为硬件 FLASH 名称，`0x1f4501` 为该 FLASH 的 jedecid，其他参数可以根据该硬件的 FLASH 添加。

```sh
//uboot-2022.10/drivers/mtd/spi/spi-nor-ids.c
const struct flash_info spi_nor_ids[] = {
     { INFO("flash_name",    0x1f4501, 0, 64 * 1024,  16, SECT_4K) },
```

**NOR 加载流程**：

- **opensbi**：由 `spl_spi_load_image()`（`common/spl/spl_mtd.c`）加载，优先查找名为 `opensbi` 的 MTD 分区（可通过 env `opensbi_partition` 覆盖），找不到则按 `opensbi_offset` 偏移加载。
- **esos / uboot**：opensbi 加载后由 `board_load_extra_fits()` 加载，支持以下两种方式：

- **分区名方式**（默认）

  SPL 通过 MTD 分区名直接获取 `esos` 和 `uboot` 分区，加载 FIT 镜像。分区名由 env 变量 `extra_esos_partition`（默认 `esos`）和 `extra_uboot_partition`（默认 `uboot`）控制。若 env 未设置，SPL 会自动尝试名为 `esos` 和 `uboot` 的 MTD 分区。

  ![alt text](static/spl-config_12.png)

- **绝对偏移方式**

  通过 env 变量 `esos_offset` 和 `uboot_offset` 指定字节偏移（默认值见 `include/configs/k3.h`）。

  开启以下配置，输入存储介质的绝对偏移地址。
  ![alt text](static/spl-config_13.png)

###### NAND 启动

K3 平台支持通过 NAND 启动，主要包括两种方式：

- **NAND（u-boot-spl / esos / uboot / opensbi）+ SSD（bootfs / rootfs）**
- **纯 NAND（u-boot-spl / esos / uboot / opensbi / kernel）**

默认情况下，NAND 启动功能是关闭的。NAND 的 opensbi/esos/uboot 加载方式与 NOR 完全相同，均由 `spl_spi_load_image()` 和 `board_load_extra_fits()` 处理，分区名由 `opensbi_partition`、`extra_esos_partition`、`extra_uboot_partition` 控制。

SPL DTS 配置如下

```sh
//uboot-2022.10/arch/riscv/dts/k3_spl.dts
        spi@d420c000 {
                status = "okay";
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_qspi>;
                u-boot,dm-spl;

                compatible = "spacemit,k3-qspi";
                spi-nand@0 {
                       compatible = "spi-nand";
                       reg = <0>;
                       spi-tx-bus-width = <1>;
                       spi-rx-bus-width = <1>;
                       spi-max-frequency = <6250000>;
                       u-boot,dm-spl;
                       status = "okay";
               };
        };

```

下面将介绍 **纯 NAND 启动配置** 步骤。

1. **配置 SPL 编译选项**
   - 执行 `make uboot_menuconfig`，进入 `SPL configuration options`，启用以下选项：

     ![alt text](static/spl-config_14.png)

     - `Support MTD drivers`
     - `Support SPI DM drivers in SPL`
     - `Support SPI drivers`
     - `Use standard NAND driver`
     - `Support simple NAND drivers in SPL`
     - `Support loading from mtd device`
     - 设置 `Partition name to use to load U-Boot from`，需与分区表保持一致（默认 `esos`）

   - K3 的 NAND 加载由 `board_load_extra_fits()` 统一处理，分区名通过 env 变量 `extra_esos_partition` 和 `extra_uboot_partition` 控制，无需额外配置 second partition。

     ![alt text](static/spl-config_15.png)

2. **配置 env 支持**
   对于 MTD 设备，需要开启 `env`，以确保 SPL 启动后能从 `env` 获取 MTD 分区信息
   - 执行 `make uboot_menuconfig`
   - 进入 `Environment` 配置页
   - 启用 SPI 的 `env` 加载支持
   - 设置偏移地址为 `0xA0000`（需与分区表一致）

3. **适配 NAND Flash 驱动**

   - 驱动需匹配实际使用的 NAND 芯片厂商。目前已支持的 NAND FLASH 如下驱动所示。

```sh
~/uboot-2022.10$ ls drivers/mtd/nand/spi/*.c
uboot-2022.10/drivers/mtd/nand/spi/core.c
uboot-2022.10/drivers/mtd/nand/spi/micron.c
uboot-2022.10/drivers/mtd/nand/spi/winbond.c
uboot-2022.10/drivers/mtd/nand/spi/gigadevice.c
uboot-2022.10/drivers/mtd/nand/spi/other.c
uboot-2022.10/drivers/mtd/nand/spi/macronix.c
uboot-2022.10/drivers/mtd/nand/spi/toshiba.c
```

- 如果未内置相关驱动，可以在 `other.c` 驱动中添加厂商的 **JEDEC ID**。
     示例：添加 FORESEE 或 Dosilicon 芯片支持：

```sh
//uboot-2022.10/drivers/mtd/nand/spi/other.c
 static int other_spinand_detect(struct spinand_device *spinand)
 {
     u8 *id = spinand->id.data;
     int ret = 0;

     /*
      * dosilicon nand flash
      */
     if (id[1] == 0xe5)
        ret = spinand_match_and_init(spinand, dosilicon_spinand_table,
                     ARRAY_SIZE(dosilicon_spinand_table),
                     id[2]);

     /*FORESEE nand flash*/
     if (id[1] == 0xcd)
        ret = spinand_match_and_init(spinand, foresee_spinand_table,
                     ARRAY_SIZE(foresee_spinand_table),
                     id[2]);
     if (ret)
         return ret;

     return 1;
 }
```

###### UFS 启动

K3 平台支持通过 UFS 介质启动，UFS 设备在 SPL 中以 SCSI 块设备形式访问（`IF_TYPE_SCSI`，设备号 0）。

SPL DTS 需开启 UFS 节点：

```sh
//uboot-2022.10/arch/riscv/dts/k3_spl.dts
        ufs: ufs@c0e00000 {
                compatible = "spacemit,k3-ufshci";
                reg = <0x0 0xc0e00000 0x0 0x30000>;
                clocks = <&ccu CLK_UFS_ACLK>;
                clock-names = "aclk";
                resets = <&reset RESET_UFS_ACLK>;
                reset-names = "aclk_reset";
                ref-clk-freq = <19200000>;
                status = "okay";
                u-boot,dm-spl;
        };
```

**UFS 加载流程**：

- **opensbi**：由 `spl_ufs_load_image()`（`common/spl/spl_ufs.c`）加载，优先按 `opensbi_offset` 偏移加载，找不到则按 `CONFIG_SYS_LOAD_IMAGE_PARTITION_NAME` 指定的分区名加载。
- **esos / uboot**：opensbi 加载后由 `board_load_extra_fits()` 加载，支持以下两种方式：

1. **分区名方式**（默认）

   SPL 扫描 UFS GPT 分区表，按 `extra_esos_partition`（默认 `esos`）和 `extra_uboot_partition`（默认 `uboot`）找到对应分区，加载 FIT 镜像。

2. **绝对偏移方式**

   通过 env 变量 `esos_offset` 和 `uboot_offset` 指定字节偏移（需对齐到 UFS 块大小）。

> 注意：UFS 启动时 SPL 会在 `board_load_extra_fits()` 之前完成 env 加载，无需重复初始化。

##### 启动 Kernel

系统启动进入 **U-Boot** 后，会根据 `env` 中的配置自动加载并启动 Kernel。开发者可以按需定制启动方式。

> 注意：`env_k3.txt` 的优先级高于 `env.bin`，会覆盖其中的同名变量。

**Kernel 启动流程说明**

在 `fsbl` 启动 `opensbi → uboot` 后，U-Boot 会根据设定的启动命令，从 FAT 或 EXT4 文件系统中加载 kernel、dtb 等镜像至内存，最终执行 `bootm` 命令启动内核。

相关配置可参考：

```sh
uboot-2022.10/board/spacemit/k3/k3.env
```

文件中已集成多种启动方案（如 eMMC/SD/NOR+BLK/UFS/NAND），并能根据实际启动介质自动适配。`autoboot` 变量根据 `boot_device` 自动选择对应的启动命令。

**启动方式示例**

1. **MMC 启动**（适用于 SD / eMMC）

   对于 SD/eMMC，都属于 `mmc_boot`

```sh
//uboot-2022.10/board/spacemit/k3/k3.env
mmc_boot=echo "Try to boot from ${boot_devname}${boot_devnum} ..."; \
        run commonargs; \
        run add_bootarg; \
        run set_mmc_args; \
        run get_esp_index; \
        if test -e ${boot_devname} ${boot_devnum}:${esp_index} ${grub_file}; then \
            run boot_grub; \
        else \
            run boot_kernel; \
        fi;
```

启动时，rootfs 由 U-Boot 传递 `bootargs` 给 Kernel，Kernel 或其 `init` 脚本解析并挂载 rootfs。rootfs 分区通过 PARTUUID 方式传递。

- **NOR + blk 启动**（如 NOR + SSD）

```sh
//uboot-2022.10/board/spacemit/k3/k3.env
nor_boot=echo "Try to boot from ${boot_devname}${boot_devnum} ..."; \
        run commonargs; \
        run add_bootarg; \
        run set_nor_args; \
        run get_esp_index; \
        if test -e ${boot_devname} ${boot_devnum}:${esp_index} ${grub_file}; then \
            run boot_grub; \
        else \
            run boot_kernel; \
        fi;
```

- **UFS 启动**

```sh
//uboot-2022.10/board/spacemit/k3/k3.env
ufs_boot=echo "Try to boot from UFS (scsi 0) ..."; \
        if test "${scsi_scanned}" != "1"; then \
            scsi scan; \
            setenv scsi_scanned 1; \
        fi; \
        setenv boot_devname scsi; \
        setenv boot_devnum 0; \
        run commonargs; \
        run add_bootarg; \
        run set_ufs_args; \
        ...
```

- **NAND 启动**

```sh
//uboot-2022.10/board/spacemit/k3/k3.env
nand_boot=echo "Try to boot from nand flash..."; \
          run commonargs; \
          run add_bootarg; \
          run set_nand_args; \
          run detect_dtb; \
          run ubifs_list; \
          run ubifs_loadimg; \
          ...
```

- **autoboot 自动选择**

```sh
//uboot-2022.10/board/spacemit/k3/k3.env
autoboot=if test ${boot_device} = nand; then \
                run nand_boot; \
        elif test ${boot_device} = nor; then \
                run nor_boot; \
        elif test ${boot_device} = mmc; then \
                run mmc_boot; \
        elif test ${boot_device} = ufs; then \
                run ufs_boot; \
        elif test ${boot_device} = nfs; then \
                run nfs_boot; \
        elif test ${boot_device} = udisk; then \
                run udisk_boot; \
        fi;

bootcmd=run autoboot; echo "run autoboot"
```

### 安全启动

安全启动基于 **FIT image** 格式实现，主要流程如下：

1. 将 ESOS,、OpenSBI、uboot 和 Kernel 各自打包为一个 FIT 镜像；
2. 打开代码中安全启动配置；
3. 使用私钥对 FIT 镜像签名，同时将公钥信息嵌入到 DTS/DTB 中，供上一级引导程序验签使用。

#### 验签流程

启动验签流程如下图所示：

![alt text](static/secure_boot.png)

签名过程描述：

- BootROM 作为信任根，固化在芯片中，不可更改；
- Hash of ROTPK 需烧录进芯片内部Efuse，只能烧录一次；
- 哈希算法：`SHA256`；
- 签名算法：`SHA256 + RSA2048`；

#### 配置

- **U-Boot 编译配置：**

```sh
CONFIG_FIT_SIGNATURE=y
CONFIG_SPL_FIT_SIGNATURE=y

CONFIG_SHA256=y
CONFIG_SPL_SHA256=y

CONFIG_RSA=y
CONFIG_SPL_RSA=y
CONFIG_SPL_RSA_VERIFY=y
CONFIG_RSA_VERIFY=y
```

- **OpenSBI 编译配置：**

```sh
CONFIG_FIT_SIGNATURE=y
```

- **Kernel 编译配置：**

```sh
CONFIG_FIT_SIGNATURE=y
```

#### 公私钥生成

使用 `openssl` 生成私钥和证书（私钥需妥善保管，仅证书公开）：

```sh
# build private key without password
openssl genrsa -out prv-rsa.key 2048
# private key parse:
openssl rsa -in prv-rsa.key -text -noout

# build certificate that expired after 365 days
openssl req -batch -new -x509 -days 365 -key prv-rsa.key -out rsa.crt
# certificate parse:
openssl x509 -in rsa.crt -text -noout

# build public key from private key:
openssl rsa -in prv-rsa.key -pubout -out pub-rsa.key
# public key parse:
openssl rsa -in pub-rsa.key -pubin -noout -text
```

#### 镜像签名

1. 修改 ITS 脚本，启用 hash 和签名配置：

```sh
/dts-v1/;

/ {
        description = "Configuration to load OpenSBI before U-Boot";
        #address-cells = <2>;
        fit,fdt-list = "of-list";

        images {
                opensbi {
                        description = "OpenSBI fw_dynamic Firmware";
                        type = "firmware";
                        os = "opensbi";
                        arch = "riscv";
                        compression = "none";
                        load = <0x0 0x0>;
                        entry = <0x0 0x0>;
                        data = /incbin/("./fw_dynamic.bin");
                        hash-1 {
                                algo = "sha256";
                        };
                };
        };
        configurations {
                default = "config_1";

                config_1 {
                        description = "opensbi FIT config";
                        firmware = "opensbi";
                        signature {
                                algo = "sha256,rsa2048";
                                key-name-hint = "uboot_key_prv";
                                sign-images = "firmware";
                        };
                };
        };
};
```

2. 使用私钥和证书，对 FIT image 文件进行签名

```sh
# build empty dtb file, for next stage public key file output
printf "/dts-v1/;\n/ {\n};" > pubkey.dts
dtc -I dts -O dtb -o pubkey.dtb pubkey.dts

# build fit image
# input: fit script, folder contain private key and certification
# output: fit image, dtb that contain public key info
mkimage -f uboot_fdt_sign.its -K pubkey.dtb -k key -r u-boot.itb

# parse dtb file that public key info
fdtdump -s pubkey.dtb
```

3. 更新公钥信息到上一级引导代码

  示例：更新 uboot 签名所用私钥对应的公钥信息到 FSBL 设备树中

```sh
/ {
        signature {
                key-uboot_key_prv {
                        required = "conf";
                        algo = "sha256,rsa2048";
                        rsa,r-squared = <0x5353bc86 0x7070d595 0xe2ea6280 0xb9887ae1 0xf69bb145 0x161e6675 0x6f9d37dc 0x29646b18 0x0ecc66d1 0x0ef7fa25 0xddc925cf 0xf068e5e4 0x78e5b40b 0x124095c6 0x1282d13c 0x1bdf09d0 0x7ddf7bf4 0xb4e61d0b 0x8d68f15d 0xb77282df 0xb0b371d8 0xd887288d 0x6c2ee06e 0x4124c030 0xbcdb8688 0x13a6ea0a 0xbb8dc9d1 0xd4b8a0fd 0x141c1e45 0x91c77190 0xf2685d1e 0xa44e33eb 0x38a90bdf 0x671b076b 0x0efb5223 0x72762fd2 0xcbf35219 0x833553c7 0x91382847 0xa3806134 0xb785d6f6 0x64ba98d7 0x4f01bc2e 0x78e320dc 0x9233332c 0x8be5ebec 0x60605d78 0xd5e5741c 0x2980546e 0x6332d458 0x73023036 0xb5e64449 0xc3f81911 0xc7d57cad 0xf17d98b1 0x139801a2 0x778632bd 0xfc15d9ca 0x4f5fc152 0xa49e2b4f 0x6f09a6b5 0xecd52030 0x19022428 0x5907c874>;
                        rsa,modulus = <0xaa282eab 0xc7d0a288 0x5eee2ea1 0xd7d11bc5 0xaf57d029 0x4ad6c85f 0xedc802b1 0x227775cc 0x0d57d3de 0xc8e6113c 0xd3c238fd 0x03eecd4c 0x6983e4e0 0xd71eba6b 0xcdcc3c7f 0x6f602163 0x71e25d7e 0xd3ade9b9 0x25c9b950 0x4bf4d0a5 0xa067ca9c 0x64397ed2 0xd07dfa01 0x29102b9c 0x6008c40e 0xc55cc431 0xf3422d16 0xb8ade9d2 0xa8e5d3d1 0x40aca443 0x91603617 0x4159c91f 0xa10e3ef9 0xa21c40c7 0x377dfcc6 0xd831b829 0xd645d1b1 0xb04c534e 0xfd3352ef 0xdfe19a7d 0xf90c4295 0x7e753266 0x398ade75 0x85427a33 0x79412712 0x5dcd236d 0x015d8fb6 0xdde963ad 0xb8730cf5 0x45fc281b 0x1e40a1de 0xcd1d2af6 0x45ce6740 0x42e1e705 0x274af16a 0x50a66381 0xbb815c44 0x5222fe56 0x826e4475 0xd2193598 0x967573fd 0xc814bed6 0x95db8fae 0xe519808f>;
                        rsa,exponent = <0x00000000 0x00010001>;
                        rsa,n0-inverse = <0xfba86191>;
                        rsa,num-bits = <0x00000800>;
                        key-name-hint = "uboot_key_prv";
                };
        };
};

```

## U-Boot 功能与配置

本章节主要介绍 U-Boot 的功能以及常用的配置方法。

### 功能介绍

U-Boot 的主要功能有以下几点：

- **加载启动内核**

  U-Boot 从存储介质（eMMC / SD / NAND / NOR / SSD / UFS 等），加载内核镜像到内存指定位置，并启动内核。

- **fastboot 刷机功能**
  通过 fastboot 工具，烧写镜像到指定的分区位置。

- **开机 logo**
  U-Boot 启动阶段显示启动 logo 以及 boot menu 。

- **驱动调试**
  基于 U-Boot 调试设备驱动，如 MMC / SPI / NAND / NOR / NVME 等驱动，U-Boot 提供 shell 命令行对各个驱动进行功能调试。
  所有 U-Boot 驱动在 `drivers/` 目录下。

### 编译

本章节介绍基于 U-Boot 代码环境，编译生成 U-Boot 的镜像文件。

- **编译配置**
  首次编译或更换方案前，请选择对应的编译配置（以 k3 为例）：

  ```shell
  cd ~/uboot-2022.10
  make ARCH=riscv k3_defconfig -C ~/uboot-2022.10/
  ```

  可视化更改编译配置：

  ```shell
  make ARCH=riscv menuconfig
  ```

  ![a](static/uboot_menuconfig_0.png)

  通过键盘 "Y"/"N" 以 开启/关闭 相关的功能配置。保存后会更新到 U-Boot 根目录的 `.config` 文件。

- **编译 U-Boot**

  ```shell
  cd ~/uboot-2022.10
  GCC_PREFIX=riscv64-unknown-linux-gnu-
  make ARCH=riscv CROSS_COMPILE=${GCC_PREFIX} -C ~/uboot-2022.10 -j4
  ```

- **编译产物说明**

```shell
~/uboot-2022.10$ ls u-boot* -l
u-boot
u-boot.bin           # uboot镜像
u-boot.dtb           # 设备树文件
u-boot-dtb.bin       # 包含设备树的完整 U-Boot 镜像
u-boot.itb           # FIT 格式镜像（含 U-Boot 和多方案设备树）
u-boot-nodtb.bin
bootinfo_block.bin   # 用于 eMMC/SD/UFS 启动时记录 SPL 位置的信息
bootinfo_spinand.bin
bootinfo_spinor.bin
FSBL.bin             # u-boot-spl.bin 加上头信息, 由 brom 加载启动
k3_deb1.dtb          # 方案 deb1 的设备树
k3_spl.dtb           # SPL 的设备树
```

### DTS 配置说明

U-Boot 的设备树文件位于：

```shell
~/uboot-2022.10/arch/riscv/dts/
```

根据所使用的方案（如 deb1）修改对应的 DTS 文件：

```shell
~/uboot-2022.10$ ls arch/riscv/dts/k3*.dts -l
arch/riscv/dts/k3_com260.dts
arch/riscv/dts/k3_com260_kit_v02.dts
arch/riscv/dts/k3_dc_board.dts
arch/riscv/dts/k3_deb1.dts
arch/riscv/dts/k3_evb.dts
arch/riscv/dts/k3_evb2-1.dts
arch/riscv/dts/k3_evb2-2.dts
arch/riscv/dts/k3_spl.dts
```

## U-Boot 驱动开发调试

本章节主要介绍 U-Boot 的驱动使用和调试方法。
默认情况下，所有驱动已在构建配置中启用，无需手动启用。

### 启动 Linux 内核（Boot Kernel）

本节介绍如何通过 U-Boot 启动 Linux 内核(FIT格式，需包含dtb)，包括分区的自定义配置和启动过程。

**步骤 1：进入 U-Boot Shell**
开发板上电启动后，立即按下键盘上的 `s` 键，进入 U-Boot Shell。

**步骤 2：进入 Fastboot 模式**
在 U-Boot Shell 中，执行以下命令进入 Fastboot 模式：

```shell
=> fastboot 0
```

设备进入 Fastboot 模式，等待接收文件。

**步骤 3：下载 Kernel 镜像**
在 **PC 端**，执行以下命令将 Kernel 镜像发送到开发板：

```shell
C:\Users>fastboot stage Z:\k3\output\Image.itb
```

在 **开发板** 的 U-Boot 中，接收镜像数据：


**预期输出**：

```shell
Starting download of 50687488 bytes
...
downloading/uploading of 50687488 bytes finished
```

PC 端输出：

```shell
Sending 'Z:\k3\output\Image.itb' (49499 KB)           OKAY [  1.934s]
Finished. Total time: 3.358s
```

**步骤 4：退出 Fastboot 模式**
下载完成后，在 U-Boot Shell 中，通过键盘输入 `CTRL+C` 退出 Fastboot 模式。

**步骤 5：启动 Kernel**
执行 `booti` 启动 kernel：

```shell
=> bootm 0x140000000
```

**预期输出**：

```shell
   Loading fdt from 0x116aaa3dc to 0x128000000
   Booting using the fdt blob at 0x128000000
   Loading Kernel Image
   Loading Device Tree to 00000003ff6be000, end 00000003ff6dd774 ... OK

Starting kernel ...

[    0.000000] Booting Linux on hartid 0
[    0.000000] Linux version 6.18.3-g61bd18de599b-dirty (zhouxl@LT-ZHOUXIAOLEI) (riscv64-unknown-linux-gnu-gcc (gd094d3a8c4f) 15.2.0, GNU ld (GNU Binutils) 2.45.0.20251020) #8 SMP PREEMPT Sat Jan 24 16:52:02 CST 2026
[    0.000000] random: crng init done
[    0.000000] Machine model: spacemit,k3-deb1
```

### 直接加载启动介质上 FIT 格式镜像

**步骤 1：检查存储介质中的文件**
假设 eMMC 中分区 2 为 FAT32 文件系统，且里面保存 `uImage.itb` 文件。在 U-Boot Shell 中，执行以下命令列出文件：

```shell
=> ls mmc 2:2
```

**预期输出**：

```shell
sdh@d4281000: 74 clk wait timeout(100)
 50896911   uImage.itb
     4671   env_k3.txt

2 file(s), 0 dir(s)
```

**步骤 2：加载 FIT 格式镜像**
执行以下命令加载 FIT 格式镜像：

```shell
=> load mmc 2:2 0x140000000 uImage.itb
```

**预期输出**：

```shell
50896911 bytes read in 339 ms (143.2 MiB/s)
```

**步骤 3：启动 Kernel**
执行 `bootm` 启动 FIT 格式镜像：

```shell
=> bootm 0x140000000
```

**预期输出**：

```shell
## Loading kernel from FIT Image at 140000000 ...
Boot from fit configuration k3_deb1
   Using 'conf_3' configuration
   Trying 'kernel' kernel subimage
     Description:  Vanilla Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x1400000e8
     Data Size:    50687488 Bytes = 48.3 MiB
     Architecture: RISC-V
     OS:           Linux
     Load Address: 0x02000000
     Entry Point:  0x02000000
   Verifying Hash Integrity ... OK
## Loading fdt from FIT Image at 140000000 ...
   Using 'conf_3' configuration
   Trying 'fdt_3' fdt subimage
     Description:  Flattened Device Tree blob for k3_deb1
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x143067c90
     Data Size:    68940 Bytes = 67.3 KiB
     Architecture: RISC-V
     Load Address: 0x128000000
   Verifying Hash Integrity ... OK
   Loading fdt from 0x43067c90 to 0x128000000
   Booting using the fdt blob at 0x128000000
   Loading Kernel Image
   Using Device Tree in place at 0000000128000000, end 0000000128013d4b

Starting kernel ...

[    0.000000] Booting Linux on hartid 0
[    0.000000] Linux version 6.18.3-g61bd18de599b-dirty (zhouxl@LT-ZHOUXIAOLEI) (riscv64-unknown-linux-gnu-gcc (gd094d3a8c4f) 15.2.0, GNU ld (GNU Binutils) 2.45.0.20251020) #8 SMP PREEMPT Sat Jan 24 16:52:02 CST 2026
[    0.000000] random: crng init done
[    0.000000] Machine model: spacemit,k3-deb1
[    0.000000] SBI specification v2.0 detected
```

### 配置启动环境变量`env`

本章节介绍如何在 U-Boot 启动阶段，从指定存储介质加载环境变量 `env`。

1. **进入 `make menuconfig`**
   在构建 U-Boot 时，执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

2. **进入 Environment 配置**
   在 `make menuconfig` 菜单中，选择 **Environment** 选项，进入环境变量配置界面。
   ![alt text](static/uboot_menuconfig_1.png)

   ![atl text](static/uboot_menuconfig_2.png)

   **支持的存储介质**：
   目前支持的存储介质包括：
   - **MMC 设备**（如 SD 卡或 eMMC）
   - **MTD 设备**（包括 SPI NOR 和 SPI NAND）

3. **配置环境变量的偏移地址**
   环境变量的偏移地址需要根据分区表的配置来确定。默认偏移地址为 **`0xA0000`**。具体配置如下：

   - **对于 SPI NOR 设备**：

     ```shell
     (0xA0000) Environment address       # SPI NOR 的 env 偏移地址
     ```

   - **对于 MMC 设备**：

     ```shell
     (0xA0000) Environment offset        # MMC 设备的 env 偏移地址
     ```

### MMC 驱动配置与调试

本章节介绍如何配置和调试 U-Boot 中的 MMC 驱动，包括 eMMC 和 SD 卡的配置。

eMMC 和 SD 卡都使用 MMC 驱动，设备编号分别为：

- **eMMC**: dev number = 2
- **SD 卡**: dev number = 0

1. **config 配置**
   - **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

     ```shell
     make menuconfig
     ```

   - **进入 MMC Host controller Support**
   在 `make menuconfig` 菜单中，选择 **Device Drivers** -> **MMC Host controller Support**，开启以下配置：

      ![alt text](static/uboot_menuconfig_3.png)

2. **dts 配置**
在 U-Boot 的设备树配置中，需要为 eMMC 和 SD 卡配置设备树节点。
**示例配置**：

```c
//uboot-2022.10/arch/riscv/dts/k3.dtsi
        sdhci0: sdh@d4280000 {
            compatible = "spacemit,k3-sdhci";
            reg = <0x0 0xd4280000 0x0 0x200>;
            resets = <&reset RESET_SDH_AXI>,
                     <&reset RESET_SDH0>;
            reset-names = "sdh_axi", "sdh0";
            clocks = <&ccu CLK_SDH0>,
                     <&ccu CLK_SDH_AXI>;
            clock-names = "sdh-io", "sdh-core";
            status = "disabled";
        };

        sdhci2: sdh@d4281000 {
            compatible = "spacemit,k3-sdhci";
            reg = <0x0 0xd4281000 0x0 0x200>;
            resets = <&reset RESET_SDH_AXI>,
                     <&reset RESET_SDH2>;
            reset-names = "sdh_axi", "sdh2";
            clocks = <&ccu CLK_SDH2>,
                     <&ccu CLK_SDH_AXI>;
            clock-names = "sdh-io", "sdh-core";
            status = "disabled";
        };

//uboot-2022.10/arch/riscv/dts/k3_deb1.dts
/* SD */
&sdhci0 {
        bus-width = <4>;
        broken-cd;
        no-1-8-v;
        cap-sd-highspeed;
        sdh-phy-module = <0>;
        status = "okay";
};

/* eMMC */
&sdhci2 {
        bus-width = <8>;
        non-removable;
        mmc-hs400-1_8v;
        mmc-hs400-enhanced-strobe;
        sdh-phy-module = <1>;
        status = "okay";
};
```

3. **调试验证**
U-Boot Shell 提供了命令行工具用于调试 MMC 驱动。需要开启编译配置项 `CONFIG_CMD_MMC`。

```shell
=> mmc list
sdh@d4280000: 0 (SD)
sdh@d4281000: 2 (eMMC)
=> mmc dev 2 #切换到emmc
switch to partitions #0, OK
mmc2(part 0) is current device

#read 0偏移的0x1000个blk_cnt到内存 0x140000000
=> mmc read 0x140000000 0 0x1000

MMC read: dev # 2, block # 0, count 4096 ... 4096 blocks read: OK

#从内存地址0x40000000 写到0x1000个blk_cnt到0偏移
=> mmc write 0x140000000 0 0x1000

MMC write: dev # 2, block # 0, count 4096 ... 4096 blocks written: OK

#其他用法可参考mmc -h
```

4. **常用接口**
参考 `cmd/mmc.c` 中的接口实现，了解更多信息。

### NVMe 驱动配置与调试

NVMe 驱动主要用于调试 SSD 硬盘。以下内容将介绍如何配置和调试 NVMe 驱动。

1. **config 配置**
   **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

   **进入 Device Drivers**
   在 `make menuconfig` 菜单中，进入 **Device Drivers**，开启以下配置：

   ![a](static/uboot_menuconfig_4.png)

   ![a](static/uboot_menuconfig_5.png)

2. **dts 配置**
在 U-Boot 的设备树配置中，需要为 NVMe 驱动配置设备树节点。
**示例配置**：

```c
//uboot-2022.10/arch/riscv/dts/k3.dtsi
		pcie0_rc: pcie@80000000 {
			compatible = "spacemit,k3-pcie";
			reg = <0x0 0x80000000 0x0 0x00001000>, /* dbi */
			      <0x0 0x80100000 0x0 0x00001000>, /* dbi2 */
			      <0x0 0x80300000 0x0 0x00003f20>, /* atu registers */
			      <0x11 0x00000000 0x0 0x00010000>, /* config space */
			      <0x0 0xd42829f0 0x0 0x00000008>, /* k3 soc config addr */
			      <0x0 0x82900000 0x0 0x00001000>; /* phy ahb */
			reg-names = "dbi", "dbi2", "atu", "config", "app", "phy_ahb";

			spacemit,pcie-mgmt = <0xd42829d8>;
			clocks = <&ccu CLK_PCIEA>;
			resets = <&reset RESET_PCIE_PORTA>;
			reset-names = "pcie-reset";

			spacemit,pcie-port = <0>;
			bus-range = <0x00 0xff>;
			max-link-speed = <3>;
			num-lanes = <8>;

			device_type = "pci";
			#address-cells = <3>;
			#size-cells = <2>;
			ranges = <0x01000000 0x0 0x00010000 0x11 0x00010000 0x0 0x00100000>,
				 <0x02000000 0x0 0x00110000 0x11 0x00110000 0x0 0x7fef0000>,
				 <0x43000000 0x18 0x00000000 0x18 0x00000000 0x1 0x00000000>;
			linux,pci-domain = <0>;
			status = "disabled";
		};

		pcie1_rc: pcie@80400000 {
			compatible = "spacemit,k3-pcie";
			reg = <0x0 0x80400000 0x0 0x00001000>, /* dbi */
			      <0x0 0x80500000 0x0 0x00001000>, /* dbi2 */
			      <0x0 0x80700000 0x0 0x00003f20>, /* atu registers */
			      <0x11 0x80000000 0x0 0x00010000>, /* config space */
			      <0x0 0xd42829d0 0x0 0x00000008>, /* k3 soc config addr */
			      <0x0 0x82c00000 0x0 0x00001000>; /* phy ahb */
			reg-names = "dbi", "dbi2", "atu", "config", "app", "phy_ahb";

			spacemit,pcie-mgmt = <0xd42829d8>;
			clocks = <&ccu CLK_PCIEB>;
			resets = <&reset RESET_PCIE_PORTB>;
			reset-names = "pcie-reset";

			spacemit,pcie-port = <1>;
			bus-range = <0x00 0xff>;
			max-link-speed = <3>;
			num-lanes = <2>;

			device_type = "pci";
			#address-cells = <3>;
			#size-cells = <2>;
			ranges = <0x01000000 0x0 0x00010000 0x11 0x80010000 0x0 0x00100000>,
				 <0x02000000 0x0 0x80110000 0x11 0x80110000 0x0 0x7fef0000>,
				 <0x43000000 0x16 0x00000000 0x16 0x00000000 0x1 0x00000000>;
			linux,pci-domain = <1>;
			status = "disabled";
		};

//uboot-2022.10/arch/riscv/dts/k3_deb1.dts
&pcie0_rc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pcie0_1>;
	num-lanes = <4>;
	phys = <&pcie_phy0>, <&pcie_phy1>;
	phy-names = "pcie-phy0", "pcie-phy1";
	spacemit,device-detect = <&gpio 89 0>;

	status = "okay";
};

&pcie1_rc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pcie1_1>;

	phys = <&pcie_phy1>;
	phy-names = "pcie-phy0";
	status = "okay";
};
```

3. **调试验证**
需要开启编译配置 `CONFIG_CMD_NVME`，调试方法如下：

```shell
=> nvme scan
=> nvme detail
Blk device 0: Optional Admin Command Support:
        Namespace Management/Attachment: no
        Firmware Commit/Image download: yes
        Format NVM: yes
        Security Send/Receive: yes
Blk device 0: Optional NVM Command Support:
        Reservation: yes
        Save/Select field in the Set/Get features: yes
        Write Zeroes: yes
        Dataset Management: yes
        Write Uncorrectable: yes
Blk device 0: Format NVM Attributes:
        Support Cryptographic Erase: No
        Support erase a particular namespace: Yes
        Support format a particular namespace: Yes
Blk device 0: LBA Format Support:
        LBA Foramt 0 Support: (current)
                Metadata Size: 0
                LBA Data Size: 512
                Relative Performance: Good
Blk device 0: End-to-End DataProtect Capabilities:
        As last eight bytes: No
        As first eight bytes: No
        Support Type3: No
        Support Type2: No
        Support Type1: No
Blk device 0: Metadata capabilities:
        As part of a separate buffer: No
        As part of an extended data LBA: No
=> nvme read/write addr blk_off blk_cnt
```

4. **常用接口**
参考 `cmd/nvme.c` 中的代码接口，了解更多信息。

### 网络配置（Net）

本节介绍如何配置和调试 U-Boot 中的网络功能，包括以太网接口的配置和调试。

1. **config 配置**

   **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

   **开启网络相关配置**
   在 `make menuconfig` 菜单中，进入 **Device Drivers**，开启以下配置：

   ![a](static/uboot_menuconfig_6.png)

   ![a](static/uboot_menuconfig_7.png)

2. **dts 配置**
在 U-Boot 的设备树配置中，需要为以太网接口配置设备树节点。
**示例配置**：

```c
//uboot-2022.10/arch/riscv/dts/k3.dtsi
        eth0: ethernet@cac80000 {
			compatible = "spacemit,k3-dwmac-eqos";
			reg = <0x00000000 0xcac80000 0x00000000 0x00002000>;
			apmu-base-reg = <0xd4282800>;
			ctrl-reg = <0x3e4>;
			dline-reg = <0x3e8>;
			clocks = <&ccu CLK_EMAC0_BUS>;
			clock-names = "master_bus_clk";
			resets = <&reset RESET_EMAC0>;
			reset-names = "mac_reset";
			status = "disabled";
        };

//uboot-2022.10/arch/riscv/dts/k3_deb1.dts
&eth0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_gmac0>;

	phy-reset-pin = <15>;
	max-speed = <1000>;
	clk_tuning_enable;
	clk-tuning-by-delayline;

	tx-phase = <73>;
	rx-phase = <61>;

	phy-mode = "rgmii";
	phy-handle = <&rgmii0>;

	status = "okay";

	mdio {
		#address-cells = <0x1>;
		#size-cells = <0x0>;
		rgmii0: phy@0 {
			compatible = "ethernet-phy-ieee802.3-c22";
			device_type = "ethernet-phy";
			reg = <0x1>;
		};
	};
};
```

3. **调试验证**
需要先开启编译配置 `CONFIG_CMD_NET`，并确保网线已连接到开发板的网口，且已经准备好 TFTP 服务器（TFTP 服务器的搭建方法可参考网上资料，这里不做介绍）。
**调试命令示例**：

```shell
=> dhcp #执行dhcp后，如果返回地址，表示与网络服务器联通。其他情况为连接失败
ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
emac_adjust_link link:1 speed:1000 duplex:full
BOOTP broadcast 1
BOOTP broadcast 2
BOOTP broadcast 3
BOOTP broadcast 4
BOOTP broadcast 5
BOOTP broadcast 6
BOOTP broadcast 7
DHCP client bound to address 10.0.92.130 (7982 ms)

=> tftpboot 0x140000000 site11/uImage.itb
ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
emac_adjust_link link:1 speed:1000 duplex:full
Using ethernet@cac80000 device
TFTP from server 10.0.92.134; our IP address is 10.0.92.130
Filename 'site11/uImage.itb'.
Load address: 0x140000000
Loading: ##############################################################
         ########
         1.1 MiB/s
done
Bytes transferred = 66900963 (3fcd3e3 hex)
=>

#启动kernel
=>bootm 0x140000000
```

4. **常用接口**

参考 `cmd/net.c` 中的代码接口，了解更多信息。

### SPI 配置与调试

SPI（Serial Peripheral Interface）是一种常用的串行通信协议，用于连接微控制器和各种外围设备。在 U-Boot 中，SPI 驱动用于支持 NAND 或 NOR Flash。

1. **config 配置**
**进入 `make menuconfig`**
   执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

   **进入 Device Drivers**
   在 `make menuconfig` 菜单中，进入 **Device Drivers**，开启以下配置：

   ![a](static/uboot_menuconfig_8.png)

   ![a](static/uboot_menuconfig_9.png)

2. **dts 配置**
   在 U-Boot 的设备树配置中，需要为 SPI 接口配置设备树节点。

   **示例配置**：

```c
//k3.dtsi
        qspi: spi@d420c000 {
			compatible = "spacemit,k3-qspi";
			#address-cells = <1>;
			#size-cells = <0>;
			reg = <0x0 0xd420c000 0x0 0x1000>,
			      <0x0 0xb8000000 0x0 0x4100000>;
			reg-names = "qspi-base", "qspi-mmap";
			qspi-sfa1ad = <0x800000>;
			qspi-sfa2ad = <0x1000000>;
			qspi-sfb1ad = <0x1800000>;
			qspi-sfb2ad = <0x2000000>;
			qspi-pmuap-reg = <0xd4282860>;
			qspi-mpmu-acgr-reg = <0xd4051024>;
			qspi-ahbread = <1>;
			qspi-freq = <26500000>;
			qspi-id = <4>;
			clocks = <&ccu CLK_QSPI>,
			          <&ccu CLK_QSPI_BUS>;
			clock-names = "qspi_clk", "qspi_bus_clk";
			resets = <&reset RESET_QSPI>,
				<&reset RESET_QSPI_BUS>;
			reset-names = "qspi_reset", "qspi_bus_reset";
			status = "disabled";
        };

//k3_deb1.dts
&qspi {
	status = "okay";

	flash@0 {
		compatible = "jedec,spi-nor";
		reg = <0>;
		spi-max-frequency = <26500000>;
		m25p,fast-read;
		broken-flash-reset;
		status = "okay";
	};
};
```

3. **调试验证**

   需要开启 U-Boot Shell 中的 `sspi` 命令配置 `CONFIG_CMD_SPI`。

   **调试命令：**

```c
sspi -h

    "SPI utility command",
    "[<bus>:]<cs>[.<mode>][@<freq>] <bit_len> <dout> - Send and receive bits\n"
    "<bus>     - Identifies the SPI bus\n"
    "<cs>      - Identifies the chip select\n"
    "<mode>    - Identifies the SPI mode to use\n"
    "<freq>    - Identifies the SPI bus frequency in Hz\n"
    "<bit_len> - Number of bits to send (base 10)\n"
    "<dout>    - Hexadecimal string that gets sent"

```

4. **常用接口**

   参考 `cmd/spi.c` 中的代码接口，了解更多信息。

### NAND 配置与调试

NAND 驱动基于 SPI 接口实现，因此需要先开启 SPI 驱动功能。以下内容将介绍如何配置和调试 NAND 驱动。

1. **config 配置**
  执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

   **进入 Device Drivers -> MTD Support**
   在 `make menuconfig` 菜单中，进入 **Device Drivers** -> **MTD Support**，开启以下配置：

   ![a](static/uboot_menuconfig_10.png)

   若需要新增一个 NAND Flash，可以根据已支持的厂商驱动，添加该 NAND Flash 的 JEDEC ID。

   **示例**：

```shell
~/uboot-2022.10$ ls drivers/mtd/nand/spi/*.c -l
drivers/mtd/nand/spi/core.c
drivers/mtd/nand/spi/gigadevice.c
drivers/mtd/nand/spi/macronix.c
drivers/mtd/nand/spi/micron.c
drivers/mtd/nand/spi/other.c
drivers/mtd/nand/spi/toshiba.c
drivers/mtd/nand/spi/winbond.c
```

如在 `gigadevice` 添加新的 Flash:

```c
//uboot-2022.10/drivers/mtd/nand/spi/gigadevice.c
 static const struct spinand_info gigadevice_spinand_table[] = {
     SPINAND_INFO("GD5F1GQ4UExxG", 0xd1,
              NAND_MEMORG(1, 2048, 128, 64, 1024, 1, 1, 1),
              NAND_ECCREQ(8, 512),
              SPINAND_INFO_OP_VARIANTS(&gd5fxgq4_read_cache_variants,
                           &write_cache_variants,
                           &update_cache_variants),
              0,
              SPINAND_ECCINFO(&gd5fxgqxxexxg_ooblayout,
                      gd5fxgq4xexxg_ecc_get_status)),
     SPINAND_INFO("GD5F1GQ5UExxG", 0x51,
              NAND_MEMORG(1, 2048, 128, 64, 1024, 1, 1, 1),
              NAND_ECCREQ(4, 512),
              SPINAND_INFO_OP_VARIANTS(&gd5f1gq5_read_cache_variants,
                           &write_cache_variants,
                           &update_cache_variants),
              0,
              SPINAND_ECCINFO(&gd5fxgqxxexxg_ooblayout,
                      gd5fxgq5xexxg_ecc_get_status)),
 };
```

**说明**：如果是其他品牌的 NAND Flash，可以参考 `gigadevice` 的驱动代码实现。

2. **dts 配置**

   NAND 驱动挂在 SPI 驱动下，因此需要在 SPI 节点下配置。

   **示例配置**：

```c
 &qspi {
     status = "okay";
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_qspi>;

     spi-nand@0 {
         compatible = "spi-nand";
         reg = <0>;
         spi-tx-bus-width = <1>;
         spi-rx-bus-width = <1>;
         spi-max-frequency = <6250000>;
         u-boot,dm-spl;
         status = "okay";
     };
 };
```

3. **调试验证**
NAND 驱动可以基于 MTD 命令进行调试。需要开启 U-Boot Shell 中的 `CONFIG_CMD_MTD` 配置。
**调试命令**：

```shell
=> mtd
mtd - MTD utils

Usage:
mtd - generic operations on memory technology devices

mtd list
mtd read[.raw][.oob]                  <name> <addr> [<off> [<size>]]
mtd dump[.raw][.oob]                  <name>        [<off> [<size>]]
mtd write[.raw][.oob][.dontskipff]    <name> <addr> [<off> [<size>]]
mtd erase[.dontskipbad]               <name>        [<off> [<size>]]

Specific functions:
mtd bad                               <name>

With:
        <name>: NAND partition/chip name (or corresponding DM device name or OF path)
        <addr>: user address from/to which data will be retrieved/stored
        <off>: offset in <name> in bytes (default: start of the part)
                * must be block-aligned for erase
                * must be page-aligned otherwise
        <size>: length of the operation in bytes (default: the entire device)
                * must be a multiple of a block for erase
                * must be a multiple of a page otherwise (special case: default is a page with dump)

The .dontskipff option forces writing empty pages, don't use it if unsure.

=> mtd list
[RESET]spacemit_reset_set assert=1, id=77
[RESET]spacemit_reset_set assert=1, id=78
clk qspi_bus_clk already disabled
clk qspi_clk already disabled
ccu_mix_set_rate of qspi_clk timeout
[RESET]spacemit_reset_set assert=0, id=77
[RESET]spacemit_reset_set assert=0, id=78
SF: Detected w25q32 with page size 256 Bytes, erase size 64 KiB, total 4 MiB
Could not find a valid device for spi-nand
List of MTD devices:
* nor0
  - device: flash@0
  - parent: spi@d420c000
  - driver: jedec_spi_nor
  - path: /soc/spi@d420c000/flash@0
  - type: NOR flash
  - block size: 0x10000 bytes
  - min I/O: 0x1 bytes
  - 0x000000000000-0x000000400000 : "nor0"
          - 0x0000000a0000-0x000000100000 : "opensbi"
          - 0x000000100000-0x000000300000 : "uboot"
=> mtd read/write partname addr off size
```

4. 常用接口
参考 `cmd/mtd.c` 中的代码接口，了解更多信息。

### NOR 配置与调试

NOR 驱动基于 SPI 接口实现，因此需要先开启 SPI 驱动功能。以下内容将介绍如何配置和调试 NOR 驱动。

1. **config 配置**

   **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

   ```shell
   make menuconfig
   ```

   **进入 Device Drivers -> MTD Support -> SPI Flash Support**
   在 `make menuconfig` 菜单中，进入 **Device Drivers** -> **MTD Support** -> **SPI Flash Support**，开启以下配置：

![a](static/uboot_menuconfig_11.png)

![a](static/uboot_menuconfig_12.png)

**添加一个新的  SPI NOR Flash**

- 对于已支持的厂商 NOR Flash，可以直接开启对应的编译配置。例如，对于 GigaDevice 厂商的 Flash，可以开启对应的配置。
- **检查 JEDEC ID 列表**：SPI Flash 的 JEDEC ID 列表在 `uboot-2022.10/drivers/mtd/spi/spi-nor-ids.c` 文件中维护。如果列表中没有特定的 NOR Flash JEDEC ID，可以自行添加。（JEDEC ID 为 SPI Flash 对应的厂商代号，可根据 NOR Flash 的 数据手册 中查找 `manufac` 关键字，如 winbond 为 `0xfe`）

2. **dts 配置**
   NOR 驱动依赖 SPI 驱动接口，因此需要在 SPI 节点下添加 NOR Flash 的设备树节点。

   **示例配置**：

```c
//k3/uboot-2022.10/arch/riscv/dts/k3_deb1.dts
 &qspi {
     status = "okay";
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_qspi>;

     flash@0 {
         compatible = "jedec,spi-nor";
         reg = <0>;
         spi-max-frequency = <26500000>;
         m25p,fast-read;
         broken-flash-reset;
         status = "okay";
     };
 };
```

3. **调试验证**
   NOR 驱动可以通过 U-Boot 命令行的 `mtd` 和 `sf` 命令进行调试。需要确保编译配置中开启了 `CONFIG_CMD_MTD=y` 和 `CONFIG_CMD_SF`。

   基于 `mtd` 命令读写 NOR Flash

```shell
=> mtd list
List of MTD devices:
* nor0
  - device: flash@0
  - parent: spi@d420c000
  - driver: jedec_spi_nor
  - path: /soc/spi@d420c000/flash@0
  - type: NOR flash
  - block size: 0x1000 bytes
  - min I/O: 0x1 bytes
  - 0x000000000000-0x000000400000 : "nor0"
          - 0x000000020000-0x000000040000 : "fsbl"
          - 0x0000000a0000-0x0000000b0000 : "env"
          - 0x0000000b0000-0x0000001b0000 : "opensbi"
          - 0x0000001b0000-0x000000400000 : "uboot"
=>

=> mtd
mtd - MTD utils

Usage:
mtd - generic operations on memory technology devices

mtd list
mtd read[.raw][.oob]                  <name> <addr> [<off> [<size>]]
mtd dump[.raw][.oob]                  <name>        [<off> [<size>]]
mtd write[.raw][.oob][.dontskipff]    <name> <addr> [<off> [<size>]]
mtd erase[.dontskipbad]               <name>        [<off> [<size>]]

Specific functions:
mtd bad                               <name>

With:
        <name>: NAND partition/chip name (or corresponding DM device name or OF path)
        <addr>: user address from/to which data will be retrieved/stored
        <off>: offset in <name> in bytes (default: start of the part)
                * must be block-aligned for erase
                * must be page-aligned otherwise
        <size>: length of the operation in bytes (default: the entire device)
                * must be a multiple of a block for erase
                * must be a multiple of a page otherwise (special case: default is a page with dump)

The .dontskipff option forces writing empty pages, don't use it if unsure.

=>
=> mtd read uboot 0x140000000
Reading 1048576 byte(s) at offset 0x00000000
=> mtd dump uboot 0 0x10
Reading 16 byte(s) at offset 0x00000000

Dump 16 data bytes from 0x0:
0x00000000:     d0 0d fe ed 00 0d e8 95  00 00 00 38 00 0d e4 44
=>
```

基于 `sf` 命令读写 NOR Flash

```shell
=> sf
sf - SPI flash sub-system

Usage:
sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus
                                  and chip select
sf read addr offset|partition len       - read `len' bytes starting at
                                          `offset' or from start of mtd
                                          `partition'to memory at `addr'
sf write addr offset|partition len      - write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf erase offset|partition [+]len        - erase `len' bytes from `offset'
                                          or from start of mtd `partition'
                                         `+len' round up `len' to block size
sf update addr offset|partition len     - erase and write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf protect lock/unlock sector len       - protect/unprotect 'len' bytes starting
                                          at address 'sector'
=> sf probe
SF: Detected w25q32 with page size 256 Bytes, erase size 4 KiB, total 4 MiB
=> sf read 0x140000000 0 0x10
device 0 offset 0x0, size 0x10
SF: 16 bytes @ 0x0 Read: OK
=>
```

4. **常用接口**

```c
include <spi.h>
#include <spi_flash.h>

struct udevice *new, *bus_dev;
int ret;
static struct spi_flash *flash;

//bus,cs对应spi的bus和cs编号，如0，0
ret = spi_find_bus_and_cs(bus, cs, &bus_dev, &new);
flash = spi_flash_probe(bus, cs, speed, mode);

ret = spi_flash_read(flash, offset, len, buf);
```

### HDMI 配置与调试

本小节主要介绍如何开启 HDMI 驱动。

1. **config 配置**

   **进入 `make uboot_menuconfig`**
   执行以下命令进入配置菜单：

   ```shell
   make uboot_menuconfig
   ```

   **进入 Device Drivers -> Graphics support**
   在 `make uboot_menuconfig` 菜单中，进入 **Device Drivers** -> **Graphics support**，开启以下配置(默认情况下已开启)。

![a](static/uboot_menuconfig_13.png)

![a](static/uboot_menuconfig_14.png)

![a](static/uboot_menuconfig_15.png)

![a](static/uboot_menuconfig_16.png)

![a](static/uboot_menuconfig_17.png)

2. **dts 配置**

   在设备树配置中，需要为 HDMI 驱动配置设备树节点。

   **示例配置**：

```c
&dpu {
        status = "okay";
};

&hdmi {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_hdmi_0>;
        status = "okay";
};
```

### Boot Logo 配置与显示

本小节主要介绍如何在 U-Boot 启动阶段显示 Boot Logo。

1. **config 配置**

   - **开启 HDMI 支持**: 首先，确保 U-Boot 已开启 HDMI 支持。具体步骤可以参考 **HDMI 配置与调试** 小节。

   - **开启 Boot Logo 支持**
   执行以下命令进入配置菜单：

     ```shell
     make menuconfig
     ```

     在 `make menuconfig` 菜单中，进入 **Device Drivers** -> **Graphics support**，开启以下选项：

     ![a](static/uboot_menuconfig_18.png)

2. **env 配置**

   在 U-Boot 的配置文件中，需要添加 Boot Logo 所需的环境变量。这些变量包括 `splashimage`、`splashpos` 和 `splashfile`。

   **示例配置**：在 uboot-2022.10\include\configs 目录下的 `k3.h` 增加 Boot Logo 所需的 3 个 env 变量：`splashimage`、`splashpos` 和 `splashfile`。

```c
//uboot-2022.10/include/configs/k3.h
    ... ...
    ... ...
 #define CONFIG_EXTRA_ENV_SETTINGS \
        ... ...
        ... ...
     "splashimage=" __stringify(CONFIG_FASTBOOT_BUF_ADDR) "\0" \
     "splashpos=m,m\0" \
     "splashfile=bianbu.bmp\0" \

        ... ...
        ... ...
     BOOTENV_DEVICE_CONFIG \
     BOOTENV

 #endif /* __CONFIG_H */
```

**说明**：

- `splashimage`：Boot Logo 图片加载到内存的地址。
- `splashpos`：图片显示的位置。`"m,m"` 表示图片显示在屏幕正中间。
- `splashfile`：将要显示的 BMP 文件名。该文件需要放在 bootfs 所在的分区。

3. **打包 BMP 图片进 BootFS**

   将 Boot Logo 的 BMP 图片打包进 BootFS：

   - **准备图片文件**
   将 `bianbu.bmp` 文件放在 `./buildroot-ext/board/spacemit/k3` 目录下。文件名需要与 `buildroot-ext/board/spacemit/k3/prepare_img.sh` 中的 `UBOOT_LOGO_FILE` 和环境变量 `splashfile` 保持一致。

   - **修改打包脚本**
   确保 `prepare_img.sh` 脚本中正确引用了 `bianbu.bmp` 文件：

   - **编译打包**
   在编译打包后，BMP 图片将被打包进 BootFS。

```shell
//buildroot-ext/board/spacemit/k3/prepare_img.sh
#!/bin/bash

######################## Prepare sub-iamges and pack ####################
#$0 is this file path
#$1 is buildroot output images dir

IMGS_DIR=$1
DEVICE_DIR=$(dirname $0)

... ...

UBOOT_LOGO_FILE="$DEVICE_DIR/bianbu.bmp"

```

4. 修改 Boot Logo

   如果需要修改 Boot Logo, 直接替换 `buildroot-ext/board/spacemit/k3/` 目录中的 `bianbu.bmp` 文件。

### Boot Menu 配置与使用

本小节主要介绍如何开启 U-Boot 的 Boot Menu 功能。

1. **config 配置**

   - **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

     ```shell
     make menuconfig
     ```

   - **进入 Command line interface -> Boot commands**
   在 `make menuconfig` 菜单中，进入 **Command line interface** -> **Boot commands**，开启以下配置：

     ![a](static/uboot_menuconfig_19.png)

   - **进入 Boot options -> Autoboot options**
   再进入 **Boot options** -> **Autoboot options**，开启以下选项：
     ![a](static/uboot_menuconfig_20.png)

2. **env 配置**
在 `buildroot-ext/board/spacemit/k3/env_k3.txt` 文件中，需要添加 `bootdelay` 和 `bootmenu_delay` 环境变量。
例如：
   - `bootmenu_delay=5` 表示系统会在启动菜单界面等待 5 秒，用户可以选择启动选项。
   - `bootdelay=5` 表示选择启动方式后，系统在真正启动前再等待 5 秒。

```c
//buildroot-ext/board/spacemit/k3/env_k3.txt
bootdelay=5

# Boot menu definitions
boot_default=echo "Current Boot Device: ${boot_device}"
flash_default=echo "Returning to Boot Menu..."
bootmenu_delay=5
bootmenu_0="-------- Boot Options --------"=run boot_default
bootmenu_1="Boot from Nor"=run nor_boot
bootmenu_2="Boot from Nand"=run nand_boot
bootmenu_3="Boot from MMC"=run mmc_boot
bootmenu_4="Boot from UFS"=run ufs_boot
bootmenu_5="Autoboot"=run autoboot
```

3. **进入 Boot Menu**
   开发板上电启动后，立即按住键盘上的 `Esc` 键，进入 Boot Menu。

### Fastboot Command 配置与使用

本小节主要介绍 K3-DEB1 方案支持的 Fastboot 命令。

#### 编译配置
开启 Fastboot 支持如下步骤：
- **进入 `make menuconfig`**
   执行以下命令进入配置菜单：

     ```shell
     make menuconfig
     ```

- **进入 Device Drivers -> Fastboot support**
   在 `make menuconfig` 菜单中，进入 **Device Drivers** -> **Fastboot support**，开启以下编译配置：

     ![a](static/uboot_menuconfig_21.png)

- **开启 USB 支持**
   Fastboot 依赖 USB 驱动，需要开启 USB 的配置 **USB support**：

     ![a](static/uboot_menuconfig_22.png)

     ![a](static/uboot_menuconfig_23.png)

#### 进入 Fastboot 模式

**方法 1**： 通过 U-Boot Shell 进入 Fastboot 模式

- **进入 U-Boot Shell**
     系统启动后，按 `s` 键进入 U-Boot Shell。

- **执行 Fastboot 命令**
     执行以下命令进入 Fastboot 模式：

       ```shell
       => fastboot 0
       ```
  系统默认的 Fastboot 缓冲区地址和大小由宏定义 `CONFIG_FASTBOOT_BUF_ADDR` 和 `CONFIG_FASTBOOT_BUF_SIZE` 指定。

```shell
# => fastboot -l 0x130000000 -s 0x10000000 0，指定缓冲区地址和大小

=> fastboot 0 # 进入 Fastboot 模式

# 预期输出：
dwc3 udc: phy_init
dwc3 udc probe
dwc3 udc: pullup 1
-- suspend --
handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x40
handle setup SET_ADDRESS, 0x0, 0x5 index 0x0 value 0x22 length 0x0
handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x12
..
```

**方法 2**：**通过 Bianbu OS 进入 Fastboot 模式**
 设备启动到 Bianbu OS 后，发送以下命令使系统重启进入 Fastboot 模式：

   ```shell
   adb reboot bootloader
   ```

   **注**：某些方案可能不支持此功能。

#### 支持的 Fastboot 命令

电脑端的 Fastboot 环境配置，请参考 **电脑环境安装** 章节。

```shell
#fastboot原生协议命令
fastboot devices              #显示可用的设备
fastboot reboot               #重启设备
fastboot getvar [version/product/serialno/max-download-size]
fastboot flash partname image #烧写image镜像到partname分区
fastboot erase partname       #擦除partname分区
fastboot stage file           #下载文件file到内存的buff addr

#oem厂商自定义的命令和功能
fastboot getvar [mtd-size/blk-size] #获取mtd/blk设备size，没有则返回NULL
fastboot oem read part              #读取part中的数据到buff addr
fastboot get_staged file       #上传数据并命名为file。依赖oem read part命令
```

### 文件系统操作

- **FAT**

```shell
=> fat
  fatinfo fatload fatls fatmkdir fatrm fatsize fatwrite
=> fatls mmc 2:5
 50896911   uImage.itb
     4671   env_k3.txt

2 file(s), 0 dir(s)

=> fatload mmc 2:5 0x140000000 uImage.itb #load uImage.itb到0x40000000
50896911 bytes read in 339 ms (143.2 MiB/s)
=>
```

- **EXT4**
类似 `fat` 指令

```shell
=> ext4
  ext4load ext4ls ext4size
=> ext4load
ext4load - load binary file from a Ext4 filesystem

Usage:
ext4load <interface> [<dev[:part]> [addr [filename [bytes [pos]]]]]
    - load binary file 'filename' from 'dev' on 'interface'
      to address 'addr' from ext4 filesystem
=>
```

### 常用 U-Boot 命令

本小节主要介绍 U-Boot 中的一些常用命令。

1. **常用命令**

```shell
printenv  - print environment variables
md        - memory display
mw        - memory write (fill)
fdt       - flattened device tree utility commands



help      - print command description/usage
```

2. **fdt 命令**

`fdt` 命令主要用于操作和打印设备树（Device Tree）的内容，例如 U-Boot 启动后加载的 DTB 文件。

```shell
=> fdt
fdt - flattened device tree utility commands

Usage:
fdt addr [-c] [-q] <addr> [<size>]  - Set the [control] fdt location to <addr>
fdt move   <fdt> <newaddr> <length> - Copy the fdt to <addr> and make it active
fdt resize [<extrasize>]            - Resize fdt to size + padding to 4k addr + some optional <extrasize> if needed
fdt print  <path> [<prop>]          - Recursive print starting at <path>
fdt list   <path> [<prop>]          - Print one level starting at <path>
fdt get value <var> <path> <prop> [<index>] - Get <property> and store in <var>
                                      In case of stringlist property, use optional <index>
                                      to select string within the stringlist. Default is 0.
fdt get name <var> <path> <index>   - Get name of node <index> and store in <var>
fdt get addr <var> <path> <prop>    - Get start address of <property> and store in <var>
fdt get size <var> <path> [<prop>]  - Get size of [<property>] or num nodes and store in <var>
fdt set    <path> <prop> [<val>]    - Set <property> [to <val>]
fdt mknode <path> <node>            - Create a new node after <path>
fdt rm     <path> [<prop>]          - Delete the node or <property>
fdt header [get <var> <member>]     - Display header info
                                      get - get header member <member> and store it in <var>
fdt bootcpu <id>                    - Set boot cpuid
fdt memory <addr> <size>            - Add/Update memory node
fdt rsvmem print                    - Show current mem reserves
fdt rsvmem add <addr> <size>        - Add a mem reserve
fdt rsvmem delete <index>           - Delete a mem reserves
fdt chosen [<start> <size>]         - Add/update the /chosen branch in the tree
                                        <start>/<size> - initrd start addr/size
NOTE: Dereference aliases by omitting the leading '/', e.g. fdt print ethernet0.
=>

# 示例操作：
=> fdt addr $fdtcontroladdr # 设置控制设备树的地址
=> fdt print                # 打印整个设备树内容
/ {
        compatible = "spacemit,k3", "riscv";
        #address-cells = <0x00000002>;
        #size-cells = <0x00000002>;
        model = "spacemit k3 deb1 board";
... ...
        memory@0 {
                device_type = "memory";
                reg = <0x00000000 0x00000000 0x00000000 0x80000000>;
        };
        chosen {
                bootargs = "earlycon=sbi console=ttyS0,115200 debug loglevel=8,initcall_debug=1 rdinit=/init.tmp";
                stdout-path = "serial0:115200n8";
        };
};

=> fdt print /chosen # 打印特定路径的内容
chosen {
        bootargs = "earlycon=sbi console=ttyS0,115200 debug loglevel=8,initcall_debug=1 rdinit=/init.tmp";
        stdout-path = "serial0:115200n8";
};
=>
```

3. **Shell 命令**
U-Boot 支持 Shell 风格的命令，如 `if/fi` 和 `echo` 等。

```shell
=> if test ${boot_device} = nand; then echo "nand boot"; else echo "not nand boot";fi
not nand boot
=> printenv boot_device
boot_device=nor
=> if test ${boot_device} = nor; then echo "nor boot"; else echo "not nor boot";fi
nor boot
=>
```

## OpenSBI 功能与配置

本章节介绍 OpenSBI 的编译配置

### OpenSBI 编译

```shell
cd ~/opensbi/ # 进入 OpenSBI 目录

# 注意：确保使用正确的编译工具链。这里使用的工具链需要由 SpacemiT 提供，否则可能会引起编译异常

GCC_PREFIX=riscv64-unknown-linux-gnu- # 设置编译工具链

CROSS_COMPILE=${GCC_PREFIX} PLATFORM=generic \
PLATFORM_DEFCONFIG=k3_deb1_defconfig \
PLATFORM_RISCV_ISA=rv64gc \
FW_TEXT_START=0x0  \
make
```

### 生成的编译文件

编译完成后，生成的文件位于 `build/platform/generic/firmware/` 目录下：

```shell
~/opensbi$ ls build/platform/generic/firmware/ -l
fw_dynamic.bin     #dynamic镜像跳转会传递配置参数
fw_dynamic.elf
fw_dynamic.elf.dep
fw_dynamic.itb     #将fw_dynamic.bin打包成fit格式
fw_dynamic.its
fw_dynamic.o
fw_jump.bin       #jump镜像仅做跳转
fw_payload.bin    #payload镜像会包含uboot镜像
```

### OpenSBI 功能配置

可以通过执行 `menuconfig`，开启或关闭某些功能

```shell
make PLATFORM=generic PLATFORM_DEFCONFIG=k3_deb1_defconfig menuconfig
```

示例配置：
![a](static/uboot_menuconfig_24.png)

## FAQ

本章节介绍常见问题以及解决方式，或者常用的调试手段以及容易出错的问题记录。

### 用 TitanFlash 烧写固件时，没有检测到设备

- **检查 USB 连接**
   确保 USB 线已接入电脑，且串口打印正常, 如下所示：
   ![alt text](static/flash_tool_3.png)

如果串口打印显示设备已连接，但 TitanFlash 仍然无法检测到设备，请检查以下内容：

- **检查设备管理器**
  在 Windows 设备管理器中，检查是否存在 ADB 设备。如果没有，则需要安装相应的驱动程序。

![alt text](static/flash_tool_4.png)

- **参考安装指南**
  如果设备管理器中没有显示 ADB 设备，请参考 **电脑环境安装章节** 中的 Fastboot 环境安装部分。

### 更新代码后有涉及到def_config的改动，编译的时候没有生效

需要执行 `make menuconfig` 更新到 `.config`，编译才生效。

### 更改 FSBL 信息

FSBL的位置、长度信息在bootinfo中指定，如需更改 FSBL 信息，需按如下步骤完成修改：

**1. 更新bootinfo**

根据启动介质，修改对应的bootinfo_*.json文件，更新

```c
// board/spacemit/k3/configs/bootinfo_*.json
            "spl0_offset, 0x180000, 4",
            "spl1_offset, 0x200000, 4",
            "spl_size_limit, 0x74000, 4",
```

**2. 更新分区表**

根据启动介质，修改对应分区表中FSBL分区起始地址。

```c
// buildroot-ext/board/spacemit/k3/partition_*.json
    {
      "name": "fsbl",
      "comment": "its offset was designated by bootinfo",
      "hidden": true,
      "offset": "1536K",
      "size": "512K",
      "image": "factory/FSBL.bin"
    },
```

**3. 更新mtdparts**

对于mtd设备，还需通过menuconfig，更新MTDPARTS_DEFAULT配置(CONFIG_MTDPARTS_DEFAULT).

![a](static/uboot_MTDPARTS_DEFAULT.png)

### 如何设置隐藏分区

分区表 `partition_universal.json` 支持隐藏分区的功能，`hidden` 标签用来标注隐藏分区。刷机启动后，隐藏分区不会出现在 GPT 表中。

存在隐藏分区的分区表，需要用 TitanFlasher 工具烧写镜像，或者执行 `fastboot flash gpt partition_universal.json` 后，才能使用 Fastboot 命令烧写隐藏分区的镜像。

目前仅支持 blk 设备的隐藏分区功能，如 eMMC、SSD、UFS 等。

带隐藏分区的分区表参考如下：不添加 `hidden` 标签默认为显示分区。
`cat k3/common/flash_config/partition_universal.json`

```sh
    {
      "name": "fsbl",
      "comment": "its offset was designated by bootinfo",
      "hidden": true,
      "offset": "1536K",
      "size": "512K",
      "image": "factory/FSBL.bin"
    },
```

### 不支持烧写 gzip 格式镜像文件

SpacemiT 刷机机制会将大文件镜像压缩成 gzip 格式，以解决 USB 传输大文件数据慢的问题。刷机服务会默认检测数据是否为 gzip 格式，并进行解压处理。如果需要烧写 gzip 格式的镜像到自定义分区，可以采用以下方法：

- 方式一：对指定的分区不做gzip检查，如分区usbfs

```c
diff --git a/drivers/fastboot/fb_mmc.c b/drivers/fastboot/fb_mmc.c
index 88d8778376..a37cbde596 100644
--- a/drivers/fastboot/fb_mmc.c
+++ b/drivers/fastboot/fb_mmc.c
@@ -689,7 +689,7 @@ void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
            fastboot_mmc_get_part_info(cmd, &dev_desc, &info, response) < 0)
                return;

-       if (check_gzip_format((uchar *)download_buffer, src_len) >= 0) {
+       if (check_gzip_format((uchar *)download_buffer, src_len) >= 0 && strcmp("usbfs", cmd) != 0) {
                /*is gzip data and equal part name*/
                gzip_image = true;
                if (strcmp(cmd, part_name_t)){


```

- 方式二：关闭压缩镜像刷机功能(***不推荐，导致刷机时间更长***)

```c
//关闭刷机服务压缩镜像的检查
//uboot-2022.10
diff --git a/drivers/fastboot/fb_mmc.c b/drivers/fastboot/fb_mmc.c
index 88d8778376..5f8d4f8931 100644
--- a/drivers/fastboot/fb_mmc.c
+++ b/drivers/fastboot/fb_mmc.c
@@ -689,7 +689,7 @@ void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
            fastboot_mmc_get_part_info(cmd, &dev_desc, &info, response) < 0)
                return;

-       if (check_gzip_format((uchar *)download_buffer, src_len) >= 0) {
+       if (false) {
                /*is gzip data and equal part name*/
                gzip_image = true;
                if (strcmp(cmd, part_name_t)){


//关闭刷机工具对镜像的压缩处理
//buildroot-ext，可直接改刷机包里面的partition_universal.json，注意json格式最后一行不能带逗号
diff --git a/board/spacemit/k3/partition_universal.json b/board/spacemit/k3/partition_universal.json
index e02a1d0..e9da5a9 100644
--- a/board/spacemit/k3/partition_universal.json
+++ b/board/spacemit/k3/partition_universal.json
@@ -36,15 +36,13 @@
             "name": "bootfs",
             "offset": "4M",
             "size": "256M",
-            "image": "bootfs.img",
-            "compress": "gzip-5"
+            "image": "bootfs.img"
         },
         {
             "name": "rootfs",
             "offset": "260M",
             "size": "-",
-            "image": "rootfs.ext4",
-            "compress": "gzip-5"
+            "image": "rootfs.ext4"
         }
     ]
 }
```
