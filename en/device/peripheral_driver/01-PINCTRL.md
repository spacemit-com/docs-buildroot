# PINCTRL

PIN Functionality and Usage Guide.

## Overview

PINCTRL is the **controller for the PIN module**.

### Function Description

![](static/linux_pinctrl.png)  
 
The Linux pinctrl module consists of two parts: **pinctrl core** and **pin controller driver**.

1. **pinctrl core** primarily has two functions:
   - Provides pinctrl function interfaces for use by other drivers
   - Offers registration and deregistration interfaces for pin controller devices

2. **pinctrl controller driver** main functions:
   - Drive the pin controller hardware
   - Implement the management and configuration of pins

### Source Code Structure

The controller driver code is located in the `drivers/pinctrl` directory:

```
drivers/pinctrl
|-- pinctrl-single.c
```

## Key Features


| Feature | Description |
| :-----| :----|
| Support for pin multiplexing selection | Allows configuring a pin for one of its multiplexed functions |  
| Support for setting pin attributes | Supports configuring pin attributes such as edge detection, pull-up, pull-down, and drive strength |

## Configuration

It mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

- **CONFIG_PINCTRL**: Provides support for pin controllers, with a default value of `Y`.

```
Device Drivers
        Pin controllers (PINCTRL [=y])
```

- **CONFIG_PINCTRL_SINGLE**: Provides support for the K1 pinctrl controller, with a default value of `Y`.

```
Device Drivers  
        Pin controllers (PINCTRL [=y])
                One-register-per-pin type device tree based pinctrl driver (PINCTRL_SINGLE [=y])
```

## Pin Usage Instructions

Introduces how to use pins in the device tree (DTS) device nodes.

### Pin configuration parameter

Define the **pin id**, **multiplexing function**, and **attributes**

Detailed definitions in the kernel directory `include/dt-bindings/pinctrl/k1-x-pinctrl.h`.

#### PIN ID

This is the pin number.

The K1 pin numbers range is **1 - 147**, corresponding to the macro definitions from `GPIO_00` to `GPIO_127`.

#### pin function

K1 pins support multiplexing selection.

The K1 pin multiplexing function list can be found at [K1 Pin Multiplex](https://developer.spacemit.com/documentation?token=CzJlwnDYNigRgDk7qS2cvYHPnkh&type=file).

The multiplexing function numbers for pins range from **0 to 7**, defined as `MUX_MODE0 to MUX_MODE7`.

#### pin attributes

The attributes of a pin include **edge detection, pull-up/pull-down**, and **drive strength**.

##### Edge Detection

When using a functional pin to wake up the system, configure the signal detection method that generates the wake-up event.

Four modes are supported:

- Edge detection disabled: `EDGE_NONE`
- Rising edge detection: `EDGE_RISE`
- Falling edge detection: `EDGE_FALL`
- Both rising and falling edges: `EDGE_BOTH`

##### Pull-up/Pull-down

Three modes are supported:

- Pull-up/pull-down disabled: `PULL_DIS`
- Pull-up: `PULL_UP`
- Pull-down: `PULL_DOWN`

##### Drive Strength

1. **Pin voltage at 1.8V**: There are four levels, with higher values indicating stronger drive capabilities.

   - PAD_1V8_DS0
   - PAD_1V8_DS1
   - PAD_1V8_DS2
   - PAD_1V8_DS3

2. **Pin voltage at 3.3V**: There are seven levels, with higher values indicating stronger drive capabilities.

   - PAD_3V_DS0
   - PAD_3V_DS1
   - PAD_3V_DS2
   - PAD_3V_DS3
   - PAD_3V_DS4
   - PAD_3V_DS5
   - PAD_3V_DS6
   - PAD_3V_DS7

### Pin Configuration Definitions

#### Single Pin Configuration

Configure the pin function, edge detection, pull-up/pull-down, and drive strength.

Use the macro `K1X_PADCONF` for configuration, with the format **pin_id, mux_mode, pin_config**.

Example: Set pin GPIO_00 to the gmac0 rxdv function, disable edge detection, disable pull-up/pull-down, and set the drive strength to 2 (1.8V).

Check the K1 pin multiplexing function list [K1 Pin Multiplex] (https://developer.spacemit.com/documentation?token=CzJlwnDYNigRgDk7qS2cvYHPnkh&type=file). To set GPIO_00 to the gmac0 rxdv function, the function mode needs to be set to 1, that is, MUX_MODE1.

Configure as follows:

```c
K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
```

#### Define a Group of Pins

Configure the **functional pin group** used by the controller (such as GMAC, PCIE, USB, and eMMC, etc.).

The default functional pin group definitions are located in the kernel directory: `arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi`.

1. Check if the functional pin group is defined in `k1-x_pinctrl.dtsi`. If it is defined and meets the configuration requirements, it can be used directly. Otherwise, follow Step-2.
2. Set up the pin group used by the controller.

Taking eth0 as an example, assume that the eth0 pin group on the development board uses GPIO00 to GPIO14 and GPIO45, and pull-up is also enabled by TX.

Default definition of gmac0 pins in `k1-x_pinctrl.dtsi`

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

The pull-up/pull-down function of the tx pin is not satisfied. The default definition is to disable pull-up/pull-down, but currently, pull-up needs to be enabled.

There are two methods:

- **Method 1:** Rewrite the default pin group definition in the dts.
- **Method 2:** Add a new pin group definition in the dts.

The following sections will introduce each method respectively.

1. Rewrite the default pin group definition.

   Add the following configuration in the solution DTS file to rewrite the default gmac0 configuration and set gmac0 tx to pull-up.

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

2. Define a new gmac0 pin group.

   Add the following configuration in the solution DTS file to set gmac0 tx to pull-up.

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

### Pin Usage Example

eth0 references the `pinctrl_gmac0` that is redefined in the solution.

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0>;
};
```

Or it references the newly added `pinctrl_gmac0_1` in the solution.

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0_1>;
};
```

## Interface

### API

- **Obtaining and releasing the pinctrl handle for a device**

```
struct pinctrl *devm_pinctrl_get(struct device *dev);  
```

- **Release the pinctrl handle for a device**

```
void devm_pinctrl_put(struct pinctrl *p);
```

- **Find pinctrl state**
  Look up the corresponding pin control state in the pin control state holder based on state_name.

```
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
       const char *name)
```

- **Set pinctrl state**
  Set the pinctrl state for the device's pins

```
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
```

### Demo Example

#### Using Linux Default-Defined Pin States

Linux defines four standard pin states: **default, init, idle**, and **sleep**. These are managed by the kernel framework, and module drivers do not need to operate on them  

- **default**: The default state for device pins.
- **init**: The initialization state for device pins during the probe phase of the device driver.
- **sleep**: The pin state when the device is in sleep mode during the PM (Power Management) process, set during suspend.
- **idle**: The pin state when the device is in runtime suspend, set during pm_runtime_suspend or pm_runtime_idle.

For example, if the gmac0 controller uses pins defined in the **default** state, the gmac controller driver does not need to perform any operations. The kernel framework will handle the setup of the eth0 pins. 
The DTS configuration is as follows:


```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0_1>;
};
```

#### Custom Pin States

Taking the K1 SD card controller as an example, the K1 SD card controller defines three pin states: **default**, **fast**, and **debug**. 

The definitions and references in the dts are as follows

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

The K1 SD controller driver `sdhci-of-k1x.c` manages the above pins.

```c
/* Get the pinctrl handler */
spacemit->pinctrl = devm_pinctrl_get(&pdev->dev);
...

/* Find and set the pin configuration for fast/default/debug states */
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

## Debugging

### sysfs

View the system's current **pinctrl control details** and **pin configuration settings**.

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

- `d401e000.pinctrl-pinctrl-single`: D401e000 pinctrl manages **pin details**. Please refer to the **debugfs** section here for details.

- `pinctrl-devices`: Information about all **pinctrl controllers** in the system.

- `pinctrl-handles/pinctrl-maps`: Displays information about the **pin function groups** that have been requested by the system.

### debugfs

Used to view the current pin configuration information of the solution. This includes the usage of all pins in the system, which ones are used as gpio, and which ones are used as functional pins.

```
/sys/kernel/debug/pinctrl/d401e000.pinctrl-pinctrl-single
|-- gpio-ranges         //Configured as gpio
|-- pingroups           //Functional pin groups
|-- pinmux-functions
|-- pinmux-pins
|-- pinmux-select      
|-- pins               //Usage of all pins
```

## Testing
 
Use the `devmem` tool to check the register value associated with a pin:

```
devmem reg_addr
```

## FAQ
