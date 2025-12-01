# Graphics Driver Framework

## Overall Framework

![Linux Graphical Framework](./static/linuxGraphicsFramework_en.png#pic_center)

## Environment Setup and Dependencies

### Kernel

Enter the Linux Kernel source directory and enable **PowerVR GPU driver** in the config:

```bash
CONFIG_POWERVR_ROGUE=y # Enable PowerVR Rogue
```

You can also enable it through `make menuconfig` in the graphical interface.  
Navigate to **Device Drivers -> Graphics support -> PowerVR GPU**, and select the **PowerVR GPU** option:

```shell
<*>   Lontium LT9711 DSI to DP
< > null drm disp
<*> PowerVR GPU # Enable PowerVR GPU
[ ] Enable legacy drivers (DANGEROUS)  ----
    Frame buffer Devices  --->
```

### PVR DDK

Due to copyright restrictions, the DDK (Device Development Kit) for the PVR GPU cannot be provided directly.  
User-space closed-source code is provided as shared libraries (`.so`) offering OpenGLES, Vulkan, and OpenCL APIs.

The way to enable the PVR GPU driver on buildroot and bianbu is as follows:

#### Buildroot
Enable it in the configuration file when compiling Buildroot:

```shell
BR2_PACKAGE_IMG_GPU_POWERVR=y
```

Or manually select it via `make menuconfig`:
**External options -> Bianbu config -> img-gpu-powervr**

  ```shell
  [*] rtk hciattach
      *** spacemit mpp package ***
  -*- spacemit mpp
  [*] img-gpu-powervr # Enable `img-gpu-powervr`
      Output option (Wayland)  --->
  [ ]   install examples
  ```

#### Bianbu

Due to licensing issues, the closed-source GPU code is provided as `.so` shared libraries via the **img-gpu-powervr** `.deb` package.  
Users can install the PVR GPU driver with:

```bash
sudo apt install img-gpu-powervr
```

> **Note:** The GPU driver consists of a kernel-level driver and a user-level driver; both must have matching versions, otherwise GPU functionality may be abnormal.
  
Method to check the version:

  ```bash
    # Check the kernel-level driver
    ➜  ~ journalctl -b | grep "Initialized pvr"
    11月 21 14:11:37 spacemit-k1-x-MUSE-Pi-board kernel: [drm] Initialized pvr 24.2.6603887 20170530 for cac00000.imggpu on minor 1 # Version 24.2
    # Check the user-level driver
    ➜  ~ dpkg -s img-gpu-powervr
    Package: img-gpu-powervr
    Status: install ok installed
    Priority: optional
    Section: graphics
    Installed-Size: 70314
    Maintainer: bianbu <bo.deng@spacemit.com>
    Architecture: all
    Version: 24.2-6603887bb1 # Version 24.2
  ```

### Mesa3D

Mesa3D provides the OpenGL ES API interface and software rendering implementation.  

It bridges applications that use OpenGL ES APIs with the underlying PowerVR GPU driver.

#### Buildroot

Enable the following options in the configuration file to activate Mesa3D:

  ```bash
    Symbol: BR2_PACKAGE_MESA3D [=y]
    Symbol: BR2_PACKAGE_MESA3D_DRIVER [=y] 
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER [=y]
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_PVR [=y]
    Symbol: BR2_PACKAGE_MESA3D_GBM [=y]
    Symbol: BR2_PACKAGE_MESA3D_OPENGL_EGL [=y]
  ```

Or manually select from **make menuconfig**:
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

Calling the PowerVR GPU hardware depends on the Mesa `.deb` packages from the bianbu source, which should be installed in the following order:

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

Also need to install **libglvnd**:

  ```bash
    sudo apt install libglvnd
  ```

## Others

### BlobCache Usage

The **GPU BlobCache** is a data caching mechanism used to store and reuse GPU compilation results.  
This caching mechanism aims to improve performance by reducing redundant computations and data transfers, thereby speeding up rendering.

**Configuration Method**

1. Open the configuration file:

   ```bash
   vim /etc/powervr.ini
   ```

2. Set as follows to enable the BlobCache feature:

```bash
[default]
EnableBlobCache=1 # Enable the BlobCache feature

[mpv]
EnableWorkaroundMPVScreenTearing=1

[totem]
EnableWorkaroundMPVScreenTearing=1

[gst-launch-1.0]
EnableWorkaroundMPVScreenTearing=1
```

### 3.2 GPU Node Debugging

By monitoring the GPU node, you can view the GPU’s running status in real time:

```shell
~ su
~ cd /sys/kernel/debug/pvr/
~ watch cat status (Press Ctrl+C to exit)
```

Example output as shown below：

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
