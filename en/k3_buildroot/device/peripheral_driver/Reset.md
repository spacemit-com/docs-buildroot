# Reset

This document describes the K3 platform reset system, including its architecture, driver implementation, device tree configuration, usage, and common debugging methods.

## Module Overview

The reset system controls reset signals for internal SoC modules. It returns modules to a known state and ensures correct initialization.

### Functional Overview

#### Reset Framework

Linux provides a reset management layer called the Reset Controller Framework. It gives device drivers a unified reset interface, so they do not need to handle hardware-specific reset logic directly.

![](static/RESET.png)

The Reset Controller Framework includes the following core components:

- **reset provider**: A reset controller that provides the reset signals required by the system.
- **reset consumer**: A device driver that uses reset signals. It acquires and controls resets through the common APIs provided by the Reset Controller Framework.
- **reset framework**: The core of the Reset Controller Framework. It provides generic reset APIs to consumers, implements the central reset management logic, and delegates hardware-specific reset operations to the reset provider.
- **device tree**: The Reset Controller Framework allows reset signals and their device associations to be declared in the device tree.
        - The `resets` property identifies the reset resource used by a device through a `provider + ID` pair.
        - The `reset-names` property provides readable names for those resources, allowing device drivers to acquire and manage them by name.

#### Reset Functions

The reset system provides the following main functions:

- **Assert**: Places a module in the reset state and stops it from running.
- **Deassert**: Releases a module from the reset state and allows normal operation.
- **Reset**: Performs a complete reset cycle by asserting and then deasserting the reset signal.
- **Status**: Checks whether the module is currently in the reset state.

#### K3 Reset Architecture

On the K3 platform, the reset architecture is tightly coupled with the clock architecture. It is divided into multiple reset providers by address domain, and each reset provider maps to a matching clock provider.

| Reset provider node | Address | Typical usage |
| :--- | :--- | :--- |
| `syscon_mpmu` | `0xd4050000` | Basic resets related to MPMU |
| `syscon_apmu` | `0xd4282800` | APMU domain resets, such as QSPI, SDH, USB, and CPU |
| `syscon_apbc` | `0xd4015000` | APB peripheral resets, such as UART, PWM, I2C, SPI, and RTC |
| `syscon_apbc2` | `0xf0610000` | Security-domain APB peripheral resets |
| `syscon_dciu` | `0xd8440000` | DCIU-related control |
| `syscon_rcpu_sysctrl` | `0xc0880000` | R-domain system control related resets |
| `syscon_rcpu_uartctrl` | `0xc0881f00` | R-domain UART |
| `syscon_rcpu_i2sctrl` | `0xc0882000` | R-domain I2S |
| `syscon_rcpu_spictrl` | `0xc0885f00` | R-domain SPI |
| `syscon_rcpu_i2cctrl` | `0xc0886f00` | R-domain I2C |
| `syscon_rpmu` | `0xc088c000` | R-domain PMU |
| `syscon_rcpu_pwmctrl` | `0xc088d000` | R-domain PWM |

The following is a common reset reference pattern in K3 DTS files:

```dts
resets = <&syscon_apbc RESET_APBC_CAN0>;
```

This means:

- The reset provider is `syscon_apbc`
- The reset ID is `RESET_APBC_CAN0`

Select the correct reset provider and reset ID based on the actual driver implementation.

#### Reset and Clock Coordination

Reset control must be coordinated with clocks to initialize hardware modules correctly. A typical K3 workflow is shown below.

**Module initialization flow**

1. Release reset (`reset_control_deassert`)
2. Enable the clock (`clk_prepare_enable`)
3. Configure the operating clock rate (`clk_set_rate`)
4. Configure module registers
5. Start the module

**Module shutdown flow**

When the module enters sleep mode or is shut down, perform the steps in reverse order:

1. Stop the module
2. Disable the clock (`clk_disable_unprepare`)
3. Assert reset for the module (`reset_control_assert`)

## Source Code Structure

### Source Locations

K3 reset-related code is mainly located in the following paths:

```text
linux-6.18/
|-- drivers/reset/
|   `-- reset-spacemit.c          # K3 reset controller driver
|-- include/dt-bindings/clock/
|   `-- spacemit,k3-syscon.h      # K3 reset and clock ID definitions
|-- include/soc/spacemit
|   `-- k3-syscon.h               # K3 reset and clock register definitions
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3*.dts
```

The SpacemiT K3 reset driver implements the standard Reset Controller Framework interfaces:

- `reset_control_assert()`: Sets the reset signal to the asserted state and resets the module.
- `reset_control_deassert()`: Sets the reset signal to the deasserted state and releases the module from reset.

## Configuration

### Kconfig Settings

- `CONFIG_RESET_CONTROLLER`: Enables Reset Controller Framework support and provides the base reset framework for the system.

```text
-> Device Drivers│
  │       -> Reset Controller Support (RESET_CONTROLLER [=y])│
```

- `CONFIG_SPACEMIT_RESET`: Enables the SpacemiT reset driver and provides the platform-specific reset controller implementation.

```text
-> Device Drivers│
  │       -> Reset Controller Support (RESET_CONTROLLER [=y])│
  │              -> SpacemiT SoC reset controller (SPACEMIT_RESET [=y])│
```

### DTS Configuration

K3 reset controllers are divided into multiple reset controller instances by address domain. Each reset controller manages reset signals within one address domain. These reset controller nodes are defined in `k3.dtsi`, and they also act as clock controllers. The DTS configuration is shown below:

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
These nodes declare themselves as reset providers through the `#reset-cells = <1>` property. Individual device nodes reference the required reset resources through the `resets` and `reset-names` properties.

## API Description

The Reset Controller Framework provides a unified reset API for device drivers. These interfaces are declared in `include/linux/reset.h`. Common APIs are listed below.

- `get`: Acquire a reset handle

```c
/**
 * devm_reset_control_get - get reset control
 * @dev: device
 * @id: reset name
 * Returns a reset control or IS_ERR() condition containing errno.
 */
struct reset_control *devm_reset_control_get(struct device *dev, const char *id);

/**
 * of_reset_control_get_by_name - get reset control by name
 * @node: device
 * @name: reset name
 * Returns a reset control or IS_ERR() condition containing errno.
 */
struct reset_control *of_reset_control_get_by_name(struct device_node *node,
                                                   const char *name);
```
If the second parameter is omitted, these interfaces return the first entry in the DTS `resets` list by default.

- `put`: Release a reset handle

```c
/**
 * reset_control_put - release a reset control
 * @rstc: reset control
 */
void reset_control_put(struct reset_control *rstc);
```

- `assert`: Assert reset

```c
/**
 * reset_control_assert - asserts the reset line
 * @rstc: reset control
 * Returns 0 on success or negative errno on failure.
 */
int reset_control_assert(struct reset_control *rstc);
```

- `deassert`: Deassert reset

```c
/**
 * reset_control_deassert - deasserts the reset line
 * @rstc: reset control
 * Returns 0 on success or negative errno on failure.
 */
int reset_control_deassert(struct reset_control *rstc);
```

- `reset`: Perform a complete reset cycle

```c
/**
 * reset_control_reset - perform a complete reset of the device
 * @rstc: reset control
 * Returns 0 on success or negative errno on failure.
 * This function will assert, then deassert the reset line.
 */
int reset_control_reset(struct reset_control *rstc);
```

## Usage

This section describes reset usage from both the **provider** and **consumer** perspectives.

### Provider

K3 reset providers are already defined in `k3.dtsi`. In most cases, no additional provider configuration is required in the board-level DTS.

### Consumer

A reset consumer typically needs to:

- Find the correct reset provider and reset ID, and configure `resets` in the DTS node.
- Configure `reset-names` in the DTS node.
        - The `reset-names` values vary by device and depend on the purpose of each reset.
        - They can be customized if needed.
        - Drivers use these names when acquiring reset handles.
        - Common values include `reset`, `rst`, and `sys_rst`.
- Acquire reset handles in the device driver through the reset APIs and perform the required reset operations.

Most K3 peripheral DTS nodes use a reset configuration similar to the following:

```dts
xxx: device@addr {
	resets = <&ResetProvider RESET_ID>;
	status = "disabled";
};
```

The reset provider can usually be identified from the prefix of the reset ID. The mapping is shown below:

| Reset ID prefix | Corresponding provider node |
| :--- | :--- |
| RESET_MPMU_ | syscon_mpmu |
| RESET_APMU_ | syscon_apmu |
| RESET_APBC_ | syscon_apbc |
| RESET_APBC2_ | syscon_apbc2 |
| RESET_DCIU_ | syscon_dciu |
| RESET_RCPU_SYSCTRL_ | syscon_rcpu_sysctrl |
| RESET_RCPU_UARTCTRL_ | syscon_rcpu_uartctrl |
| RESET_RCPU_I2SCTRL_ | syscon_rcpu_i2sctrl |
| RESET_RCPU_SPICTRL_ | syscon_rcpu_spictrl |
| RESET_RCPU_I2CCTRL_ | syscon_rcpu_i2cctrl |
| RESET_RPMU_ | syscon_rpmu |
| RESET_RCPU_PWMCTRL_ | syscon_rcpu_pwmctrl |

## Usage Example

To use reset functionality, a module must define `resets` and `reset-names` in DTS, then perform reset-related operations in the driver through the Reset Controller Framework APIs.

- **Configure DTS**

   Find the corresponding reset ID in `include/dt-bindings/clock/spacemit,k3-syscon.h`, and add it to the module DTS.

   Take `can0` as an example. `can0` has only one reset ID: `RESET_APBC_CAN0`. The prefix `RESET_APBC_` maps to the provider `syscon_apbc`, so the DTS configuration is as follows:

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
        reset-names = "reset";
        status = "disabled";
    };
```

- **Add the header file and `reset_control` structure**

   Add the following header file and structure definition to the driver code:
```c
#include <linux/reset.h>
```

```c
struct flexcan_priv {
        struct reset_control *reset;
};
```

- **Acquire the reset handle**

   Use `devm_reset_control_get_optional` to acquire the reset handle:

```c
reset = devm_reset_control_get_optional(&pdev->dev, "reset");
        if(IS_ERR(reset)) {
                dev_err(&pdev->dev, "flexcan get reset failed\n");
                return PTR_ERR(reset);
        }
```

- **Release reset**

   Use `reset_control_deassert` to release reset:

```c
        if (priv->reset) {
                err = reset_control_deassert(priv->reset);
                if (err && priv->clk_ipg && priv->clk_per) {
                        clk_disable_unprepare(priv->clk_per);
                        clk_disable_unprepare(priv->clk_ipg);
                }
        }
```

- **Assert reset**

   Use `reset_control_assert` to reset the module:
```c
        if (priv->reset)
                reset_control_assert(priv->reset);
```

## FAQ

### 1. The module is not working. How should reset-related issues be investigated?

- Check whether the DTS configuration is correct:
  - Is `resets` configured?
  - Are the provider and ID correct?
  - Does `reset-names` match the driver?
- Check whether the provider node includes the `#reset-cells = <1>` property.
- Check which name the driver actually passes to `devm_reset_control_get()`, and whether any errors are reported, such as `failed to get reset` or `failed to deassert reset`.
- Check whether the driver deasserts reset correctly.
- Check whether the reset and clock sequence is correct.
- Check whether any other dependencies are missing, such as clock, pinctrl, power-domain, or regulator.

### 2. Why do some modules use `syscon_apbc` while others use `syscon_apmu`?

Because they belong to different clock, power, or bus domains. In general:

- Low-speed APB peripherals are more commonly associated with `syscon_apbc`.
- High-speed or more complex modules are more commonly associated with `syscon_apmu`.
- R-domain modules are more commonly associated with the `syscon_rcpu_*` provider series.

See the method described in the [Usage](#usage) section.

### 3. What is the difference between `reset_control_get` and `reset_control_get_optional`?

- `reset_control_get`: Returns an error if the `resets` property is not configured in DTS.
- `reset_control_get_optional`: Returns `NULL` instead of an error if the `resets` property is not configured in DTS, which means the reset signal is optional.

For modules where the reset signal is optional, use the `_optional` interface. This allows the driver to continue working even if no reset signal is configured.

### 4. When should `reset_control_assert` be called?

`reset_control_assert` is typically used in the following scenarios:

- When a module is unloaded or shut down, to reset it and reduce power consumption
- When a module encounters an error and needs to be reinitialized
- When the system enters a low-power mode

In most cases, it is sufficient to call `reset_control_deassert` during initialization to release reset.
