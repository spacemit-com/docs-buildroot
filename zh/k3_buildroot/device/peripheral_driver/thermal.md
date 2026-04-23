# Thermal

介绍 K3 SoC Thermal 的功能和使用方法。

## 模块介绍

Thermal 特指一套关于温控机制的驱动框架。Linux Thermal 框架是 Linux 系统下用于温度控制的一套架构，主要用于解决随着设备性能不断增强而引起的日益严重的发热问题。

### 功能介绍

![](static/thermal.png)

1. **thermal_cooling_device**：对应实施冷却措施的驱动，是温控的执行者。
2. **thermal core**：Thermal 的核心程序，负责驱动初始化，维护 thermal_zone、governor 和 cooling device 三者的关系，并通过 sysfs 和用户空间交互。
3. **thermal governor**：温度控制算法，解决温控发生时 cooling device 应该选择哪个 cooling state。
4. **thermal zone device**：主要用来创建 thermal zone 节点和连接 thermal sensor。节点位于 `/sys/class/thermal` 目录下，由 DTS 文件配置生成。
5. **thermal sensor**：温度传感器，主要是给 thermal 提供温度数据。

### 源码结构介绍

Thermal 驱动目录如下：

```
drivers/thermal/
├── cpufreq_cooling.c
├── cpuidle_cooling.c
├── devfreq_cooling.c
├── gov_bang_bang.c
├── gov_fair_share.c
├── gov_power_allocator.c
├── gov_step_wise.c
├── gov_user_space.c
├── k3-thermal.c    --> K3 平台驱动
├── k3-thermal.h
├── thermal_core.c
├── thermal_core.h
├── thermal_helpers.c
├── thermal_hwmon.c
├── thermal_hwmon.h
├── thermal_of.c
├── thermal_sysfs.c
```

### K3 Thermal Sensor 概述

K3 SoC 内置 8 个温度传感器（sensor 0 ~ sensor 7），分布在芯片不同区域，用于监测各功能模块的温度。驱动通过 `sensor_range` 和 `tsensor_map` 属性确定哪些 sensor 被启用。

DTS 中 tsensor 节点配置了 `sensor_range = <0x0 0x7>` 和 `tsensor_map = <1 0 1 1 1 1 1 1>`，表示 sensor 0、2~7 被启用，sensor 1 未启用。

各 sensor 对应的监测区域（以 k3_evb 为例）：

| Sensor ID | Thermal Zone | 监测区域 |
| :--- | :--- | :--- |
| 0 | thermal_top | 芯片顶层 |
| 2 | thermal_vpu | VPU 区域 |
| 3 | thermal_gpu | GPU 区域 |
| 4 | thermal_cluster0 | CPU Cluster 0（cpu_0~cpu_3） |
| 5 | thermal_cluster1 | CPU Cluster 1（cpu_4~cpu_7） |
| 6 | thermal_cluster2 | CPU Cluster 2（cpu_8~cpu_11） |
| 7 | thermal_cluster3 | CPU Cluster 3（cpu_12~cpu_15） |

## 关键特性

### 特性

- 支持多区域温度监测
- 支持 CPU 温度控制
- 支持 115°C 过温关机

### 测试方法

利用外部测温设备或运行高负载应用，制造温度变化的环境，检查 thermal 和 cpufreq 节点，确认温控是否符合预期。

1. 查看所有 thermal zone 温度和类型：
   ```
   cat /sys/class/thermal/thermal_zone1/temp
   ```

2. 查看 CPU 调频节点（确认温控降频是否生效）：
   ```
   cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
   cat /sys/devices/system/cpu/cpufreq/policy8/scaling_cur_freq
   ```

3. 查看 GPU cooling 状态（需使能 `CONFIG_POWERVR_THERMAL`）：
   ```
   cat /sys/class/thermal/cooling_device*/type   # 找到 type 为 devfreq 的设备
   cat /sys/class/thermal/cooling_device*/cur_state
   ```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

THERMAL 配置如下：

```
CONFIG_K3_THERMAL:
Enable this option if you want to have support for thermal management
controller present in Spacemit SoCs

  Symbol: K3_THERMAL [=y]
  Type  : tristate
  Defined at drivers/thermal/Kconfig:420
  Prompt: Spacemit K3 Thermal Support
  Depends on: OF [=y] && SOC_SPACEMIT [=y]
  Location:
   -> Device Drivers
    -> Thermal drivers (THERMAL [=y])
     -> Spacemit K3 Thermal Support (K3_THERMAL [=y])
```

### DTS 配置

tsensor 硬件节点位于 `k3.dtsi`：

```dts
thermal: thermal@d4018000 {
    compatible = "spacemit,k3-tsensor";
    reg = <0x0 0xd4018000 0x0 0x100>;
    interrupt-parent = <&saplic>;
    interrupts = <61 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "tsensor";
    clocks = <&syscon_apbc CLK_APBC_TSEN>,
             <&syscon_apbc CLK_APBC_TSEN_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_apbc RESET_APBC_TSEN>;
    sensor_range = <0x0 0x7>;
    tsensor_map = <1 0 1 1 1 1 1 1>;
    temperature_offset = <274>;
    #thermal-sensor-cells = <1>;
    status = "okay";
};
```

thermal zone 配置在各板级 DTS 中（以 k3_evb.dts 为例，此处仅展示 cluster1 部分）：

```dts
&thermal_zones {
    thermal_cluster1 {
        polling-delay = <1000>;
        polling-delay-passive = <250>;
        thermal-sensors = <&thermal 5>;

        trips {
            thermal_cluster1_trip0: thermal_cluster1-trip0 {
                temperature = <85000>;
                hysteresis = <2000>;
                type = "active";
            };

            thermal_cluster1_trip1: thermal_cluster1-trip1 {
                temperature = <95000>;
                hysteresis = <2000>;
                type = "passive";
            };

            thermal_cluster1_trip2: thermal_cluster1-trip2 {
                temperature = <105000>;
                hysteresis = <2000>;
                type = "passive";
            };

            thermal_cluster1_trip3: thermal_cluster1-trip3 {
                temperature = <115000>;
                hysteresis = <2000>;
                type = "critical";
            };
        };

        cooling-maps {
            map0 {
                trip = <&thermal_cluster1_trip0>;
                cooling-device = <&cpu_0 2 2>,
                        <&cpu_1 2 2>,
                        ...
                        <&cpu_7 2 2>;
            };
            ...
        };
    };
};
```

## 接口介绍

### API 介绍

请参考内核目录下面的文档：
```
Documentation/driver-api/thermal/
```

## Debug 介绍

### sysfs

所有 thermal zone 节点位于 `/sys/class/thermal/` 目录下。

| 节点 | 作用说明 |
| :--- | :--- |
| thermal\_zoneX/temp | 当前温度（单位：毫摄氏度） |
| thermal\_zoneX/type | thermal zone 名称 |
| thermal\_zoneX/trip\_point\_Y\_temp | 第 Y 个 trip point 的温度阈值 |
| thermal\_zoneX/trip\_point\_Y\_type | 第 Y 个 trip point 的类型 |
| thermal\_zoneX/policy | 当前使用的 thermal governor |
| cooling\_deviceX/cur\_state | cooling device 当前状态 |
| cooling\_deviceX/max\_state | cooling device 最大状态 |

更多 sysfs 接口请参考：
```
Documentation/driver-api/thermal/sysfs-api.rst
```

## 测试介绍

请按照上述 **测试方法** 部分描述，进行 thermal 驱动的测试验证。

## FAQ
