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

也可通过 `make menuconfig` 在图形界面中启用。在 **Device Drivers -> Graphics support -> Direct Rendering Manager** 路径下，选择 **PowerVR GPU** 选项：

```shell
<*>   PowerVR GPU
<*>     PowerVR GPU DVFS
<*>       PowerVR GPU Thermal
< >     null drm disp
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
    [drm] Initialized pvr 24.2.6603887 for cac00000.imggpu on minor 0 # 版本为24.2
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

### GPU DVFS

K3 平台的 GPU 内核驱动通过 Linux devfreq 框架实现了 DVFS（动态频率电压调节），默认使用 `simple_ondemand` 调频策略，根据 GPU 负载动态调整频率（硬件实现上仅支持动态调频）。

支持以下频率档位，通过 DTS 中的 OPP（Operating Performance Points）表进行配置：

| 档位 | 频率 |
|------|------|
| 1 | 409 MHz |
| 2 | 491 MHz |
| 3 | 600 MHz |
| 4 | 614 MHz |
| 5 | 750 MHz |
| 6 | 819 MHz |
| 7 | 1000 MHz |
| 8 | 1228 MHz |

DTS 配置示例（`k3.dtsi`）：

```dts
imggpu: imggpu@cac00000 {
    /* ... 其他属性 ... */
    clocks = <&syscon_apmu CLK_APMU_GPU>;
    clock-names = "clk_rgx";
    resets = <&syscon_apmu RESET_APMU_GPU>;
    #cooling-cells = <2>;
    thermal-zone = "thermal_gpu";
    operating-points-v2 = <&imggpu_opp_table>;
    status = "okay";

    imggpu_opp_table: opp-table {
        compatible = "operating-points-v2";

        opp-409000000 {
            opp-hz = /bits/ 64 <409000000>;
            opp-microvolt = <800000>; /* OPP 框架要求的固定值，实际仅调频 */
            clock-latency-ns = <100000>;
        };

        opp-614000000 {
            opp-hz = /bits/ 64 <614000000>;
            opp-microvolt = <900000>;
            clock-latency-ns = <100000>;
        };

        opp-819000000 {
            opp-hz = /bits/ 64 <819000000>;
            opp-microvolt = <900000>;
            clock-latency-ns = <100000>;
        };

        opp-1000000000 {
            opp-hz = /bits/ 64 <1000000000>;
            opp-microvolt = <950000>;
            clock-latency-ns = <100000>;
        };

        opp-1228000000 {
            opp-hz = /bits/ 64 <1228000000>;
            opp-microvolt = <950000>;
            clock-latency-ns = <100000>;
        };
    };
};
```

可通过修改 DTS 中的 OPP 表实现频率档位的定制化。

**DVFS 调试**

可通过 devfreq 节点查看和调整 GPU 频率：

```shell
# 查看当前频率
cat /sys/class/devfreq/cac00000.imggpu/cur_freq

# 查看可用频率
cat /sys/class/devfreq/cac00000.imggpu/available_frequencies

# 查看当前调频策略
cat /sys/class/devfreq/cac00000.imggpu/governor

# 查看可用调频策略
cat /sys/class/devfreq/cac00000.imggpu/available_governors
```

### GPU 温控

K3 平台的 GPU 温控通过 Linux thermal 框架与 devfreq cooling 机制实现。当 GPU 温度达到阈值时，系统会自动降低 GPU 频率以控制温度。

温控策略通过 DTS 中的 thermal zone 配置，包含多级温度阈值和对应的降频映射。以 `k3_com260.dts` 为例：

| 级别 | 温度阈值 | 回滞温度 | 类型 | 降频等级 |
|------|----------|----------|------|----------|
| gpu_alert0 | 65°C | 2°C | passive | cooling 0~1 |
| gpu_alert1 | 75°C | 2°C | passive | cooling 1~2 |
| gpu_alert2 | 85°C | 2°C | passive | cooling 2~3 |
| gpu_crit | 105°C | 2°C | critical | 关闭 GPU |

DTS 配置示例（`k3_com260.dts`）：

```dts
thermal_gpu {
    polling-delay = <1000>;
    polling-delay-passive = <500>;
    thermal-sensors = <&thermal 3>;

    trips {
        gpu_alert0: gpu-alert0 {
            temperature = <65000>;
            hysteresis = <2000>;
            type = "passive";
        };

        gpu_alert1: gpu-alert1 {
            temperature = <75000>;
            hysteresis = <2000>;
            type = "passive";
        };

        gpu_alert2: gpu-alert2 {
            temperature = <85000>;
            hysteresis = <2000>;
            type = "passive";
        };

        gpu_crit: gpu-crit {
            temperature = <105000>;
            hysteresis = <2000>;
            type = "critical";
        };
    };

    cooling-maps {
        map0 {
            trip = <&gpu_alert0>;
            cooling-device = <&imggpu 0 1>;
        };

        map1 {
            trip = <&gpu_alert1>;
            cooling-device = <&imggpu 1 2>;
        };

        map2 {
            trip = <&gpu_alert2>;
            cooling-device = <&imggpu 2 3>;
        };
    };
};
```

> **注意**：不同板型的温控阈值可能不同。例如 `k3_deb1.dts` 中的阈值为 80°C / 85°C / 90°C / 105°C（critical），具体以实际板型 DTS 配置为准。

**温控调试**

可通过 thermal 节点查看 GPU 温度和温控状态：

```shell
# 查看 GPU 当前温度
cat /sys/class/thermal/thermal_zone*/type  # 找到 thermal_gpu 对应的 zone
cat /sys/class/thermal/thermal_zone*/temp  # 查看温度（单位：毫摄氏度）
```

### GPU 节点调试

通过对 GPU 节点的监控，可以实时查看 GPU 的运行状态：

```shell
~ su
~ cd /sys/kernel/debug/pvr/
~ watch cat status (按 Ctrl + C 退出)
```

如下所示：

```shell
Every 2.0s: cat /sys/kernel/debug/pvr/status                bianbu-pc: Thu Mar 05 10:00:00 2026

Driver Status:   OK

Device ID: 0:128
Firmware Status: OK
Server Errors:   0
HWR Event Count: 0
CRR Event Count: 0
SLR Event Count: 0
WGP Error Count: 0
TRP Error Count: 0
FWF Event Count: 0
APM Event Count: 2026
GPU Utilisation: 0%
DM Utilisation:  VM0
           2D:   0%
         GEOM:   0%
           3D:   0%
          CDM:   0%
          RAY:   0%
        GEOM2:   0%
```
