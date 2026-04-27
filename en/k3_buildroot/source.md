---
sidebar_position: 3
---

# Source Code

This document describes the SDK source code development environment, as well as the procedures for downloading and building the source tree.

## Development Environment

### Hardware Requirements

Recommended configuration:

- CPU: 12th Gen Intel(R) Core(TM) i5 or higher
- Memory: 16 GB or more
- Disk: SSD, 256 GB or more

### Operating System

Ubuntu 20.04 or a later LTS release is recommended. Other Linux distributions with Docker support are also supported.

### Installing Dependencies

The K3 Buildroot SDK supports container-based builds by default, so only [Docker CE](https://docs.docker.com/engine/install/) needs to be installed.

To build directly on the host system, install the required dependencies listed below.

Ubuntu 16.04 and 18.04:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

Ubuntu 20.04 or later:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 python-is-python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

## Download

### Preparation

The Buildroot source code is hosted on both Gitee and GitHub. It consists of multiple repositories managed through `repo`. Before downloading the source, complete the following setup steps:

1. Configure SSH keys

   - For downloads from Gitee, refer to [this document](https://gitee.com/help/articles/4191) to configure SSH keys.
   - For downloads from GitHub, refer to [this document](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) to configure SSH keys.

2. Install `repo`

   - If Google services are accessible, refer to [this document](https://gerrit.googlesource.com/git-repo/+/refs/heads/main/README.md#install).
   - Alternatively, follow the instructions in the [Git Repo mirror usage guide](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/).

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

The `main` branch of the `manifests` repository defines the `manifest.xml` files for different releases. Each XML file specifies the repository paths and branches to use.

| Version | `manifest.xml` | Branch |
| ---- | ------------- | ------------ |
| v1.0 | `k3-br-v1.0.y.xml` | `k3-br-v1.0.y` |


### Downloading the Source Tree

For example, to download the source tree for version 1.0:

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

To download a different branch, specify the corresponding `manifest.xml` file with the `-m` option.

After downloading the source tree, it is recommended to pre-download the third-party packages required by Buildroot and distribute them internally to avoid congestion on the main server.

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl/
```

## Directory Structure

```shell
├── bsp-src
│   ├── linux-6.18 # Linux kernel source code
│   ├── opensbi # OpenSBI source code
│   └── uboot-2022.10 # U-Boot source code
├── buildroot # Main Buildroot directory
├── buildroot-ext # Custom extensions, including the board, configs, package, and patches subdirectories
├── Makefile # Top-level Makefile
├── package-src # Directory for SpacemiT customized source packages
│   ├── drm-test # DRM unit test program
│   ├── esos # ESOS real-time code for the auxiliary CPU
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
├── scripts # Build scripts
```

## Cross-Compilation

### Full Build for Buildroot 1.0

A full build using `make envconfig` is recommended for the initial build.

If `buildroot-ext/configs/spacemit_k3_defconfig` has been modified, `make envconfig` must be used.

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

To build Buildroot 1.0, enter `2` and press Enter to start the build.

**Notes:**

1. Container-based build is enabled by default. To build directly on the host system, set the environment variable `export DIRECT_BUILD=1`. When switching between container builds and host builds, clean the `output` directory first.
2. A new command set has been introduced. It is not compatible with the legacy commands. To use the new commands, delete the `env.mk` file generated by the legacy command set in the project root directory. Then run `make help` to view the new command list. For example, a specific solution can be built directly with `make k3-build`:

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

The build process may need to download third-party packages, so total build time depends on your network environment. If the Buildroot third-party packages have been downloaded in advance, the recommended hardware configuration typically completes the build in about one hour.

Build artifacts are generated in `/output/k3/images/`. A successful packaging log looks like this:

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


`Buildroot-K3-xxx.zip` can be used with [Titan Flasher](https://www.spacemit.com/community/document/info?lang=en&nodepath=tools/user_guide/flasher_user_guide.md), or it can be extracted and flashed with `fastboot`. `Buildroot-K3-xxx-sdcard.img` is an SD card firmware image. After extraction, it can be written to an SD card using the `dd` command or [balenaEtcher](https://etcher.balena.io/).

The default firmware login credentials are: 
- Username: `root`
- Password `bianbu`.

### Configuration

#### Buildroot

Buildroot configuration:

```shell
make menuconfig
```

To save the configuration, which is written by default to `buildroot-ext/configs/spacemit_k3_defconfig`:

```shell
make savedefconfig
```

#### Linux

Linux kernel configuration:

```shell
make linux-menuconfig
```

To save the configuration, which is written by default to `bsp-src/linux-6.18/arch/riscv/configs/k3_bianbu_defconfig`:

```shell
make linux-update-defconfig
```

#### U-Boot

U-Boot configuration:

```shell
make uboot-menuconfig
```

To save the configuration, which is written by default to `bsp-src/uboot-2022.10/configs/k3_defconfig`:

```shell
make uboot-update-defconfig
```

### Full Rebuild

If a full build has already been completed with `make envconfig`, `make` can be run directly for a subsequent full build.

### Building a Specific Package

Buildroot supports building individual packages. Run `make help` for detailed guidance.

Common commands:

- Remove the build directory of `<pkg>`: `make <pkg>-dirclean`
- Build `<pkg>`: `make <pkg>`
- Rebuild `<pkg>`: `make <pkg>-rebuild`

#### U-Boot

```shell
# build
make uboot

# clean
make uboot-dirclean

# rebuild
make uboot-rebuild
```

#### Linux

The default kernel configuration is `k3_bianbu_defconfig`.

```shell
# build
make linux

# clean
make linux-dirclean

# rebuild
make linux-rebuild
```

#### OpenSBI

```shell
# build
make opensbi

# clean
make opensbi-dirclean

# rebuild
make opensbi-rebuild
```

After building a specific package, it can be deployed to the device for verification, or included in the firmware image by running:

```shell
make
```

## Standalone Build

For an `OS Vendor` adapting a custom OS, building the full SDK is unnecessary. Individual components can be built separately, or an independent build flow can be used.

Standalone build uses local cross-compilation with the [cross toolchain](http://archive.spacemit.com/toolchain/) provided by SpacemiT. `K3 Buildroot SDK` uses `spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz`, which corresponds to the `gcc15` release.

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

### Building OpenSBI

```shell
cd bsp-src/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k3_defconfig PLATFORM=generic
```

The final build artifact is `build/platform/generic/firmware/fw_dynamic.itb`.

### Building U-Boot

```shell
cd bsp-src/uboot-2022.10
make k3_defconfig
make -j$(nproc)
```

The build generates `u-boot-env-default.bin` based on `board/spacemit/k3/k3.env`, which corresponds to the `env` partition image in the partition table. It also generates `bootinfo.bin`, `FSBL.bin`, and `u-boot.itb`.

### Building Linux

```shell
cd bsp-src/linux-6.18
make k3_bianbu_defconfig
LOCALVERSION="" make -j$(nproc)
```

### Building ESOS

Refer to the [ESOS Development Guide](https://spacemit.com/community/document/info?lang=zh&nodepath=software/SDK/buildroot/k3_buildroot/esos/esos_dev_guide.md).