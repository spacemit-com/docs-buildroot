# Clock

介绍 K3 平台时钟系统的组成、驱动实现、设备树配置方式、使用方法以及常用调试方法。

## 模块介绍

Clock 系统负责给 SoC 内部各个模块提供工作时钟，并且支持时钟的选择、分频和使能等功能。

### 功能介绍

#### 时钟框架

Linux 为了做好时钟管理，提供了一个时钟管理框架 Common Clock Framework（以下简称CCF），为设备驱动提供统一的操作接口，使设备驱动不必关心时钟硬件实现的具体细节。

![](static/CLOCK.png)

CCF 框架包括以下核心组成部分：

- **clock provider**：对应上图的右侧部分，即 clock controller，负责提供系统所需的各种时钟。

- **clock consumer**：对应上图的左侧部分，即使用时钟的一些设备驱动。通过 CCF 提供的通用 API 获取、配置和使能时钟。

- **clock framework**：CCF 的核心部分，向 clock consumers 提供操作 clock 的通用 API；实现时钟管理的核心逻辑，将与硬件相关的 clock 控制逻辑封装成操作函数集，交由 clock provider 实现。

- **device tree**：CCF 允许在设备树中声明可用的时钟与设备的关联。设备树中的 `clocks` 属性通过 “provider + ID” 指定了设备使用哪个时钟资源，`clock-names` 属性为这些时钟资源提供了一个易于理解的名称。设备驱动通过这些名称获取对应的时钟资源，并进行配置和管理。

#### 时钟结构

时钟系统通常包含以下器件：
- Oscillator / Crystal：有源或无源晶振，作为时钟源头。
- PLL（Phase Locked Loop）：锁相环，用于倍频，提高基础频率。
- Divider：分频器，提供不同频率。
- MUX：多路选择器，切换时钟源。
- GATE：时钟开关，控制时钟通断。

系统中可能存在很多个这样的硬件模块，呈树形结构，Linux 将它们管理成一个时钟树（clock-tree），系统中的时钟树（Clock Tree）通常以晶振（Oscillator/Crystal）为起点，逐层经由 PLL（倍频）、MUX（选择）、DIV（分频）及 GATE（控制）等节点，最终输出给设备使用。

K3 Clock 子系统负责给 SoC 内部各个模块提供工作时钟，包含以下几项工作：

- 从固定时钟源或 PLL 派生出各种频率，作为各模块的时钟源；
- 通过 MUX 为各模块提供时钟源选择；
- 通过 Divider 为各模块提供分频功能；
- 通过 Gate 为各模块提供时钟开关；

CCF 实现了多种基础时钟类型，例如:

- 固定速率时钟 fixed_rate clock
- 门控时钟 gate clock
- 分频器时钟 divider clock
- 复用器时钟 mux clock

#### K3 时钟体系

K3 平台的时钟体系并不是单一一个 clock provider，而是按地址域拆成多个 clock provider。

| clock provider 节点 | 地址 | 典型用途 |
| :--- | :--- | :--- |
| `pll` | `0xd4090000` | PLL时钟 |
| `syscon_mpmu` | `0xd4050000` | MPMU 相关基础时钟/低速时钟 |
| `syscon_apmu` | `0xd4282800` | APMU 域时钟，如 QSPI、SDH、USB、CPU 等 |
| `syscon_apbc` | `0xd4015000` | APB 外设时钟，如 UART/PWM/I2C/SPI/RTC |
| `syscon_apbc2` | `0xf0610000` | 安全域APB外设时钟 |
| `syscon_dciu` | `0xd8440000` | DCIU 相关控制 |
| `syscon_rcpu_sysctrl` | `0xc0880000` | R-domain 系统控制相关时钟/复位 |
| `syscon_rcpu_uartctrl` | `0xc0881f00` | R-domain UART |
| `syscon_rcpu_i2sctrl` | `0xc0882000` | R-domain I2S |
| `syscon_rcpu_spictrl` | `0xc0885f00` | R-domain SPI |
| `syscon_rcpu_i2cctrl` | `0xc0886f00` | R-domain I2C |
| `syscon_rpmu` | `0xc088c000` | R-domain PMU |
| `syscon_rcpu_pwmctrl` | `0xc088d000` | R-domain PWM |

所以 K3 DTS 里常见的时钟引用形式是：

```dts
clocks = <&syscon_apbc CLK_APBC_SPI0>, <&syscon_apbc CLK_APBC_SPI0_BUS>;
```

这表示：

- Clock Provider 是 `syscon_apbc`
- 第一个 Clock ID 是 `CLK_APBC_SPI0`
- 第二个 Clock ID 是 `CLK_APBC_SPI0_BUS`

需要根据实际的驱动实现来确定对应的 Clock provider和 Clock ID。

#### 时钟与复位

时钟需要与复位信号配合使用才能正确初始化硬件模块。K3 的典型操作流程如下：

**模块初始化流程**

1. 释放复位 (reset_control_deassert)
2. 使能时钟 (clk_prepare_enable)
3. 配置工作时钟频率 (clk_set_rate)
4. 配置模块寄存器
5. 模块开始工作

**模块关闭流程**

当模块进入休眠或关闭时，按相反顺序操作：

1. 停止模块工作
2. 关闭时钟 (clk_disable_unprepare)
3. 复位模块 (reset_control_assert)

## 源码结构介绍

### 源码位置

K3 时钟复位相关代码主要位于：

```text
linux-6.18/
|-- drivers/clk/spacemit/
|   |-- ccu_common.h              # 通用时钟结构定义
|   |-- ccu_pll.c                 # PLL 时钟驱动源码
|   |-- ccu_pll.h                 # PLL 时钟头文件
|   |-- ccu_mix.c                 # MIX 时钟驱动源码
|   |-- ccu_mix.h                 # MIX 时钟头文件
|   |-- ccu_ddn.c                 # DDN 时钟驱动源码
|   |-- ccu_ddn.h                 # DDN 时钟头文件
|   |-- ccu-k3.c                  # K3 时钟控制器驱动
|   |-- Kconfig                   # 配置文件
|   `-- Makefile                  # 编译makefile
|-- include/dt-bindings/clock/
|   `-- spacemit,k3-syscon.h      # K3 Clock 和 Reset ID 定义
|-- include/soc/spacemit
|   `-- k3-syscon.h               # K3 Clock 和 Reset 寄存器定义
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3*.dts
```

可以看到，SpacemiT 实现了三种时钟类型：

- PLL 类型，锁相环类型（包含 PLL 和 PLLA 两种类型，K3 使用 PLLA 类型）
- DDN 类型，分数 divider，有一级除频，对应分母，一级倍频，对应分子
- MIX 类型，混合类型，支持 gate/mux/divider 的任一种或者任意组合

#### 自定义时钟类型

K3 Clock 驱动实现了三种主要的时钟类型：

##### PLL 类型

**功能**：锁相环（Phase Locked Loop），用于倍频和频率合成。

**特点**：
- 支持两种 PLL 类型：标准 PLL 和 PLLA（精细可调 PLL）
- PLLA 是 K3 新增的类型，支持更精细的频率调节
- 包含多个预定义的工作频率点

##### DDN 类型

**功能**：分数分频器，支持分子和分母的精细调节。

**特点**：
- 一级除频（对应分母），一级倍频（对应分子）
- 支持精细的频率调节
- 多用于音频时钟和其他需要精密频率的场景

##### MIX 类型

**功能**：混合时钟类型，支持 GATE、MUX、DIVIDER 的任意组合。

**特点**：
- 支持 GATE（门控）、MUX（多路选择）、DIVIDER（分频）的任意组合
- 灵活性最高，适用于多数外设时钟
- 配置最为复杂

## 配置介绍

主要包括 **内核 CONFIG 配置** 和 **设备树配置**

### CONFIG 配置

- `CONFIG_COMMON_CLK`：为 Common Clock Framework 提供支持，默认情况下，此选项为 `Y`

```text
-> Device Drivers│
  │       -> Common Clock Framework (COMMON_CLK [=y])│
```

- `CONFIG_SPACEMIT_CCU`：为 Spacemit SoC 提供 Clock 类型驱动支持，CONFIG_ARCH_SPACEMIT = Y 情况下，此选型为 `Y`

```text
-> Device Drivers│
  │       -> Common Clock Framework (COMMON_CLK [=y])│
  │              -> Clock support for SpacemiT SoCs (SPACEMIT_CCU [=y])│
```

- `CONFIG_SPACEMIT_K3_CCU`：为 K3 Clock 控制器驱动提供支持，CONFIG_SOC_SPACEMIT_K3 = Y 情况下，此选型为 `Y`

```text
-> Device Drivers│
  │       -> Common Clock Framework (COMMON_CLK [=y])│
  │              -> Clock support for SpacemiT SoCs (SPACEMIT_CCU [=y])│
  │                     -> Support for SpacemiT K3 SoC (SPACEMIT_K3_CCU [=y])
```

### DTS 配置

K3 时钟控制器按不同地址域分成了多个 clock controller，每个 clock controller 负责管理一个地址域内的时钟，clock controller 节点在 `k3.dtsi` 中定义，这些节点同时也是 reset controller（pll除外），DTS 配置如下：

```dts
/ {
        soc: soc {
                ...
                syscon_mpmu: system-controller@d4050000 {
                        compatible = "spacemit,k3-syscon-mpmu";
                        reg = <0x0 0xd4050000 0x0 0x10000>;
                        clocks = <&osc_32k>, <&vctcxo_1m>, <&vctcxo_3m>,
                                 <&vctcxo_24m>, <&reserved_clk>, <&external_clk>;
                        clock-names = "osc_32k", "vctcxo_1m", "vctcxo_3m", "vctcxo_24m",
                                      "reserved_clk", "external_clk";
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                pll: clock-controller@d4090000 {
                        compatible = "spacemit,k3-pll";
                        reg = <0x0 0xd4090000 0x0 0x10000>;
                        clocks = <&vctcxo_24m>;
                        spacemit,mpmu = <&syscon_mpmu>;
                        #clock-cells = <1>;
                };

                syscon_apmu: system-controller@d4282800 {
                        compatible = "spacemit,k3-syscon-apmu";
                        reg = <0x0 0xd4282800 0x0 0x400>;
                        clocks = <&osc_32k>, <&vctcxo_1m>, <&vctcxo_3m>, <&vctcxo_24m>,
                                 <&reserved_clk>, <&external_clk>;
                        clock-names = "osc_32k", "vctcxo_1m", "vctcxo_3m", "vctcxo_24m",
                                      "reserved_clk", "external_clk";
                        #clock-cells = <1>;
                        #power-domain-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_apbc: system-controller@d4015000 {
                        compatible = "spacemit,k3-syscon-apbc";
                        reg = <0x0 0xd4015000 0x0 0x1000>;
                        clocks = <&osc_32k>, <&vctcxo_1m>, <&vctcxo_3m>, <&vctcxo_24m>,
                                 <&reserved_clk>, <&external_clk>;
                        clock-names = "osc_32k", "vctcxo_1m", "vctcxo_3m", "vctcxo_24m",
                                      "reserved_clk", "external_clk";
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_apbc2: system-controller@f0610000 {
                        compatible = "spacemit,k3-syscon-apbc2";
                        reg = <0x0 0xf0610000 0x0 0x2000>;
                        clocks = <&osc_32k>, <&vctcxo_1m>, <&vctcxo_3m>, <&vctcxo_24m>,
                                 <&reserved_clk>, <&external_clk>;
                        clock-names = "osc_32k", "vctcxo_1m", "vctcxo_3m", "vctcxo_24m",
                                      "reserved_clk", "external_clk";
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_dciu: system-controller@d8440000 {
                        compatible = "spacemit,k3-syscon-dciu";
                        reg = <0x0 0xd8440000 0x0 0xc000>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_sysctrl: system-controller@c0880000 {
                        compatible = "spacemit,k3-syscon-rcpu-sysctrl";
                        reg = <0x0 0xc0880000 0x0 0x1000>;
                        clocks = <&vctcxo_24m>, <&external_clk>;
                        clock-names = "vctcxo_24m", "external_clk";
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_uartctrl: system-controller@c0881f00 {
                        compatible = "spacemit,k3-syscon-rcpu-uartctrl";
                        reg = <0x0 0xc0881f00 0x0 0x100>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_i2sctrl: system-controller@c0882000 {
                        compatible = "spacemit,k3-syscon-rcpu-i2sctrl";
                        reg = <0x0 0xc0882000 0x0 0x1000>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_spictrl: system-controller@c0885f00 {
                        compatible = "spacemit,k3-syscon-rcpu-spictrl";
                        reg = <0x0 0xc0885f00 0x0 0x100>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_i2cctrl: system-controller@c0886f00 {
                        compatible = "spacemit,k3-syscon-rcpu-i2cctrl";
                        reg = <0x0 0xc0886f00 0x0 0x100>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rpmu: system-controller@c088c000 {
                        compatible = "spacemit,k3-syscon-rpmu";
                        reg = <0x0 0xc088c000 0x0 0x800>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };

                syscon_rcpu_pwmctrl: system-controller@c088d000 {
                        compatible = "spacemit,k3-syscon-rcpu-pwmctrl";
                        reg = <0x0 0xc088d000 0x0 0x100>;
                        #clock-cells = <1>;
                        #reset-cells = <1>;
                };
                ...
        };
};

```
这些节点通过 `#clock-cells = <1>` 属性声明自己为 Clock Provider。各个设备节点通过 `clocks` 和 `clock-names` 属性引用对应的 Clock 资源。

## 接口描述

### Clock API 介绍

CCF 为设备驱动提供了通用的时钟操作的接口，可在 `include/linux/clk.h` 中查看，以下是一些常用接口：

- `get`：获取时钟句柄

```c
/*
* clk_get - get clk
* @dev: device
* @id: clock name of dts "clock-names"
*/
struct clk *clk_get(struct device *dev, const char *id);

/*
* devm_clk_get - get clk
* @dev：device
* @id：clock name of dts "clock-names"
*/
struct clk *devm_clk_get(struct device *dev, const char *id);

/*
* of_clk_get_by_name - get clk by name
* @np：device_node
* @id：clock name of dts "clock-names"
*/
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
```

上述接口，第 2 个参数缺省时，默认返回 DTS `clocks` 列表中的第 1 个时钟。

- `put`：释放时钟句柄

```c
/*
* clk_put - put clk
* @clk: clock source
*/
void clk_put(struct clk *clk);

/*
* devm_clk_put - put clk
* @dev: device
* @clk: clock source
*/
void devm_clk_put(struct device *dev, struct clk *clk);
```

- `prepare`：prepare 时钟，通常是在 enable 操作之前进行的准备步骤。

```c
/**
 * clk_prepare - prepare a clock source
 * @clk: clock source
 * This prepares the clock source for use.
 * Must not be called from within atomic context.
 */
int clk_prepare(struct clk *clk);
```

- `unprepare`：unprepare 时钟，通常是在 disable 操作之后的一些善后工作

```c
/**
 * clk_unprepare - undo preparation of a clock source
 * @clk: clock source
 * This undoes a previously prepared clock. The caller must balance
 * the number of prepare and unprepare calls.
 * Must not be called from within atomic context.
 */
void clk_unprepare(struct clk *clk);
```

- `enable`：使能时钟

```c
/**
 * clk_enable - inform the system when the clock source should be running.
 * @clk: clock source
 * If the clock can not be enabled/disabled, this should return success.
 * May be called from atomic contexts.
 * Returns success (0) or negative errno.
 */
int clk_enable(struct clk *clk);
```

- `disable`：关闭时钟

```c
/**
 * clk_disable - inform the system when the clock source is no longer required.
 * @clk: clock source
 * Inform the system that a clock source is no longer required by
 * a driver and may be shut down.
 * May be called from atomic contexts.
 * Implementation detail: if the clock source is shared between
 * multiple drivers, clk_enable() calls must be balanced by the
 * same number of clk_disable() calls for the clock source to be
 * disabled.
 */
void clk_disable(struct clk *clk);
```

`clk_prepare_enable` 是 `clk_prepare` 和 `clk_enable` 的组合，`clk_disable_unprepare`是 `clk_unprepare` 和 `clk_disable` 的组合。
`clk_prepare` 在可睡眠上下文中执行（如申请 regulator），`clk_enable` 在原子上下文中快速执行（操作寄存器），推荐统一使用组合接口。

- `set rate`：设置时钟频率

```c
/**
 * clk_set_rate - set the clock rate for a clock source
 * @clk: clock source
 * @rate: desired clock rate in Hz
 * Updating the rate starts at the top-most affected clock and then
 * walks the tree down to the bottom-most clock that needs updating.
 * Returns success (0) or negative errno.
 */
int clk_set_rate(struct clk *clk, unsigned long rate);
```

- `get rate`：获取当前时钟频率

```c
/**
 * clk_get_rate - obtain the current clock rate (in Hz) for a clock source.
 * This is only valid once the clock source has been enabled.
 * @clk: clock source
 */
unsigned long clk_get_rate(struct clk *clk);

```

- `set parent`：设置父时钟

```c
/**
 * clk_set_parent - set the parent clock source for this clock
 * @clk: clock source
 * @parent: parent clock source
 * Returns success (0) or negative errno.
 */
int clk_set_parent(struct clk *clk, struct clk *parent);

```

- `get parent`：获取当前父时钟句柄

```c
/**
 * clk_get_parent - get the parent clock source for this clock
 * @clk: clock source
 * Returns struct clk corresponding to parent clock source, or
 * valid IS_ERR() condition containing errno.
 */
struct clk *clk_get_parent(struct clk *clk);
```

- `round rate`：获取与目标频率接近并且时钟控制器可以提供的频率

```c
/**
 * clk_round_rate - adjust a rate to the exact rate a clock can provide
 * @clk: clock source
 * @rate: desired clock rate in Hz
 * This answers the question "if I were to pass @rate to clk_set_rate(),
 * what clock rate would I end up with?" without changing the hardware
 * in any way. In other words:
 *   rate = clk_round_rate(clk, r);
 * and:
 *   clk_set_rate(clk, r);
 *   rate = clk_get_rate(clk);
 * are equivalent except the former does not modify the clock hardware
 * in any way.
 * Returns rounded clock rate in Hz, or negative errno.
 */
long clk_round_rate(struct clk *clk, unsigned long rate);
```

## 使用介绍

主要以 **Provider** 和 **Consumer** 两个角色角度来介绍。

### Provider

K3 Clock Provider 在 `k3.dtsi` 里已经定义好，板级 DTS 一般不需要重新创建 Provider，这部分一般没有什么工作。

### Consumer

Clock Consumer一般需要完成以下工作：

- 找到正确的 Clock Provider 和 Clock ID，在DTS节点中配置 `clocks`；
- 在 DTS 节点中配置 `clock-names`：
  - 不同设备的 `clock-names`会有区别，具体看 clock 的用途，也可以根据实际的情况自定义，主要是驱动获取句柄时需要使用，`clock-names` 常见的取值有`func`、`bus`、`clk`、`gate`等；
- 必要时，在 DTS 节点中配置 `assigned-clocks` / `assigned-clock-parents` / `assigned-clock-rates`；
- 设备驱动中通过接口获取 Clock 句柄，并进行 Clock 相关的操作。

大多数 K3 外设 DTS 节点 Clock 配置写法都类似：

```dts
xxx: device@addr {
	clocks = <&ClockProvider CLK_ID0>, <&ClockProvider CLK_ID1>;
	clock-names = "func", "bus";
	status = "disabled";
};
```
Clock Provider 和 Clock ID 可以通过ID的前缀来对应，具体对应关系如下：

| Clock id前缀 | 前缀对应的 Provider 节点 |
|---------------------|-----------------------|
| CLK_PLL_       | pll                   |
| CLK_MPMU_     | syscon_mpmu           |
| CLK_APBC_     | syscon_apbc           |
| CLK_APMU_     | syscon_apmu           |
| CLK_DCIU_      | syscon_dciu           |
| CLK_RCPU_SYSCTRL_ | syscon_rcpu_sysctrl  |
| CLK_RCPU_UARTCTRL_ | syscon_rcpu_uartctrl |
| CLK_RCPU_I2SCTRL_ | syscon_rcpu_i2sctrl  |
| CLK_RCPU_SPICTRL_ | syscon_rcpu_spictrl  |
| CLK_RCPU_I2CCTRL_ | syscon_rcpu_i2cctrl  |
| CLK_RPMU_     | syscon_rpmu           |
| CLK_RCPU_PWMCTRL_ | syscon_rcpu_pwmctrl  |
| CLK_APBC2_    | syscon_apbc2          |


## 使用示例

模块如要使用 clock 功能，需要在 DTS 配置 clocks 和 clock-names 属性，然后在驱动中通过CCF API 进行 Clock 相关的操作。

- 配置 DTS
在 `include/dt-bindings/clock/spacemit,k3-syscon.h` 找到对应的Clock ID，配置到模块 DTS 中。
以`can0`为例：
can0 的 Clock ID 有两个：CLK_APBC_CAN0 和 CLK_APBC_CAN0_BUS，分别对应 func clk 和 bus clk，前缀 CLK_APBC_ 对应的 Provider 为 syscon_apbc，所以其DTS 配置如下：

```dts
    flexcan0: fdcan@d4028000 {
            compatible = "spacemit,k1-flexcan";
            reg = <0x0 0xd4028000 0x0 0x4000>;
            interrupts = <161 IRQ_TYPE_LEVEL_HIGH>;
            interrupt-parent = <&saplic>;
            fsl,clk-source = <0>;
            clocks = <&syscon_apbc CLK_APBC_CAN0>,<&syscon_apbc CLK_APBC_CAN0_BUS>;
            clock-names = "per","ipg";
            resets = <&syscon_apbc RESET_APBC_CAN0>;
            status = "disabled";
    };
```

- 加头文件和 `clk` 结构体
在驱动代码中添加头文件和结构体定义：
```c
#include <linux/clk.h>
```

```c
struct flexcan_priv {

        struct clk *clk_ipg;
        struct clk *clk_per;
};
```

- 获取时钟
一般在驱动 probe 阶段通过 `devm_clk_get` 获取时钟句柄，当驱动 probe 失败或者 remove 时，驱动自动释放对应的时钟句柄

```c
        clk_ipg = devm_clk_get(&pdev->dev, "ipg");               // 获取总线时钟CLK_APBC_CAN0_BUS对应的时钟句柄
        if (IS_ERR(clk_ipg)) {
                dev_err(&pdev->dev, "no ipg clock defined\n");
                return PTR_ERR(clk_ipg);
        }

        clk_per = devm_clk_get(&pdev->dev, "per");               // 获取工作时钟CLK_APBC_CAN0对应的时钟句柄
        if (IS_ERR(clk_per)) {
                dev_err(&pdev->dev, "no per clock defined\n");
                return PTR_ERR(clk_per);
        }

```

- 使能时钟
通过 `clk_prepare_enable` 使能时钟节点

```c
        if (priv->clk_ipg) {
                err = clk_prepare_enable(priv->clk_ipg);         // 使能总线时钟CLK_APBC_CAN0_BUS
                if (err)
                        return err;
        }

        if (priv->clk_per) {
                err = clk_prepare_enable(priv->clk_per);         // 使能工作时钟CLK_APBC_CAN0
                if (err)
                        clk_disable_unprepare(priv->clk_ipg);
        }

```

- 获取时钟频率
通过 `clk_get_rate` 获取时钟频率

```c
clock_freq = clk_get_rate(clk_per);                  // 获取工作时钟CLK_APBC_CAN0当前频率
```

- 设置时钟频率
通过 `clk_set_rate` 修改时钟频率，第一个参数是时钟句柄 struct clk*，第二个参数是目标频率

```c
clk_set_rate(clk_per, clock_freq);                   // 设置工作时钟CLK_APBC_CAN0频率
```

- 关闭时钟
通过 `clk_disable_unprepare` 关闭时钟

```c
clk_disable_unprepare(priv->clk_per);                // 关闭工作时钟CLK_APBC_CAN0
clk_disable_unprepare(priv->clk_ipg);                // 关闭总线时钟CLK_APBC_CAN0_BUS
```

## 调试介绍

Clock 可以通过 debugfs 进行调试。

### clk_summary
`/sys/kernel/debug/clk/clk_summary` 常用于打印完整时钟树，查看各个时钟节点的状态，频率，父时钟等信息。

```
root# cat /sys/kernel/debug/clk/clk_summary
```

### 时钟节点信息
还可以单独查看具体时钟节点的状态，频率，父时钟等信息。以 `can0_clk` 为例：

```
root:/sys/kernel/debug/clk/can0_clk # ls -l
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_accuracy
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_duty_cycle
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_enable_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_flags
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_max_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_min_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_notifier_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_parent
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_phase
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_possible_parents
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_enable
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_protect_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_rate
root:/sys/kernel/debug/clk/can0_clk# cat clk_prepare_count          # 查看enable状态
0
root:/sys/kernel/debug/clk/can0_clk# cat clk_rate                   # 查看当前频率
20000000
root:/sys/kernel/debug/clk/can0_clk# cat clk_parent                 # 查看当前父时钟
pll3_20
root:/sys/kernel/debug/clk/can0_clk#
```

### 动态操作时钟节点

在 `drivers/clk/clk.c` 中加上 `CLOCK_ALLOW_WRITE_DEBUGFS` 宏定义，就可以对 debugfs 下的一些 clk 节点进行写操作，否则只有读操作权限。

```
/sys/kernel/debug/clk/can0_clk # ls -l
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_accuracy
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_duty_cycle
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_enable_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_flags
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_max_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_min_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_notifier_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_parent              # 可读可写
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_phase
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_possible_parents
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_enable      # 可读可写
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_protect_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_rate                # 可读可写
/sys/kernel/debug/clk/can0_clk # cat clk_rate                                    # 查看频率
20000000
/sys/kernel/debug/clk/can0_clk # echo 40000000 > clk_rate                        # 设置频率为40MHz
/sys/kernel/debug/clk/can0_clk # cat clk_rate                                    # 确认设置结果
40000000
/sys/kernel/debug/clk/can0_clk # cat clk_parent                                  # 查看父时钟
pll3_40
/sys/kernel/debug/clk/can0_clk # echo 0 > clk_parent                             # 设置父时钟为index为0的时钟源
/sys/kernel/debug/clk/can0_clk # cat clk_parent                                  # 确认设置结果
pll3_20
/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # 查看prepare_enable状态
0
/sys/kernel/debug/clk/can0_clk # echo 1 > clk_prepare_enable                     # prepare并enable时钟节点
/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # 确认设置结果
1
/sys/kernel/debug/clk/can0_clk # echo 0 > clk_prepare_enable                     # unprepare并disable时钟节点

/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # 确认设置结果
0
/sys/kernel/debug/clk/can0_clk #
```

## FAQ

### 1.模块不工作，应该如何排查？

- DTS 配置是否正确：`clocks` 是否配置，Provider 和 ID 是否正确；`clock-names` 顺序对不对；
- 驱动里实际 `devm_clk_get()` 请求的名字是什么，是否有报错，例如 `failed to get clk` / `failed to enable clk`；
- 时钟树是否正常：通过 debugfs 看时钟树，确认时钟是否 enable，父时钟是谁，频率是不是预期的值；
- 驱动是否正确释放复位；
- 复位与时钟的时序是否正确；
- 是否还有其他依赖没满足，pinctrl / power-domain / regulator 等；

### 2.为什么有些模块挂 `syscon_apbc`，有些挂 `syscon_apmu`？

因为它们处于不同时钟/电源/总线域。通常来说：

- 低速 APB 外设更多挂在 `syscon_apbc`；
- 高速或更复杂模块更多挂在 `syscon_apmu`；
- R-domain 模块则更多挂在 `syscon_rcpu_*` 系列 provider。

可参照 `使用介绍` 章节的对应方法。
