# Thermal

介绍 Thermal 的功能和使用方法。

## 模块介绍

Thermal 特指一套关于温控机制的驱动框架。Linux Thermal 框架是 Linux 系统下用于温度控制的一套架构，主要用于解决随着设备性能不断增强而引起的日益严重的发热问题。

### 功能介绍

![](static/thermal.png)

1. **thermal_cooling_device**：对应系实施冷却措施的驱动，是温控的执行者。  
2. **thermal core**：Thermal的只要程序，负责驱动初始化，维护 thermal_zone，governor 和 cooling device 三者的关系，并通过 sysfs 和用户空间交互。
3. **thermal governor**：温度控制算法，解决温控发生时 cooling device 应该选择哪个cooling state。
4. **thermal zone device**：主要用来创建 thermal zone 结点和连接 thermal sensor。节点位于 `/sys/class/thermal` 目录下，由 DTS 文件配置生成。
5. **thermal sensor**：温度传感器，主要是给 thermal 提供温度数据。

### 源码结构介绍

CPU 调频平台驱动目录如下：

```
drivers/thermal/
├── cpufreq_cooling.c
├── cpufreq_cooling.o
├── cpuidle_cooling.c
├── devfreq_cooling.c
├── gov_bang_bang.c
├── gov_fair_share.c
├── gov_power_allocator.c
├── gov_step_wise.c
├── gov_user_space.c
├── k1x-thermal.c    ---> 平台驱动
├── k1x-thermal.h
├── thermal_core.c
├── thermal_core.h
├── thermal_helpers.c
├── thermal_hwmon.c
├── thermal_hwmon.h
├── thermal_of.c
├── thermal_sysfs.c
```

## 关键特性

### 特性

- 支持 CPU 温度控制
- 支持 115°C 过温关机

**测试方法**

利用外部测温设备或运行高负载应用，制造温度变化的环境，检查 thermal 和 cpufreq 节点，确认 CPU 调温是否符合预期。
1. 查看 thermal sensor 节点：
```
cat /sys/class/thermal/thermal_zone1/temp  
```

2. 查看 CPU 调频节点：
```
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

THERMAL 配置如下：

```
CONFIG_K1X_THERMAL:
Enable this option if you want to have support for thermal management
controller present in Spacemit SoCs

 Symbol: K1X_THERMAL [=y]
 Type  : tristate
 Defined at drivers/thermal/Kconfig:450
 Prompt: Spacemit K1X Thermal Support
 Depends on: THERMAL [=y] && OF [=y] && SOC_SPACEMIT [=y]
 Location:
  -> Device Drivers
   -> Thermal drivers (THERMAL [=y])
    -> Spacemit K1X Thermal Support (K1X_THERMAL [=y]) 
```

### DTS 配置

```
&thermal_zones {
        cluster0_thermal {
                polling-delay = <0>;
                polling-delay-passive = <0>;
                thermal-sensors = <&thermal 3>;

                thermal0_trips: trips {
                        cls0_trip0: cls0-trip-point0 {
                                temperature = <75000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip1: cls0-trip-point1 {
                                temperature = <85000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip2: cls0-trip-point2 {
                                temperature = <95000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip3: cls0-trip-point3 {
                                temperature = <105000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip4: cls0-trip-point4 {
                                temperature = <115000>;
                                hysteresis = <5000>;
                                type = "critical";
                        };
                };

                cooling-maps {
                        map0 {
                                trip = <&cls0_trip0>;
                                cooling-device = <&cpu_0 0 0>,
                                                 <&cpu_1 0 0>,
                                                 <&cpu_2 0 0>,
                                                 <&cpu_3 0 0>,
                                                 <&cpu_4 0 0>,
                                                 <&cpu_5 0 0>,
                                                 <&cpu_6 0 0>,
                                                 <&cpu_7 0 0>;
                        };

                        map1 {
                                trip = <&cls0_trip1>;
                                cooling-device = <&cpu_0 1 1>,
                                                 <&cpu_1 1 1>,
                                                 <&cpu_2 1 1>,
                                                 <&cpu_3 1 1>,
                                                 <&cpu_4 1 1>,
                                                 <&cpu_5 1 1>,
                                                 <&cpu_6 1 1>,
                                                 <&cpu_7 1 1>;
                        };

                        map2 {
                                trip = <&cls0_trip2>;
                                cooling-device = <&cpu_0 2 3>,
                                                 <&cpu_1 2 3>,
                                                 <&cpu_2 2 3>,
                                                 <&cpu_3 2 3>,
                                                 <&cpu_4 2 3>,
                                                 <&cpu_5 2 3>,
                                                 <&cpu_6 2 3>,
                                                 <&cpu_7 2 3>;
                        };

                        map3 {
                                trip = <&cls0_trip3>;
                                cooling-device = <&cpu_0 4 5>,
                                                 <&cpu_1 4 5>,
                                                 <&cpu_2 4 5>,
                                                 <&cpu_3 4 5>,
                                                 <&cpu_4 4 5>,
                                                 <&cpu_5 4 5>,
                                                 <&cpu_6 4 5>,
                                                 <&cpu_7 4 5>;
                        };
                };
        };

        cluster1_thermal {
                polling-delay = <0>;
                polling-delay-passive = <0>;
                thermal-sensors = <&thermal 4>;

                thermal1_trips: trips {
                        cls1_trip0: cls1-trip-point0 {
                                temperature = <75000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip1: cls1-trip-point1 {
                                temperature = <85000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip2: cls1-trip-point2 {
                                temperature = <95000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip3: cls1-trip-point3 {
                                temperature = <105000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip4: cls1-trip-point4 {
                                temperature = <115000>;
                                hysteresis = <5000>;
                                type = "critical";
                        };
                };
        };
};

```

## 接口介绍

### API介绍

请参考内核目录下面的文档：  
```
Documentation/driver-api/thermal/
```

## Debug 介绍

### sysfs

请参考内核目录下面的文档：
```
Documentation/driver-api/thermal/sysfs-api.rst
```

## 测试介绍

请按照上述 **测试方** 部分描述，进行 thermal 驱动的测试验证。

## FAQ
