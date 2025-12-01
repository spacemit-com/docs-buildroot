---
sidebar_position: 1.5
---

# Source

This document provides an overview of the development environment, as well as instructions for downloading and compiling the SDK source code.

## Development Environment

### Hardware Configuration

Recommended configuration:

- CPU: 12th Gen Intel(R) Core(TM) i5 or above
- Memory: 16GB or above
- Disk: SSD, 256GB or above

### Operating System

- For Buildroot 2.2.7 and later

  Recommended: Ubuntu 20.04 or a newer LTS version, or any Linux distribution that supports Docker.

- For Buildroot 2.2.6 and earlier

  Recommended: Ubuntu 20.04 or a newer LTS version; other Linux distributions are untested.

### Install Dependencies

For Buildroot 2.2.7 and later, containerized builds are supported by default, so you only need to [install Docker CE](https://docs.docker.com/engine/install/)

If you prefer to build directly on the host, install the dependencies using the guide below:

Ubuntu 16.04 and 18.04:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

Ubuntu 20.04 or higher:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 python-is-python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

## Download

### Preparation

The Buildroot code is hosted on Gitee and Github, consists of multiple repositories managed using `repo`. Before downloading, you need to:

1. If downloading from Gitee, please refer to [this document](https://gitee.com/help/articles/4191) to set up SSH Keys first; if downloading from Github, please refer to [this document](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) to set up SSH Keys first.

2. Install repo

   If you have access to Google, please refer to [this document](https://gerrit.googlesource.com/git-repo/+/refs/heads/main/README.md#install) for installation; otherwise, refer to [Git Repo Mirror Usage Guide](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/) for installation.

   The repo version must be >= 2.41, otherwise the download process may fail.

```shell
repo --version
repo version v2.48
       (from https://mirrors.tuna.tsinghua.edu.cn/git/git-repo)
       (tracking refs/heads/stable)
       (Mon, 7 Oct 2024 18:44:19 +0000)
repo launcher version 2.50
       (from /usr/bin/repo)
       (currently at 2.48)
repo User-Agent git-repo/2.48 (Linux) git/2.25.1 Python/3.8.10
git 2.25.1
git User-Agent git/2.25.1 (Linux) git-repo/2.48
Python 3.8.10 (default, Nov  7 2024, 13:10:47)
[GCC 9.4.0]
OS Linux 5.4.0-196-generic (#216-Ubuntu SMP Thu Aug 29 13:26:53 UTC 2024)
CPU x86_64 (x86_64)
Bug reports: https://issues.gerritcodereview.com/issues/new?component=1370071
```

If the version does not meet the requirement, or if the source is not the Tsinghua mirror, refer to [Git Repo Mirror Usage Help](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/) for installation.

### Version Branches

manifests repository defines the `manifest.xml` files for different versions. The xml files specify the paths and branches of each repository.

| Version | manifest.xml | Branch |
| ------- | ------------- | ------ |
| v1.0    | bl-v1.0.y.xml | bl-v1.0.y |
| v2.0    | bl-v2.0.y.xml | bl-v2.0.y |
| v2.1    | k1-bl-v2.1.y.xml | k1-bl-v2.1.y |
| v2.2    | k1-bl-v2.2.y.xml | k1-bl-v2.2.y |

**Notice**

- Due to Gitee's single repository capacity limitations, the bl-v2.0.y and k1-bl-v2.1.y branches of the linux-6.6 repository have been moved to separate repositories: [linux-6.6-v2.0.y](https://gitee.com/bianbu-linux/linux-6.6-v2.0.y) and [linux-6.6-v2.1.y](https://gitee.com/bianbu-linux/linux-6.6-v2.1.y). If you are using v2.0 or v2.1, you will not be able to repo sync or git pull the linux-6.6 repository and will need to re-download. We apologize for any inconvenience this may cause.
- Github only hosts v2.2 and later versions.

### Download Code

For example, to download the code for version 2.2:

#### Download from Gitee

```shell
mkdir ~/buildroot-sdk-2.2
cd ~/buildroot-sdk-2.2
repo init -u git@gitee.com:bianbu-linux/manifests.git -b main -m k1-bl-v2.2.y.xml
repo sync
repo start k1-bl-v2.2.y --all
```

#### Download from Github

```shell
mkdir ~/buildroot-sdk-2.2
cd ~/buildroot-sdk-2.2
repo init -u git@github.com:spacemit-com/manifests.git -b main -m k1-bl-v2.2.y.xml
repo sync
repo start k1-bl-v2.2.y --all
```

To download a different branch, specify another `manifest.xml` file using the `-m` option.

After downloading the source code, it is recommended to download the third-party software packages required by buildroot in advance and distribute them within the team to avoid network congestion on the main server.

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl
```

## Directory structure

```shell
├── bsp-src               # Stores linux kernel、uboot、opensbi source
│   ├── linux-6.6
│   ├── opensbi
│   └── uboot-2022.10
├── buildroot             # buildroot main directory
├── buildroot-ext         # Custom extensions, including board, configs, package, patches subdirectories
├── Makefile              # Top-level Makefile
├── package-src           # Directory for locally expanded application or library source code
│   ├── ai-support
│   ├── drm-test
│   ├── img-gpu-powervr
│   ├── k1x-cam
│   ├── k1x-jpu
│   ├── k1x-vpu-firmware
│   ├── k1x-vpu-test
│   ├── mesa3d
│   ├── mpp
│   └── v2d-test
└── scripts               # Scripts used for compilation
```

## Cross Compilation

### Buildroot 2.x First Complete Compilation

For the first compilation, it is recommended to use `make envconfig` for a complete build.

If you have modified `buildroot-ext/configs/spacemit_<solution>_defconfig`, use `make envconfig` to compile.

In other cases, use `make` to compile.

```shell
cd ~/buildroot-sdk
make envconfig
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_upstream_defconfig
  2. spacemit_k1_minimal_defconfig
  3. spacemit_k1_plt_defconfig
  4. spacemit_k1_rt_defconfig
  5. spacemit_k1_v2_defconfig


your choice (1-5):

```

To compile Buildroot 2.x version, enter `5` and press Enter to start compiling.

Note: Starting with Buildroot 2.2.7

1. Builds default to running inside a container. If you still want to build on the host, set the environment variable `export DIRECT_BUILD=1`. When switching between container and host builds you must clean the `output` directory.
2. A new set of commands has been added and is not compatible with the old commands. To use the new commands, delete the `env.mk` file generated by the old commands in the project root, then run `make help` to see the new command list. For example, you can run `make k1_v2-build` to build a specific configuration directly.

The compilation process may require downloading some third-party software packages, and the time required depends on the network environment. If you have downloaded the third-party software packages required by buildroot in advance, the recommended hardware configuration compilation time is about 1 hour.

After the compilation is complete, you will see:

```shell
Images successfully packed into /home/username/bianbu-linux/output/k1_v2/images/bianbu-linux-k1_v2.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/home/username/bianbu-linux/output/k1_v2/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/home/username/bianbu-linux/output/k1_v2/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/home/username/bianbu-linux/output/k1_v2/images"" (stderr):
INFO: hdimage(sdcard.img): adding partition 'bootinfo' from 'factory/bootinfo_sd.bin' ...
INFO: hdimage(sdcard.img): adding partition 'fsbl' (in MBR) from 'factory/FSBL.bin' ...
INFO: hdimage(sdcard.img): adding partition 'env' (in MBR) from 'env.bin' ...
INFO: hdimage(sdcard.img): adding partition 'opensbi' (in MBR) from 'opensbi.itb' ...
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
Successfully generated at /home/username/work/bianbu-linux/output/k1_v2/images/bianbu-linux-k1_v2-sdcard.img
```

- `bianbu-linux-k1_v2.zip` is suitable for use with Titan Flasher. Alternatively, you can unzip it and flash it using Fastboot.
- `bianbu-linux-k1_v2-sdcard.img` is the sdcard firmware, which can be written to the sdcard using the dd command or [balenaEtcher](https://etcher.balena.io/) after unzipping.

> Titan Flasher User Guide: [Flashing Tool User Guide](https://developer.spacemit.com/documentation?token=B9JCwRM7RiBapHku6NfcPCstnqh)

The default username for the firmware is `root`, and the password is `bianbu`.

### Bianbu PREEMPT_RT Linux First Complete Compilation

For the first compilation, it is recommended to use `make envconfig` for a complete build.

If you have modified `buildroot-ext/configs/spacemit_<solution>_defconfig`, use `make envconfig` to compile.

In other cases, use `make` to compile.

```shell
cd ~/buildroot-sdk
make envconfig
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_defconfig
  2. spacemit_k1_upstream_defconfig
  3. spacemit_k1_minimal_defconfig
  4. spacemit_k1_plt_defconfig
  5. spacemit_k1_rt_defconfig
  6. spacemit_k1_v2_defconfig


your choice (1-6):

```

To compile Bianbu PREEMPT_RT Linux 2.0 version, enter `5`, then press Enter to start compiling, the compile process will apply PREEMPT_RT patch

```shell
buildroot-ext/configs//spacemit_k1_rt_defconfig
Patching linux with PREEMPT_RT patch
Applying rt-linux-support.patch using patch:
...
```

After the compilation is complete, you will see:

```shell
Images successfully packed into /home/username/bianbu-linux/output/k1_rt/images/bianbu-linux-k1_rt.zip
...
Successfully generated at /home/username/work/bianbu-linux/output/k1_rt/images/bianbu-linux-k1_rt-sdcard.img
```

### Configuration

#### buildroot

Config:

```shell
make menuconfig
```

Save the configuration, which is saved by default to `buildroot-ext/configs/spacemit_k1_v2_defconfig`:

```shell
make savedefconfig
```

#### linux

Config:

```shell
make linux-menuconfig
```

Save the configuration, which is saved by default to `bsp-src/linux-6.6/arch/riscv/configs/k1_defconfig`:

```shell
make linux-update-defconfig
```

#### u-boot

Config:

```shell
make uboot-menuconfig
```

Save the configuration, which is saved by default to `bsp-src/uboot-2022.10/configs/k1_defconfig`:

```shell
make uboot-update-defconfig
```

### Recompile Completely

If you have already compiled completely using `make envconfig`, you can directly use `make` to recompile completely.

### Compile Specific Packages

buildroot supports compiling specific packages. You can view the guide with `make help`.

Common commands:

- Delete the build directory of `<pkg>`: `make <pkg>-dirclean`
- Compile `<pkg>`: `make <pkg>`

To compile the kernel:

```shell
make linux-rebuild
```

To compile u-boot:

```shell
make uboot-rebuild
```

After compiling the specified package, you can download it to the device for verification or compile it into the firmware:

```shell
make
```

## Standalone Compilation

The cross-compiler download URL is `http://archive.spacemit.com/toolchain/`, and it can be used after decompression.

Such as `spacemit-toolchain-linux-glibc-x86_64-v1.0.0.tar.xz`:

```shell
sudo tar -Jxf /path/to/spacemit-toolchain-linux-glibc-x86_64-v1.0.0.tar.xz -C /opt
```

Set environment variables:

```shell
export PATH=/opt/spacemit-toolchain-linux-glibc-x86_64-v0.3.3/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

### Compile opensbi

```shell
cd bsp-src/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
```

The compilation will finally generate `platform/generic/firmware/fw_dynamic.itb`.

### Compile u-boot

```shell
cd bsp-src/uboot-2022.10
make k1_defconfig
make -j$(nproc)
```

The compilation will generate `u-boot-env-default.bin` based on `board/spacemit/k1-x/k1-x.env`, which corresponds to the image of the `env` partition in the partition table, and will also generate `FSBL.bin` and `u-boot.itb`.

### Compile linux

```shell
cd bsp-src/linux-6.6
make k1_defconfig
LOCALVERSION="" make -j$(nproc)
```
