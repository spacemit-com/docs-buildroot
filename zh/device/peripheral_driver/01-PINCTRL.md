# PINCTRL

介绍 **PIN** 的功能和使用方法。

## 模块介绍

PINCTRL 是 **PIN模块的控制器**。

### 功能介绍

![](static/linux_pinctrl.png)  

Linux pinctrl模块包括两部分: **pinctrl core** 和 **pin 控制器驱动**。  

1. **pinctrl core** 主要有两个功能:  
   - 提供 pinctrl 功能接口给其它驱动使用  
   - 提供 pin 控制器设备注册与注销接口  

2. **pinctrl 控制器驱动** 主要功能:  
   - 驱动 pin 控制器硬件  
   - 实现 pin 的管理和配置

### 源码结构介绍

控制器驱动代码在 `drivers/pinctrl` 目录下：

```
drivers/pinctrl
|-- pinctrl-single.c
```

## 关键特性


| 特性 | 特性说明 |
| :-----| :----|
| 支持pin复用选择 | 支持将pin设置成复用功能中一种 |  
| 支持设置pin的属性 | 支持设置pin的边沿检测、上下拉和驱动能力 |

## 配置介绍

主要包括 **驱动使能配置** 和 **dts配置**

### CONFIG配置

- **CONFIG_PINCTRL**: 为 pin 控制器提供支持，默认值为 `Y`

```
Device Drivers
        Pin controllers (PINCTRL [=y])
```

- **CONFIG_PINCTRL_SINGLE**: 为 K1 pinctrl 控制器提供支持，默认值为 `Y`

```
Device Drivers  
        Pin controllers (PINCTRL [=y])
                One-register-per-pin type device tree based pinctrl driver (PINCTRL_SINGLE [=y])
```

## pin使用说明

介绍在dts设备节点里使用pin。

### pin 配置参数

对 **pin id**、**复用功能** 和 **属性** 进行定义。

详细定义内核目录 `include/dt-bindings/pinctrl/k1-x-pinctrl.h`。

#### pin id

即 pin 编号。

K1 pin 编号范围 **1~147**，对应宏定义 `GPIO_00 ~ GPIO_127`。

#### pin 功能

K1 pin 支持复用选择。

K1 pin 复用功能列表见 [K1 Pin Multiplex](https://developer.spacemit.com/documentation?token=CzJlwnDYNigRgDk7qS2cvYHPnkh&type=file)。

pin 的复用功能号为 **0~7**，分别定义为 `MUX_MODE0 ~ MUX_MODE7`。

#### pin 属性

pin 的属性包括 **边沿检测**、**上下拉** 和 **驱动能力**。

##### 边沿检测

采用功能 pin 唤醒系统时，设置产生唤醒事件的信号检测方式。

支持如下四种模式：

- 边沿检测关闭：`EDGE_NONE`
- 上升沿检测：`EDGE_RISE`
- 下降沿检测：`EDGE_FALL`
- 上升和下降沿：`EDGE_BOTH`

##### 上下拉

支持如下三种模式：

- 上下拉禁止：`PULL_DIS`
- 上拉：`PULL_UP`
- 下拉：`PULL_DOWN`

##### 驱动能力

1. **pin 电压为 1.8V**: 分为 4 级，值越大，驱动能力越强。

   - PAD_1V8_DS0
   - PAD_1V8_DS1
   - PAD_1V8_DS2
   - PAD_1V8_DS3

2. **pin 电压为 3.3V**: 分为 7 级，值越大，驱动能力越强。

   - PAD_3V_DS0
   - PAD_3V_DS1
   - PAD_3V_DS2
   - PAD_3V_DS3
   - PAD_3V_DS4
   - PAD_3V_DS5
   - PAD_3V_DS6
   - PAD_3V_DS7

### pin 配置定义

#### 单个 pin 配置

选定 pin 功能，设置 pin 的边沿检测，上下拉和驱动能力。

采用宏 `K1X_PADCONF` 进行设置, 格式为 **pin_id, mux_mode, pin_config**

示例：将 pin GPIO_00 设置为 gmac0 rxdv 功能，且关闭边沿检测，关闭上下拉，驱动能力设置为 2 (1.8V)。

查看 K1 pin 功能复用列表 [K1 Pin Multiplex](https://developer.spacemit.com/documentation?token=CzJlwnDYNigRgDk7qS2cvYHPnkh&type=file)，GPIO_00 要设置成 gmac0 rxdv 功能，需要设置功能模式为 1, 即 MUX_MODE1。

设置如下:

```c
K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
```

#### 定义一组 pin

对控制器(如 gmac、pcie、usb 和 emmc 等)使用的 **功能 pin 组** 进行配置。

默认的功能 pin 组定义，内核目录：`arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi`。

1. 功能 pin 组是否在 `k1-x_pinctrl.dtsi` 有定义，如果已定义且满足配置，直接使用；如果未定义或配置不满足，则按照第 2 步进行设置；
2. 设置控制器使用的 pin 组

以 eth0 为例，假设开发板 eth0 pin 组使用 GPIO00~GPIO14、GPIO45，且 tx 需要使能上拉。

`k1-x_pinctrl.dtsi` 中 gmac0 pins 默认定义

```c
pinctrl_gmac0: gmac0_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
};
```

tx pin 的上下拉功能不满足，默认定义为关闭上下拉，当前需要使能上拉。

有两种方法：

1. 方案 dts 重写 pin 组默认定义
2. 方案 dts 增加一组 pin 定义

下面分别进行介绍。

1. 重写 pin 组默认定义
   在方案 dts 文件中增加如下配置，重写 gmac0 默认配置，将 gmac0 tx 设置为上拉。

```c
&pinctrl {
    pinctrl_gmac0: gmac0_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
    };
};
```

2. 新定义 gmac0 pin 组
   在方案 dts 文件中增加如下配置，将 gmac0 tx 设置为上拉。

```c
&pinctrl {
    pinctrl_gmac0_1: gmac0_1_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
    };
};
```

### pin 使用示例

eth0 引用方案中重写定义的 `pinctrl_gmac0`

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0>;
};
```

或者引用方案新增加的 `pinctrl_gmac0_1`

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0_1>;
};
```

## 接口介绍

### API介绍

- **获取和释放设备 pinctrl 句柄**

```
struct pinctrl *devm_pinctrl_get(struct device *dev);  
```

- **释放设备pinctrl句柄**

```
void devm_pinctrl_put(struct pinctrl *p);
```

- **查找pinctrl state**
  根据state_name 在pin control state holder中查找对应的pin control state.

```
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
       const char *name)
```

- **设定pinctrl state**  
  对设备pins设置pinctrl state.

```
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
```

### demo示例

#### 使用 Linux 默认定义的 pins 状态

linux定义了 **defaul**、**init**、**idle** 和 **sleep** 四种标准 pins 状态，kernel 框架层会进行管理，模块驱动不用操作。  

- **default**: 设备pins默认状态  
- **init**:    设备驱动probe阶段初始化状态
- **sleep**:   PM(电源管理)流程设备睡眠状态时pins状态, suspend时设置
- **idle**:    runtime suspend时pins状态，pm_runtime_suspend或pm_runtime_idle时设置

如 gmac0 控制器使用 pins 定义为 **default** 状态, gmac 控制器驱动不用做任何操作，kernel 框架会完成 eth0 pins 的设置。
dts配置如下:

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0_1>;
};
```

#### 自定义 pins 状态 

以 K1 SD卡控制器举例，K1 SD卡控制器定义了 3 种 pins状态 **default**、**fast** 和 **debug**。  
dts 中定义和引用如下:

```c
&pinctrl {
    ...
    pinctrl_mmc1: mmc1_grp {
  pinctrl-single,pins = <
   K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d3 */
   K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d2 */
   K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d1 */
   K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d0 */
   K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_cmd */
   K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_3V_DS4)) /* mmc1_clk */
  >;
 };

 pinctrl_mmc1_fast: mmc1_fast_grp {
  pinctrl-single,pins = <
   K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_1V8_DS3)) /* mmc1_d3 */
   K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_1V8_DS3)) /* mmc1_d2 */
   K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_1V8_DS3)) /* mmc1_d1 */
   K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_1V8_DS3)) /* mmc1_d0 */
   K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_1V8_DS3)) /* mmc1_cmd */
   K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS3)) /* mmc1_clk */
  >;
 };

    pinctrl_mmc1_debug: mmc1_debug_grp {
  pinctrl-single,pins = <
   K1X_PADCONF(MMC1_DAT3, MUX_MODE3, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* uart0_txd */
   K1X_PADCONF(MMC1_DAT2, MUX_MODE3, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* uart0_rxd */
   K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d1 */
   K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_d0 */
   K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP   | PAD_3V_DS4)) /* mmc1_cmd */
   K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_3V_DS4)) /* mmc1_clk */
  >;
 };
    ...
};

&sdhci0 {
 pinctrl-names = "default","fast","debug";
 pinctrl-0 = <&pinctrl_mmc1>;
 pinctrl-1 = <&pinctrl_mmc1_fast>;
 pinctrl-2 = <&pinctrl_mmc1_debug>;
    ...
};

```

K1 SD 控制器驱动
`sdhci-of-k1x.c` 管理上述pins

```c
/* 获取pinctrl handler */
spacemit->pinctrl = devm_pinctrl_get(&pdev->dev);
...

/* 查找fast/default/debug状态的pins配置，并进行设置 */
if (spacemit->pinctrl && !IS_ERR(spacemit->pinctrl)) {
        if (clock >= 200000000) {
       spacemit->pin = pinctrl_lookup_state(spacemit->pinctrl, "fast");
       if (IS_ERR(spacemit->pin))
            pr_warn("could not get sdhci fast pinctrl state.\n");
       else
            pinctrl_select_state(spacemit->pinctrl, spacemit->pin);
  } else if (clock == 0) {
       spacemit->pin = pinctrl_lookup_state(spacemit->pinctrl, "debug");
       if (IS_ERR(spacemit->pin))
            pr_debug("could not get sdhci debug pinctrl state. ignore it\n");
       else
            pinctrl_select_state(spacemit->pinctrl, spacemit->pin);
  } else {
       spacemit->pin = pinctrl_lookup_state(spacemit->pinctrl, "default");
       if (IS_ERR(spacemit->pin))
            pr_warn("could not get sdhci default pinctrl state.\n");
       else
            pinctrl_select_state(spacemit->pinctrl, spacemit->pin);
  }
}
...

```

## Debug介绍

### sysfs

查看系统当前 **pinctrl 控制信息** 和 **pin配置信息**

```
/sys/kernel/debug/pinctrl
|-- d401e000.pinctrl-pinctrl-single
|   |-- gpio-ranges
|   |-- pingroups
|   |-- pinmux-functions
|   |-- pinmux-pins
|   |-- pinmux-select
|   `-- pins
|-- pinctrl-devices
|-- pinctrl-handles
|-- pinctrl-maps
`-- spacemit-pinctrl@spm8821
    |-- gpio-ranges
    |-- pinconf-groups
    |-- pinconf-pins
    |-- pingroups
    |-- pinmux-functions
    |-- pinmux-pins
    |-- pinmux-select
    `-- pins
```

- `d401e000.pinctrl-pinctrl-single`: d401e000 pinctrl 管理的 **pin 详细信息**。详见本文的 **debugfs** 小节说明  

- `pinctrl-devices`: 系统中所有 **pinctrl 控制器信息**  

- `pinctrl-handles/pinctrl-maps`: 显示系统已请求的 **pin 功能组信息**

### debugfs

用于查看当前方案pin配置信息。包括系统中所有pins的使用情况，哪些用于gpio，哪些用于功能pin。

```
/sys/kernel/debug/pinctrl/d401e000.pinctrl-pinctrl-single
|-- gpio-ranges         //配置成gpio
|-- pingroups           //功能pin组
|-- pinmux-functions
|-- pinmux-pins
|-- pinmux-select      
|-- pins               //所有pins使用情况
```

## 测试方法  

使用 `devmem` 工具查看某 pin 对应寄存器值： 

```
devmem reg_addr
```

## FAQ
