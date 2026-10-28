# CPUFreq

## 模块介绍

### 功能介绍

CPUFreq 是 Linux 内核的 CPU 频率管理子系统，用于根据系统负载、用户设置及功耗和温度限制，调整 CPU 工作频率。

K3 的设备树调频方案由通用 `cpufreq-dt` 驱动和 Spacemit K3 平台适配代码共同实现：

- **CPUFreq core**：管理调频策略、频率范围、频率切换通知及 sysfs 接口。
- **Governor**：根据负载或用户请求，在当前允许范围内选择目标频率。
- **cpufreq-dt**：将目标频率转换为 OPP 请求，通过 clock 和 regulator 框架执行调整。
- **K3 平台适配代码**：选择 OPP 表、配置共享调频域及多路时钟，并处理 PLL 切换。
- **OPP 框架**：管理频率及对应电压配置。
- **CPUFreq stats**：提供频点驻留时间、切换次数等统计信息，需要单独启用。

本文介绍 K3 的设备树启动及 `cpufreq-dt` 调频路径。使用其他 CPUFreq 驱动的系统，应根据实际驱动确认配置和接口行为。

### K3 调频流程

K3 的主要调用关系如下：

```text
Governor / 用户空间 sysfs
           |
      CPUFreq core
           |------------------ 频率切换通知
           |                         |
           |                  K3 平台适配代码
           |                  处理 PLL 等时钟切换
           |
   cpufreq-dt：set_target()
           |
   dev_pm_opp_set_rate()
           |
           +-- Clock 框架
           |      |
           |   K3 CPU 时钟驱动
           |
           +-- Regulator 框架（X100）
                  |
             RPMI regulator
                  |
             OpenSBI / RCPU
                  |
                 PMIC
```

CPUFreq 使用的频率单位为 **kHz**，`cpufreq-dt` 调用 OPP 接口时将其转换为 **Hz**。

X100 的电压调整通过 RPMI regulator 完成；CPU 时钟由 Linux clock 框架及 K3 时钟驱动管理。

## 关键特性

### 调频域划分

当前设备树将 CPU 划分为两组共享调频域：

| 调频域 | CPU 范围 | 普通频点范围 | Boost 频点 | 电压管理 |
|---|---|---|---|---|
| X100 | CPU0～CPU7 | 614.4 MHz～2.15 GHz | 2.2、2.3、2.4 GHz | 通过 `edcdc1` 配置电压 |
| A100 | CPU8～CPU15 | 614.4 MHz～1.8 GHz | 1.85、1.9、2.0 GHz | 当前 OPP 表不配置电压 |

上述范围由若干离散频点组成，不表示支持范围内任意频率。

同一调频域内的 CPU 共享一个 CPUFreq policy。修改该 policy 的 governor、频率或限制，会影响该域内的全部 CPU。

在完整启用上述 CPU 的常见配置中，通常可以看到：

```text
/sys/devices/system/cpu/cpufreq/policy0
/sys/devices/system/cpu/cpufreq/policy8
```

其中，`policy0` 通常对应 X100，`policy8` 通常对应 A100。实际对应关系应通过 `related_cpus` 确认。

每个调频域配置了 `cls0`、`cls1` 两路 cluster 核心时钟。这里的“两组调频域”描述的是共享调频关系，不能据此将 K3 的物理拓扑简单理解为只有两个 cluster。

### X100 OPP 表选择

当前源码为 X100 提供两张 OPP 表，两张表的频点相同，普通频点的目标电压不同。

驱动读取最终设备树中 `/cpus` 节点的 `pmic-compatible` 属性，选择对应表：

| `pmic-compatible` 属性 | OPP 索引 | OPP 表 | 普通频点目标电压 |
|---|---:|---|---:|
| `spacemit,au4562` | 1 | `clst_core_opp_table1_x100` | 0.91 V |
| `spacemit,tda38740` | 0 | `clst_core_opp_table0_x100` | 0.95 V |
| 其他值或属性不存在 | 0 | `clst_core_opp_table0_x100` | 0.95 V |

当前 U-Boot 代码会读取板级 EEPROM 中的 `TLV_CODE_PMIC_TYPE`，尝试将 PMIC 类型写入 `/cpus/pmic-compatible`。Linux 根据接收到的最终属性进行判断。

当前 K3 CPUFreq 驱动不再使用原先的四张 X100 OPP 表，也不通过 `svt-dro` 选择这些表。

> au4562 表的普通频点目标电压以当前设备树配置的 **910000 µV，即 0.91 V** 为准。

### X100 频率与电压

当前两张 X100 OPP 表均包含以下 19 个频点，按频率从高到低排列：

| 频率（MHz） | CPUFreq 频率（kHz） | table0 目标电压（V） | table1 目标电压（V） | Boost |
|---:|---:|---:|---:|---|
| 2400 | 2400000 | 1.00 | 1.00 | 是 |
| 2300 | 2300000 | 1.00 | 1.00 | 是 |
| 2200 | 2200000 | 0.95 | 0.95 | 是 |
| 2150 | 2150000 | 0.95 | 0.91 | 否 |
| 2100 | 2100000 | 0.95 | 0.91 | 否 |
| 2000 | 2000000 | 0.95 | 0.91 | 否 |
| 1900 | 1900000 | 0.95 | 0.91 | 否 |
| 1850 | 1850000 | 0.95 | 0.91 | 否 |
| 1800 | 1800000 | 0.95 | 0.91 | 否 |
| 1700 | 1700000 | 0.95 | 0.91 | 否 |
| 1600 | 1600000 | 0.95 | 0.91 | 否 |
| 1500 | 1500000 | 0.95 | 0.91 | 否 |
| 1400 | 1400000 | 0.95 | 0.91 | 否 |
| 1300 | 1300000 | 0.95 | 0.91 | 否 |
| 1200 | 1200000 | 0.95 | 0.91 | 否 |
| 1100 | 1100000 | 0.95 | 0.91 | 否 |
| 1000 | 1000000 | 0.95 | 0.91 | 否 |
| 819.2 | 819200 | 0.95 | 0.91 | 否 |
| 614.4 | 614400 | 0.95 | 0.91 | 否 |

表中电压为 `opp-microvolt` 的目标值。实际配置还包含最小、最大允许电压，最终电压设置受 regulator 能力和板级约束影响。

**2.2 GHz 在当前设备树中同样带有 `turbo-mode`，属于 Boost 频点。**

### A100 频率配置

当前 A100 使用一张 OPP 表，包含以下 14 个频点，按频率从高到低排列：

| 频率（MHz） | CPUFreq 频率（kHz） | Boost |
|---:|---:|---|
| 2000 | 2000000 | 是 |
| 1900 | 1900000 | 是 |
| 1850 | 1850000 | 是 |
| 1800 | 1800000 | 否 |
| 1700 | 1700000 | 否 |
| 1600 | 1600000 | 否 |
| 1500 | 1500000 | 否 |
| 1400 | 1400000 | 否 |
| 1300 | 1300000 | 否 |
| 1200 | 1200000 | 否 |
| 1100 | 1100000 | 否 |
| 1000 | 1000000 | 否 |
| 819.2 | 819200 | 否 |
| 614.4 | 614400 | 否 |

A100 的 OPP 表未配置 `opp-microvolt`，K3 平台适配代码也不为 CPU8 及以上 CPU 配置 OPP regulator。

因此，本文所述 A100 CPUFreq 路径执行调频，不通过该路径调整电压。

### 高频 PLL 切换

K3 平台代码在 `CPUFREQ_PRECHANGE` 通知中处理高频 PLL：

1. 当目标频率大于 **1 GHz** 时，设置该调频域对应的两个 PLL。
2. 如果旧频率也大于 **1 GHz**，则先将 OPP 管理的两路 cluster 核心时钟切换到 **819.2 MHz**，再调整 PLL。
3. 随后由调频流程继续完成目标 OPP 设置。

源码中的 `TURBO0_FREQUENCY` 为 1 GHz，是这段 PLL 切换逻辑的判断阈值。

CPUFreq 的 Boost 频点由设备树中的 `turbo-mode` 标记决定，两者含义不同。例如，X100 的 1.6 GHz 和 2.15 GHz 都会进入上述高频 PLL 处理路径，但属于普通频点。

## 配置介绍

### 内核配置

设备树调频路径涉及以下配置：

```text
CONFIG_CPU_FREQ=y
CONFIG_SPACEMIT_K3_CPUFREQ=y
CONFIG_CPUFREQ_DT=y
CONFIG_CPUFREQ_DT_PLATDEV=y

CONFIG_PM_OPP=y
CONFIG_COMMON_CLK=y

CONFIG_REGULATOR=y
CONFIG_REGULATOR_RPMI=y
CONFIG_MAILBOX=y
CONFIG_RISCV_SBI_MPXY_MBOX=y
```

还需要启用 K3 对应的时钟驱动，并由设备树提供 CPU 时钟和供电配置。

当前工作区启用的 governor 包括：

```text
CONFIG_CPU_FREQ_GOV_PERFORMANCE=y
CONFIG_CPU_FREQ_GOV_POWERSAVE=y
CONFIG_CPU_FREQ_GOV_USERSPACE=y
CONFIG_CPU_FREQ_GOV_ONDEMAND=y
CONFIG_CPU_FREQ_GOV_CONSERVATIVE=y
```

当前工作区的默认 governor 为：

```text
CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y
```

当前未启用：

```text
# CONFIG_CPU_FREQ_GOV_SCHEDUTIL is not set
# CONFIG_CPU_FREQ_STAT is not set
```

以上是当前工作区配置，不代表所有 K3 镜像采用相同默认值。运行时应查看 `scaling_governor` 和 `scaling_available_governors`。

常见 governor 的作用如下：

| Governor | 行为 |
|---|---|
| `performance` | 请求当前 policy 允许的最高频率 |
| `powersave` | 请求当前 policy 允许的最低频率 |
| `userspace` | 由用户空间通过 `scaling_setspeed` 请求频率 |
| `ondemand` | 根据 CPU 负载动态调整频率 |
| `conservative` | 根据 CPU 负载逐步调整频率 |
| `schedutil` | 利用调度器负载信息选择频率，需要单独启用 |

`performance` 仍受 Boost 状态、频率范围和温控等约束，并不保证始终运行在设备树定义的最高频点。

如需频率统计功能，可启用：

```text
CONFIG_CPU_FREQ_STAT=y
```

如需 CPUFreq 温控限频，还需要相应 thermal 配置及板级 cooling 绑定。当前工作区已启用：

```text
CONFIG_THERMAL=y
CONFIG_CPU_FREQ_THERMAL=y
```

### DTS 配置

OPP 表集中定义在 [k3_opp_table.dtsi](/data/home/liulixin/workspace/k3/linux-6.18/arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi)。

#### CPU 与 OPP 表绑定

X100 CPU 节点配置两张 OPP 表，由 K3 平台代码选择其中一张。以下以 CPU0 为例：

```dts
&cpu_0 {
	clst-supply = <&edcdc1>;

	clocks = <&syscon_apmu CLK_APMU_CPU_C0_CORE>,
		 <&syscon_apmu CLK_APMU_CPU_C1_CORE>;
	clock-names = "cls0", "cls1";

	operating-points-v2 = <&clst_core_opp_table0_x100>,
			      <&clst_core_opp_table1_x100>;
};
```

CPU1～CPU7 使用相同的供电、时钟和 OPP 表绑定。

A100 CPU 节点仅配置一张 OPP 表。以下以 CPU8 为例：

```dts
&cpu_8 {
	clocks = <&syscon_apmu CLK_APMU_CPU_C2_CORE>,
		 <&syscon_apmu CLK_APMU_CPU_C3_CORE>;
	clock-names = "cls0", "cls1";

	operating-points-v2 = <&clst_core_opp_table0_a100>;
};
```

CPU9～CPU15 使用相同的时钟和 OPP 表绑定。

#### OPP 属性

以下为 au4562 表中 2.15 GHz 普通频点的配置：

```dts
opp2150000000 {
	opp-hz = /bits/ 64 <2150000000>,
		 /bits/ 64 <2150000000>;
	opp-microvolt = <910000 910000 914000>;
	clock-latency-ns = <200000>;
};
```

各属性含义如下：

| 属性 | 含义 |
|---|---|
| `opp-hz` | 目标频率，单位 Hz；这里的两个值分别对应两路 OPP 时钟 |
| `opp-microvolt` | 电压配置，三元组依次为目标、最小、最大电压，单位 µV |
| `clock-latency-ns` | 设备树声明的时钟切换延迟，单位 ns |
| `turbo-mode` | 将该 OPP 标记为 Boost 频点 |
| `opp-shared` | 位于 OPP 表节点中，表示使用该表的设备共享 OPP 状态 |

Boost 频点示例：

```dts
opp2200000000 {
	opp-hz = /bits/ 64 <2200000000>,
		 /bits/ 64 <2200000000>;
	opp-microvolt = <950000 950000 954000>;
	clock-latency-ns = <200000>;
	turbo-mode;
};
```

`clock-latency-ns = <200000>` 表示配置值为 200 µs，不能据此认定包含 PLL、电压通信等操作的整次调频实测耗时一定为 200 µs。

板级设备树可能覆盖公共配置，运行时结果应以实际启动使用的设备树为准。

## Debug 介绍

### 确认调频域和驱动

以 root 用户执行：

```sh
ls /sys/devices/system/cpu/cpufreq/
```

查看各 policy 的共享 CPU、在线 CPU、驱动和 governor：

```sh
for p in /sys/devices/system/cpu/cpufreq/policy*; do
	[ -d "$p" ] || continue

	printf '\nPolicy: %s\n' "$p"

	printf 'related_cpus: '
	cat "$p/related_cpus"

	printf 'affected_cpus: '
	cat "$p/affected_cpus"

	printf 'scaling_driver: '
	cat "$p/scaling_driver"

	printf 'scaling_governor: '
	cat "$p/scaling_governor"
done
```

当前设备树调频路径的 `scaling_driver` 预期为：

```text
cpufreq-dt
```

`related_cpus` 表示该 policy 的共享 CPU；`affected_cpus` 表示当前受该 policy 管理的在线 CPU。

### 常用 sysfs 接口

下面的接口位于各 `policyX` 目录中，频率值单位均为 **kHz**。

| 节点 | 作用 |
|---|---|
| `related_cpus` | 与该 policy 共享调频关系的 CPU |
| `affected_cpus` | 当前受该 policy 管理的在线 CPU |
| `scaling_driver` | 当前 CPUFreq 驱动 |
| `scaling_available_governors` | 当前可选择的 governor |
| `scaling_governor` | 当前使用的 governor |
| `scaling_available_frequencies` | 频率表中的普通频点 |
| `scaling_boost_frequencies` | 频率表中带有 Boost 标记的频点 |
| `scaling_min_freq` | 当前 policy 生效的频率下限 |
| `scaling_max_freq` | 当前 policy 生效的频率上限 |
| `scaling_cur_freq` | CPUFreq 报告的当前频率 |
| `cpuinfo_min_freq` | 驱动报告的频率下界信息 |
| `cpuinfo_max_freq` | 驱动报告的频率上界信息 |
| `cpuinfo_cur_freq` | 通过驱动读取的当前频率 |
| `scaling_setspeed` | 在 `userspace` 下请求目标频率 |
| `boost` | 当前 policy 的 Boost 开关 |

全局 Boost 开关位于：

```text
/sys/devices/system/cpu/cpufreq/boost
```

查看当前频率时，需要区分接口来源：

- 当前 K3 `cpufreq-dt` 路径下，`scaling_cur_freq` 通常返回 CPUFreq 维护的 `policy->cur`。
- `cpuinfo_cur_freq` 通过驱动的 `.get` 回调读取；该驱动使用 `cpufreq_generic_get()`，最终调用 `clk_get_rate(policy->clk)`。

因此，这些接口适合检查 CPUFreq 和 clock 框架报告的频率状态，不能直接视为独立硬件计数器测得的实时执行频率。

### 确认 PMIC 及 OPP 表选择

读取运行时设备树中的 PMIC 属性：

```sh
PMIC_PROP=/sys/firmware/devicetree/base/cpus/pmic-compatible

if [ -r "$PMIC_PROP" ]; then
	tr -d '\000' < "$PMIC_PROP"
	printf '\n'
else
	printf 'pmic-compatible 不存在，K3 驱动默认选择 X100 table0\n'
fi
```

查看内核日志：

```sh
dmesg | grep -iE 'Spacemit K3|cpufreq|OPP|rpmi|regulator'
```

当前驱动会打印类似以下信息：

```text
Spacemit K3: pmic-compatible="spacemit,au4562", using X100 OPP table1
```

这是源码中的日志格式示例，具体内容以板端输出为准。

### 频率范围与温控

当前能够使用的频率受以下因素共同约束：

- 设备树提供的 OPP。
- Boost 开关状态。
- 用户设置的频率上下限。
- 温控及其他 frequency QoS 请求。
- 时钟和 regulator 的实际能力。

排查频率未达到预期时，先查看：

```sh
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

如果板级配置启用了 CPUFreq cooling，温控可以限制有效频率上限。`userspace` 和 `performance` 同样需要遵守该限制。

可读取 thermal zone 的类型及温度辅助排查：

```sh
for z in /sys/class/thermal/thermal_zone*; do
	[ -d "$z" ] || continue

	printf '\nThermal zone: %s\n' "$z"
	cat "$z/type"
	cat "$z/temp"
done
```

thermal zone 的 `temp` 通常以毫摄氏度表示，例如 `65000` 表示 65 ℃。

### 频率统计

启用 `CONFIG_CPU_FREQ_STAT` 后，可以查看：

```sh
cat /sys/devices/system/cpu/cpufreq/policy0/stats/time_in_state
cat /sys/devices/system/cpu/cpufreq/policy0/stats/total_trans
```

当前工作区未启用该配置，因此不能将 `stats` 目录不存在直接判断为 CPUFreq 驱动异常。

## 测试介绍

### 测试前检查

以下示例假定已经确认：

- `policy0` 对应 X100。
- `policy8` 对应 A100。
- 当前可用 governor 包含 `userspace`。

如果实际 policy 名称不同，需要替换为对应路径。

```sh
X100=/sys/devices/system/cpu/cpufreq/policy0
A100=/sys/devices/system/cpu/cpufreq/policy8

cat "$X100/related_cpus"
cat "$A100/related_cpus"

cat "$X100/scaling_available_governors"
cat "$A100/scaling_available_governors"
```

测试前应记录原有 governor、频率限制和 Boost 状态，以便测试结束后恢复。

### 普通频点调频

#### X100 设置为 1.6 GHz

先检查可用普通频点和当前有效范围：

```sh
cat "$X100/scaling_available_frequencies"
cat "$X100/scaling_min_freq"
cat "$X100/scaling_max_freq"
```

确认 `1600000` 位于当前允许范围后执行：

```sh
echo userspace > "$X100/scaling_governor"
echo 1600000 > "$X100/scaling_setspeed"

cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"
```

该操作影响共享此 policy 的 X100 CPU。

#### A100 设置为 1.2 GHz

```sh
cat "$A100/scaling_available_frequencies"
cat "$A100/scaling_min_freq"
cat "$A100/scaling_max_freq"
```

确认 `1200000` 位于当前允许范围后执行：

```sh
echo userspace > "$A100/scaling_governor"
echo 1200000 > "$A100/scaling_setspeed"

cat "$A100/scaling_cur_freq"
cat "$A100/cpuinfo_cur_freq"
```

`scaling_setspeed` 是目标频率请求，CPUFreq 会根据当前范围及离散频率表解析该请求。应写入表中存在的频点，并通过频率回读和日志确认结果。

### Boost 使用方法

#### Boost 与频率列表

当前实现分别通过两个文件列出普通频点和 Boost 频点：

```sh
cat "$X100/scaling_available_frequencies"
cat "$X100/scaling_boost_frequencies"
```

两份列表按频点的 Boost 标记分类：

- `scaling_available_frequencies` 列出普通频点。
- `scaling_boost_frequencies` 列出 Boost 频点。

**读取 Boost 列表不要求先打开 Boost。打开 Boost 后，Boost 频点也不会因此合并到普通频点列表中。**

这些文件描述频率表中的档位，不代表每个列出的频点此刻都允许选用。实际还需要检查 Boost 状态及 `scaling_min_freq`、`scaling_max_freq`。

#### 全局与 per-policy 开关

查看当前状态：

```sh
cat /sys/devices/system/cpu/cpufreq/boost
cat "$X100/boost"
cat "$A100/boost"
```

开关含义如下：

| 全局 Boost | 当前 policy Boost | 当前 policy 的 Boost 状态 |
|---:|---:|---|
| 0 | 0 | 关闭 |
| 1 | 0 | 关闭 |
| 1 | 1 | 允许使用 Boost 频点，仍受其他限制约束 |

当前实现还有以下行为：

- 写全局 Boost 开关，会同步支持 Boost 的各个活动 policy。
- 全局 Boost 关闭时，写 per-policy Boost 开关会返回错误。
- 需要分别控制两组 CPU 时，应先打开全局 Boost，再调整各 policy 的开关。
- 后续再次写全局开关，会重新同步各 policy 的 Boost 状态。

该 `cpufreq-dt` 实现初始未开启 Boost，启动脚本可能修改其状态，运行时应以实际读取结果为准。

#### X100 请求 2.4 GHz

打开全局 Boost：

```sh
echo 1 > /sys/devices/system/cpu/cpufreq/boost
```

此操作会同步开启支持 Boost 的其他活动 policy。如本次只测试 X100，可以再关闭 A100 的 Boost：

```sh
echo 0 > "$A100/boost"
```

确认 X100 的 Boost 状态、频点和有效上限：

```sh
cat "$X100/boost"
cat "$X100/scaling_boost_frequencies"
cat "$X100/scaling_max_freq"
```

当 `2400000` 位于当前有效范围时，请求 2.4 GHz：

```sh
echo userspace > "$X100/scaling_governor"
echo 2400000 > "$X100/scaling_setspeed"

cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"
```

如果频率未达到预期，应检查温控限制、regulator 配置及调频日志。

#### A100 请求 2.0 GHz

在全局 Boost 已打开的前提下：

```sh
echo 1 > "$A100/boost"

cat "$A100/scaling_max_freq"

echo userspace > "$A100/scaling_governor"
echo 2000000 > "$A100/scaling_setspeed"

cat "$A100/scaling_cur_freq"
cat "$A100/cpuinfo_cur_freq"
```

同样需要确认 `2000000` 位于当前有效范围。

#### 关闭 Boost

只关闭 X100 的 Boost，且保留其他 policy 的设置：

```sh
echo 0 > "$X100/boost"
```

关闭全局 Boost：

```sh
echo 0 > /sys/devices/system/cpu/cpufreq/boost
```

关闭后，驱动会重新计算允许范围，普通频点的最高值为：

- X100：2.15 GHz。
- A100：1.8 GHz。

实际生效上限还可能受到其他限制。关闭 Boost 不会删除 `scaling_boost_frequencies` 中的档位。

### 测试结果判断与恢复

调频测试不能仅以 `echo` 命令是否成功作为判断依据。

当前代码会将目标频率限制到 policy 允许范围，并映射到频率表中的档位。例如，Boost 未开启时，请求一个 Boost 频率可能被限制到当前允许的上限。

此外，当前 `scaling_setspeed` 的 sysfs 写入实现没有将 governor 回调的返回值直接返回给用户空间。因此，写入成功也不能单独证明底层切换成功。

测试后应检查：

```sh
cat "$X100/scaling_governor"
cat "$X100/scaling_min_freq"
cat "$X100/scaling_max_freq"
cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"

dmesg | grep -iE 'cpufreq|OPP|rpmi|regulator'
```

测试结束后，按测试前记录恢复 Boost 状态、频率限制及 governor。恢复 per-policy Boost 状态时，应先处理全局开关，再处理各 policy 开关。

## FAQ

### 为什么没有出现 policy 目录？

依次检查：

1. 是否启用了 `CONFIG_CPU_FREQ`、`CONFIG_SPACEMIT_K3_CPUFREQ` 和 `CONFIG_CPUFREQ_DT`。
2. 当前使用的设备树是否包含 CPU 的 `operating-points-v2` 配置。
3. CPU 时钟、X100 regulator 及其依赖服务是否初始化成功。
4. 内核日志是否包含 OPP 初始化失败或驱动探测失败信息。

可先查看：

```sh
dmesg | grep -iE 'cpufreq|Spacemit K3|OPP|rpmi|regulator'
```

### 为什么修改一个 CPU 的频率，会影响其他 CPU？

同一调频域内的 CPU 共享一个 policy。当前设备树将 CPU0～CPU7 归为 X100 调频域，CPU8～CPU15 归为 A100 调频域。

可通过 `related_cpus` 查看共享关系。

### 为什么不能写 scaling_setspeed？

首先检查当前 governor：

```sh
cat "$X100/scaling_governor"
```

该接口需要 governor 提供相应回调。手动指定频率时，应先切换到 `userspace`，并确认它出现在 `scaling_available_governors` 中。

### 为什么 Boost 列表中有 2.4 GHz，却无法运行到该频率？

Boost 列表表示频率表中存在该档位。还需要满足：

- 全局 Boost 已打开。
- 当前 policy 的 Boost 已打开。
- 当前有效频率上限允许该频点。
- 时钟和 regulator 能完成目标设置。

应结合 `boost`、`scaling_max_freq`、频率回读及日志判断。

### 为什么请求频率与回读频率不一致？

常见原因包括：

- 请求值不在离散频率表中。
- 请求超出当前 policy 范围。
- Boost 未开启。
- 温控或其他 QoS 请求限制了上限。
- 当前使用动态 governor，频率在读取前已经再次变化。
- 底层时钟或电压调整失败。

建议先使用 `userspace`，请求一个明确存在且处于当前允许范围内的普通频点，再检查结果。

### 为什么 X100 的普通频点电压没有随频率变化？

当前 table0 将全部普通频点的目标电压配置为 0.95 V，table1 将全部普通频点的目标电压配置为 0.91 V。

这是当前 OPP 表的配置结果。支持 DVFS 并不要求每两个相邻频点都使用不同电压。

### 为什么没有 schedutil 或 stats？

两者取决于内核配置：

- `schedutil` 需要启用 `CONFIG_CPU_FREQ_GOV_SCHEDUTIL`。
- `stats` 需要启用 `CONFIG_CPU_FREQ_STAT`。

当前工作区未启用这两个选项，应根据实际镜像配置判断。

### 为什么 performance 下仍然低于最高 Boost 频率？

`performance` 请求当前 policy 允许的最高频率。

Boost 关闭、温控限频或其他上限请求，都可能使当前允许上限低于设备树中的最高频点。应查看 `scaling_max_freq`，不能仅根据 governor 名称判断最终频率。

## 源码参考

| 文件 | 主要作用 |
|---|---|
| [drivers/cpufreq/cpufreq.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/cpufreq/cpufreq.c) | CPUFreq core、policy、sysfs 和 Boost 控制 |
| [drivers/cpufreq/cpufreq-dt.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/cpufreq/cpufreq-dt.c) | 设备树 CPUFreq 驱动，通过 OPP 设置目标频率 |
| [drivers/cpufreq/spacemit-k3-cpufreq.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/cpufreq/spacemit-k3-cpufreq.c) | K3 OPP 选表、多时钟和 regulator 配置、PLL 通知处理 |
| [drivers/cpufreq/freq_table.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/cpufreq/freq_table.c) | 频率表处理及普通、Boost 频率列表 |
| [drivers/cpufreq/cpufreq_userspace.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/cpufreq/cpufreq_userspace.c) | userspace governor |
| [arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi](/data/home/liulixin/workspace/k3/linux-6.18/arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi) | X100、A100 OPP 表和 CPU 绑定 |
| [drivers/clk/spacemit/ccu-k3.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/clk/spacemit/ccu-k3.c) | K3 时钟控制器 |
| [drivers/regulator/regulator-rpmi.c](/data/home/liulixin/workspace/k3/linux-6.18/drivers/regulator/regulator-rpmi.c) | Linux RPMI regulator 客户端 |
| [board/spacemit/k3/k3.c](/data/home/liulixin/workspace/k3/uboot-2022.10/board/spacemit/k3/k3.c) | U-Boot 板级设备树修正，包括 PMIC 属性写入 |
