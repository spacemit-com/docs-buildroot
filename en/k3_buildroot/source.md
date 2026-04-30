---
sidebar_position: 3
---

# Source Code

This document describes the development environment, source download process, and build methods for the SDK source tree.

## Development Environment

### Hardware Requirements

Recommended system configuration:

- CPU: 12th Gen Intel(R) Core(TM) i5 or higher
- Memory: 16 GB or more
- Storage: SSD, 256 GB or more

### Operating System

Ubuntu 20.04 LTS or a later LTS release is recommended. Other Linux distributions with Docker support can also be used.

### Install Dependencies

The K3 Buildroot SDK supports container-based builds by default. In most cases, only [Docker CE](https://docs.docker.com/engine/install/) needs to be installed.

For direct host-side builds, install the required dependencies as described below.

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

## Download the Source Code

### Preparation

The Buildroot source code is hosted on both Gitee and GitHub. The complete source tree consists of multiple repositories managed through `repo`.

Before downloading the source tree, complete the following preparation steps:

1. Configure SSH keys.

   - For Gitee access, refer to the [Gitee SSH key guide](https://gitee.com/help/articles/4191).
   - For GitHub access, refer to the [GitHub SSH key guide](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).

2. Install `repo`.

   - If Google services are accessible, refer to the official [`repo` installation guide](https://gerrit.googlesource.com/git-repo/+/refs/heads/main/README.md#install).
   - A mirrored installation method is also available in the [Git Repo mirror usage guide](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/).

   `repo` version 2.41 or later is required. Earlier versions may cause download failures.

```shell
$ repo --version
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

### Release Branches

The `main` branch of the `manifests` repository defines the `manifest.xml` files for different releases. Each XML file specifies the repository paths and corresponding branches.

| Version | `manifest.xml` | Branch |
| ---- | ------------- | ------------ |
| v1.0 | `k3-br-v1.0.y.xml` | `k3-br-v1.0.y` |


### Download the Code

The following examples download the source code for release 1.0.

#### Download from Gitee

```shell
mkdir ~/k3-buildroot-sdk-1.0
cd ~/k3-buildroot-sdk-1.0
repo init -u git@gitee.com:spacemit-buildroot/manifests.git -b main -m k3-br-v1.0.y.xml
repo sync
repo start k3-br-v1.0.y --all
```

#### Download from GitHub

```shell
mkdir ~/k3-buildroot-sdk-1.0
cd ~/k3-buildroot-sdk-1.0
repo init -u git@github.com:spacemit-com/manifests.git -b main -m k3-br-v1.0.y.xml
repo sync
repo start k3-br-v1.0.y --all
```

To download a different release branch, specify a different `manifest.xml` file with the `-m` option.

After the source tree has been downloaded, pre-downloading the third-party packages required by Buildroot is recommended. Distributing those packages within the development team can reduce load on the primary server and help avoid network congestion.

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl/
```

## Directory Structure

```shell
├── bsp-src
│   ├── linux-6.18 # Linux kernel source
│   ├── opensbi # OpenSBI source
│   └── uboot-2022.10 # U-Boot source
├── buildroot # Main Buildroot directory
├── buildroot-ext # Custom extensions, including board, configs, package, and patches subdirectories
├── Makefile # Top-level Makefile
├── package-src # SpacemiT custom source package directory
│   ├── drm-test # DRM unit test program
│   ├── esos # ESOS real-time firmware for the auxiliary core
│   ├── factorytest # Factory test code
│   ├── glmark2
│   ├── gpu-test # GPU unit test program
│   ├── img-gpu-powervr # GPU DDK
│   ├── k3x-vpu-firmware
│   ├── k3x-vpu-test # Video Processing Unit test program
│   ├── k3x-cam # CSI unit test program
│   ├── mesa
│   ├── mpp # Media Processing Platform
│   ├── rtk_hciattach
│   └── v2d-test # 2D unit test program
├── scripts # Build-related scripts
```

## Cross Compilation

### First Full Build for Buildroot 1.0

For the initial build, a full build with `make envconfig` is recommended.

If `buildroot-ext/configs/spacemit_k3_defconfig` has been modified, `make envconfig` should also be used.

In all other cases, `make` is sufficient.

```shell
$ cd ~/k3-buildroot-sdk-1.0
$ make envconfig
Available buildroot-ext/configs:
  1. spacemit_k3_ci_defconfig
  2. spacemit_k3_defconfig
  3. spacemit_k3_fpga_defconfig
  4. spacemit_k3_plt_defconfig

Your choice (1-4):

```

To build Buildroot 1.0, enter `2` and press Enter.

**Notes:**

1. Builds run in a container by default. For direct host-side builds, set the environment variable `export DIRECT_BUILD=1`. When switching between container builds and host builds, clean the `output` directory first.
2. A new command set is provided and is not compatible with the legacy command set. To use the new commands:
   - Delete the `env.mk` file generated by the legacy flow from the project root.
   - Run `make help` to view the available commands.
   - Use commands such as `make k3-build` to build a specific solution directly.

   Example:

   ```shell
   $ cd ~/k3-buildroot-sdk-1.0
   $ rm -rf env.mk
   $ make help
        Buildroot Build System - Help

        Available solutions:
        k3 k3_ci k3_fpga k3_plt

        Development Commands:
        make vars                          # Show project information
        make <solution>-supported          # Check if solution is supported
        make <solution>-config             # Apply solution defconfig
        make <solution>-menuconfig         # Configure buildroot for solution
        make <solution>-linux-menuconfig   # Configure Linux kernel for solution
        make <solution>-uboot-menuconfig   # Configure U-Boot for solution
        make <solution>-busybox-menuconfig # Configure BusyBox for solution
        make <solution>-build              # Build specified solution
        make <solution>-pkg PKG=<package>  # Build specified package for solution
        make <solution>-shell              # Enter build container for solution
        make <solution>-source             # Download all source packages for solution
        make <solution>-clean              # Clean solution build artifacts
        make <solution>-cleanbuild         # Clean and rebuild solution
        make build-docker-image            # Build Docker image
        make update-docker-image           # Update Docker image

        Quick Start: Run 'make <solution>-build' to get started, e.g.:
        make k3-build
    ```

The build process may download third-party packages, so the total build time depends on network conditions. When the Buildroot dependency packages have been downloaded in advance, a system that meets the recommended hardware configuration typically completes the build in about one hour.

Build artifacts are generated in `/output/k3/images/`. An example of a successful packaging log is shown below:

```shell
Images successfully packed into /k3/images/Buildroot-k3-20260425142457.zip


Generating sdcard image...................................
INFO: cmd: "mkdir -p "/k3/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/k3/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/k3/images"" (stderr):
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'env' from 'env.bin' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'bootinfo' from 'factory/bootinfo_block.bin' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'fsbl' from 'factory/FSBL.bin' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'esos' from 'esos.itb' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'opensbi' from 'fw_dynamic.itb' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'uboot' from 'u-boot.itb' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'bootfs' (in MBR) from 'bootfs.img' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition 'rootfs' (in MBR) from 'rootfs.ext4' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition '[MBR]' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition '[GPT header]' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition '[GPT array]' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): adding primary partition '[GPT backup]' ...
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): writing GPT
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): writing protective MBR
INFO: hdimage(Buildroot-k3-20260425142457-sdcard.img): writing MBR
INFO: cmd: "rm -rf "/k3/build/genimage.tmp/"" (stderr):
Successfully generated at /k3/images/Buildroot-k3-20260425142457-sdcard.img

```


`Buildroot-K3-xxx.zip` is intended for use with [Titan Flasher](https://www.spacemit.com/community/document/info?lang=en&nodepath=tools/user_guide/flasher_user_guide.md). It can also be extracted and flashed with `fastboot`.

`Buildroot-K3-xxx-sdcard.img` is an SD card image. After extraction, it can be written to an SD card with `dd` or [balenaEtcher](https://etcher.balena.io/).

The default firmware credentials are:

- Username: `root`
- Password: `bianbu`

### Configuration

#### Buildroot

Open the configuration menu:

```shell
make menuconfig
```

Save the configuration. By default, it is written to `buildroot-ext/configs/spacemit_k3_defconfig`:

```shell
make savedefconfig
```

#### Linux

Open the kernel configuration menu:

```shell
make linux-menuconfig
```

Save the configuration. By default, it is written to `bsp-src/linux-6.18/arch/riscv/configs/k3_bianbu_defconfig`:

```shell
make linux-update-defconfig
```

#### U-Boot

Open the U-Boot configuration menu:

```shell
make uboot-menuconfig
```

Save the configuration. By default, it is written to `bsp-src/uboot-2022.10/configs/k3_defconfig`:

```shell
make uboot-update-defconfig
```

### Run a Full Build Again

If a full build has already been completed with `make envconfig`, subsequent full builds can be started directly with `make`.

### Build a Specific Package

Buildroot supports building individual packages. Refer to `make help` for the complete command reference.

Common commands:

- Remove the build directory for `<pkg>`: `make <pkg>-dirclean`
- Build `<pkg>`: `make <pkg>`
- Rebuild `<pkg>`: `make <pkg>-rebuild`

#### U-Boot

```
# Build
make uboot

# Clean
make uboot-dirclean

# Rebuild
make uboot-rebuild
```

#### Linux

The default kernel configuration is `k3_bianbu_defconfig`.

```
# Build
make linux

# Clean
make linux-dirclean

# Rebuild
make linux-rebuild
```

#### OpenSBI

```
# Build
make opensbi

# Clean
make opensbi-dirclean

# Rebuild
make opensbi-rebuild
```

After building a specific package, it can be deployed to a device for validation or included in a full firmware build:

```
make
```

## Standalone Compilation

For an `OS Vendor` adapting a different operating system, a full SDK build is not required. Standalone compilation or an existing internal build flow can be used instead.

Standalone compilation uses a local cross-compilation flow with the [cross-compilation toolchain](http://archive.spacemit.com/toolchain/) provided by SpacemiT. The `K3 Buildroot SDK` uses `spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz`, which corresponds to the GCC 15 toolchain release.

Download and extract the toolchain:

```shell
wget http://archive.spacemit.com/toolchain/spacemit-toolchain-elf-newlib-x86_64-v1.2.2.tar.xz
sudo tar -Jxf /path/to/spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz -C /opt
```

Set the environment variables:

```shell
export PATH=/opt/spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

### Build OpenSBI

```shell
cd bsp-src/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k3_defconfig PLATFORM=generic
```

The final output is `build/platform/generic/firmware/fw_dynamic.itb`.

### Build U-Boot

```shell
cd bsp-src/uboot-2022.10
make k3_defconfig
make -j$(nproc)
```

The build generates `u-boot-env-default.bin` from `board/spacemit/k3/k3.env`. This file corresponds to the `env` partition image in the partition layout. The build also generates `bootinfo.bin`, `FSBL.bin`, and `u-boot.itb`.

### Build Linux

```shell
cd bsp-src/linux-6.18
make k3_bianbu_defconfig
LOCALVERSION="" make -j$(nproc)
```

### Build ESOS

Refer to the [ESOS Development Guide](https://spacemit.com/community/document/info?lang=en&nodepath=software/SDK/buildroot/k3_buildroot/esos/esos_dev_guide.md).