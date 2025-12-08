---
sidebar_position: 1.5
---

# 源码

本文档介绍SDK源码的开发环境、下载和编译方式。

## 开发环境

### 硬件配置

推荐配置：

- CPU：12th Gen Intel(R) Core(TM) i5或以上
- Memory：16GB或以上
- Disk：SSD，256GB或以上

### 操作系统

- Buildroot 2.2.7或之后的版本

  推荐Ubuntu 20.04或更新LTS版本，或支持Docker的Linux发行版。

- Buildroot 2.2.6或之前的版本

  推荐Ubuntu 20.04或更新LTS版本，其他Linux发行版本没有测试。

### 安装依赖

Buildroot 2.2.7或之后的版本默认支持在容器里编译，因此只需要[安装Docker CE](https://docs.docker.com/engine/install/)。

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

1. 如果从 Gitee 下载，先参考[这篇文档](https://gitee.com/help/articles/4191)设置 SSH Keys；如果从 Github 下载，先参考[这篇文档](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)设置 SSH Keys。

2. 安装 repo

   如果您能访问 Google，请参考[这篇文档](https://gerrit.googlesource.com/git-repo/+/refs/heads/main/README.md#install)安装；否则参考[Git Repo 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/)安装。

   要求 repo 版本 >= 2.41，否则下载过程会异常。

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

### 版本分支

manifests 仓库的 main 分支定义了不同版本的 manifest.xml，xml 文件指定了各仓库的路径和分支。

| 版本 | manifest.xml | 分支 |
| ---- | ------------- | ------------ |
| v1.0 | bl-v1.0.y.xml | bl-v1.0.y    |
| v2.0 | bl-v2.0.y.xml | bl-v2.0.y    |
| v2.1 | k1-bl-v2.1.y.xml | k1-bl-v2.1.y |
| v2.2 | k1-bl-v2.2.y.xml | k1-bl-v2.2.y |

**注意事项：**

- 由于 Gitee 单仓库容量限制，linux-6.6 仓库的 bl-v2.0.y 和 k1-bl-v2.1.y 分支分别被移动 [linux-6.6-v2.0.y](https://gitee.com/spacemit-buildroot/linux-6.6-v2.0.y) 和 [linux-6.6-v2.1.y](https://gitee.com/spacemit-buildroot/linux-6.6-v2.1.y) 仓库，如果您在使用 v2.0 或 v2.1，将无法 repo sync 或 git pull linux-6.6 仓库，只能重新下载，给您造成的不便，敬请原谅。
- Github 只托管了 v2.2 及以后的版本。

### 下载代码

例如下载 2.2 版本的代码：

#### 从 Gitee 下载

```shell
mkdir ~/buildroot-sdk-2.2
cd ~/buildroot-sdk-2.2
repo init -u git@gitee.com:spacemit-buildroot/manifests.git -b main -m k1-bl-v2.2.y.xml
repo sync
repo start k1-bl-v2.2.y --all
```

#### 从 Github 下载

```shell
mkdir ~/buildroot-sdk-2.2
cd ~/buildroot-sdk-2.2
repo init -u git@github.com:spacemit-com/manifests.git -b main -m k1-bl-v2.2.y.xml
repo sync
repo start k1-bl-v2.2.y --all
```

如需下载其他分支，通过 `-m` 指定不同的 manifest.xml 即可。

下载完源码，推荐提前下载buildroot依赖的第三方软件包，并在团队内部分发，避免主服务器网络拥塞。

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl
```

## 目录结构

```shell
├── bsp-src               # 存放linux kernel、uboot、opensbi源码
│   ├── linux-6.6
│   ├── opensbi
│   └── uboot-2022.10
├── buildroot             # buildroot主目录
├── buildroot-ext         # 客制化扩展，包含board、configs、package、patches子目录
├── Makefile              # 顶层Makefile
├── package-src           # 本地展开的应用或库源码目录
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
└── scripts               # 编译使用到的脚本
```

## 交叉编译

### Buildroot 2.x 首次完整编译

首次编译，建议使用`make envconfig`完整编译。

修改了`buildroot-ext/configs/spacemit_<solution>_defconfig`，要使用`make envconfig`编译。

其他情况，使用`make`编译即可。

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

编译Buildroot 2.x版本，输入`5`，然后回车即开始编译。

注意：自Buildroot 2.2.7 开始

1. 默认在容器中构建。如需在宿主机上构建，请配置环境变量 `export DIRECT_BUILD=1`，切换容器中构建和宿主机构建时需要清理output目录。
2. 新增一组命令。与旧命令不兼容，如需使用新命令，需删除项目根目录下旧命令生成的env.mk文件，之后可用`make help`查看新命令列表，如可使用`make k1_v2-build`直接编译指定方案。

编译过程可能需要下载一些第三方软件包，具体耗时和网络环境相关。如果提前下载buildroot依赖的第三方软件包，推荐硬件配置编译耗时约为1小时。

```shell
Images successfully packed into /home/username/buildroot-sdk/output/k1_v2/images/buildroot-k1_v2.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/home/username/buildroot-sdk/output/k1_v2/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/home/username/buildroot-sdk/output/k1_v2/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/home/username/buildroot-sdk/output/k1_v2/images"" (stderr):
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
Successfully generated at /home/username/work/buildroot-sdk/output/k1_v2/images/buildroot-k1_v2-sdcard.img
```

其中`buildroot-k1_v2.zip`适用于Titan Flasher，或者解压后用fastboot刷机；`buildroot-k1_v2-sdcard.img`为sdcard固件，解压后可以用dd命令或者[balenaEtcher](https://etcher.balena.io/)写入sdcard。

> Titan Flasher使用指南：[刷机工具使用指南](https://developer.spacemit.com/#/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d)

固件默认用户名：`root`，密码：`bianbu`。

### Bianbu PREEMPT_RT Linux 首次完整编译

首次编译，建议使用`make envconfig`完整编译。

修改了`buildroot-ext/configs/spacemit_<solution>_defconfig`，要使用`make envconfig`编译。

其他情况，使用`make`编译即可。

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

Buildroot 2.0支持实时Linux(PREEMPT_RT)内核编译，输入`5`,然后回车即开始编译，首次编译过程中会自动打上PREEMPT_RT补丁

```shell
buildroot-ext/configs//spacemit_k1_rt_defconfig
Patching linux with PREEMPT_RT patch
Applying rt-linux-support.patch using patch:
...
```

编译完成，可以看到：

```shell
Images successfully packed into /home/username/buildroot-sdk/output/k1_rt/images/buildroot-k1_rt.zip
...
Successfully generated at /home/username/work/buildroot-sdk/output/k1_rt/images/buildroot-k1_rt-sdcard.img
```

### 配置

#### buildroot

配置：

```shell
make menuconfig
```

保存配置，默认保存到`buildroot-ext/configs/spacemit_k1_v2_defconfig`：

```shell
make savedefconfig
```

#### linux

配置：

```shell
make linux-menuconfig
```

保存配置，默认保存到`bsp-src/linux-6.6/arch/riscv/configs/k1_defconfig`：

```shell
make linux-update-defconfig
```

#### u-boot

配置：

```shell
make uboot-menuconfig
```

保存配置，默认保存到`bsp-src/uboot-2022.10/configs/k1_defconfig`：

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

编译内核：

```shell
make linux-rebuild
```

编译u-boot：

```shell
make uboot-rebuild
```

编译指定包之后，可以单独下载到设备验证，或者编进固件：

```shell
make
```

## 单独编译

交叉编译器下载地址：`http://archive.spacemit.com/toolchain/`，解压即可使用。

例如`spacemit-toolchain-linux-glibc-x86_64-v1.0.0.tar.xz`：

```shell
sudo tar -Jxf /path/to/spacemit-toolchain-linux-glibc-x86_64-v1.0.0.tar.xz -C /opt
```

设置环境变量：

```shell
export PATH=/opt/spacemit-toolchain-linux-glibc-x86_64-v0.3.3/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

### 编译 opensbi

```shell
cd bsp-src/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
```

编译最终生成 `platform/generic/firmware/fw_dynamic.itb`。

### 编译 u-boot

```shell
cd bsp-src/uboot-2022.10
make k1_defconfig
make -j$(nproc)
```

编译会根据 `board/spacemit/k1-x/k1-x.env` 生成 `u-boot-env-default.bin`，对应分区表 `env` 分区的镜像，以及生成 `FSBL.bin` 和 `u-boot.itb`。

### 编译linux

```shell
cd bsp-src/linux-6.6
make k1_defconfig
LOCALVERSION="" make -j$(nproc)
```
