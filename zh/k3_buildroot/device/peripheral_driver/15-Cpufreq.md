# CPUFREQ

介绍 K3 平台 CPUFREQ 的功能、设备树配置方式、用户空间调频接口与常见调试方法。

## 模块介绍

CPUFREQ 子系统负责在系统运行过程中动态调整 CPU 频率，并在需要时配合 OPP（Operating Performance Points）表、时钟和电源约束一起工作，在性能与功耗之间取得平衡。

对用户来说，CPUFREQ 文档最需要回答的是这几个问题：

- K3 用的是哪套 cpufreq 驱动；
- 频点是写死在驱动里，还是来自 DTS / OPP 表；
- 哪些 CPU 共用一套策略；
- 怎么查看当前频率、支持频点、governor；
- 如果频率拉不上去，应该先查哪里。

### 功能介绍

![](static/cpufreq.png)

Linux CPUFREQ 体系通常分为四层：

1. **cpufreq core**  
   提供调频框架、sysfs 接口和通知机制；
2. **cpufreq driver**  
   负责平台相关的频率切换逻辑；
3. **governor**  
   负责决定什么时候升频、什么时候降频；
4. **OPP / clock / regulator**  
   描述不同频率档位下所需的时钟与电压约束。

K3 平台的 CPUFREQ 不是单纯“改一个频率值”这么简单，而是和以下内容一起联动：

- `drivers/cpufreq/spacemit-k3-cpufreq.c`
- `cpufreq-dt`
- `operating-points-v2`
- CPU cluster 的 PLL / clock parent
- `clst-supply` 电源约束（部分 cluster）

### 源码结构介绍

K3 CPUFREQ 相关代码主要位于：

```text
linux-6.18/
|-- drivers/cpufreq/
|   |-- cpufreq.c
|   |-- cpufreq-dt.c
|   |-- cpufreq-dt-platdev.c
|   |-- cpufreq_governor.c
|   |-- cpufreq_ondemand.c
|   |-- cpufreq_userspace.c
|   |-- cpufreq_performance.c
|   |-- cpufreq_powersave.c
|   |-- cpufreq_stats.c
|   `-- spacemit-k3-cpufreq.c          # K3 平台相关 cpufreq 扩展驱动
`-- arch/riscv/boot/dts/spacemit/
    |-- k3_opp_table.dtsi              # K3 OPP 表
    |-- k3.dtsi                        # CPU 基础节点
    `-- k3*.dts                        # 板级 DTS，通常通过 include OPP 表生效
```

从当前 SDK 看，K3 不是完全自定义一整套 cpufreq 框架，而是：

- 使用 **`cpufreq-dt`** 作为基础调频驱动；
- 再通过 **`spacemit-k3-cpufreq.c`** 提前配置 OPP / clock / regulator 相关上下文；
- 通过 `operating-points-v2` 和 OPP 表驱动实际频点切换。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 基于 OPP 表调频 | 频率档位主要来自 `k3_opp_table.dtsi` |
| 基于 cpufreq-dt | 底层复用标准 `cpufreq-dt` 路径 |
| K3 平台扩展驱动 | `spacemit-k3-cpufreq.c` 在 probe 前预配置 OPP / clk / regulator |
| cluster 级共享策略 | OPP 表使用 `opp-shared`，同一 cluster 的 CPU 共用频率策略 |
| 频率与时钟联动 | OPP 表中同时关联 PLL/clock 信息 |
| 部分 cluster 具备电压约束 | `cpu_0 ~ cpu_7` 通过 `clst-supply = <&edcdc1>` 参与调压 |

### K3 当前频点现状

从 `k3_opp_table.dtsi` 可以看到，K3 当前至少定义了两套 cluster 频点表：

- `clst_core_opp_table0_x100`
- `clst_core_opp_table0_a100`

其中：

- `cpu_0 ~ cpu_7` 绑定 `clst_core_opp_table0_x100`
- `cpu_8 ~ cpu_15` 绑定 `clst_core_opp_table0_a100`

#### `clst_core_opp_table0_x100` 示例频点

| 频率 | 电压 |
| :--- | :--- |
| 2400000000 | 1000000 uV |
| 2300000000 | 880000 uV |
| 2200000000 | 880000 uV |
| 2100000000 | 880000 uV |
| 2000000000 | 880000 uV |
| 1800000000 | 880000 uV |
| 1600000000 | 880000 uV |
| 1500000000 | 850000 uV |
| 1400000000 | 850000 uV |
| 1300000000 | 850000 uV |
| 1200000000 | 850000 uV |
| 1100000000 | 850000 uV |
| 1000000000 | 800000 uV |
| 819200000 | 800000 uV |
| 614400000 | 800000 uV |

#### `clst_core_opp_table0_a100` 示例频点

这套表也覆盖了：

- `2000000000`
- `1900000000`
- `1850000000`
- `1800000000`
- `1700000000`
- `1600000000`
- `1500000000`
- `1400000000`
- `1300000000`
- `1200000000`
- `1100000000`
- `1000000000`
- `819200000`
- `614400000`

但当前 DTS 片段里没有像 `x100` 那样逐项列出 `opp-microvolt`，因此文档里不应擅自补充不存在的电压值。

### 从用户角度最值得关注的点

#### 1. K3 不是所有 CPU 各自独立调频

K3 当前 DTS 使用的是 cluster 级共享 OPP 设计，`opp_table` 节点带有：

```dts
opp-shared;
```

这意味着同一组共享策略的 CPU，不是你单独调一个核就能完全独立跑一个频率，而是按 policy / related CPUs 的方式成组工作。

#### 2. 频率是否可达，不只取决于 governor

如果一个频点没有出现在：

- `operating-points-v2`
- OPP 表
- 对应供电/时钟配置

那么就算你在 userspace 里强行写目标频率，也未必能真正切过去。

#### 3. K3 cpufreq 驱动做了额外的 OPP / 时钟处理

`spacemit-k3-cpufreq.c` 不只是注册一个名字，它做了几件关键事：

- 在 `cpufreq-dt` platform device 绑定时，提前为各 CPU 初始化 OPP 上下文；
- 通过 `dev_pm_opp_set_config()` 指定 regulator 名称 `clst` 和 clock 名称 `cls0` / `cls1`；
- 对 `cpu >= 8` 的 cluster，不再配置 regulator 名称；
- 通过 notifier 在 PRECHANGE 阶段处理 cluster PLL / parent 切换。

也就是说，K3 这套方案是 **cpufreq-dt + SpacemiT 平台补丁层**，不是完全 generic 的最小实现。

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS / OPP 配置**。

### CONFIG 配置

K3 相关配置的核心是：

- `CONFIG_CPU_FREQ`
- `CONFIG_CPUFREQ_DT`
- `CONFIG_CPUFREQ_DT_PLATDEV`
- `CONFIG_SPACEMIT_K3_CPUFREQ`
- 以及常用 governor：
  - `CONFIG_CPU_FREQ_GOV_PERFORMANCE`
  - `CONFIG_CPU_FREQ_GOV_POWERSAVE`
  - `CONFIG_CPU_FREQ_GOV_USERSPACE`
  - `CONFIG_CPU_FREQ_GOV_ONDEMAND`
  - `CONFIG_CPU_FREQ_GOV_CONSERVATIVE`

K3 平台驱动的 Kconfig 片段如下：

```text
config SPACEMIT_K3_CPUFREQ
	tristate "CPU frequency scaling driver for Spacemit K1X"
	depends on OF && COMMON_CLK
	select CPUFREQ_DT
	select CPUFREQ_DT_PLATDEV
	help
	  This adds the CPUFreq driver support for Spacemit K3 SoC
	  which are capable of changing the CPU's frequency dynamically.
```

> 注意：这里的 prompt 还写着 `Spacemit K1X`，但 help 和实际文件名已经明确这是 K3 路径。写文档时应按 **K3 实际实现** 说明，而不是照着 prompt 机械照抄。

### DTS / OPP 配置

#### 1. OPP 表位置

K3 完整 OPP 表位于：

```text
arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi
```

多个板级 DTS 直接 include 该文件，例如：

- `k3_evb.dts`
- `k3_deb1.dts`
- `k3_com260.dts`
- `k3_com260_kit_v02.dts`
- `k3_gemini_c0.dts`
- `k3_dc_board.dts`

这说明 K3 当前 CPUFREQ 频点定义是**集中维护**的，而不是每个板级 DTS 单独写一份。

#### 2. OPP 表的基本写法

示例：

```dts
clst_core_opp_table0_x100: opp_table0_x100 {
	compatible = "operating-points-v2";
	opp-shared;

	clocks = <&pll CLK_PLL3>, <&pll CLK_PLL4>,
		 <&pll CLK_PLL3_D1>, <&syscon_apmu CLK_APMU_CPU_C1_PLL_SRC>;
	clock-names = "pll_clst0", "pll_clst1", "pll_src", "clt_pll_src";

	opp2400000000 {
		opp-hz = /bits/ 64 <2400000000>, /bits/ 64 <2400000000>;
		opp-microvolt = <1000000>;
		clock-latency-ns = <200000>;
	};
};
```

这里最关键的字段包括：

| 属性 | 作用 |
| :--- | :--- |
| `compatible = "operating-points-v2"` | 使用标准 OPP v2 binding |
| `opp-shared` | 表示该 OPP 表由多个 CPU 共享 |
| `opp-hz` | 目标频率 |
| `opp-microvolt` | 目标电压（如定义） |
| `clock-latency-ns` | 频点切换时延约束 |
| `clocks` / `clock-names` | OPP 表关联的 PLL / source 时钟 |

#### 3. CPU 节点如何引用 OPP 表

以 `cpu_0` 为例：

```dts
&cpu_0 {
	clst-supply = <&edcdc1>;
	clocks = <&syscon_apmu CLK_APMU_CPU_C0_CORE>,
		 <&syscon_apmu CLK_APMU_CPU_C1_CORE>;
	clock-names = "cls0", "cls1";
	operating-points-v2 = <&clst_core_opp_table0_x100>;
};
```

而 `cpu_8` 则引用另一套表：

```dts
&cpu_8 {
	clocks = <&syscon_apmu CLK_APMU_CPU_C2_CORE>,
		 <&syscon_apmu CLK_APMU_CPU_C3_CORE>;
	clock-names = "cls0", "cls1";
	operating-points-v2 = <&clst_core_opp_table0_a100>;
};
```

可以看到，K3 当前至少存在两组 cluster：

- 一组带 `clst-supply = <&edcdc1>`
- 一组只给出 clock / OPP 关联

这也是为什么 `spacemit-k3-cpufreq.c` 里会对 `cpu >= 8` 单独处理 regulator 配置。

## 接口描述

### 用户空间常用 sysfs 路径

CPUFREQ 常见节点位于：

```text
/sys/devices/system/cpu/cpufreq/policy0/
```

如果系统有多组 policy，也可能看到：

- `/sys/devices/system/cpu/cpufreq/policy0/`
- `/sys/devices/system/cpu/cpufreq/policy8/`

具体以实际 cluster / related_cpus 划分为准。

常用节点如下：

| 节点 | 作用 |
| :--- | :--- |
| `affected_cpus` / `related_cpus` | 查看该 policy 覆盖哪些 CPU |
| `scaling_driver` | 查看当前使用的调频驱动 |
| `scaling_governor` | 当前 governor |
| `scaling_available_governors` | 可用 governor 列表 |
| `scaling_available_frequencies` | 可用频点 |
| `scaling_cur_freq` | 当前逻辑频率 |
| `cpuinfo_cur_freq` | 当前硬件频率 |
| `scaling_min_freq` | 软件最小频率限制 |
| `scaling_max_freq` | 软件最大频率限制 |
| `scaling_setspeed` | userspace governor 下手动设频 |
| `cpuinfo_transition_latency` | 频率切换时延 |
|

### 常见命令示例

#### 1. 查看有哪些 policy

```shell
ls /sys/devices/system/cpu/cpufreq/
```

#### 2. 查看当前驱动

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver
```

#### 3. 查看当前 governor

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

#### 4. 查看支持的频率列表

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
```

#### 5. 切到 userspace governor

```shell
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

#### 6. 手动设置频率

```shell
echo 1600000 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
```

> 这里 sysfs 通常使用的是 **kHz** 单位，因此 `1600000` 表示 `1.6 GHz`。

#### 7. 查看当前频率是否切换成功

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq
```

## Debug 介绍

### 1. 先看驱动是否加载成功

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver
```

如果返回为空，或者压根没有 `policy0`，优先检查：

- `CONFIG_CPU_FREQ`
- `CONFIG_SPACEMIT_K3_CPUFREQ`
- `CONFIG_CPUFREQ_DT`
- DTS 是否带了 `operating-points-v2`
- OPP 表是否被正确 include

### 2. 看 policy 是怎么分组的

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/related_cpus
cat /sys/devices/system/cpu/cpufreq/policy8/related_cpus
```

这一步很重要，因为 K3 大概率不是 16 个 CPU 各自独立一个 policy，而是 cluster 共享。

### 3. 看可用频点是否符合 OPP 表

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
```

如果实际频点比 DTS 里少，常见原因有：

- 某些 OPP 没有被内核接受；
- 对应 regulator / clock 约束不满足；
- thermal / QoS / 平台限制压住了最大频率；
- cluster 当前策略未开放更高频点。

### 4. 用 userspace 模式做静态验证

```shell
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
echo 1200000 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

这个方法最适合先验证：

- OPP 表是否生效；
- 频点切换路径是否通；
- 不是 governor 算法本身导致的“没升频”。

### 5. 配合负载观察 governor 行为

例如切到 `ondemand` 或 `schedutil` 后，用压测工具制造负载，再观察频率变化：

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
watch -n 0.5 cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

## 测试介绍

建议按下面的顺序验证 K3 CPUFREQ：

1. **确认 policy 节点存在**  
   确认 cpufreq 驱动已经工作；
2. **确认 `scaling_driver` 正常**  
   确认不是空壳 sysfs；
3. **确认 `scaling_available_frequencies`**  
   对照 `k3_opp_table.dtsi`；
4. **用 userspace 逐档切频**  
   先验证静态频点切换；
5. **再验证 governor 自动调频**  
   观察空闲与高负载切换行为；
6. **必要时联动电源和温控一起看**  
   避免把 thermal / regulator 限制误判成 cpufreq 问题。

## FAQ

### 1. 为什么明明 OPP 表里有高频，sysfs 里却看不到？

优先检查：

- 板级 DTS 是否 include 了 `k3_opp_table.dtsi`
- CPU 节点是否正确写了 `operating-points-v2`
- 对应 cluster 的 `clst-supply` / clock 名称是否匹配驱动预期
- 内核日志里是否有 OPP 初始化失败信息

### 2. 为什么写了 `scaling_setspeed` 但频率没变？

先确认你当前 governor 是：

```shell
userspace
```

如果还在 `ondemand` / `schedutil` / `performance`，那 `scaling_setspeed` 通常不会按你预期工作。

### 3. 为什么不同 CPU 的频率看起来一起变？

这是正常现象。K3 当前是按 cluster / policy 共享 OPP 和调频策略，不是每个核完全独立。

### 4. 为什么 Kconfig 里写的是 `Spacemit K1X`，但实际上是 K3？

这是当前 SDK 中的文案残留。判断应以这些内容为准：

- 文件名：`spacemit-k3-cpufreq.c`
- help 文本：`Spacemit K3 SoC`
- DTS / OPP 实际内容：`k3_opp_table.dtsi`

所以文档应按 **K3 实际平台实现** 来理解和描述。
