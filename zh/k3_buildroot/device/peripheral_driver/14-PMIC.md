# PMIC

介绍 K3 平台 PMIC 相关实现、驱动分层、DTS 配置方式，以及它与 regulator / RTC / 系统供电管理的关系。

## 模块介绍

PMIC（Power Management IC）负责为系统提供多路可控电源轨，并常常同时集成以下功能：

- BUCK / LDO / Switch regulator
- RTC
- 电源时序控制
- 掉电保持
- 部分平台上的按键/中断/复位辅助能力

在 K3 当前 SDK 里，PMIC 不能只理解成“一个 regulator 芯片”。实际看到的是 **多类电源管理器件并存**：

1. `spacemit,p1`：带 regulator + RTC 子功能的 PMIC
2. `spacemit,mpq8655`：单路外部 buck regulator
3. 其他固定电源 / GPIO regulator / 板级电源轨命名

所以对用户来说，PMIC 文档最重要的是先分清：

- 哪些器件属于真正的 PMIC / sub-PMIC；
- 哪些只是独立 regulator；
- 哪些 DTS 节点负责“电源芯片本体”，哪些节点负责“导出的电源轨”；
- CPU / 外设 DTS 里看到的 `*-supply = <&xxx>` 最终接到了哪一路 regulator。

## 功能介绍

![](static/pmic.png)

Linux 里的 PMIC 支持通常分成三层：

1. **MFD / parent device**  
   负责把一个复合电源芯片拆分成多个子设备；
2. **Regulator / RTC / 其他 function driver**  
   分别管理芯片内的不同功能块；
3. **Board DTS 电源拓扑**  
   给每一路输出命名、设置电压范围、绑定 consumer。

K3 当前最典型的一条 PMIC 路线就是：

- I2C 识别 `compatible = "spacemit,p1"`
- `simple-mfd-i2c.c` 创建共享 regmap
- 再拆出两个子设备：
  - `spacemit-p1-regulator`
  - `spacemit-p1-rtc`

这也是为什么前面 RTC 文档里会看到 P1 RTC，而它本质上又属于 PMIC 体系的一部分。

## 源码结构介绍

K3 PMIC 相关代码主要位于：

```text
linux-6.18/
|-- drivers/mfd/
|   `-- simple-mfd-i2c.c              # P1 PMIC 通过这里拆分子设备
|-- drivers/regulator/
|   |-- spacemit-p1.c                 # P1 PMIC regulator 驱动
|   |-- spacemit-mpq8655.c            # MPQ8655 regulator 驱动
|   |-- fixed.c
|   `-- gpio-regulator.c
|-- drivers/rtc/
|   `-- rtc-spacemit-p1.c             # P1 PMIC RTC 子功能
`-- arch/riscv/boot/dts/spacemit/
    `-- k3*.dts
```

本轮和 K3 PMIC 最相关的几个对象是：

- `drivers/mfd/simple-mfd-i2c.c`
- `drivers/regulator/spacemit-p1.c`
- `drivers/regulator/spacemit-mpq8655.c`
- `drivers/rtc/rtc-spacemit-p1.c`
- `arch/riscv/boot/dts/spacemit/k3_evb.dts`

### `simple-mfd-i2c.c` 在 P1 路线里的作用

P1 PMIC 不是直接由 regulator 驱动单独探测，而是先通过 `simple-mfd-i2c.c` 做 parent device。

代码里可以直接看到：

```c
static const struct mfd_cell spacemit_p1_cells[] = {
	{ .name = "spacemit-p1-regulator", },
	{ .name = "spacemit-p1-rtc", },
};
```

以及：

```c
{ .compatible = "spacemit,p1", .data = &spacemit_p1, },
```

这说明：

- `spacemit,p1` 对应的不是单个功能驱动；
- 它先被当作 MFD parent；
- 再拆成 regulator + RTC 两个子设备。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| K3 PMIC 不是单一器件 | 当前可见 P1 PMIC + MPQ8655 + 板级固定电源共同构成供电体系 |
| P1 走 MFD 架构 | `simple-mfd-i2c.c` 先建 regmap，再拆子设备 |
| P1 同时导出 regulator 和 RTC | `spacemit-p1-regulator` + `spacemit-p1-rtc` |
| P1 regulator 数量较多 | 6 路 BUCK、4 路 ALDO、7 路 DLDO |
| MPQ8655 是独立 buck regulator | 当前实现导出 `edcdc1` |
| consumer 通过 `*-supply` 连接 | 比如 CPUFREQ/OPP 里的 `clst-supply = <&edcdc1>` |
| 电压策略主要写在 DTS | 各 rail 的 min/max/boot-on/always-on 由板级 DTS 描述 |

### P1 PMIC 当前 regulator 组成

`drivers/regulator/spacemit-p1.c` 里直接定义了这些 regulator：

- BUCK：
  - `buck1`
  - `buck2`
  - `buck3`
  - `buck4`
  - `buck5`
  - `buck6`
- ALDO：
  - `aldo1`
  - `aldo2`
  - `aldo3`
  - `aldo4`
- DLDO：
  - `dldo1`
  - `dldo2`
  - `dldo3`
  - `dldo4`
  - `dldo5`
  - `dldo6`
  - `dldo7`

也就是说，P1 这一颗 PMIC 本身就导出了：

- **6 路 buck**
- **11 路 LDO（4 ALDO + 7 DLDO）**

### K3 上 `dldo` 的供电关系有特殊分支

这个点挺关键。

`spacemit-p1.c` 里有：

```c
#ifdef CONFIG_SOC_SPACEMIT_K3
#define P1_DLDO_DESC(_n) \
	P1_REG_DESC(DLDO, dldo, _n, "buck4", 0x67, LDO_MASK, 128, p1_ldo_ranges)
#else
#define P1_DLDO_DESC(_n) \
	P1_REG_DESC(DLDO, dldo, _n, "buck5", 0x67, LDO_MASK, 128, p1_ldo_ranges)
#endif
```

这说明：

- 在 **K3** 上，`dldo*` 的上游 supply_name 挂的是 `buck4`
- 不同于非 K3 路线里的 `buck5`

这类细节非常像“平台电源拓扑知识”，文档里应该明确写出来，不然很容易照着 K1 经验误判。

### P1 电压范围

从 `spacemit-p1.c` 可见：

#### BUCK 线性范围

```c
REGULATOR_LINEAR_RANGE(500000, 0, 170, 5000),
REGULATOR_LINEAR_RANGE(1375000, 171, 254, 25000),
```

表示大致支持：

- `500000uV` 起步
- 前半段 `5mV` 步进
- 高电压段 `25mV` 步进

#### LDO 线性范围

```c
REGULATOR_LINEAR_RANGE(225000, 0, 127, 25000),
```

表示大致支持：

- `225000uV` 起步
- `25mV` 步进

### MPQ8655 当前实现

`drivers/regulator/spacemit-mpq8655.c` 里当前导出的是：

- `edcdc1`

它的描述符写法为：

```c
MPQ8655_REG_DESC(BUCK, edcdc, _n, "vcc", 0x21, BUCK_MASK, 501, mpq8655_buck_ranges)
```

对应电压线性范围：

```c
REGULATOR_LINEAR_RANGE(0, 0, 0x1f4, 2000),
```

这说明 MPQ8655 在 K3 方案里扮演的是**额外的外部可调 buck**，不是像 P1 那样的复合 PMIC。

## 配置介绍

PMIC 相关配置要分成：

1. MFD / parent 节点
2. regulator 子节点
3. consumer 的 `*-supply` 引用

### 1. P1 PMIC 本体配置

板级 DTS 示例可以看到：

```dts
p1@41 {
	compatible = "spacemit,p1";
	reg = <0x41>;
	status = "disabled";
	vcc-supply = <&p4v>;

	regulators {
		compatible = "spacemit-p1-regulator";
		...
	};
};
```

这说明：

- P1 是一个 I2C 从设备；
- 上游输入电源通过 `vcc-supply` 指定；
- regulator 子功能放在 `regulators {}` 子节点中；
- 真正的注册工作由 MFD + regulator 子驱动共同完成。

### 2. P1 regulator 子节点配置

P1 的每一路 regulator 都可以在 DTS 中声明自己的约束，例如：

```dts
buck3: buck3 {
	regulator-min-microvolt = <800000>;
	regulator-max-microvolt = <800000>;
	regulator-ramp-delay = <5000>;
	regulator-always-on;
	regulator-boot-on;
};
```

常见属性包括：

| 属性 | 作用 |
| :--- | :--- |
| `regulator-min-microvolt` | 最小电压约束 |
| `regulator-max-microvolt` | 最大电压约束 |
| `regulator-ramp-delay` | 升降压斜率/延时约束 |
| `regulator-always-on` | 不允许运行中关闭 |
| `regulator-boot-on` | 启动阶段默认开启 |

### 3. MPQ8655 本体配置

板级 DTS 里还能看到：

```dts
mpq8655: mpq8655@30 {
	compatible = "spacemit,mpq8655";
	reg = <0x30>;
	vcc-supply = <&p12v>;

	regulators {
		compatible = "spacemit,regulator,mpq8655";

		edcdc1: edcdc1 {
			regulator-min-microvolt = <534000>;
			regulator-max-microvolt = <1000000>;
			regulator-ramp-delay = <10>;
			regulator-always-on;
			regulator-boot-on;
		};
	};
};
```

这一路在当前 K3 文档里尤其重要，因为前面 Cpufreq 文档里已经看到：

```dts
clst-supply = <&edcdc1>;
```

所以至少部分 cluster CPU 电压就是挂在这颗外部 regulator 上。

### 4. consumer 侧如何引用 PMIC 输出

对用户来说，PMIC 的真正落点不是“定义了多少 buck/ldo”，而是后面谁在用它。

例如：

- CPU 节点：`clst-supply = <&edcdc1>`
- 外设节点：`xxx-supply = <&buck3>` / `<&aldo2>` / `<&dldo4>`
- 板级供电名：`pwr_x100` / `pwr_a100` / `pwr_ext`

所以调 PMIC 时，通常要做两步：

1. 看 PMIC/regulator 节点本身有没有注册成功；
2. 再看 consumer 的 `*-supply` 是否真的绑到了期望 rail。

## 使用与调试介绍

### 1. 先看 regulator 是否枚举成功

常见接口：

```text
/sys/class/regulator/
```

例如：

```shell
ls /sys/class/regulator/
```

可以继续查看：

```shell
for r in /sys/class/regulator/regulator*; do
	echo "== $r =="
	cat $r/name 2>/dev/null || true
done
```

### 2. 看具体 rail 的状态

常见可查看属性包括：

- `name`
- `microvolts`
- `state`
- `num_users`

例如：

```shell
cat /sys/class/regulator/regulator0/name
cat /sys/class/regulator/regulator0/microvolts
cat /sys/class/regulator/regulator0/state
```

### 3. 结合 consumer 一起看最有效

单看 PMIC 自己有没有起来，不够。

更实用的是反查 consumer，例如：

- CPUFREQ 异常时，查 `edcdc1`
- RTC 路线异常时，查 `spacemit,p1` 是否探测成功
- 某外设没起来时，查它的 `xxx-supply` 是不是连到了被禁用的 rail

### 4. P1 路线排查顺序

P1 这条链路建议按下面顺序查：

1. I2C 控制器是否起来；
2. `p1@41` 是否探测成功；
3. `simple-mfd-i2c` 是否把它拆成：
   - `spacemit-p1-regulator`
   - `spacemit-p1-rtc`
4. regulator 子设备是否成功注册多路 rail；
5. RTC 子设备是否成功注册 `/dev/rtcN`。

### 5. MPQ8655 路线排查顺序

MPQ8655 相对简单，重点查：

1. I2C 设备是否探测成功；
2. `edcdc1` 是否成功注册；
3. CPU 节点 `clst-supply` 是否真的引用它；
4. OPP 切换时 regulator 电压是否能满足约束。

## FAQ

### 1. K3 的 PMIC 是只有一颗 P1 吗？

不是。当前 SDK / DTS 里至少能看到：

- `spacemit,p1`
- `spacemit,mpq8655`
- 以及若干板级固定电源轨

所以 K3 电源体系是组合式的，不是单一 PMIC 全包。

### 2. 为什么 P1 不是直接一个 regulator 驱动探测完事？

因为它是复合器件，当前走的是 MFD 路线。先由 `simple-mfd-i2c.c` 建共享 regmap，再拆出 regulator 和 RTC 两个子功能。

### 3. K3 上 `dldo*` 的 supply_name 为什么值得特别写？

因为在 `CONFIG_SOC_SPACEMIT_K3` 下，`dldo*` 的 `supply_name` 是 `buck4`，不是别的平台上的 `buck5`。这属于很典型的“平台电源拓扑差异”，不写清楚后面很容易误配。

### 4. PMIC 文档里最该记住哪条经验？

**不要只看 PMIC 芯片节点，要顺着 `*-supply` 一直追到具体 consumer。**

因为对用户真正有价值的不是“这个 PMIC 有多少路输出”，而是“哪一路输出在给哪个模块供电，以及它现在是不是满足约束”。
