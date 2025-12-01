# Clock

Clock Functionality and Usage Guide.

## Overview

The Clock module manages and controls system clock signals.

### Functional Description

![](static/CLOCK.png)

To manage clocks effectively, Linux provides the Common Clock Framework (CCF) for centralized clock management. This framework offers a unified interface for device drivers, allowing them to operate without needing to know the specific hardware implementation details of the clock. The structure includes the following components:
- **clock provider:** Corresponding to the right side of the diagram, the clock controller, which is responsible for providing various clocks required by the system.
- **clock consumer:** Corresponding to the left side of the diagram, the device drivers that use the clocks.
- **clock framework:** The CCF core that exposes unified APIs for clock operations to consumers; implementing the core logic of clock management, encapsulating hardware-related clock control logic into a set of operation functions, and leaving their implementation to the clock provider.
- **device tree:** CCF allows the declaration of available clocks and their associations with devices in the device tree

Clock-related components in the system include:
- Oscillator (active, aka resonator) or Crystal (passive, aka crystal oscillator) for generating clocks.
- PLL (Phase Locked Loop) for frequency multiplication.
- Divider for frequency division.
- Mux for clock source selection.
- Gate for clock enable/disable control.

There may be many such hardware modules in the system, organized in a tree structure. Linux manages them as a clock tree, with the root node typically being a crystal oscillator, followed by PLLs, then muxes or dividers, and finally gate controllers as leaf nodes. CCF implements several basic clock types, such as fixed-rate clocks, gate clocks, divider clocks, and mux clocks. Generally, to facilitate usage, additional clock types are implemented based on the clock tree design.

### Source Code Structure

#### Clock Controller Driver Source Code
The Clock controller driver code is located in the `drivers/clk/spacemit` directory:

```
drivers/clk/spacemit
|-- ccu_ddn.c                   # Source code for ddn clock type
|-- ccu_ddn.h
|-- ccu_ddr.c                   # Source code for ddr clock type
|-- ccu_ddr.h
|-- ccu_dpll.c                  # Source code for dpll clock type
|-- ccu_dpll.h
|-- ccu_mix.c                   # Source code for mix clock type
|-- ccu_mix.h
|-- ccu_pll.c                   # Source code for pll clock type
|-- ccu_pll.h
|-- ccu-spacemit-k1x.c          # k1 clock controller driver
|-- ccu-spacemit-k1x.h
|-- Kconfig
|-- Makefile
```

The clock controller driver implements five types of clocks:
- **pll type:** Phase Locked Loop (PLL) type.
- **dpll type:** DDR-related PLL type.
- **ddn type:** Fractional divider type, with one level of division (denominator) and one level of multiplication (numerator).
- **mix type:** Mixed type, supporting any combination of gate, mux, and divider.
- **ddr type:** Special clock type related to DDR.

#### Clock Index Definitions

The clock index definitions are located in the dt-bindings directory:

```
include/dt-bindings/clock/spacemit-k1x-clock.h
```

## Configuration

The configuration mainly includes driver enabling and DTS (Device Tree Source) configuration.

### CONFIG Configuration

CONFIG_COMMON_CLK: This option provides support for the Common Clock Framework (CCF). By default, this option is enabled (`y`).

```
Device Drivers
 Common Clock Framework (COMMON_CLK[=y])
```

CONFIG_SPACEMIT_K1X_CCU: This option provides support for the K1 Clock controller driver. By default, this option is enabled  (`y`).

```
 Device Drivers
 Common Clock Framework (COMMON_CLK[=y])
         Clock support for Spacemit k1x SoCs (SPACEMIT_K1X_CCU [=y])
```

### DTS Configuration
The DTS configuration for the clock controller is as follows:

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

## Interface

### API
The Common Clock Framework (CCF) provides a set of generic clock operation interfaces for device drivers.

- get  
Obtain a clock handle

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

For the aforementioned interface, if the second parameter is omitted, it will default to obtaining the first clock configured in the "clocks" item in the DTS.
- put  
Release the clock handle.

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

- prepare  
Prepare the clock, which usually involves some preliminary work before enabling the clock.

```c
/**
 * clk_prepare - prepare a clock source
 * @clk: clock source
 * This prepares the clock source for use.
 * Must not be called from within atomic context.
 */
int clk_prepare(struct clk *clk);
```

- unprepare  
Unprepare the clock, which usually involves some cleanup work after disabling the clock.

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

- enable  
Clock enable

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

- disable  
Disable the clock.

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

**clk_prepare_enable:** This is a combination of clk_prepare and clk_enable. It is recommended to use this interface for enabling the clock after preparing it.
**clk_disable_unprepare:** This is a combination of clk_unprepare and clk_disable. It is recommended to use this interface for disabling the clock and performing any necessary cleanup.

- set rate  
Set the clock frequency.

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

- get rate  
Obtain the current clock frequency.

```c
/**
 * clk_get_rate - obtain the current clock rate (in Hz) for a clock source.
 *                This is only valid once the clock source has been enabled.
 * @clk: clock source
 */
unsigned long clk_get_rate(struct clk *clk);

```

- set parent  
Set the parent clock.

```c
/**
 * clk_set_parent - set the parent clock source for this clock
 * @clk: clock source
 * @parent: parent clock source
 * Returns success (0) or negative errno.
 */
int clk_set_parent(struct clk *clk, struct clk *parent);

```

- get parent  
Obtain the handle of the current parent clock.

```c
/**
 * clk_get_parent - get the parent clock source for this clock
 * @clk: clock source
 * Returns struct clk corresponding to parent clock source, or
 * valid IS_ERR() condition containing errno.
 */
struct clk *clk_get_parent(struct clk *clk);
```

- round rate  
Obtain the frequency that is closest to the target frequency and that the clock controller can provide.

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

### Usage Example

To use the clock functionality in a module, you need to configure the clocks and clock-names properties in the DTS file, and then use the CCF API for clock-related operations in the driver.
- Configure dts
  Locate the corresponding clock indices in include/dt-bindings/clock/spacemit-k1x-clock.h and configure them in the module's DTS file.
  Taking can0 as an example, can0 has two clocks: one is the module operation clock CLK_CAN0, and the other is the bus clock CLK_CAN0_BUS. The DTS configuration is as follows:

   ```
                  flexcan0: fdcan@d4028000 {
                          compatible = "spacemit,k1x-flexcan";
                          reg = <0x0 0xd4028000 0x0 0x4000>;
                          interrupts = <16>;
                          interrupt-parent = <&intc>;
                          clocks = <&ccu CLK_CAN0>,<&ccu CLK_CAN0_BUS>; # Configure the clock indices for can0
                          clock-names = "per","ipg";                    # Configure the names corresponding to the clocks, which  can be used by the driver to obtain the corresponding clocks
                          resets = <&reset RESET_CAN0>;
                          fsl,clk-source = <0>;
                          status = "disabled";
                };

  ```

- Add Header Files and Clock Handles 

  ```
  #include <linux/clk.h>
  ```

  ```
  struct flexcan_priv {

          struct clk *clk_ipg;
          struct clk *clk_per;
  };
  ````

- Obtaining Clocks  
  Typically, during the driver's probe stage, clock handles are obtained using devm_clk_get. If the driver's probe fails or during the remove stage, the driver automatically releases the corresponding clock handles.

  ```
          clk_ipg = devm_clk_get(&pdev->dev, "ipg");               # Obtain the clock handle corresponding to the bus clock CLK_CAN0_BUS.
          if (IS_ERR(clk_ipg)) {
                  dev_err(&pdev->dev, "no ipg clock defined\n");
                  return PTR_ERR(clk_ipg);
          }

          clk_per = devm_clk_get(&pdev->dev, "per");               # Obtain the clock handle corresponding to the operating clock CLK_CAN0.
          if (IS_ERR(clk_per)) {
                  dev_err(&pdev->dev, "no per clock defined\n");
                  return PTR_ERR(clk_per);
          }

  ```

- Enable the clock.
  Enable the clock node using clk_prepare_enable

  ```
          if (priv->clk_ipg) {
                  err = clk_prepare_enable(priv->clk_ipg);         # Enable the bus clock CLK_CAN0_BUS.
                  if (err)
                          return err;
          }

          if (priv->clk_per) {
                  err = clk_prepare_enable(priv->clk_per);         # Enable the operating clock CLK_CAN0.
                  if (err)
                          clk_disable_unprepare(priv->clk_ipg);
          }

  ```

- Obtain the clock frequency.
Get the clock frequency using clk_get_rate.

  ```
  clock_freq = clk_get_rate(clk_per);                  # Obtain the current frequency of the operating clock CLK_CAN0.
  ```

- Set the clock frequency. 
  Modify the clock frequency using clk_set_rate, where the first parameter is the clock handle struct clk*, and the second parameter is the target frequency.

  ```
  clk_set_rate(clk_per, clock_freq);                   # Set the frequency of the operating clock CLK_CAN0.
  ```

- Disable the clock. 
  Disable the clock using clk_disable_unprepare.

  ```
  clk_disable_unprepare(priv->clk_per);                # Disable the operating clock CLK_CAN0.
  clk_disable_unprepare(priv->clk_ipg);                # Disable the bus clock CLK_CAN0_BUS.
  ```

## Debugging

  You can use debugfs for debugging purposes.

- Print the Clock Tree
  The file /sys/kernel/debug/clk/clk_summary is commonly used to print the clock tree structure. It provides information on the status, frequency, parent clocks, and other details of each clock node

  ```
  root# cat /sys/kernel/debug/clk/clk_summary
  ```

- Viewing Specific Clock Nodes
  You can also view the status, frequency, parent clock, and other information for specific clock nodes. For example, to view details for the can0_clk:

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
  root:/sys/kernel/debug/clk/can0_clk# cat clk_prepare_count          # Viewing Enable Status
  0
  root:/sys/kernel/debug/clk/can0_clk# cat clk_rate                   # Viewing Current Frequency
  20000000
  root:/sys/kernel/debug/clk/can0_clk# cat clk_parent                 # Viewing the Current Parent Clock
  pll3_20
  root:/sys/kernel/debug/clk/can0_clk#
  ```

- Modifying Clock Configuration
  Define CLOCK_ALLOW_WRITE_DEBUGFS in drivers/clk/clk.c to enable write access to debugfs clock nodes. Otherwise, you only have read permissions.

  ```
  /sys/kernel/debug/clk/can0_clk # ls -l
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_accuracy
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_duty_cycle
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_enable_count
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_flags
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_max_rate
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_min_rate
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_notifier_count
  -rw-r--r--    1 root     root             0 Jan  1 08:03 clk_parent              # Read/write
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_phase
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_possible_parents
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_count
  -rw-r--r--    1 root     root             0 Jan  1 08:03 clk_prepare_enable      # Read/write
  -r--r--r--    1 root     root             0 Jan  1 08:03 clk_protect_count
  -rw-r--r--    1 root     root             0 Jan  1 08:03 clk_rate                # Read/write
  /sys/kernel/debug/clk/can0_clk # cat clk_rate                                    # View the frequency.
  20000000
  /sys/kernel/debug/clk/can0_clk # echo 40000000 > clk_rate                        # Set the frequency to 40 MHz.
  /sys/kernel/debug/clk/can0_clk # cat clk_rate                                    # Confirm the setting result
  40000000
  /sys/kernel/debug/clk/can0_clk # cat clk_parent                                  # View the parent clock.
  pll3_40
  /sys/kernel/debug/clk/can0_clk # echo 0 > clk_parent                             # Set the parent clock to the clock source with index 0.
  /sys/kernel/debug/clk/can0_clk # cat clk_parent                                  # Confirm the setting result.
  pll3_20
  /sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # View the prepare_enable status.
  0
  /sys/kernel/debug/clk/can0_clk # echo 1 > clk_prepare_enable                     # Prepare and enable the clock node.
  /sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # Confirm the setting result
  1
  /sys/kernel/debug/clk/can0_clk # echo 0 > clk_prepare_enable                     # Unprepare and disable the clock node.
  /sys/kernel/debug/clk/can0_clk # cat clk_prepare_enable                          # Confirm the setting result.
  0
  /sys/kernel/debug/clk/can0_clk #
  ```

## FAQ
