# Thermal

介绍 K3 平台 Thermal 的功能、驱动实现、DTS 配置方式以及常见调试与测试方法。

## 模块介绍

Thermal 是 Linux 用来做温度采集、温控策略和降温联动的一整套框架。对 K3 来说，它不是单独一个“温度传感器驱动”就结束了，而是至少包含下面几层：

- 片上温度传感器驱动；
- thermal zone 描述；
- trip point（温控触发点）；
- cooling device（降温执行者）；
- thermal governor（温控策略）；
- 和 cpufreq / 风扇 / 其他 cooling device 的联动。

从用户角度，Thermal 文档最重要的不是抽象概念，而是这几个问题：

- K3 实际用的是哪个温度传感器驱动；
- DTS 里 thermal sensor、thermal zone、trip、cooling-maps 怎么连；
- K3 当前是靠 CPU 降频降温，还是靠风扇，还是两者都可能；
- 温度读数、trip 生效、降温动作到底怎么验证。

### 功能介绍

![](static/thermal.png)

Linux Thermal 框架通常由五部分组成：

1. **thermal sensor**  
   提供原始温度数据；
2. **thermal zone device**  
   把某个 sensor 封装成 thermal zone，提供 sysfs 节点；
3. **trip points**  
   设定不同温度阈值及触发类型，如 `passive`、`critical`；
4. **cooling device**  
   执行降温动作，比如 cpufreq 降频、风扇提速；
5. **thermal governor**  
   决定在不同温度区间下，cooling device 应该进入什么 state。

K3 当前 SDK 里的 Thermal 路线很清楚：

- 传感器驱动：`drivers/thermal/k3-thermal.c`
- DTS 传感器节点：`compatible = "spacemit,k3-tsensor"`
- thermal zone / trip / cooling-maps：在 DTS 中定义
- cooling device：至少可见风扇类设备（如 `ctf2301`），也可和 CPU cooling 结合

## 源码结构介绍

K3 Thermal 相关代码主要位于：

```text
linux-6.18/
|-- drivers/thermal/
|   |-- k3-thermal.c              # K3 平台温度传感器驱动
|   |-- k3-thermal.h
|   |-- thermal_core.c
|   |-- thermal_of.c
|   |-- thermal_sysfs.c
|   |-- cpufreq_cooling.c
|   |-- devfreq_cooling.c
|   |-- gov_step_wise.c
|   |-- gov_power_allocator.c
|   `-- ...
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3_com260.dts
    `-- k3*.dts
```

当前 K3 平台自己的 thermal 传感器驱动是：

```text
drivers/thermal/k3-thermal.c
```

对应头文件：

```text
drivers/thermal/k3-thermal.h
```

### `k3-thermal.c` 做了什么

这份驱动的核心逻辑并不复杂，但几个点很关键：

- 从 DTS 读取 `sensor_range`；
- 从 DTS 读取 `tsensor_map`；
- 获取并使能两路时钟：`func`、`bus`；
- 获取 reset 并 `deassert`；
- 初始化传感器寄存器；
- 为每个启用的 sensor 调用 `devm_thermal_of_zone_register()`；
- 再通过 `devm_thermal_add_hwmon_sysfs()` 暴露 hwmon 节点。

驱动里实际可直接看到：

- `compatible = "spacemit,k3-tsensor"`
- `devm_clk_get(dev, "func")`
- `devm_clk_get(dev, "bus")`
- `devm_reset_control_get_optional()`
- `devm_thermal_of_zone_register()`
- `devm_thermal_add_hwmon_sysfs()`

这也说明 K3 Thermal 不是“驱动里自己硬编码几个 thermal zone”，而是：

- 驱动只负责 sensor 采样；
- zone / trip / cooling-maps 主要由 DTS 描述。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| 支持最多 8 路 sensor | `MAX_SENSOR_NUMBER = 8` |
| 传感器启用可由 DTS 选择 | 通过 `sensor_range` + `tsensor_map` 控制 |
| 温度通过 raw data 转换得到 | 驱动读取寄存器后结合 offset 转成毫摄氏度 |
| 标准 thermal zone 注册方式 | 使用 `devm_thermal_of_zone_register()` |
| 支持 hwmon 导出 | 使用 `devm_thermal_add_hwmon_sysfs()` |
| 依赖 clock/reset | 节点要提供 `func` / `bus` clocks 和 reset |
| 降温动作由 DTS 决定 | trip 与 cooling-maps 不写在驱动里，而写在 DTS 里 |

### K3 传感器节点当前配置

`k3.dtsi` 里的 K3 thermal 节点是：

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

这个节点里最关键的属性有：

| 属性 | 作用 |
| :--- | :--- |
| `compatible = "spacemit,k3-tsensor"` | 绑定 K3 传感器驱动 |
| `clocks` / `clock-names` | 提供 `func` 与 `bus` 两路时钟 |
| `resets` | 提供 TSEN reset |
| `sensor_range` | 指定可枚举的 sensor 下标范围 |
| `tsensor_map` | 指定哪些 sensor 实际启用 |
| `temperature_offset` | 温度校正偏移 |
| `#thermal-sensor-cells = <1>` | thermal zone 里按 `<&thermal sensor_id>` 引用 |

### DTS 里的 `tsensor_map` 值很关键

当前默认值：

```dts
tsensor_map = <1 0 1 1 1 1 1 1>;
```

这表示：

- 总范围是 sensor `0 ~ 7`
- 但不是每个 sensor 都启用
- 第 1 号 sensor 当前是关闭的

驱动 probe 时会按 `sensor_range` 遍历，再根据 `tsensor_map[i]` 决定是否注册对应 thermal zone sensor。

所以如果某个 thermal zone 一直起不来，除了检查 zone 本身，也要反查：

- 该 zone 用的 sensor 编号是否在 `sensor_range` 内；
- `tsensor_map` 是否真的启用了这个 sensor。

## 配置介绍

Thermal 配置主要包括两部分：

1. **驱动使能配置**
2. **DTS thermal 节点 / thermal-zones 配置**

### CONFIG 配置

K3 thermal 驱动的 Kconfig 为：

```text
config K3_THERMAL
	tristate "Spacemit K3 Thermal Support"
	depends on OF && SOC_SPACEMIT
	help
	  Enable this option if you want to have support for thermal management
	  controller present in Spacemit SoCs
```

Makefile 中可见：

```text
obj-$(CONFIG_K3_THERMAL) += k3-thermal.o
```

常见还会联动这些 thermal 组件：

- `CONFIG_THERMAL`
- `CONFIG_THERMAL_OF`
- `CONFIG_CPU_THERMAL`
- `CONFIG_THERMAL_GOV_STEP_WISE`
- `CONFIG_CPU_FREQ_THERMAL`
- `CONFIG_HWMON`

> 具体内核配置名以实际 defconfig 为准，但 K3 平台传感器本身的核心开关就是 `CONFIG_K3_THERMAL`。

### DTS 配置

#### 1. thermal sensor 节点

K3 片上传感器节点已经在 `k3.dtsi` 里定义，通常板级 DTS 不需要重建，只需要正确引用它。

#### 2. thermal-zones 节点

K3 的 thermal zone 在 `k3.dtsi` 里预留了空壳：

```dts
thermal_zones: thermal-zones {

};
```

也就是说，**真正的 zone/trip/cooling-map 往往由板级 DTS 填充**。

#### 3. K3 COM260 板级示例

从 `k3_com260.dts` 可见，板级直接补充了多组 thermal zone，例如：

- `thermal_cluster0`
- `thermal_cluster1`
- `thermal_cluster2`
- `thermal_cluster3`

其中 `thermal_cluster1` 的写法例如：

```dts
thermal_cluster1 {
	polling-delay = <1000>;
	polling-delay-passive = <250>;
	thermal-sensors = <&thermal 5>;

	trips {
		thermal_cluster1_trip0: thermal_cluster1-trip0 {
			temperature = <40000>;
			hysteresis = <5000>;
			type = "passive";
		};
	};
};
```

这说明：

- zone 名字可以按板级方案自定义；
- `thermal-sensors = <&thermal N>` 中的 `N` 对应 K3 tsensor 的 sensor index；
- trip 的温度单位是毫摄氏度；
- `passive` 一般表示进入被动降温流程。

#### 4. 风扇 cooling-maps 示例

`k3_com260.dts` 还展示了一种很典型的 K3 用法：**温控直接映射到风扇 PWM/转速等级**。

例如：

```dts
cooling-maps {
	map0 {
		trip = <&thermal_cluster0_trip0>;
		cooling-device = <&ctf2301 0 50>;
	};

	map1 {
		trip = <&thermal_cluster0_trip1>;
		cooling-device = <&ctf2301 64 75>;
	};
};
```

这说明 K3 当前并不只是 CPU 降频一条散热路线，板级完全可以：

- 用风扇作 cooling device；
- 按不同 trip 配不同 cooling state 区间。

### Thermal 与 Cpufreq 的关系

虽然当前看到的 `k3_com260.dts` 示例重点是风扇 cooling，但 thermal 框架本身也完全可以和 CPU cooling 配合。

结合上一轮 Cpufreq 文档，用户应当理解：

- Thermal 负责“何时该降温”；
- Cpufreq cooling 负责“怎样通过降频降温”；
- 风扇 cooling 负责“怎样通过提风量降温”；
- 最终采用哪种 cooling-device，由 DTS `cooling-maps` 决定。

## 接口介绍

### sysfs 节点

Thermal 相关节点通常位于：

```text
/sys/class/thermal/
```

常见包括：

- `thermal_zone0/`
- `thermal_zone1/`
- `cooling_device0/`
- `cooling_device1/`

每个 thermal zone 常见属性：

- `type`
- `temp`
- `policy`
- `available_policies`
- `trip_point_*_temp`
- `trip_point_*_type`

### hwmon 节点

由于驱动里调用了：

```c
devm_thermal_add_hwmon_sysfs()
```

因此很多情况下也能在：

```text
/sys/class/hwmon/
```

下看到对应温度输入节点。

## 使用与调试介绍

### 1. 先看系统注册了哪些 thermal zone

```shell
ls /sys/class/thermal/
```

### 2. 看各 zone 名称和温度

```shell
cat /sys/class/thermal/thermal_zone0/type
cat /sys/class/thermal/thermal_zone0/temp
```

可以循环查看：

```shell
for z in /sys/class/thermal/thermal_zone*; do
	echo "== $z =="
	cat $z/type
	cat $z/temp
done
```

### 3. 看 cooling device

```shell
for c in /sys/class/thermal/cooling_device*; do
	echo "== $c =="
	cat $c/type
done
```

这一步能帮助你确认当前系统挂上的 cooling device 到底是：

- CPU frequency
- 风扇
- 还是其他设备

### 4. 看 trip 点是否和 DTS 一致

```shell
cat /sys/class/thermal/thermal_zone0/trip_point_0_temp
cat /sys/class/thermal/thermal_zone0/trip_point_0_type
```

如果实际 sysfs 里的 trip 和 DTS 不一致，优先检查：

- 板级 DTS 是否真的生效；
- 当前 thermal zone 是否来自你以为的那个 sensor；
- 编译进去的是不是对应板子的 dtb。

### 5. 联动 cpufreq 一起观察

如果 cooling-device 是 CPU 降频，建议同时观察：

```shell
watch -n 0.5 cat /sys/class/thermal/thermal_zone0/temp
watch -n 0.5 cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

这样能直接看到：

- 温度是否在升；
- 到达 trip 后是否开始降频；
- thermal 与 cpufreq 的联动是否真的发生。

### 6. 如果是风扇 cooling，就看风扇 state

如果板级像 `k3_com260.dts` 一样把 `ctf2301` 挂进 cooling-map，那调试重点就不是 cpufreq，而是：

- 风扇设备是否 probe 成功；
- thermal trip 到来后 cooling state 是否变化；
- PWM / 风扇转速是否真的提升。

## 测试介绍

建议按下面顺序测试 K3 Thermal：

1. **确认 thermal 驱动已注册**  
   看 `/sys/class/thermal/thermal_zone*` 是否存在；
2. **确认温度读数正常**  
   读取 `temp`，确认不是固定值或异常值；
3. **确认 trip 点正确**  
   核对 sysfs 与 DTS；
4. **确认 cooling device 枚举正确**  
   看 `cooling_device*` 类型；
5. **制造温升场景**  
   运行高负载程序或外部加热；
6. **观察 cooling 动作**  
   看是否降频、升风扇、或进入其他 cooling state；
7. **必要时观察恢复行为**  
   温度下降后 state 是否恢复。

### 简单测试命令

#### 查看温度

```shell
cat /sys/class/thermal/thermal_zone0/temp
```

#### 查看 CPU 当前频率

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

#### 持续观察温度变化

```shell
watch -n 0.5 cat /sys/class/thermal/thermal_zone0/temp
```

## Debug 介绍

### 1. 没有 thermal zone

优先检查：

- `CONFIG_K3_THERMAL` 是否打开；
- `thermal@d4018000` 节点是否存在且 `status = "okay"`；
- `clocks` / `clock-names` / `resets` 是否正确；
- DTS 中是否真的定义了 `thermal-zones`。

因为 K3 驱动只是注册 sensor，**如果 DTS 没写 thermal zone，用户空间就看不到完整 thermal 策略结构**。

### 2. 某个 zone 没有温度

先反查：

- `thermal-sensors = <&thermal N>` 的 `N` 是否越界；
- `N` 是否被 `tsensor_map` 启用；
- `sensor_range` 是否覆盖了这个编号。

K3 这里不是所有 sensor 都默认有效，这点非常关键。

### 3. 温度值明显不对

当前 K3 驱动里温度转换流程是：

- 读取 `REG_TSEN_LITE_TEMP_DATA`
- `& BITS_TEMP_DATA`
- `/ TEMP_RAW_DATA_DIV`
- `- temp_offset`
- `* 1000`

而 DTS 默认又给了：

```dts
temperature_offset = <274>;
```

所以如果温度异常偏高或偏低，要优先检查：

- `temperature_offset` 是否被改错；
- 原始寄存器读数是否稳定；
- 板级传感器校准是否匹配当前芯片。

### 4. trip 到了但没有降温动作

先分两种情况：

- zone 本身有没有更新；
- cooling device 有没有挂进去。

重点检查：

- `cooling-maps` 是否存在；
- `cooling-device = <...>` 是否引用对了；
- 风扇或 cpufreq cooling device 是否已经注册；
- 当前 governor / cooling state 是否真的允许动作。

### 5. 不要把 thermal 问题都归到 cpufreq 上

这点很常见。

在 K3 上，thermal 可能驱动的是：

- CPU 降频
- 风扇提速
- 或两者结合

所以看到“温度高了但没降频”，不一定是 thermal 坏了，也可能是：

- 当前 cooling-map 根本没绑 CPU；
- 板级方案主要靠风扇；
- 实际触发的是别的 cooling device。

## FAQ

### 1. K3 当前 thermal 的核心驱动是哪一个？

当前平台自己的片上传感器驱动是：

```text
drivers/thermal/k3-thermal.c
```

对应 DTS compatible 为：

```text
spacemit,k3-tsensor
```

### 2. K3 的 thermal zone 是驱动里写死的吗？

不是。驱动负责 sensor 注册，zone / trip / cooling-map 主要由 DTS 描述。

### 3. 为什么有的板子是降频，有的板子是拉风扇？

因为 cooling-device 是板级策略的一部分，由 `cooling-maps` 决定，不是 K3 thermal 驱动强行规定的。

### 4. 文档里最该记住哪条经验？

**先看 DTS 里的 `thermal-sensors`、`trips`、`cooling-maps`，再看驱动。**

因为 K3 这套 thermal 机制里，策略大头在 DTS，不在 `k3-thermal.c` 本身。
