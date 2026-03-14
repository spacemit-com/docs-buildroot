# PINCTRL

介绍 **K3 PINCTRL** 的功能和使用方法。

## 模块介绍

PINCTRL 是 **PIN 模块的控制器**，负责完成 K3 SoC 上各个管脚的复用选择、电气属性配置以及部分唤醒中断相关能力。

### 功能介绍

Linux pinctrl 模块包括两部分：**pinctrl core** 和 **pin 控制器驱动**。

1. **pinctrl core** 主要功能：
   - 提供 pinctrl 功能接口给其它驱动使用
   - 提供 pin 控制器设备注册与注销接口
   - 解析 dts 中的 pinctrl state，并在设备驱动中按 state 进行切换

2. **K3 pinctrl 控制器驱动** 主要功能：
   - 驱动 K3 pin 控制器硬件
   - 实现 pin 复用功能配置
   - 实现 pin 电气属性配置，如上下拉、驱动能力、施密特输入等
   - 对外部电压域 pin 配置 IO 电源电压
   - 提供基于 pin 的边沿唤醒中断支持

### 源码结构介绍

K3 pinctrl 驱动代码位于内核目录：

```text
drivers/pinctrl/spacemit/
|-- Kconfig
|-- Makefile
|-- pinctrl-k1.c
`-- pinctrl-k1.h
```

虽然源文件名仍然为 `pinctrl-k1.c`，但当前驱动实际同时支持 **K1/K3**，K3 的 dts `compatible` 为：

```dts
compatible = "spacemit,k3-pinctrl";
```

K3 默认 pin 配置定义位于：

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

K3 pinctrl 控制器节点位于：

```text
arch/riscv/boot/dts/spacemit/k3.dtsi
```

对应节点示例：

```dts
pinctrl: pinctrl@d401e000 {
    compatible = "spacemit,k3-pinctrl";
    reg = <0x0 0xd401e000 0x0 0x400>,
          <0x0 0xd401e800 0x0 0x34>;
    clocks = <&syscon_apbc CLK_APBC_AIB>,
             <&syscon_apbc CLK_APBC_AIB_BUS>;
    clock-names = "func", "bus";
    interrupt-controller;
    #interrupt-cells = <2>;
    interrupts = <60 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    spacemit,gpio-edge = <&syscon_gpio_edge>;
    spacemit,apbc = <&syscon_apbc 0x50>;
    spacemit,apmu = <&syscon_apmu>;
};
```

## 关键特性

| 特性 | 特性说明 |
| :----- | :---- |
| 支持 pin 复用选择 | 支持将 pin 设置为某一种复用功能 |
| 支持 pin 电气属性配置 | 支持上下拉、强上拉、驱动能力、施密特输入等 |
| 支持外部 IO 电压域 | 对 External 类型 pin，可通过 `power-source` 指定 1.8V 或 3.3V |
| 支持边沿唤醒中断 | 支持 rise / fall / both 方式的 pin 唤醒检测 |
| 支持分组配置 | 支持在一个 pin group 中统一设置多个 pin 的 mux 和属性 |
| 支持多 state 切换 | 支持 `default`、`uhs`、`debug` 等多个 pinctrl state |

## 配置介绍

主要包括 **驱动使能配置** 和 **dts 配置**。

### CONFIG 配置

- **CONFIG_PINCTRL**：为 pin 控制器提供通用支持，默认值为 `Y`
- **CONFIG_PINCTRL_SPACEMIT_K1**：为 SpacemiT K1/K3 pinctrl 控制器提供支持

Kconfig 中对应定义如下：

```text
config PINCTRL_SPACEMIT_K1
    bool "SpacemiT K1/K3 SoC Pinctrl driver"
    depends on SOC_SPACEMIT || COMPILE_TEST
    depends on OF
    default ARCH_SPACEMIT
```

## pin 使用说明

介绍在 dts 设备节点中使用 K3 pinctrl 的方式。

### pin 配置参数

K3 pinctrl 的配置由三部分组成：

- **pin id**
- **复用功能**
- **属性配置**

与 K1 不同，K3 在 dts 中不再使用 `K1X_PADCONF(pin, mux, config)` 这种将 mux 和电气配置全部编码进一个宏的方式，而是采用：

- `pinmux` 只编码 **pin id + mux mode**
- 上下拉、驱动能力、电压等使用 **标准 pinconf 属性** 单独描述

#### pinmux 编码宏

K3 默认使用如下宏定义：

```c
#define K3_PADCONF(pin, func) (((pin) << 16) | (func))
```

含义如下：

- 高 16 位：pin 编号
- 低 16 位：mux function 编号

驱动中对应解析方式：

- `pin = value >> 16`
- `mux = value & GENMASK(15, 0)`

### pin id

K3 pin 编号范围为 **0 ~ 163**，共 **164 个 pin**。

驱动中的 K3 pin 名称定义位于 `pinctrl-k1.c` 的 `k3_pin_desc[]`。

典型 pin 名称包括：

- `GPIO_00 ~ GPIO_127`
- `PWR_SCL`
- `PWR_SDA`
- `VCXO_EN`
- `PMIC_INT_N`
- `MMC1_DAT3 ~ MMC1_CLK`
- `QSPI_DAT0 ~ QSPI_CLK`
- `PRI_TDI ~ PWR_SSP_RXD`
- `EMMC_D0 ~ EMMC_CMD`

### pin 电气类型

驱动中为每个 pin 定义了 IO 类型，不同类型决定该 pin 的驱动能力映射和是否需要 `power-source`。

支持的类型如下：

- `IO_TYPE_1V8`
- `IO_TYPE_3V3`
- `IO_TYPE_EXTERNAL`

其中 K3 实际使用的主要分布如下：

| 管脚范围 | 名称/Bank | IO 类型 |
| :-- | :-- | :-- |
| 0 ~ 20 | GPIO1 | EXTERNAL |
| 21 ~ 41 | GPIO2 | EXTERNAL |
| 42 ~ 75 | GPIO3 | 1.8V |
| 76 ~ 98 | GPIO4 | EXTERNAL |
| 99 ~ 127 | GPIO5 | EXTERNAL |
| 128 ~ 131 | PMIC/PWR | 1.8V |
| 132 ~ 137 | MMC1 | EXTERNAL |
| 138 ~ 144 | QSPI | EXTERNAL |
| 145 ~ 152 | JTAG / PWR_SSP | 1.8V |
| 153 ~ 163 | EMMC | 特殊（仅 eMMC/GPIO 两种功能） |

> 说明：对 `IO_TYPE_EXTERNAL` 类型 pin，如果 dts 中配置了 `drive-strength`，则必须同时提供 `power-source = <1800>` 或 `power-source = <3300>`，否则驱动会判定配置错误。

### pin 功能

K3 pin 支持复用功能选择，mux 功能号直接写在 `K3_PADCONF(pin, func)` 中。

例如：

```dts
pinmux = <K3_PADCONF(149, 2)>, <K3_PADCONF(150, 2)>;
```

表示：

- pin 149 选择 mux function 2
- pin 150 选择 mux function 2

驱动中实际将 mux 值写入 pad 寄存器的 `mux mode` 位域。

### pin 属性

K3 通过标准 pinconf 属性描述 pin 的电气配置。实际驱动代码和 dts 用法表明，K3 当前主要支持以下属性。

#### 上下拉

支持如下配置：

- `bias-disable;`
- `bias-pull-down;`
- `bias-pull-up;`

其中：

- `bias-disable`：关闭上下拉
- `bias-pull-down`：使能下拉
- `bias-pull-up`：使能上拉

驱动内部还支持 **强上拉** 概念：

- `PIN_CONFIG_BIAS_PULL_UP` 参数值为 `0`：normal pull-up
- `PIN_CONFIG_BIAS_PULL_UP` 参数值为 `1`：strong pull-up

默认 dts 示例中常见的是普通上拉。

#### 驱动能力

通过属性：

```dts
drive-strength = <mA>;
```

其中参数单位为 **mA**，不是 DS 编号。驱动会根据 pin 的电压类型，将 mA 自动映射到对应寄存器值。

##### 1.8V pin 的驱动能力映射

K3 1.8V 驱动能力支持 16 档：

| 寄存器值 | 电流(mA) |
| :-- | :-- |
| 0 | 2 |
| 1 | 4 |
| 2 | 6 |
| 3 | 7 |
| 4 | 9 |
| 5 | 11 |
| 6 | 13 |
| 7 | 14 |
| 8 | 21 |
| 9 | 23 |
| 10 | 25 |
| 11 | 26 |
| 12 | 28 |
| 13 | 30 |
| 14 | 31 |
| 15 | 33 |

##### 3.3V pin 的驱动能力映射

K3 3.3V 驱动能力支持 16 档：

| 寄存器值 | 电流(mA) |
| :-- | :-- |
| 0 | 3 |
| 1 | 5 |
| 2 | 7 |
| 3 | 9 |
| 4 | 11 |
| 5 | 13 |
| 6 | 15 |
| 7 | 17 |
| 8 | 25 |
| 9 | 27 |
| 10 | 29 |
| 11 | 31 |
| 12 | 33 |
| 13 | 35 |
| 14 | 37 |
| 15 | 38 |

驱动使用“**不小于目标值的最小档位**”进行匹配。

例如：

- `drive-strength = <25>;` 会映射到 25mA 对应档位
- `drive-strength = <8>;` 对 3.3V pin 会映射到不小于 8mA 的最小档位，即 9mA 档

#### 输入施密特触发

驱动支持：

```dts
input-schmitt = <value>;
```

K3 中对应寄存器位宽为 1 bit，驱动会将该属性写入 `schmitt` 位域。

#### 电压源选择

对 `IO_TYPE_EXTERNAL` pin，可使用：

```dts
power-source = <1800>;
```

或：

```dts
power-source = <3300>;
```

作用：

- 指定该 pin group 使用 1.8V 还是 3.3V IO 电压
- 让驱动按正确的驱动能力表计算 `drive-strength`
- 同时驱动会写入对应的 IO power domain 配置寄存器

如果 external pin group 使用了 `drive-strength` 但未提供合法的 `power-source`，驱动会报错并拒绝该配置。

#### 边沿检测

驱动底层硬件支持：

- 上升沿检测
- 下降沿检测
- 双边沿检测
- 清除边沿状态

这部分主要用于 **wake irq** 功能，不是在 `k3-pinctrl.dtsi` 中通过通用 pinconf 属性普遍配置，而是由 pinctrl 控制器中断域和 irq type 设置流程完成。

### pin 配置定义

#### 单个 pin 配置

K3 的单个 pin 通常通过 `pinmux` + 独立属性方式配置。

示例：将 pin 149、150 配置为 uart0 功能，并设置普通上拉与 25mA 驱动能力。

```dts
uart0_0_cfg: uart0-0-cfg {
    uart0-0-pins {
        pinmux = <K3_PADCONF(149, 2)>,    /* uart0 tx */
                 <K3_PADCONF(150, 2)>;    /* uart0 rx */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

#### 定义一组 pin

K3 默认的功能 pin 组位于：

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

与 K1 相比，K3 的默认 pin 组有两个明显特点：

1. **命名统一采用 `xxx_cfg`**，如 `uart0_0_cfg`、`gmac0_cfg`、`mmc1_cfg`
2. **一个 state 下允许包含多个 child group**，每个 child group 可以有独立的 `pinmux` 和电气属性

例如 `mmc1_cfg` 下，数据线/命令线与时钟线分别放在两个子组中：

```dts
mmc1_cfg: mmc1-cfg {
    mmc1-0-pins {
        pinmux = <K3_PADCONF(132, 0)>,    /* mmc1 dat3 */
                 <K3_PADCONF(133, 0)>,    /* mmc1 dat2 */
                 <K3_PADCONF(134, 0)>,    /* mmc1 dat1 */
                 <K3_PADCONF(135, 0)>,    /* mmc1 dat0 */
                 <K3_PADCONF(136, 0)>;    /* mmc1 cmd */

        bias-pull-up;
        drive-strength = <8>;
        power-source = <3300>;
    };

    mmc1-1-pins {
        pinmux = <K3_PADCONF(137, 0)>;    /* mmc1 clk */

        bias-pull-down;
        drive-strength = <8>;
        power-source = <3300>;
    };
};
```

#### 复写默认 pin 组

如果 `k3-pinctrl.dtsi` 中已有默认定义，但不满足板级需求，可以在板级 dts 中：

1. 直接重写已有 `xxx_cfg` 节点；或者
2. 新增一个新的 `xxx_cfg` 节点，再在设备节点中引用新的 state

建议优先采用 **新增 state** 的方式，便于和原始 SoC dtsi 中的默认配置区分。

### pin 使用示例

#### UART 使用示例

`k3-pinctrl.dtsi` 中定义：

```dts
uart2_0_cfg: uart2-0-cfg {
    uart2-0-pins {
        pinmux = <K3_PADCONF(134, 2)>,    /* uart2 tx */
                 <K3_PADCONF(135, 2)>,    /* uart2 rx */
                 <K3_PADCONF(136, 2)>,    /* uart2 cts */
                 <K3_PADCONF(137, 2)>;    /* uart2 rts */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

设备节点引用：

```dts
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_0_cfg>;
    status = "disabled";
};
```

#### SD 卡多 state 示例

K3 SD 卡控制器定义了 3 组 pinctrl state：

- `default`
- `uhs`
- `debug`

其中：

- `default`：普通 3.3V 工作模式
- `uhs`：UHS/1.8V 工作模式
- `debug`：复用部分 MMC1 pin 给 uart0 调试功能

对应 dts 定义如下：

```dts
mmc1_cfg: mmc1-cfg {
    mmc1-0-pins {
        pinmux = <K3_PADCONF(132, 0)>, <K3_PADCONF(133, 0)>,
                 <K3_PADCONF(134, 0)>, <K3_PADCONF(135, 0)>,
                 <K3_PADCONF(136, 0)>;
        bias-pull-up;
        drive-strength = <8>;
        power-source = <3300>;
    };

    mmc1-1-pins {
        pinmux = <K3_PADCONF(137, 0)>;
        bias-pull-down;
        drive-strength = <8>;
        power-source = <3300>;
    };
};

mmc1_uhs_cfg: mmc1-uhs-cfg {
    mmc1-0-pins {
        pinmux = <K3_PADCONF(132, 0)>, <K3_PADCONF(133, 0)>,
                 <K3_PADCONF(134, 0)>, <K3_PADCONF(135, 0)>,
                 <K3_PADCONF(136, 0)>;
        bias-pull-up;
        drive-strength = <7>;
        power-source = <1800>;
    };

    mmc1-1-pins {
        pinmux = <K3_PADCONF(137, 0)>;
        bias-pull-down;
        drive-strength = <6>;
        power-source = <1800>;
    };
};

mmc1_debug_cfg: mmc1-debug-cfg {
    mmc1-0-pins {
        pinmux = <K3_PADCONF(132, 2)>,    /* uart0 tx */
                 <K3_PADCONF(133, 2)>,    /* uart0 rx */
                 <K3_PADCONF(134, 0)>,
                 <K3_PADCONF(135, 0)>,
                 <K3_PADCONF(136, 0)>;
        bias-pull-up;
        drive-strength = <8>;
        power-source = <3300>;
    };

    mmc1-1-pins {
        pinmux = <K3_PADCONF(137, 0)>;
        bias-pull-down;
        drive-strength = <8>;
        power-source = <3300>;
    };
};
```

设备节点引用如下：

```dts
&sdcard {
    pinctrl-names = "default", "uhs", "debug";
    pinctrl-0 = <&mmc1_cfg>;
    pinctrl-1 = <&mmc1_uhs_cfg>;
    pinctrl-2 = <&mmc1_debug_cfg>;
    bus-width = <4>;
    status = "disabled";
};
```

#### GMAC 使用示例

K3 GMAC 的 pin 组定义较细，按 base pin、RGMII 扩展 pin、MII 扩展 pin、可选中断 pin、可选 PPS pin、可选 refclk pin 分组。

例如 `gmac0_cfg`：

```dts
gmac0_cfg: gmac0-cfg {
    gmac0_base_pins: gmac0-0-pins {
        pinmux = <K3_PADCONF(0, 1)>, <K3_PADCONF(1, 1)>,
                 <K3_PADCONF(2, 1)>, <K3_PADCONF(3, 1)>,
                 <K3_PADCONF(6, 1)>, <K3_PADCONF(7, 1)>,
                 <K3_PADCONF(11, 1)>, <K3_PADCONF(12, 1)>,
                 <K3_PADCONF(13, 1)>;
        bias-disable;
        drive-strength = <25>;
    };

    gmac0_rgmii_add_pins: gmac0-1-pins {
        pinmux = <K3_PADCONF(4, 1)>, <K3_PADCONF(5, 1)>,
                 <K3_PADCONF(8, 1)>, <K3_PADCONF(9, 1)>,
                 <K3_PADCONF(10, 1)>;
        bias-disable;
        drive-strength = <25>;
    };

    gmac0_int_pins: gmac0-3-pins {
        pinmux = <K3_PADCONF(14, 1)>;
        bias-disable;
        drive-strength = <25>;
    };
};
```

设备侧引用：

```dts
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&gmac0_cfg>;
    phy-mode = "rgmii";
    status = "okay";
};
```

#### PCIe 使用示例

K3 PCIe 的 PERST#/WAKE#/CLKREQ# 等控制 pin 也通过 pinctrl 配置，例如：

```dts
pcie0_1_cfg: pcie0-1-cfg {
    pcie0-0-pins {
        pinmux = <K3_PADCONF(79, 5)>,    /* pcie0 perst */
                 <K3_PADCONF(80, 5)>,    /* pcie0 wake */
                 <K3_PADCONF(81, 5)>;    /* pcie0 clkreq */

        bias-disable;
        drive-strength = <25>;
        power-source = <3300>;
    };
};
```

设备节点引用：

```dts
&pcie0_rc {
    pinctrl-names = "default";
    pinctrl-0 = <&pcie0_1_cfg>;
    status = "okay";
};
```

#### USB Host 驱动脚示例

例如 USB2 host drive pin：

```dts
usb20_host_drv_3_cfg: usb20_host_drv-3-cfg {
    usb20_host_drv-pins {
        pinmux = <K3_PADCONF(103, 3)>;
        bias-disable;
        drive-strength = <25>;
    };
};
```

在板级 dts 中引用：

```dts
&usb3_portc {
    pinctrl-names = "default";
    pinctrl-0 = <&usb20_host_drv_3_cfg>;
};
```

## K3 pinctrl dts 写法说明

### 基本结构

K3 pinctrl state 一般放在 `&pinctrl` 节点下，结构通常如下：

```dts
&pinctrl {
    xxx_cfg: xxx-cfg {
        xxx-pins {
            pinmux = <K3_PADCONF(pin, func)>, ...;
            bias-pull-up;
            drive-strength = <25>;
            power-source = <1800>;
        };
    };
};
```

其含义为：

- `xxx_cfg`：供外设节点引用的 pinctrl state 名称
- `xxx-cfg`：dts 节点名
- `xxx-pins`：该 state 下的一个子 group
- `pinmux`：该组包含的 pin 与 mux 设置
- 其它标准 pinconf 属性：该组统一的电气属性

### 多子组结构

当同一个 state 中不同 pin 需要不同电气属性时，可以拆成多个 `xxx-pins` 子节点。

例如 MMC1：

- 数据线/命令线使用 `bias-pull-up`
- 时钟线使用 `bias-pull-down`

因此拆成 `mmc1-0-pins` 和 `mmc1-1-pins` 两组。

### 设备节点引用方式

外设节点通过标准 pinctrl 方式引用：

```dts
pinctrl-names = "default";
pinctrl-0 = <&xxx_cfg>;
```

多 state 情况下：

```dts
pinctrl-names = "default", "uhs", "debug";
pinctrl-0 = <&state0_cfg>;
pinctrl-1 = <&state1_cfg>;
pinctrl-2 = <&state2_cfg>;
```

## 调试说明

### debugfs 查看 pinctrl 信息

使能 debugfs 后，可通过如下路径查看 pinctrl 配置：

```text
/sys/kernel/debug/pinctrl/
```

可用于查看：

- pin group 注册情况
- function 与 group 映射关系
- pin 当前 mux 配置
- pin 当前 bias / drive-strength 等信息

驱动中实现了 `pin_dbg_show` 和 `pin_config_dbg_show`，可输出如下信息：

- pin 寄存器 offset
- IO type
- 当前 mux 值
- 上下拉状态
- drive strength
- 原始寄存器值

### wake irq 说明

K3 pinctrl 节点本身还是一个中断控制器：

- `interrupt-controller;`
- `#interrupt-cells = <2>;`

驱动 probe 时会：

1. 获取 pinctrl 控制器中断
2. 清空所有 pin 的 edge 状态
3. 注册 threaded irq handler
4. 建立线性 irq domain
5. 根据 irq type 设置 rise/fall/both 检测方式

这部分能力主要用于 SoC 休眠/唤醒场景中的 GPIO/pin 边沿唤醒。

## K1 与 K3 差异说明

虽然 K1 与 K3 使用同一套 pinctrl IP 演进而来，但两者在 dts 写法上已有明显差异：

2. **K3 使用 `compatible = "spacemit,k3-pinctrl"`**
3. **K1 常见写法是 `K1X_PADCONF(pin, mux, config)` 一次编码全部属性**
4. **K3 改为 `K3_PADCONF(pin, func)` + 标准 pinconf 属性分离描述**
5. **K3 对 external IO 更强调 `power-source` 与驱动能力联动**
6. **K3 默认 dtsi 中更大量使用分组、分 state、分子组组织方式**

因此在为 K3 新增或修改 pinctrl 配置时，建议优先参考：

- `drivers/pinctrl/spacemit/pinctrl-k1.c`
- `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi`
- 对应板级 dts 中的实际引用方式

## 接口介绍

### API 介绍

- **获取设备 pinctrl 句柄**

```c
struct pinctrl *devm_pinctrl_get(struct device *dev);
```

- **释放设备 pinctrl 句柄**

```c
void devm_pinctrl_put(struct pinctrl *p);
```

- **查找 pinctrl state**

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                           const char *name);
```

- **切换 pinctrl state**

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state);
```

### demo 示例

#### 使用 Linux 默认定义的 pins 状态

Linux 定义了 **default**、**init**、**idle** 和 **sleep** 四种标准 pin 状态，kernel 框架层会进行管理，普通驱动一般不需要手动操作。

- **default**：设备默认 pin 状态
- **init**：设备 probe 初始化阶段状态
- **sleep**：设备 suspend 时 pin 状态
- **idle**：runtime suspend / idle 时 pin 状态

例如：

```dts
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&gmac0_cfg>;
};
```

这种情况下，内核框架会在设备初始化时自动应用 `default` 状态。

#### 自定义 pins 状态

K3 SD 卡控制器就是一个典型例子，定义了 `default`、`uhs` 和 `debug` 三种状态，由驱动在不同工作场景下进行切换。

```dts
&sdcard {
    pinctrl-names = "default", "uhs", "debug";
    pinctrl-0 = <&mmc1_cfg>;
    pinctrl-1 = <&mmc1_uhs_cfg>;
    pinctrl-2 = <&mmc1_debug_cfg>;
};
```

驱动中通常按如下方式使用：

```c
pinctrl = devm_pinctrl_get(dev);
state_default = pinctrl_lookup_state(pinctrl, "default");
state_uhs = pinctrl_lookup_state(pinctrl, "uhs");
state_debug = pinctrl_lookup_state(pinctrl, "debug");

pinctrl_select_state(pinctrl, state_default);
```

## 总结

K3 pinctrl 文档编写和配置时，建议把握以下几点：

1. **K3 的核心写法是 `K3_PADCONF(pin, func)` + 标准 pinconf 属性**
2. **外部电压域 pin 配置驱动能力时，要同时设置 `power-source`**
3. **优先复用 `k3-pinctrl.dtsi` 中已有的 `xxx_cfg` 定义**
4. **当默认配置不满足需求时，可在板级 dts 中新增或重写 pin group**
5. **分析问题时以驱动代码和实际 dts 用法为准， binding 仅作参考**
