# PWM

This document describes the functionality and usage of PWM.

## Module Overview

PWM (Pulse Width Modulation) outputs pulse signals with configurable period and duty cycle, and is commonly used for scenarios such as backlight dimming, motor speed control, and buzzer driving.

### Functional Overview

The Linux PWM subsystem typically consists of two parts:

- **PWM controller driver**: Responsible for accessing hardware registers and configuring the period, duty cycle, enable state, and related settings.
- **PWM consumer driver**: Other functional modules request PWM through the `pwms` property and configure output parameters according to their own use cases.

On K3, the PWM controller driver reuses the PXA-series PWM driver framework, which SpacemiT has extended to support the PWM controllers on K1/K3 SoCs.

### Source Code Structure

The K3 PWM-related code and description files are mainly located at:

```text
Linux-6.18/
|-- drivers/pwm/pwm-pxa.c
|-- drivers/pwm/Kconfig
|-- drivers/pwm/Makefile
|-- Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3-pinctrl.dtsi
```

Where:

- `drivers/pwm/pwm-pxa.c` is the actual driver file.
- `Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml` is the dt-binding currently in use.
- `k3.dtsi` defines the PWM controller nodes in the AP domain.
- `k3-rdomain.dtsi` defines the RPWM controller nodes in the RCPU domain.
- `k3-pinctrl.dtsi` provides the pin multiplexing configuration for PWM/RPWM.

## Relationship Between the K1 and K3 Drivers

This point needs to be distinguished clearly.

In the old K1 kernel (for example, `k1-sdk/linux-6.6`), the common `compatible` value for PWM nodes is:

```dts
compatible = "spacemit,k1x-pwm";
```

The corresponding driver is still:

```text
drivers/pwm/pwm-pxa.c
```

In the current K3 kernel, `k3-sdk/linux-6.18`, the PWM nodes use:

```dts
compatible = "spacemit,k1-pwm", "marvell,pxa910-pwm";
```

The corresponding driver is still:

```text
drivers/pwm/pwm-pxa.c
```

In addition, SpacemiT's extended `compatible` string has also been merged into the dt-binding:

```text
Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml
```

Therefore:

- **K1 and K3 currently use the same PWM driver implementation.**
- **K3 does not have a separately written PWM driver.**
- The differences are mainly reflected in **DTS `compatible` strings, clock/reset resources, number of nodes, power domains, and board-level pinmux usage**.

## Key Features

| Feature | Description |
| :----- | :----- |
| Period configuration support | Supports setting the PWM period in nanoseconds |
| Duty-cycle configuration support | Supports setting the duty cycle |
| Enable/disable support | Controls output uniformly through the PWM framework |
| Multi-channel PWM controller support | K3 provides multiple PWM channels in both the AP and RCPU domains |
| Standard consumer usage support | Other devices can request PWM through `pwms = <...>` |
| Normal polarity only | The driver accepts only `PWM_POLARITY_NORMAL` |

## Configuration

This mainly includes driver `CONFIG` enablement and DTS configuration.

### CONFIG Configuration

The core configuration options required by K3 PWM are:

- `CONFIG_PWM`
- `CONFIG_PWM_SYSFS` (if debugging uses sysfs)
- `CONFIG_PWM_PXA`

In `drivers/pwm/Kconfig`:

```text
config PWM_PXA
	tristate "PXA PWM support"
	depends on ARCH_PXA || ARCH_MMP || ARCH_SPACEMIT || COMPILE_TEST
```

This indicates that the SpacemiT platform uses the `PWM_PXA` driver.

### DTS Configuration

#### 1. PWM Controller Node

The AP-domain PWM nodes on K3 are defined in `arch/riscv/boot/dts/spacemit/k3.dtsi`, for example:

```dts
pwm0: pwm@d401a000 {
    compatible = "spacemit,k1-pwm", "marvell,pxa910-pwm";
    reg = <0x0 0xd401a000 0x0 0x10>;
    clocks = <&syscon_apbc CLK_APBC_PWM0>,<&syscon_apbc CLK_APBC_PWM0_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_apbc RESET_APBC_PWM0>;
    #pwm-cells = <3>;
    k1,pwm-disable-fd;
    status = "disabled";
};
```
The RCPU-domain RPWM nodes are defined in `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`, for example:

```dts
rpwm0: pwm@c088d100 {
    compatible = "spacemit,k1-pwm", "marvell,pxa910-pwm";
    reg = <0x0 0xc088d100 0x0 0x10>;
    clocks = <&syscon_rcpu_pwmctrl CLK_RCPU_PWMCTRL_RPWM0>,
             <&syscon_rcpu_pwmctrl CLK_RCPU_PWMCTRL_RPWM0_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_rcpu_pwmctrl RESET_RCPU_PWMCTRL_PWM0>;
    #pwm-cells = <3>;
    k1,pwm-disable-fd;
    status = "disabled";
};
```

#### 2. dt-binding Description

The current binding file is:

```text
Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml
```

The SpacemiT `compatible` definition in this binding is:

```yaml
compatible:
  oneOf:
    - items:
        - const: spacemit,k1-pwm
        - const: marvell,pxa910-pwm
```

The binding also specifies:

When `compatible` includes `spacemit,k1-pwm`, `#pwm-cells` must be set to `3`.

This is also the source of the actual setting used in the K3 `.dtsi` files.

#### 3. Meaning of `#pwm-cells = <3>`

The K3 PWM controller node uses:

```dts
#pwm-cells = <3>;
```

Therefore, the `pwms` property on the consumer side usually consists of 3 cells:

```dts
pwms = <&pwmX channel period_ns polarity>;
```

For such single-channel PWM controllers, the `channel` is usually set to `0`.

Therefore, a typical example is:

```dts
pwms = <&pwm0 0 50000 PWM_POLARITY_NORMAL>;
```

The cells indicate:

1. `&pwm0`: Which PWM controller is referenced
2. `0`: PWM channel number
3. `50000`: PWM period, in ns
4. `PWM_POLARITY_NORMAL`: PWM polarity; the driver supports only this value

Note the PWM period constraints:
- Minimum value: Must be greater than 0
- Maximum value: It is limited by the input clock frequency `f_clk`. The maximum period is calculated as: `T_max = (10^9 * 64 * 1024) / f_clk` (unit: ns)
- Overflow check: The driver internally calculates the total period cycles. If `(period_ns * f_clk) / 10^9` exceeds `65536 (64 * 1024)`, the driver returns `-EINVAL`.

#### 4. `k1,pwm-disable-fd` Property Description

PWM nodes in the K3 `.dtsi` files commonly include:

```dts
k1,pwm-disable-fd;
```

The corresponding parsing logic in `drivers/pwm/pwm-pxa.c` is:

```c
if (of_get_property(pdev->dev.of_node, "k1,pwm-disable-fd", NULL))
    pc->dcr_fd = 0;
else
    pc->dcr_fd = 1;
```

Therefore, this property is used to control whether `PWMDCR_FD`-related behavior is disabled. The K3 `.dtsi` files include this property by default, which indicates that the platform disables this behavior by default and configures PWM according to the platform-specific adaptation.

This also indicates that although the base driver comes from `pwm-pxa.c`, SpacemiT has added custom extensions. Therefore, the documentation should follow the actual DTS usage on the current platform.

## Pin Configuration

To output PWM signals to external pins, in addition to enabling the PWM controller node, the corresponding pin must also be switched to PWM function mode in the pinctrl configuration.

The K3 PWM/RPWM pinmux configurations are mainly defined in:

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

For example:

```dts
pwm0_0_cfg: pwm0-0-cfg {
    pwm0-0-pins {
        pinmux = <K3_PADCONF(0, 3)>;    /* pwm0 */
    };
};

pwm0_1_cfg: pwm0-1-cfg {
    pwm0-0-pins {
        pinmux = <K3_PADCONF(42, 6)>;   /* pwm0 */
    };
};
```

This indicates that the same PWM controller on K3 may have multiple alternative pinmux options.

Another example is in the RCPU domain:

```dts
rpwm0_0_cfg: rpwm0-0-cfg {
    rpwm0-0-pins {
        pinmux = <K3_PADCONF(15, 3)>;   /* rpwm0 */
    };
};

rpwm0_1_cfg: rpwm0-1-cfg {
    rpwm0-0-pins {
        pinmux = <K3_PADCONF(27, 3)>;   /* rpwm0 */
    };
};
```

Therefore, when using PWM in practice, both of the following must be completed:

1. Enable the corresponding PWM controller node
2. Select the correct pinctrl configuration in the device or board-level DTS

## Usage Guide

### Enable the PWM Controller

To enable a specific PWM channel, the corresponding node in the board-level `.dts` file usually needs to be set to `okay`, for example:

```dts
&pwm0 {
    status = "okay";
};
```

If an RCPU-domain PWM is used, the process is similar:

```dts
&rpwm0 {
    status = "okay";
};
```

### Selecting Pin Multiplexing

At the same time, select the correct pinctrl configuration for the corresponding peripheral device or board-level node. For example, if the board design uses the `pwm0_0_cfg` multiplexing option, the relevant pinctrl state must be referenced in the corresponding location.

If PWM is used only as a standalone output, a common practice is to reference the corresponding pinctrl state directly in the board-level file from the PWM function node or a related peripheral node.

### Using PWM as a Consumer

In the current K3 board-level DTS files, the directly visible `pwms` instances mainly come from EC-provided PWM rather than the SoC-integrated PWM. For example:

```dts
pwms = <&cros_ec_pwm 0>;
```

This indicates that:

- The SoC-integrated PWM controller nodes are already defined in `k3.dtsi` / `k3-rdomain.dtsi`
- Whether a specific SoC PWM is actually used in the board-level DTS depends on the product design

Therefore, when adding new SoC PWM functionality, customers usually need to add the `pwms` reference to the consumer node in the board-level DTS.

A conceptual example is shown below:

```dts
pwm-test {
    compatible = "pwm-backlight";
    pwms = <&pwm0 0 50000 PWM_POLARITY_NORMAL>;
    brightness-levels = <0 64 128 255>;
    default-brightness-level = <3>;
    status = "okay";
};
```

If the scenario also requires PWM0 to be output to an external pin, ensure that the corresponding pinmux is configured for the `pwm0` function.

## Driver Implementation

The core K3 PWM driver implementation is in `drivers/pwm/pwm-pxa.c`.

### 1. Period and Duty Cycle Calculation

The driver comments provide the formauls:

```c
/*
 * period_ns = 10^9 * (PRESCALE + 1) * (PV + 1) / PWM_CLK_RATE
 * duty_ns   = 10^9 * (PRESCALE + 1) * DC / PWM_CLK_RATE
 */
```

Based on the requested `period` and `duty_cycle`, the driver calculates:

- `prescale`
- `pv`
- `dc`

These values are then written to:

- `PWMCR`
- `PWMDCR`
- `PWMPCR`

### 2. Only Normal Polarity is Supported

In `pxa_pwm_apply()`:

```c
if (state->polarity != PWM_POLARITY_NORMAL)
    return -EINVAL;
```

Therefore, if the consumer side attempts to configure inverted polarity, the driver directly returns an error.

### 3. Clock Enable Sequence

Before configuring PWM parameters, the driver first does:

```c
clk_prepare_enable(pc->clk);
```

After configuration, it decides whether to disable the clock based on the current state. Therefore, if the clock resources are not configured correctly in the DTS, PWM cannot output normally.

### 4. Handling 100% Duty Cycle

When `duty_ns == period_ns`, the driver enters a special path to handle `PWMDCR_FD`-related logic.
SpacemiT has added the DTS property `k1,pwm-disable-fd` to control this behavior. Therefore, this implementation is not exactly the same as the traditional PXA documentation.

## K3 PWM Node Overview

From `k3.dtsi` and `k3-rdomain.dtsi`, K3 includes at least two types of PWM:

- **AP-domain PWM**: `pwm0` ~ `pwm19`
- **RCPU-domain PWM**: `rpwm0` ~ `rpwm9`

This indicates that K3 provides many PWM resources. In actual use, pay special attention to:

- Whether the controller being used belongs to the AP domain or the RCPU domain
- Which syscon provides the corresponding clock/reset resources
- Whether the corresponding pinmux can be routed to the required board-level pins

## Debugging Suggestions

If PWM has no output, debug it in the following order:

1. Confirm that the kernel has enabled `CONFIG_PWM_PXA`
2. Confirm that the corresponding `pwmX` or `rpwmX` node state is set to `okay`
3. Confirm that `clocks`, `clock-names`, and `resets` are configured correctly
4. Confirm that the corresponding pinctrl has been switched to the PWM function
5. Confirm that the `pwms` format in the consumer node matches `#pwm-cells = <3>`
6. Confirm that `PWM_POLARITY_INVERSED` is not used
7. For scenarios involving a 100% duty cycle, note the effect of the `k1,pwm-disable-fd` property on behavior