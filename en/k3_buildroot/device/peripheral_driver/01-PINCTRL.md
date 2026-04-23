# PINCTRL

This document describes the functionality and usage of **K3 PINCTRL**.

## Module Overview

PINCTRL is the **controller for the PIN module**. It is responsible for pin multiplexing selection, electrical attribute configuration, and some wake-up interrupt capabilities on the K3 SoC.

### Features

The Linux pinctrl module consists of two parts: the **pinctrl core** and the **pin controller driver**.

1. Main functions of the **pinctrl core**:
   - Provides pinctrl function interfaces for other drivers
   - Provides registration and deregistration interfaces for pin controller devices
   - Parses pinctrl states in DTS and switches states in device drivers

2. Main functions of the **K3 pinctrl controller driver**:
   - Drives the K3 pin controller hardware
   - Implements pin multiplexing configuration
   - Configures pin electrical attributes such as pull-up, pull-down, drive strength, and Schmitt trigger input
   - Configures IO supply voltage for pins in external voltage domains
   - Provides pin-based edge-triggered wake-up interrupt support

### Source Code Structure

The K3 pinctrl driver code is located in the kernel directory:

```text
drivers/pinctrl/spacemit/
|-- Kconfig
|-- Makefile
|-- pinctrl-k1.c
`-- pinctrl-k1.h
```

Although the source file is still named `pinctrl-k1.c`, the current driver supports both **K1** and **K3**. The K3 DTS `compatible` value is:

```dts
compatible = "spacemit,k3-pinctrl";
```

The default K3 pin configuration is defined in:

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

The K3 pinctrl controller node is located in:

```text
arch/riscv/boot/dts/spacemit/k3.dtsi
```

Example node:

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

## Key Features

| Feature | Description |
| :----- | :---- |
| Supports pin multiplexing selection | Allows a pin to be configured for one of its multiplexed functions |
| Supports pin electrical attribute configuration | Supports pull-up, pull-down, strong pull-up, drive strength, and Schmitt trigger input |
| Supports external IO voltage domains | For pins of type External, `power-source` can be used to specify 1.8 V or 3.3 V |
| Supports edge-triggered wake-up interrupts | Supports pin wake-up detection via rise / fall / both|
| Supports grouped configuration | Supports configuring mux and attributes for multiple pins in a pin group |
| Supports multi-state switching | Supports multiple pinctrl states such as `default`, `uhs`, and `debug` |

## Configuration

This mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

- **CONFIG_PINCTRL**: Provides generic support for pin controllers. The default value is `Y`.
- **CONFIG_PINCTRL_SPACEMIT_K1**: Provides support for SpacemiT K1/K3 pinctrl controllers.

The corresponding Kconfig definition is:

```text
config PINCTRL_SPACEMIT_K1
    bool "SpacemiT K1/K3 SoC Pinctrl driver"
    depends on SOC_SPACEMIT || COMPILE_TEST
    depends on OF
    default ARCH_SPACEMIT
```

## Pin Usage Guide

This section describes how to use K3 pinctrl in DTS device nodes.

### Pin Configuration Parameters

K3 pinctrl configuration consists of three parts:

- **Pin ID**
- **Mux function**
- **Attribute configuration**

Unlike K1, K3 no longer uses `K1X_PADCONF(pin, mux, config)` in DTS to encode mux and electrical settings into a single macro. Instead, it uses:

- `pinmux`: Encodes only the **pin ID + mux mode**
- **Standard pinconf properties**: Describe pull-up, drive strength, voltage, and other attributes separately

#### pinmux Encoding Macro

K3 uses the following macro definition by default:

```c
#define K3_PADCONF(pin, func) (((pin) << 16) | (func))
```

Meaning:

- Upper 16 bits: Pin number
- Lower 16 bits: Mux function number

The driver parses it as follows:

- `pin = value >> 16`
- `mux = value & GENMASK(15, 0)`

### Pin ID

The K3 pin number range is **0 ~ 163**, for a total of **164 pins**.

The K3 pin name definitions in the driver are located in `k3_pin_desc[]` in `pinctrl-k1.c`.

Typical pin names include:

- `GPIO_00 ~ GPIO_127`
- `PWR_SCL`
- `PWR_SDA`
- `VCXO_EN`
- `PMIC_INT_N`
- `MMC1_DAT3 ~ MMC1_CLK`
- `QSPI_DAT0 ~ QSPI_CLK`
- `PRI_TDI ~ PWR_SSP_RXD`
- `EMMC_D0 ~ EMMC_CMD`

### Pin Electrical Types

The driver defines an IO type for each pin. Different types determine the drive strength mapping and whether `power-source` is required.

Supported types:

- `IO_TYPE_1V8`
- `IO_TYPE_3V3`
- `IO_TYPE_EXTERNAL`

The main K3 pin distributions are as follows:

| Pin Range | Name/Bank | IO Type |
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
| 153 ~ 163 | EMMC | Special (supports only eMMC/GPIO functions) |

> **Note:** For pins of type `IO_TYPE_EXTERNAL`, if `drive-strength` is configured in DTS, `power-source = <1800>` or `power-source = <3300>` must also be provided. Otherwise, the driver treats the configuration as invalid.

### Pin Functions

K3 pins support multiplexed function selection, and the mux function number is encoded directly in `K3_PADCONF(pin, func)`.

For example:

```dts
pinmux = <K3_PADCONF(149, 2)>, <K3_PADCONF(150, 2)>;
```

This means:

- pin 149 selects mux function 2
- pin 150 selects mux function 2

In the driver, the mux value is written to the `mux mode` bit field of the pad register.

### Pin Attributes

K3 uses standard pinconf properties to describe pin electrical configuration. Based on the current driver implementation and DTS usage, K3 mainly supports the following properties.

#### Pull-Up/Pull-Down

Supported settings:

- `bias-disable;`
- `bias-pull-down;`
- `bias-pull-up;`

Description:

- `bias-disable`: Disables pull-up and pull-down
- `bias-pull-down`: Enables pull-down
- `bias-pull-up`: Enables pull-up

The driver also supports the concept of **strong pull-up**:

- `PIN_CONFIG_BIAS_PULL_UP` parameter value `0`: normal pull-up
- `PIN_CONFIG_BIAS_PULL_UP` parameter value `1`: strong pull-up

Default DTS examples typically use normal pull-up.

#### Drive Strength

The attribute is:

```dts
drive-strength = <mA>;
```

The unit is **mA**, not a DS index. The driver automatically maps the mA value to the corresponding register value according to the pin voltage type.

##### 1.8V Pin Drive Strength Mapping

K3 1.8 V pins support 16 drive strength levels:

| Register Value | Current (mA) |
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

##### 3.3V Pin Drive Strength Mapping

K3 3.3 V pins support 16 drive strength levels:

| Register Value | Current (mA) |
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

The driver selects the **smallest drive strength level that is not less than the target value**.

For example:

- `drive-strength = <25>;` maps to the level corresponding to 25 mA
- `drive-strength = <8>;` for a 3.3 V pin maps to the smallest level not less than 8 mA, which is the 9 mA level

#### Input Schmitt Trigger

The driver supports:

```dts
input-schmitt = <value>;
```

On K3, the corresponding register field is 1 bit wide, and the driver writes this property to the `schmitt` field.

#### Voltage Source Selection

For `IO_TYPE_EXTERNAL` pins, the following properties are available:

```dts
power-source = <1800>;
```

or:

```dts
power-source = <3300>;
```

Purpose:

- Specifies whether the pin group uses 1.8 V or 3.3 V IO voltage
- Allows the driver to calculate `drive-strength` using the correct drive strength table
- Causes the driver to write the corresponding configuration to the IO power-domain register

If an external pin group uses `drive-strength` without a valid `power-source`, the driver reports an error and rejects the configuration.

#### Edge Detection

The driver supports the following low-level hardware edge detection:

- Rising-edge detection
- Falling-edge detection
- Both-edge detection
- Clearing edge status

This is mainly used for the **wake irq** function. It is not generally configured through standard pinconf properties in `k3-pinctrl.dtsi`. Instead, it is handled through the pinctrl controller interrupt domain and IRQ type configuration flow.

### Pin Configuration Definitions

#### Single-Pin Configuration

On K3, a single pin is usually configured through `pinmux` plus independent attributes.

Example: Configure pins 149 and 150 for the uart0 function, with normal pull-up and 25 mA drive strength.

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

#### Defining a Group of Pins

The default K3 function pin groups are located in:

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

Compared with K1, the default K3 pin groups have two clear characteristics:

1. **Unified naming using `xxx_cfg`**, such as `uart0_0_cfg`, `gmac0_cfg`, and `mmc1_cfg`
2. **A single state can contain multiple child groups**, and each child group can have its own `pinmux` and electrical attributes

For example, under `mmc1_cfg`, the data/command lines and clock line are placed in two separate child groups:

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

#### Overriding the Default Pin Group

If the default definitions in `k3-pinctrl.dtsi` do not meet board-level requirements, you can do either of the following in the board DTS:

Method 1: Rewrite the existing `xxx_cfg` node; 
Method 2: Add a new `xxx_cfg` node and then reference the new state in the device node. 

Adding a new state (as Mothod 2) is recommended because it makes it easier to distinguish board-specific settings from the default SoC DTSI configuration.

### Pin Usage Examples

#### UART Usage Example

Definition in `k3-pinctrl.dtsi`:

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

Referenced by the device node:

```dts
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_0_cfg>;
    status = "disabled";
};
```

#### SD Card Multi-State Example

The K3 SD card controller defines three pinctrl states:

- `default`
- `uhs`
- `debug`

Where:

- `default`: standard 3.3 V operating mode
- `uhs`: UHS/1.8 V operating mode
- `debug`: multiplexes some MMC1 pins to the uart0 debug function

The corresponding DTS definitions are as follows:

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

Device node reference:

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

#### GMAC Usage Example

K3 GMAC pin groups are defined in detail and are divided into:

- base pins
- RGMII extension pins
- MII extension pins
- optional interrupt pins
- optional PPS pins
- optional refclk pins

For example, `gmac0_cfg`:

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

Referenced by the device node:

```dts
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&gmac0_cfg>;
    phy-mode = "rgmii";
    status = "okay";
};
```

#### PCIe Usage Example

Control pins of the K3 PCIe interface, such as PERST#, WAKE#, and CLKREQ#, are also configured through the pinctrl subsystem. For example:

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

Device node reference:

```dts
&pcie0_rc {
    pinctrl-names = "default";
    pinctrl-0 = <&pcie0_1_cfg>;
    status = "okay";
};
```

#### USB Host Drive Pin Example

For example, the USB2 host drive pin:

```dts
usb20_host_drv_3_cfg: usb20_host_drv-3-cfg {
    usb20_host_drv-pins {
        pinmux = <K3_PADCONF(103, 3)>;
        bias-disable;
        drive-strength = <25>;
    };
};
```

Referenced in the board DTS:

```dts
&usb3_portc {
    pinctrl-names = "default";
    pinctrl-0 = <&usb20_host_drv_3_cfg>;
};
```

## K3 Pinctrl DTS Usage Guide

### Basic Structure

K3 pinctrl states are typically placed under the `&pinctrl` node. The common structure is as follows:

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

Meaning:

- `xxx_cfg`: The name of the pinctrl state referenced by peripheral device nodes
- `xxx-cfg`: The name of the DTS node
- `xxx-pins`: A child group under the corresponding state
- `pinmux`: The pin and mux settings included in the group
- Other standard pinconf properties: Shared electrical attributes for the group

### Multi-Subgroup Structure

When different pins within the same state require different electrical attributes, the configuration can be divided into multiple `xxx-pins` child nodes.

For example, MMC1:

- data lines and command lines use `bias-pull-up`
- the clock line uses `bias-pull-down`

Therefore, the configuration is divided into two groups: `mmc1-0-pins` and `mmc1-1-pins`.

### Device Node Reference Method

Peripheral device nodes reference pinctrl states in the standard way:

```dts
pinctrl-names = "default";
pinctrl-0 = <&xxx_cfg>;
```

For multiple states:

```dts
pinctrl-names = "default", "uhs", "debug";
pinctrl-0 = <&state0_cfg>;
pinctrl-1 = <&state1_cfg>;
pinctrl-2 = <&state2_cfg>;
```

## Debugging

### Viewing pinctrl Information via debugfs

After enabling debugfs, you can check the pinctrl configuration via the following path:

```text
/sys/kernel/debug/pinctrl/
```

This can be used to inspect:

- pin group registration status
- mappings between functions and groups
- current mux settings of pins
- current pin information such as bias and drive strength

The driver implements `pin_dbg_show` and `pin_config_dbg_show`, which can output the following information:

- pin register offset
- IO type
- current mux value
- pull-up/pull-down status
- drive strength
- raw register value

### Wake IRQ

The K3 pinctrl node also acts as an interrupt controller:

- `interrupt-controller;`
- `#interrupt-cells = <2>;`

During driver probe, it:

1. Obtains the pinctrl controller interrupt
2. Clears the edge status of all pins
3. Registers the threaded IRQ handler
4. Creates a linear IRQ domain
5. Sets rising-edge, falling-edge, or both-edge detection based on the IRQ type

This functionality is mainly used for GPIO/pin edge wake-up in SoC sleep/wake-up scenarios.

## Differences Between K1 and K3

Although K1 and K3 evolved from the same pinctrl IP, their DTS syntax now has clear differences:

1. **K3 uses `compatible = "spacemit,k3-pinctrl"`**
2. **A common K1 syntax is `K1X_PADCONF(pin, mux, config)`, which encodes all attributes at once**
3. **K3 uses `K3_PADCONF(pin, func)` together with separate standard pinconf properties**
4. **K3 places more emphasis on the relationship between `power-source` and drive strength for external IO**
5. **The default K3 DTSI uses grouping, multiple states, and child-group separation more extensively**

Therefore, when adding or modifying K3 pinctrl configuration, it is recommended to first refer to:

- `drivers/pinctrl/spacemit/pinctrl-k1.c`
- `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi`
- the actual reference method in the corresponding board DTS

## Interface Description

### API Description

- **Get the pinctrl handle for a device**

```c
struct pinctrl *devm_pinctrl_get(struct device *dev);
```

- **Release the pinctrl handle for a device**

```c
void devm_pinctrl_put(struct pinctrl *p);
```

- **Look up a pinctrl state**

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                           const char *name);
```

- **Switch a pinctrl state**

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state);
```

### Demo Examples

#### Using Linux Default-Defined Pin States

Linux defines four standard pin states: **default**, **init**, **idle**, and **sleep**. These are managed by the kernel framework, and ordinary drivers usually do not need to operate on them manually.

- **default**: The default pin state of the device
- **init**: The pin state during device probe initialization
- **sleep**: The pin state when the device is suspended
- **idle**: The pin state during runtime suspend or idle

For example:

```dts
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&gmac0_cfg>;
};
```

In this case, the kernel framework automatically applies the `default` state during device initialization.

#### Custom Pin States

The K3 SD card controller is a typical example. It defines three states, `default`, `uhs`, and `debug`, which are switched by the driver for different operating scenarios.

```dts
&sdcard {
    pinctrl-names = "default", "uhs", "debug";
    pinctrl-0 = <&mmc1_cfg>;
    pinctrl-1 = <&mmc1_uhs_cfg>;
    pinctrl-2 = <&mmc1_debug_cfg>;
};
```

The driver typically uses them as follows:

```c
pinctrl = devm_pinctrl_get(dev);
state_default = pinctrl_lookup_state(pinctrl, "default");
state_uhs = pinctrl_lookup_state(pinctrl, "uhs");
state_debug = pinctrl_lookup_state(pinctrl, "debug");

pinctrl_select_state(pinctrl, state_default);
```

## Summary

When writing and configuring K3 pinctrl documentation, keep the following points in mind:

1. **The core K3 syntax is `K3_PADCONF(pin, func)` plus standard pinconf properties**
2. **When configuring drive strength for pins in external voltage domains, `power-source` must also be configured**
3. **Prefer reusing existing `xxx_cfg` definitions in `k3-pinctrl.dtsi`**
4. **If the default configuration does not meet requirements, add new pin groups or override existing ones in the board DTS**
5. **When analyzing issues, rely primarily on the driver code and actual DTS usage; binding documentation is for reference only**
