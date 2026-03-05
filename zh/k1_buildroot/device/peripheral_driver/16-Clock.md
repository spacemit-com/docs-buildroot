# Clock

介绍 Clock 的功能和使用方法。

## 模块介绍

Clock 是系统中的时钟控制器模块，负责时钟的管理与分发。

### 功能介绍

![](static/CLOCK.png)

Linux为了做好时钟管理，提供了一个时钟管理框架 Common Clock Framework（以下简称CCF），为设备驱动提供统一的操作接口，使设备驱动不必关心时钟硬件实现的具体细节。  
CCF 框架包括以下核心组成部分：
**clock provider**：对应上图的右侧部分，即 clock controller，负责提供系统所需的各种时钟。  
**clock consumer**：对应上图的左侧部分，即使用时钟的一些设备驱动。  
**clock framework**：CCF 的核心部分，向 clock consumers 提供操作 clock 的通用 API；实现时钟管理的核心逻辑，将与硬件相关的 clock 控制逻辑封装成操作函数集，交由 clock provider 实现。  
**device tree**：CCF 允许在设备树中声明可用的时钟与设备的关联。  

Clock 系统相关的器件包括：

- Oscillator / Crystal：有源或无源晶振，作为根时钟源。
- PLL（Phase Locked Loop）：锁相环倍频，提高基础频率。
- Divider：分频器，降低频率。
- MUX：多路选择器，切换时钟源。
- GATE：时钟开关，控制时钟通断。

系统中可能存在很多个这样的硬件模块，呈树形结构，Linux 将他们管理成一个时钟树（clock-tree），系统中的时钟树（Clock Tree）通常以晶振（Oscillator/Crystal）为起点，逐层经由 PLL（倍频）、MUX（选择）、DIV（分频）及 GATE（控制）等节点，最终输出给设备使用。CCF 实现了多种基础时钟类型，例如:

- 固定速率时钟 fixed_rate clock
- 门控时钟 gate clock
- 分频器时钟 divider clock
- 复用器时钟 mux clock

一般为了方便使用，会根据时钟树设计，实现一些时钟类型。

### 源码结构介绍

#### Clock 控制器驱动源码

Clock 控制器驱动代码在 `drivers/clk/spacemit` 目录下：

```
drivers/clk/spacemit
|-- ccu_ddn.c                   #ddn时钟类型源码
|-- ccu_ddn.h
|-- ccu_ddr.c                   #ddr时钟类型源码
|-- ccu_ddr.h
|-- ccu_dpll.c                  #dpll时钟类型源码
|-- ccu_dpll.h
|-- ccu_mix.c                   #mix时钟类型源码
|-- ccu_mix.h
|-- ccu_pll.c                   #pll时钟类型源码
|-- ccu_pll.h
|-- ccu-spacemit-k1x.c          #k1 clock controller驱动
|-- ccu-spacemit-k1x.h
|-- Kconfig
|-- Makefile
```

Clock 控制器驱动实现了五种类型的时钟模块：

- PLL 类型，锁相环类型
- DPLL 类型，DDR 相关的锁相环类型
- DDN 类型，分数 divider，有一级除频，对应分母，一级倍频，对应分子
- MIX 类型，混合类型，支持 gate/mux/divider 的任一种或者随意组合
- DDR 类型，DDR 相关的特殊时钟类型  

#### 各时钟index定义

各时钟 index 定义在 `dt-bindings` 下：

```
include/dt-bindings/clock/spacemit-k1x-clock.h
```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS配置**

### CONFIG 配置

- `CONFIG_COMMON_CLK`：为 Common Clock Framework 提供支持，默认情况下，此选项为 `Y`

```
Device Drivers
 Common Clock Framework (COMMON_CLK[=y])
```

- `CONFIG_SPACEMIT_K1X_CCU`：为 K1 Clock 控制器驱动提供支持，默认情况下，此选型为 `Y`

```
 Device Drivers
 Common Clock Framework (COMMON_CLK[=y])
         Clock support for Spacemit k1x SoCs (SPACEMIT_K1X_CCU [=y])
```

### DTS 配置

clock controller 的 DTS 配置如下：

```
/ {
        clocks {
                #address-cells = <0x2>;
                #size-cells = <0x2>;
                ranges;

                vctcxo_24: clock-vctcxo_24 {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <24000000>;
                        clock-output-names = "vctcxo_24";
                };
                vctcxo_3: clock-vctcxo_3 {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <3000000>;
                        clock-output-names = "vctcxo_3";
                };
                vctcxo_1: clock-vctcxo_1 {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <1000000>;
                        clock-output-names = "vctcxo_1";
                };
                pll1_2457p6_vco: clock-pll1_2457p6_vco {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <2457600000>;
                        clock-output-names = "pll1_2457p6_vco";
                };
                clk_32k: clock-clk32k {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <32000>;
                        clock-output-names = "clk_32k";
                };

                pll_clk_cluster0: clock-pll_clk_cluster0 {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <10000000>;
                        clock-output-names = "pll_clk_cluster0";
                };

                pll_clk_cluster1: clock-pll_clk_cluster1 {
                        #clock-cells = <0>;
                        compatible = "fixed-clock";
                        clock-frequency = <10000000>;
                        clock-output-names = "pll_clk_cluster1";
                };
        };

        soc: soc {
                ...
                ccu: clock-controller@d4050000 {
                        compatible = "spacemit,k1x-clock";
                        reg = <0x0 0xd4050000 0x0 0x209c>,
                                <0x0 0xd4282800 0x0 0x400>,
                                <0x0 0xd4015000 0x0 0x1000>,
                                <0x0 0xd4090000 0x0 0x1000>,
                                <0x0 0xd4282c00 0x0 0x400>,
                                <0x0 0xd8440000 0x0 0x98>,
                                <0x0 0xc0000000 0x0 0x4280>,
                                <0x0 0xf0610000 0x0 0x20>,
                                <0x0 0xc0880000 0x0 0x2050>,
                                <0x0 0xc0888000 0x0 0x30>;
                        reg-names = "mpmu", "apmu", "apbc", "apbs", "ciu", "dciu", "ddrc", "apbc2", "rcpu", "rcpu2";
                        clocks = <&vctcxo_24>, <&vctcxo_3>, <&vctcxo_1>, <&pll1_2457p6_vco>,
                                <&clk_32k>;
                        clock-names = "vctcxo_24", "vctcxo_3", "vctcxo_1", "pll1_2457p6_vco",
                                "clk_32k";
                        #clock-cells = <1>;
                        status = "okay";
                };
                ...
        };
};


```

## 接口描述

### API 介绍

CCF 为设备驱动提供了通用的时钟操作的接口：

- `get`：获取时钟句柄

```c
/*
* clk_get - get clk
* @dev: device
* @id: clock name of dts "clock-names"
*/
struct clk *clk_get(struct device *dev, const char *id);

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
* @dev: device
* @id: clock name of dts "clock-names"
*/
void clk_put(struct clk *clk);

/*
* devm_clk_put - put clk
* @dev: device
* @id: clock name of dts "clock-names"
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
 * This undoes a previously prepared clock.  The caller must balance
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

`clk_prepare_enable` 是 `clk_prepare` 和 `clk_enable` 的组合，`clk_disable_unprepare`是 `clk_unprepare` 和 `clk_disable` 的组合，推荐使用这两个接口

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
 *                This is only valid once the clock source has been enabled.
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
 * in any way.  In other words:
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

### 使用示例

模块如要使用 clock 功能，需要在 DTS 配置 clocks 和 clock-names 属性，然后在驱动中通过CCF API 进行 Clock 相关的操作。

- 配置 DTS 
在 `include/dt-bindings/clock/spacemit-k1x-clock.h` 找到对应的时钟 index，配置到模块 DTS 中。
以 `can0` 为例，`can0` 有两个时钟，一个是模块工作时钟 `CLK_CAN0`，另一个是总线时钟`CLK_CAN0_BUS`。DTS 配置如下：

```
                flexcan0: fdcan@d4028000 {
                        compatible = "spacemit,k1x-flexcan";
                        reg = <0x0 0xd4028000 0x0 0x4000>;
                        interrupts = <16>;
                        interrupt-parent = <&intc>;
                        clocks = <&ccu CLK_CAN0>,<&ccu CLK_CAN0_BUS>; #配置can0时钟的index
                        clock-names = "per","ipg";                    #配置clocks对应的名称，驱动里可以通过这个字符串获取对应时钟
                        resets = <&reset RESET_CAN0>;
                        fsl,clk-source = <0>;
                        status = "disabled";
                };

```

- 加头文件和 `clk` 句柄  

```
#include <linux/clk.h>
```

```
struct flexcan_priv {

        struct clk *clk_ipg;
        struct clk *clk_per;
};
```

- 获取时钟  
一般在驱动 probe 阶段通过 `devm_clk_get` 获取时钟句柄，当驱动 probe 失败或者 remove 时，驱动自动释放对应的时钟句柄

```
        clk_ipg = devm_clk_get(&pdev->dev, "ipg");               #获取总线时钟CLK_CAN0_BUS对应的时钟句柄
        if (IS_ERR(clk_ipg)) {
                dev_err(&pdev->dev, "no ipg clock defined\n");
                return PTR_ERR(clk_ipg);
        }

        clk_per = devm_clk_get(&pdev->dev, "per");               #或取工作时钟CLK_CAN0对应的时钟句柄
        if (IS_ERR(clk_per)) {
                dev_err(&pdev->dev, "no per clock defined\n");
                return PTR_ERR(clk_per);
        }

```

- 使能时钟  
通过 `clk_prepare_enable` 使能时钟节点

```
        if (priv->clk_ipg) {
                err = clk_prepare_enable(priv->clk_ipg);         #使能总线时钟CLK_CAN0_BUS
                if (err)
                        return err;
        }

        if (priv->clk_per) {
                err = clk_prepare_enable(priv->clk_per);         #使能工作时钟CLK_CAN0
                if (err)
                        clk_disable_unprepare(priv->clk_ipg);
        }

```

- 获取时钟频率  
通过 `clk_get_rate` 获取时钟频率

```
clock_freq = clk_get_rate(clk_per);                  #获取工作时钟CLK_CAN0当前频率
```

- 设置时钟频率  
通过 `clk_set_rate` 修改时钟频率，第一个参数是时钟句柄 struct clk*，第二个参数是目标频率

```
clk_set_rate(clk_per, clock_freq);                   #设置工作时钟CLK_CAN0频率
```

- 关闭时钟  
通过 `clk_disable_unprepare` 关闭时钟

```
clk_disable_unprepare(priv->clk_per);                #关闭工作时钟CLK_CAN0
clk_disable_unprepare(priv->clk_ipg);                #关闭总线时钟CLK_CAN0_BUS
```

## Debug 介绍

可以通过 debugfs 进行调试

- 打印时钟树  
`/sys/kernel/debug/clk/clk_summary` 常用于打印时钟树结构，查看各个时钟节点的状态，频率，父时钟等信息。

```
root# cat /sys/kernel/debug/clk/clk_summary
```

- 查看具体时钟节点  
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
root:/sys/kernel/debug/clk/can0_clk# cat clk_prepare_count          #查看enable状态
0
root:/sys/kernel/debug/clk/can0_clk# cat clk_rate                   #查看当前频率
20000000
root:/sys/kernel/debug/clk/can0_clk# cat clk_parent                 #查看当前父时钟
pll3_20
root:/sys/kernel/debug/clk/can0_clk#
```

- 改变时钟配置  
在 `driver/clk/clk.c中加上CLOCK_ALLOW_WRITE_DEBUGFS` 宏定义，就可以对 debugfs 下的一些 clk 节点进行写操作，否则只有读操作权限

```
/sys/kernel/debug/clk/can0_clk # ls -l
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_accuracy
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_duty_cycle
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_enable_count
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_flags
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_max_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_min_rate
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_notifier_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_parent              #可读可写
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_phase
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_possible_parents
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_enable      #可读可写
-r--r--r--    1 root     root             0 Jan  1 08:03 clk_protect_count
-rw-r--r--    1 root     root             0 Jan  1 08:03 clk_rate                #可读可写
/sys/kernel/debug/clk/can0_clk # cat clk_rate                                    #查看频率
20000000
/sys/kernel/debug/clk/can0_clk # echo 40000000 > clk_rate                        #设置频率为40MHz
/sys/kernel/debug/clk/can0_clk # cat clk_rate                                    #确认设置结果
40000000
/sys/kernel/debug/clk/can0_clk # cat clk_parent                                  #查看父时钟
pll3_40
/sys/kernel/debug/clk/can0_clk # echo 0 > clk_parent                             #设置父时钟为index为0的时钟源
/sys/kernel/debug/clk/can0_clk # cat clk_parent                                  #确认设置结果
pll3_20
/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          #查看prepare_enable状态
0
/sys/kernel/debug/clk/can0_clk # echo 1 > clk_prepare_enable                     #prepare并enable时钟节点
/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          #确认设置结果
1
/sys/kernel/debug/clk/can0_clk # echo 0 > clk_prepare_enable                     #unprepare并disable时钟节点
/sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          #确认设置结果
0
/sys/kernel/debug/clk/can0_clk #
```

## FAQ
