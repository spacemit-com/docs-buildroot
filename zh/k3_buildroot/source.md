---
sidebar_position: 3
---

# 源码

本文档介绍 SDK 源码的开发环境、下载和编译方式。

## 开发环境

### 硬件配置

推荐配置：

- CPU：12th Gen Intel(R) Core(TM) i5或以上
- Memory：16GB或以上
- Disk：SSD，256GB或以上

### 操作系统

  推荐Ubuntu 20.04或更新LTS版本，或支持Docker的Linux发行版。

### 安装依赖

K3 Buildroot SDK默认支持在容器里编译，因此只需要[安装 Docker CE](https://docs.docker.com/engine/install/)。

如果直接在主机上编译，按以下指南安装依赖

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

## 下载

### 准备工作

Buildroot代码托管在 Gitee 和 Github 上，包含若干个仓库，使用 repo 管理，下载前需：

1. 配置 SSH Keys

   - 如果从 Gitee 下载，先参考 [这篇文档](https://gitee.com/help/articles/4191) 设置 SSH Keys
   - 如果从 Github 下载，先参考 [这篇文档](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) 设置 SSH Keys

2. 安装 repo

   - 如果可访问 Google，请参考 [这篇文档](https://gerrit.googlesource.com/git-repo/+/refs/heads/main/README.md#install) 安装
   - 也可参考 [Git Repo 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/) 安装。

   要求 repo 版本 >= 2.41，否则下载过程会异常。

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

### 版本分支

manifests 仓库的 main 分支定义了不同版本的 `manifest.xml`，xml 文件指定了各仓库的路径和分支。

| 版本 | `manifest.xml` | 分支 |
| ---- | ------------- | ------------ |
| v1.0 | `k3-br-v1.0.y.xml` | `k3-br-v1.0.y` |


### 下载代码

例如下载 1.0 版本的代码：

#### 从 Gitee 下载

```shell
mkdir ~/k3-buildroot-sdk-1.0
cd ~/k3-buildroot-sdk-1.0
repo init -u git@gitee.com:spacemit-buildroot/manifests.git -b main -m k3-br-v1.0.y.xml
repo sync
repo start k3-br-v1.0.y --all
```

#### 从 Github 下载

```shell
mkdir ~/k3-buildroot-sdk-1.0
cd ~/k3-buildroot-sdk-1.0
repo init -u git@github.com:spacemit-com/manifests.git -b main -m k3-br-v1.0.y.xml
repo sync
repo start k3-br-v1.0.y --all
```

如需下载其他分支，通过 `-m` 指定不同的 `manifest.xml` 即可。

下载完源码，推荐提前下载 buildroot 依赖的第三方软件包，并在团队内部分发，避免主服务器网络拥塞。

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl/
```

## 目录结构

```shell
├── bsp-src
│   ├── linux-6.18 # linux kernel源码
│   ├── opensbi # opensbi源码
│   └── uboot-2022.10 # uboot源码
├── buildroot # buildroot主目录
├── buildroot-ext # 客制化扩展，包含board、configs、package、patches子目录
├── Makefile # 顶层Makefile
├── package-src # Spacemit定制源码包目录
│   ├── drm-test # drm unit test program
│   ├── esos # esos小CPU实时代码
│   ├── factorytest # 工厂测试代码
│   ├── glmark2
│   ├── gpu-test # gpu unit test program
│   ├── img-gpu-powervr # GPU DDK
│   ├── k3x-vpu-firmware
│   ├── k3x-vpu-test # Video Process Unit test program
│   ├── k3x-cam # CSI Unit test progrom
│   ├── mesa
│   ├── mpp # Media Process Platform
│   ├── rtk_hciattach
│   └── v2d-test # 2D Unit test program
├── scripts # 编译使用到的脚本
```

## 交叉编译

### Buildroot 1.0 首次完整编译

首次编译，建议使用`make envconfig`完整编译。

修改了`buildroot-ext/configs/spacemit_k3_defconfig`，要使用`make envconfig`编译。

其他情况，使用`make`编译即可。

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

编译 Buildroot 1.0 版本，输入 `2`，然后回车即开始编译。

**注意：**

1. 默认在容器中构建。如需在宿主机上构建，请配置环境变量 `export DIRECT_BUILD=1`，切换容器中构建和宿主机构建时需要清理output目录。
2. 新增一组命令。与旧命令不兼容，如需使用新命令，需删除项目根目录下旧命令生成的env.mk文件，之后可用`make help`查看新命令列表，例如可以`make k3-build`直接编译指定方案：

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

编译过程可能需要下载一些第三方软件包，具体耗时和网络环境相关。如果提前下载buildroot依赖的第三方软件包，推荐硬件配置编译耗时约为1小时。

编译产物位于`/output/k3/images/`目录下，编译打包成功log如下：

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


其中 `Buildroot-K3-xxx.zip`适用于[Titan Flasher](https://www.spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md)，或者解压后用fastboot刷机；`Buildroot-K3-xxx-sdcard.img`为sdcard固件，解压后可以用dd命令或者[balenaEtcher](https://etcher.balena.io/)写入sdcard。

固件默认用户名：`root`，密码：`bianbu`。

### 配置

#### buildroot

配置：

```shell
make menuconfig
```

保存配置，默认保存到`buildroot-ext/configs/spacemit_k3_defconfig`：

```shell
make savedefconfig
```

#### Linux

配置：

```shell
make linux-menuconfig
```

保存配置，默认保存到`bsp-src/linux-6.18/arch/riscv/configs/k3_bianbu_defconfig`：

```shell
make linux-update-defconfig
```

#### u-boot

配置：

```shell
make uboot-menuconfig
```

保存配置，默认保存到`bsp-src/uboot-2022.10/configs/k3_defconfig`：

```shell
make uboot-update-defconfig
```

### 再次完整编译

如果已经使用`make envconfig`完整编译，可以直接使用`make`再次完整编译。

### 编译指定包

buildroot支持编译指定package，可以`make help`查看指南。

常用命令：

- 删除`<pkg>`的编译目录：`make <pkg>-dirclean`
- 编译`<pkg>`：`make <pkg>`
- 重新编译`<pkg>`: `make <pkg>-rebuild`

#### u-boot

```
#编译
make uboot

#清理
make uboot-dirclean

#重新编译
make uboot-rebuild
```

#### Linux

内核默认配置为`k3_bianbu_defconfig`

```
#编译
make linux

#清理
make linux-dirclean

#重新编译
make linux-rebuild
```

#### opensbi

```
#编译
make opensbi

#清理
make opensbi-dirclean

#重新编译
make opensbi-rebuild
```

编译指定包之后，可以单独下载到设备验证，或者编进固件：

```
make
```

## 单独编译

如果您是 `OS Vendor`，要适配 OS，无需完整编译 SDK，可按照单独编译或按自己的方式编译即可。

单独编译采用本地交叉编译的方式，使用进迭时空提供的[交叉编译工具链](http://archive.spacemit.com/toolchain/)，`K3 Buildroot SDK`使用`spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz`，对应`gcc15`版本。

下载编译工具链并解压：

```shell
wget http://archive.spacemit.com/toolchain/spacemit-toolchain-elf-newlib-x86_64-v1.2.2.tar.xz
sudo tar -Jxf /path/to/spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz -C /opt
```

设置环境变量：

```shell
export PATH=/opt/spacemit-toolchain-linux-glibc-x86_64-v1.2.2.tar.xz/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

### 编译 opensbi

```shell
cd bsp-src/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k3_defconfig PLATFORM=generic
```

编译最终生成 `build/platform/generic/firmware/fw_dynamic.itb`。

### 编译 u-boot

```shell
cd bsp-src/uboot-2022.10
make k3_defconfig
make -j$(nproc)
```

编译会根据 `board/spacemit/k3/k3.env` 生成 `u-boot-env-default.bin`，对应分区表中 `env` 分区的镜像，以及生成 `bootinfo.bin`，`FSBL.bin` 和 `u-boot.itb`。

### 编译linux

```shell
cd bsp-src/linux-6.18
make k3_bianbu_defconfig
LOCALVERSION="" make -j$(nproc)
```
### 编译esos

参考[ESOS 开发指南](https://spacemit.com/community/document/info?lang=zh&nodepath=software/SDK/buildroot/k3_buildroot/esos/esos_dev_guide.md)