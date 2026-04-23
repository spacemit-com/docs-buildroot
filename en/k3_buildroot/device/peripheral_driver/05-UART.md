# UART

This document describes UART configuration and usage.

## Module Overview

UART is a universal asynchronous serial communication interface. It is commonly used for console output, debug logging, Bluetooth module communication, MCU communication, and similar scenarios.

### Functional Overview

In Linux, UART support mainly consists of the following two parts:

- **UART controller driver**: Responsible for accessing serial controller registers and configuring baud rate, interrupts, FIFO, DMA, and related functions.
- **Serial core and TTY framework**: Registers serial device nodes with the system, such as `/dev/ttyS*`.

On the K3 platform, UART currently uses the **8250 OF platform-driver framework**, together with the SoC-specific adaptation logic added by SpacemiT to the 8250 OF implementation.

### Source Code Structure

The K3 UART-related code and description files are mainly located at:

```text
Linux-6.18/
|-- drivers/tty/serial/8250/8250_of.c
|-- drivers/tty/serial/8250/Kconfig
|-- drivers/tty/serial/Kconfig
|-- Documentation/devicetree/bindings/serial/8250.yaml
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3-pinctrl.dtsi
```

Where:

- `drivers/tty/serial/8250/8250_of.c` is the driver code currently matched by the K3 DTS `compatible` strings.
- `Documentation/devicetree/bindings/serial/8250.yaml` is the dt-binding currently used for K3 UART.
- `k3.dtsi` defines the AP-domain UART controller nodes.
- `k3-rdomain.dtsi` defines the RCPU-domain UART controller nodes.
- `k3-pinctrl.dtsi` defines the pinmux schemes for UART / RUART.

## Relationship Between the K1 and K3 Drivers

This distinction must be made first; otherwise, it is easy to identify the wrong driver file.

### K1 in the Older Kernel

In `k1-sdk/linux-6.6`, common UART `compatible` strings for K1 include:

- `spacemit,pxa-uart`
- `spacemit,rcpu-pxa-uart0`
- `spacemit,rcpu-pxa-uart1`

The corresponding driver file is:

```text
drivers/tty/serial/pxa_k1x.c
```

In other words, the older K1 kernel used an earlier dedicated SpacemiT PXA UART driver.

### K3 in the Current Kernel

In `k3-sdk/linux-6.18`, the K3 UART nodes are actually defined as:

```dts
compatible = "spacemit,k1-uart",
             "intel,xscale-uart";
```

The RCPU UART nodes use the same definition:

```dts
compatible = "spacemit,k1-uart",
             "intel,xscale-uart";
```

Therefore, K3 UART documentation should use the following files as the primary references:

- `drivers/tty/serial/8250/8250_of.c`
- `Documentation/devicetree/bindings/serial/8250.yaml`
- `arch/riscv/boot/dts/spacemit/k3.dtsi`
- `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`
- `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi`

## Key Features

| Feature | Description |
| :----- | :----- |
| Supports multiple UARTs | K3 provides multiple UARTs in both the AP and RCPU domains |
| Based on the 8250 OF platform driver | The current `compatible` settings route the driver to `8250_of.c` |
| Supports FIFO configuration | `fifo-size` and `tx-threshold` can be configured in DTS |
| Supports dynamic clock adaptation | `8250 OF` includes SpacemiT clock-switching logic |
| Supports RCPU UART differentiation | RCPU UARTs can be distinguished through the `spacemit,rcpu-uart` property |
| Supports multiple pinctrl configurations | The same UART usually has multiple pinmux options |
| Supports standard console setup | Console routing is organized through `aliases`, `stdout-path`, and `ttyS*` |

## Configuration

This mainly includes driver **CONFIG enablement** and **DTS configuration**.

### CONFIG Configuration

Since K3 UART currently uses the 8250 OF framework, the following configuration options usually need to be checked:

- `CONFIG_SERIAL_8250`
- `CONFIG_SERIAL_OF_PLATFORM`
- and the enablement configuration for the SpacemiT SoC

This can be seen in `drivers/tty/serial/8250/Kconfig`:

```text
config SERIAL_OF_PLATFORM
	depends on SERIAL_8250 && OF
```

### DTS Configuration

#### 1. AP-Domain UART Nodes

K3 AP-domain UARTs are defined in `arch/riscv/boot/dts/spacemit/k3.dtsi`, for example:

```dts
uart5: serial@d4017400 {
    compatible = "spacemit,k1-uart",
                 "intel,xscale-uart";
    reg = <0x0 0xd4017400 0x0 0x100>;
    clocks = <&syscon_apbc CLK_APBC_UART5>,
             <&syscon_apbc CLK_APBC_UART5_BUS>,
             <&syscon_mpmu CLK_MPMU_SLOW_UART>;
    clock-names = "core", "bus", "gate";
    resets = <&syscon_apbc RESET_APBC_UART5>;
    reg-shift = <2>;
    reg-io-width = <4>;
    fifo-size = <256>;
    tx-threshold = <32>;
    interrupt-parent = <&saplic>;
    interrupts = <47 IRQ_TYPE_LEVEL_HIGH>;
    status = "disabled";
};
```

From the current `.dtsi` files, the common AP-domain UART properties include:

- `compatible`
- `reg`
- `clocks`
- `clock-names`
- `resets`
- `reg-shift`
- `reg-io-width`
- `fifo-size`
- `tx-threshold`
- `interrupt-parent`
- `interrupts`
- `status`

The current `k3.dtsi` defines 11 AP-domain UART controllers:

- `uart0` ~ `uart10`

#### 2. RCPU-Domain UART Nodes

K3 RCPU-domain UARTs are defined in `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`, for example:

```dts
r_uart0: serial@c0881000 {
    compatible = "spacemit,k1-uart",
                 "intel,xscale-uart";
    reg = <0x0 0xc0881000 0x0 0x100>;
    clocks = <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0>,
             <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0_BUS>,
             <&syscon_mpmu CLK_MPMU_SLOW_UART>;
    clock-names = "core", "bus", "gate";
    spacemit,rcpu-uart;
    resets = <&syscon_rcpu_uartctrl RESET_RCPU_UARTCTRL_RUART0>;
    reg-shift = <2>;
    reg-io-width = <4>;
    fifo-size = <256>;
    tx-threshold = <32>;
    interrupt-parent = <&saplic>;
    interrupts = <251 IRQ_TYPE_LEVEL_HIGH>;
    status = "disabled";
};
```

Compared with AP-domain UARTs, RCPU-domain UARTs include one additional property:

```dts
spacemit,rcpu-uart;
```

This property allows the driver to identify an RCPU-domain UART. In `8250_of.c`, SpacemiT's `spacemit_8250_set_termios()` reads this property and uses different methods to select clock frequencies for ACPU and RCPU UARTs.

The current `k3-rdomain.dtsi` defines 6 RCPU-domain UART controllers:

- `r_uart0` ~ `r_uart5`

#### 3. dt-binding Description

The binding currently used by K3 UART is:

```text
Documentation/devicetree/bindings/serial/8250.yaml
```

The SpacemiT-related `compatible` definition is:

```yaml
- items:
    - enum:
        - mrvl,mmp-uart
        - spacemit,k1-uart
    - const: intel,xscale-uart
```

The binding also specifies:

- when `compatible` includes `spacemit,k1-uart`, `clock-names` must include `core` and `bus`

The actual configuration used in the current K3 `.dtsi` files is:

```dts
clock-names = "core", "bus", "gate";
```

In other words:

- `core`: functional clock
- `bus`: bus clock
- `gate`: slow clock / gate-related clock

The corresponding acquisition logic can also be found in `drivers/tty/serial/8250/8250_of.c`:

```c
info->gate_clk = devm_clk_get_optional_enabled(dev, "gate");
info->bus_clk  = devm_clk_get_optional_enabled(dev, "bus");
info->clk      = devm_clk_get_enabled(dev, info->bus_clk ? "core" : NULL);
```

#### 4. `serial` aliases and `stdout-path`

In `k3.dtsi`:

```dts
aliases {
    serial0 = &uart0;
    serial1 = &uart1;
    ...
    serial10 = &uart10;
    serial11 = &r_uart0;
    ...
    serial16 = &r_uart5;
};
```

For example, in `k3_evb.dts`:

```dts
chosen {
    bootargs = "earlycon=sbi console=ttyS0,115200 ...";
    stdout-path = "serial0:115200";
};
```

This indicates that:

- `serial0` points to `&uart0`
- the system console corresponds to `ttyS0`

This is also where the development board's default debug UART comes from.

## Pin Configuration

For UART to actually send and receive data from external pins, the corresponding pins must also be switched to the UART function in pinctrl.

The K3 UART / RUART pinmux definitions are located in:

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

For example, `uart0_0_cfg`:

```dts
uart0_0_cfg: uart0-0-cfg {
    uart0-0-pins {
        pinmux = <K3_PADCONF(149, 2)>, /* uart0 tx */
                 <K3_PADCONF(150, 2)>; /* uart0 rx */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

The same `uart0` controller on K3 also has multiple optional pinmux configurations:

- `uart0_1_cfg`
- `uart0_2_cfg`
- `uart0_3_cfg`
- `uart0_4_cfg`

This indicates that the same UART controller can be mapped to different external pin groups.

Another example is `uart2_0_cfg`:

```dts
uart2_0_cfg: uart2-0-cfg {
    uart2-0-pins {
        pinmux = <K3_PADCONF(134, 2)>, /* uart2 tx */
                 <K3_PADCONF(135, 2)>, /* uart2 rx */
                 <K3_PADCONF(136, 2)>, /* uart2 cts */
                 <K3_PADCONF(137, 2)>; /* uart2 rts */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

This shows that K3 UART supports not only TX/RX, but also CTS/RTS hardware flow-control pinmux on some UARTs.

The RCPU domain UARTs also have corresponding configurations, for example:

- `ruart0_0_cfg`
- `ruart1_0_cfg`
- `ruart2_0_cfg`
- `ruart5_0_cfg`

Therefore, in actual use, the following steps are usually required:

1. Enable the corresponding UART/RUART node
2. Select the correct pinctrl configuration

## Usage Guide

### UART Resource Configuration in `dtsi`

The SoC `.dtsi` files already define the base addresses, clocks, reset signals, and interrupt resources for the UART controllers. Under normal circumstances, board-level users do not need to modify these settings, because they are maintained centrally by the SoC platform.

### Enabling UART in `dts`

Board-level `.dts` files usually enable serial ports by overriding nodes. For example, in `k3_evb.dts`:

```dts
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_0_cfg>;
    status = "okay";
};
```

This means:

- enables `uart0`
- selects the `uart0_0_cfg` pinmux configuration
- sets the node status to `okay`

### Console Serial Port

If a UART is used as the console output port, the following settings must also be configured together:

- `aliases`
- `chosen/stdout-path`
- `console=ttySx,baudrate` in `bootargs`

For example:

```dts
chosen {
    bootargs = "earlycon=sbi console=ttyS0,115200 ...";
    stdout-path = "serial0:115200";
};
```

### Example: Multiplexing with Peripherals Such as Bluetooth

In `k3_evb.dts`, `uart2` also retains an example Bluetooth child node:

```dts
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_0_cfg>;
    status = "disabled";

    bluetooth {
        compatible = "realtek,rtl8852bs-bt";
        device-wake-gpios = <&gpio 2 30 GPIO_ACTIVE_HIGH>;
        enable-gpios = <&gpio 2 29 GPIO_ACTIVE_HIGH>;
        host-wake-gpios = <&gpio 2 28 GPIO_ACTIVE_HIGH>;
    };
};
```

## Driver Implementation

The core driver file currently used by K3 UART is:

```text
drivers/tty/serial/8250/8250_of.c
```

### 1. Matching to 8250 OF Through `compatible`

The current `compatible` property in the K3 DTSI is:

```dts
compatible = "spacemit,k1-uart", "intel,xscale-uart";
```

`Documentation/devicetree/bindings/serial/8250.yaml` defines this `compatible` group, so the current K3 UART follows the 8250 OF platform serial driver path.

### 2. SpacemiT Clock Adaptation Logic

Under `CONFIG_SOC_SPACEMIT`, `8250_of.c` adds dedicated logic such as:

- `spacemit_acpu_match_clk_rate()`
- `spacemit_8250_set_termios()`
- `spacemit_8250_of_serial_clk_work_cb()`

Among them, `spacemit_8250_set_termios()` selects different serial clock strategies based on:

- the current baud rate
- whether `spacemit,rcpu-uart` is present

- **AP-domain UART**: Selects an appropriate clock source for common baud rates through a fixed lookup-table method
- **RCPU-domain UART**: Uses `clk_round_rate()` / `clk_set_rate()` so the clock framework can choose the most suitable frequency

This is why the `spacemit,rcpu-uart` must be emphasized in the K3 UART documentation.

### 3. FIFO / Threshold Parameters

The common configuration in the current K3 DTSI is as follows:

```dts
fifo-size = <256>;
tx-threshold = <32>;
```

These two parameters should not be understood only literally as "FIFO size" and "threshold". They must also be interpreted together with the actual processing path in the 8250 OF driver.

#### Meaning of `fifo-size`

`fifo-size` is read by the common serial-property parsing logic and written to:

```c
port->fifosize
```

In `drivers/tty/serial/8250/8250_of.c`, the driver then uses it to determine whether the UART has FIFO capability:

```c
if (port8250.port.fifosize)
    port8250.capabilities = UART_CAP_FIFO;
```

In other words:

- `fifo-size` describes the **depth of the UART controller's hardware transmit/receive FIFO**.
- For K3, `fifo-size = <256>` indicates that the UART controller is configured in the 8250 framework as having a **256-byte FIFO**.
- This value affects whether the 8250 core treats the port as FIFO-capable, as well as the subsequent transmit loading strategy.

In the 8250 core transmit path `serial8250_tx_chars()`, the driver attempts to fill the TX FIFO with `up->tx_loadsz` bytes each time:

```c
count = up->tx_loadsz;
do {
    ...
    serial_out(up, UART_TX, c);
    ...
} while (--count > 0);
```

If the port does not explicitly set `fifosize` / `tx_loadsz`, the 8250 core fills them with default values based on the port type:

```c
if (!port->fifosize)
    port->fifosize = uart_config[port->type].fifo_size;
if (!up->tx_loadsz)
    up->tx_loadsz = uart_config[port->type].tx_loadsz;
```

Explicitly setting `fifo-size = <256>` in the K3 DTS essentially tells the 8250 framework: **this port should not use a conservative default FIFO depth, but should instead be configured with the actual K3 hardware FIFO depth of 256**.

#### Meaning of `tx-threshold`

`tx-threshold` is not the total FIFO size. Instead, it represents the **low-watermark threshold** of the TX FIFO. Its handling in `8250_of.c` is straightforward:

```c
if ((of_property_read_u32(ofdev->dev.of_node, "tx-threshold",
              &tx_threshold) == 0) &&
    (tx_threshold < port8250.port.fifosize))
    port8250.tx_loadsz = port8250.port.fifosize - tx_threshold;
```

This code indicates that:

- The driver first reads `tx-threshold` from the device tree
- If the value is valid and **less than `fifo-size`**, it calculates:

```c
tx_loadsz = fifosize - tx_threshold
```

For the current K3 configuration:

- `fifo-size = 256`
- `tx-threshold = 32`

The result is:

```text
tx_loadsz = 256 - 32 = 224
```

What does this mean in practice?

It can be understood as follows:

- When the remaining TX FIFO space reaches a certain level, the 8250 transmit path fills data according to `tx_loadsz`
- For the current K3 configuration, the driver tends to load up to **224 bytes** into the TX FIFO at a time
- The remaining **32 bytes** can be regarded as reserved threshold space

Therefore, a more accurate understanding of `tx-threshold` here is:

- **A trigger/reserve threshold parameter for the transmit FIFO**
- It does not directly indicate "how much to send". Instead, it indirectly determines the `tx_loadsz` of the 8250 core through `fifosize - tx_threshold`

#### Combined Effect of the Two Parameters

Taken together, the effect is clear:

- `fifo-size = <256>`: Declares that the K3 UART hardware FIFO depth is 256 bytes
- `tx-threshold = <32>`: Specifies that the TX FIFO threshold is 32 bytes
- The driver then calculates:
  - `tx_loadsz = 224`

In other words, within the 8250 framework:

- the FIFO capability is treated as 256 bytes
- the batch write size to the FIFO is treated as 224 bytes

The purpose of this configuration is to let the driver take advantage of the large FIFO for better throughput while still preserving some threshold space, rather than filling the FIFO completely every time.

#### Practical Significance for Document Users

For customers or board-level developers, the following points are important when interpreting these two parameters:

1. `fifo-size` reflects the **hardware FIFO depth**; it is not recommended to arbitrarily increase or decrease this value.
2. `tx-threshold` reflects the **transmit threshold strategy** and directly affects the `tx_loadsz` of the 8250 driver.
3. `tx-threshold` must be less than `fifo-size`, otherwise the formula in `8250_of.c` will not take effect.
4. Incorrect configuration of these parameters may impact UART transmit performance and FIFO utilization. In severe cases, it can lead to reduced throughput or abnormal behavior.

## K3 UART Resource Overview

From the current contents of `k3.dtsi` and `k3-rdomain.dtsi`, the following can be summarized:

- **AP-domain UART**: `uart0` ~ `uart10`
- **RCPU-domain UART**: `r_uart0` ~ `r_uart5`
- **aliases**: `serial0` ~ `serial16`

Therefore, when mapping debug UARTs, device node names, and console output ports, always refer to the alias definitions.

## Debugging Suggestions

If UART is not working properly, it is recommended to check the following in order:

1. Confirm that pinctrl has been switched to the correct UART / RUART pin group
2. If it is used as a console, confirm that `stdout-path`, `aliases`, and `console=ttySx` are consistent
3. If it is an RCPU UART, confirm that the `spacemit,rcpu-uart` property exists
4. If external devices such as Bluetooth are connected, confirm that the relevant GPIOs are also configured correctly

## Summary

The key point of this K3 UART document is: **do not identify the active driver solely by looking for a file whose name appears to be UART-related**.

For the current K3 kernel, the correct references are:

- the actual `compatible` values in the DTSI:
  - `spacemit,k1-uart`
  - `intel,xscale-uart`
- the binding:
  - `Documentation/devicetree/bindings/serial/8250.yaml`
- the driver path:
  - `drivers/tty/serial/8250/8250_of.c`

Therefore, when documenting or using K3 UART, pay special attention to:

- **Distinguishing between AP-domain and RCPU-domain UARTs**
- **The `spacemit,rcpu-uart` property affects the clock-selection strategy**
- **Clock / reset / interrupt / pinctrl must be configured as a complete set**
- **Console serial ports must be understood together with `aliases` and `stdout-path`**
