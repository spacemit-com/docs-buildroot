---
sidebar_position: 2
---

# 镜像

我们每个发布的版本都会提供预编译好的镜像，供体验。Buildroot 默认的镜像是`zip`格式，适用于 Titan Flasher，亦可解压后，用 fastboot 刷机。

## 下载

镜像地址：[Buildroot Image](https://spacemit.com/community/resources-download/Images%20Collects/K3/Buildroot)。

## 刷机

Titan Flasher刷机参考[刷机工具使用手册](https://www.spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md)。

安全镜像（含 TEE 固件）在普通镜像基础上增加独立的 TEE 分区与安全打包配置，其构建、分区布局与烧录步骤见[安全 / TEE 镜像构建与烧录说明](./security/tee_image.md)，方案整体见[安全](./security/index.md)。
