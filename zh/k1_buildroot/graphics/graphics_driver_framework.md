sidebar_position: 2

# 图形驱动框架

## 整体框架

![linux图形显示框架](./static/linuxGraphicsFramework.png#pic_center)

## 环境配置及依赖

### Kernel

进入 Linux Kernel 源码目录，在 config 中启用 PowerVR：

```bash
CONFIG_POWERVR_ROGUE=y # 启用 PowerVR Rogue
```

也可通过 `make menuconfig` 在图形界面中启用。在 **Device Drivers -> Graphics support -> PowerVR GPU** 路径下，选择 **PowerVR GPU** 选项：

```shell
<*>   Lontium LT9711 DSI to DP
< > null drm disp
<*> PowerVR GPU # 启用 PowerVR GPU
[ ] Enable legacy drivers (DANGEROUS)  ----
    Frame buffer Devices  --->
```

### PVR DDK

由于版权限制，PVR GPU 的 DDK（Device Development Kit）无法直接提供。用户层闭源代码以 so 动态库形式提供 OpenGLES、Vulkan 和 OpenCL API。

在 Buildroot 和 Bianbu 上开启 PVR GPU 驱动的方式如下：

#### Buildroot
在编译 buildroot 时，将配置文件中启用：

```shell
BR2_PACKAGE_IMG_GPU_POWERVR=y
```

或通过 `make menuconfig` 手动勾选：
**External options -> Bianbu config -> img-gpu-powervr**

  ```shell
  [*] rtk hciattach
      *** spacemit mpp package ***
  -*- spacemit mpp
  [*] img-gpu-powervr # 启用 img-gpu-powervr
      Output option (Wayland)  --->
  [ ]   install examples
  ```

#### Bianbu

由于涉及版权问题，闭源 GPU 代码以 `.so` 动态库形式通过 **img-gpu-powervr** deb 包提供。
用户可通过以下命令安装 PVR GPU 驱动程序：

```bash
sudo apt install img-gpu-powervr
```

> **注意**：GPU 驱动分为 内核层驱动 和 用户层驱动，二者版本必须保持一致，否则可能导致 GPU 功能异常。
  
查看版本方法：

  ```bash
    # 查看内核层驱动
    ➜  ~ journalctl -b | grep "Initialized pvr"
    11月 21 14:11:37 spacemit-k1-x-MUSE-Pi-board kernel: [drm] Initialized pvr 24.2.6603887 20170530 for cac00000.imggpu on minor 1 # 版本为24.2
    # 查看用户层驱动
    ➜  ~ dpkg -s img-gpu-powervr
    Package: img-gpu-powervr
    Status: install ok installed
    Priority: optional
    Section: graphics
    Installed-Size: 70314
    Maintainer: bianbu <bo.deng@spacemit.com>
    Architecture: all
    Version: 24.2-6603887bb1 # 版本为24.2
  ```

### Mesa3D

Mesa3D 提供 OpenGL ES API 接口及软件渲染实现。它向上对接应用程序对 OpenGL ES 渲染 API 的调用，向下对接 PowerVR GPU 驱动。

#### Buildroot

在配置文件中启用以下选项，以使能 Mesa3D：

  ```bash
    Symbol: BR2_PACKAGE_MESA3D [=y]
    Symbol: BR2_PACKAGE_MESA3D_DRIVER [=y] 
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER [=y]
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_PVR [=y]
    Symbol: BR2_PACKAGE_MESA3D_GBM [=y]
    Symbol: BR2_PACKAGE_MESA3D_OPENGL_EGL [=y]
  ```

或在 **make menuconfig** 中手动勾选：
**Target packages -> Graphic libraries and applications -> mesa3d**

  ```shell
    -*-   Gallium pvr driver
    *** Gallium VDPAU state tracker needs X.org and gallium drivers r300, r600, radeonsi or nouveau ***
    *** Vulkan drivers ***
    *** Off-screen Rendering ***
    [ ]   OSMesa (Gallium) library
          *** OpenGL API Support ***
    -*-   gbm
    *** OpenGL GLX support needs X11 ***
    -*-   OpenGL EGL
    [ ]   OpenGL ES
  ```

#### Bianbu

调用 PowerVR GPU 硬件依赖于 bianbu 源上的 Mesa deb 包，并按以下顺序进行安装：

  ```bash
        "libgl1-mesa-dev_*_riscv64.deb"
        "libegl1-mesa_*_riscv64.deb"
        "libegl1-mesa-dev_*_riscv64.deb"
        "libglapi-mesa_*_riscv64.deb"
        "libgbm1_*_riscv64.deb"
        "libgbm-dev_*_riscv64.deb"
        "libegl-mesa0_*_riscv64.deb"
        "libgl1-mesa-dri_*_riscv64.deb"
        "libgles2-mesa_*_riscv64.deb"
        "libgles2-mesa-dev_*_riscv64.deb"
        "libgl1-mesa-glx_*_riscv64.deb"
        "libglx-mesa0_*_riscv64.deb"
        "libosmesa6_*_riscv64.deb"
        "libosmesa6-dev_*_riscv64.deb"
        "libwayland-egl1-mesa_*_riscv64.deb"
        "mesa-common-dev_*_riscv64.deb"
  ```

同时需要安装 libglvnd：

  ```bash
    sudo apt install libglvnd
  ```

## 其它

### BlobCache 使用

GPU 的 BlobCache 是一种用于存储和重用 GPU 编译结果的数据缓存机制。这种缓存机制旨在提高性能，减少重复计算和数据传输的时间，从而加快渲染速度。

**配置方法**

1. 打开配置文件：

   ```bash
   vim /etc/powervr.ini
   ```

2. 设置如下, 即可启用 BlobCache 功能

```bash
[default]
EnableBlobCache=1 # 启用 BlobCache 功能

[mpv]
EnableWorkaroundMPVScreenTearing=1

[totem]
EnableWorkaroundMPVScreenTearing=1

[gst-launch-1.0]
EnableWorkaroundMPVScreenTearing=1
```

### 3.2 GPU 节点调试

通过对 GPU 节点的监控，可以实时查看 GPU 的运行状态：

```shell
~ su
~ cd /sys/kernel/debug/pvr/
~ watch cat status (按 Ctrl + C 退出)
```

如下所示：

```shell
Every 2.0s: cat status                                    spacemit-k1-x-deb1-board: Thu Nov 28 09:00:00 2024

Driver Status:   OK

Device ID: 0:128
Firmware Status: OK
Server Errors:   24
HWR Event Count: 0
CRR Event Count: 0
SLR Event Count: 0
WGP Error Count: 0
TRP Error Count: 0
FWF Event Count: 0
APM Event Count: 0
GPU Utilisation: 0%
DM Utilisation:  VM0
           2D:   0%
         GEOM:   0%
           3D:   0%
          CDM:   0%
          RAY:   0%
        GEOM2:   0%
```
