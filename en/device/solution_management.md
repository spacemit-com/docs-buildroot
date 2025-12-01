---
sidebar_position: 2
---

# Solution Management

This document introduces how the SDK manages solutions, including solution configuration files, how to customize solutions, and how to add new solutions.

## Solution Configuration Files

Taking the default solution K1 as an example, it usually includes the following configuration files:

```shell
buildroot-ext/board/spacemit/k1/partition_*.json
buildroot-ext/board/spacemit/k1/env_k1-x.txt
buildroot-ext/board/spacemit/k1/k1-x.bmp
buildroot-ext/board/spacemit/k1/dracut.conf
buildroot-ext/board/spacemit/k1/target_overlay
buildroot-ext/configs/spacemit_k1_defconfig
bsp-src/opensbi/platform/generic/configs/k1_defconfig
bsp-src/uboot-2022.10/configs/k1_defconfig
bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig
```

**buildroot-ext/board/spacemit/k1/partition_*.json**

Partition configuration files:

- `partition_2M.json`：for 2MB flash
- `partition_universal.json`：for high capacity flash, such as eMMC, sdcard, SSD

**buildroot-ext/board/spacemit/k1/env_k1-x.txt**

The environment variables of u-boot, customizable boot parameters.

**buildroot-ext/board/spacemit/k1/bianbu.bmp**

u-boot boot logo：

- Format: BMP
- Resolution: less than or equal to screen resolution
- Bit depth: 32

**buildroot-ext/board/spacemit/k1/dracut.conf**

Dracut configuration file, Dracut is a tool for creating initramfs images.

**buildroot-ext/board/spacemit/k1/target_overlay**

This directory is an overlay for the target directory.

**buildroot-ext/configs/spacemit_k1_defconfig**

Buildroot configuration file.

**bsp-src/opensbi/platform/generic/configs/k1_defconfig**

opensbi configuration file.

**bsp-src/uboot-2022.10/configs/k1_defconfig**

u-boot configuration file.

**bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig**

Kernel configuration file.

## Default Partition Configuration

Try not to modify the partition configuration before `bootfs` to avoid hard-coded boot code, which may cause boot failure after modification.

### eMMC Partition Configuration

Configured according to the `partitions` array in the partition_universal.json file, where bootinfo and fsbl are located in the boot0 partition of the eMMC.

eMMC boot0 partition configuration is as follows:

```
-----------------------------
|/bootinfo/        | fsbl   |
-----------------------------
^         ^        ^        ^
|         |        |        |
0K        80bytes  512bytes 198.5K
```

eMMC GPP partition configuration is as follows:

```
--------------------------------------------------------------------------------------------
| gpt                            | env | opensbi | uboot   |    bootfs      |    rootfs    |
--------------------------------------------------------------------------------------------
^                                ^     ^         ^         ^                ^              ^
|                                |     |         |         |                |              |
0K                               384K  448K      832K      4M               260M           Capacity
```

**Note**

- If booting from eMMC, the Boot ROM loads the FSBL from the boot0 partition rather than the GPP partition. The FSBL in the GPP partition is ignored in this case.

### sdcard Partition Configuration

Configured entirely according to the `partitions` array in the partition_universal.json file.

Partition configuration is as follows:

```
--------------------------------------------------------------------------------------------
|/bootinfo/ gpt           | fsbl | env | opensbi | uboot   |    bootfs    |    rootfs    |
--------------------------------------------------------------------------------------------
^         ^               ^      ^     ^         ^         ^              ^              ^
|         |               |      |     |         |         |              |              |
0K        80bytes         128K   384K  448K      832K      4M             260M           Capacity
```

### SPINOR + SSD Partition Configuration

In SSD solutions, SPINOR is used for booting. All partitions before `bootfs` are stored in SPINOR, and configured according to the `partitions` array in the `partition_<SPINOR size>.json` file.

SPINOR partition configuration is as follows:

```
----------------------------------------------------
|/bootinfo/         | fsbl | env | opensbi | uboot |
----------------------------------------------------
^         ^         ^      ^     ^         ^       ^
|         |         |      |     |         |       |
0K        80bytes   128K   384K  448K      640K    Capacity
```

bootfs and rootfs are stored in SSD, and configured according to the `partitions` array in the partition_universal.json file.

```
------------------------------------------------------------------------------------------
| gpt                                                      |    bootfs    |    rootfs    |
------------------------------------------------------------------------------------------
^                                                          ^              ^              ^
|                                                          |              |              |
0K                                                         4M             260M           Capacity
```

## Customizing Solutions

## Adding New Solutions
