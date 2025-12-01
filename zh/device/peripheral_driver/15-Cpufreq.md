# CPUFREQ

介绍 CPUFREQ 的功能和使用方法。

## 模块介绍

CPUFREQ 子系统负责在 CPU 运行时通过动态调整 CPU 运行频率与电压，在确保性能需求的前提下，将功耗降至最低。

### 功能介绍

![](static/cpufreq.png)

1. **cpufreq core：** cpufreq framework 的核心模块，它主要实现三类功能：  
    - 抽象调频调压的公共逻辑接口  
    - 以 sysfs 的形式向用户空间提供统一的接口，以 notifier 的形式向其他 driver 提供频率变化的通知
    - 提供CPU频率和电压控制的驱动框架  
2. **cpufreq governor：** 负责调频调压的各种策略  
3. **cpufreq driver：** 负责平台相关调频调压机制的实现
4. **cpufreq stats：** 负责调频信息和各频点运行事件统计，提供每个CPU的cpufreq有关的统计信息

### 源码结构介绍

CPU 调频平台驱动目录如下：

```
drivers/cpufreq/
├── cpufreq.c
├── cpufreq_conservative.c
├── cpufreq-dt.c
├── cpufreq-dt.h
├── cpufreq-dt-platdev.c
├── cpufreq_governor_attr_set.c
├── cpufreq_governor.c
├── cpufreq_governor.h
├── cpufreq_ondemand.c
├── cpufreq_ondemand.h
├── cpufreq_performance.c
├── cpufreq_powersave.c
├── cpufreq_stats.c
├── cpufreq_userspace.c
├── freq_table.c
├── spacemit-cpufreq.c --> 平台驱动  
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持动态调频调压 | 实时根据负载切换频率与电压 |
| 支持频率 boost 到1.8Ghz | 最高可瞬时提升至 1.8 GHz |

### 性能参数

| 支持频率档位 | 频率档位对应电压 |  
| :-----| :----:|  
| 1600000000Hz | 1.05V |  
| 1228800000Hz | 0.95V |  
| 1000000000Hz | 0.95V |  
| 819000000Hz | 0.95V |  
| 614400000Hz | 0.95V |  

测试方法


1. 将策略修改成 userspace 模式
```
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

2. 查看所支持的频率列表
```
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
614400 819000 1000000 1228800 1600000
```

3. 设置cpu频率
```
echo 1228800 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
```

4. 查看频率是否设置成功
```
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

CPUFREQ 配置如下：

```
CONFIG_SPACEMIT_K1X_CPUFREQ:

 This adds the CPUFreq driver support for Freescale QorIQ SoCs
 which are capable of changing the CPU's frequency dynamically.
 
 Symbol: SPACEMIT_K1X_CPUFREQ [=y]
 Type  : tristate
 Defined at drivers/cpufreq/Kconfig:315
 Prompt: CPU frequency scaling driver for Spacemit K1X
 Depends on: CPU_FREQ [=y] && OF [=y] && COMMON_CLK [=y]
 Location:
  -> CPU Power Management
   -> CPU Frequency scaling
    -> CPU Frequency scaling (CPU_FREQ [=y])
     -> CPU frequency scaling driver for Spacemit K1X (SPACEMIT_K1X_CPUFREQ [=y])
 Selects: CPUFREQ_DT [=y] && CPUFREQ_DT_PLATDEV [=y]    
```

### DTS 配置

完整 OPP 表位于
`arch/riscv/boot/dts/spacemit/k1-x_opp_table.dtsi`
（文件较大，此处仅引用路径）

## Debug 介绍

### sysfs

所有节点位于：
`/sys/devices/system/cpu/cpufreq/policy0/`

|节点                            | 作用说明                       |
|------------------------------  | -------------------------- |
|affected\_cpus, related\_cpus   | 查看系统支持的策略             |
|scaling\_governor               | 当前生效的调频策略                  |
|boost                           | 支持boost的节点               |
|scaling\_available\_frequencies | 查看系统支持的频率点           |
|scaling\_max\_freq              | 软件允许的最大频率限制                |
|cpuinfo\_cur\_freq              | CPU 硬件实时频率             |
|scaling\_available\_governors   | 查看系统支持的策略       |
|scaling\_min\_freq              | 软件允许的最小频率限制                |
|cpuinfo\_max\_freq              | 硬件支持的最大频率              |
|scaling\_boost\_frequencies     | boost 模式下 CPU 支持的频率           |
|scaling\_setspeed               | userspace 模式是设置 CPU 频率的接口 |
|cpuinfo\_min\_freq              | 硬件支持的最小频率              |
|scaling\_cur\_freq, cpuinfo\_transition\_latency | 查看当前CPU跑的频率           |
|scaling\_driver                 | 当前cpu调频驱动的名称         |


## 测试介绍

测试调频调压，可以按照上述总结的 “Debug 方法” 循环测试设置不同频率的正确性

## FAQ
