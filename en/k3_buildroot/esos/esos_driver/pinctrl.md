# PINCTRL

This document describes the pinctrl module on the K3 secondary-core platform (ESOS/RTT), including its role, software architecture, device tree usage, and application programming model.

## Module Overview

The pinctrl subsystem is responsible for SoC PAD multiplexing and electrical configuration. On the K3 ESOS secondary-core platform, the pinctrl module provides the following functions:

- Looking up the state associated with a device through `pinctrl-*` properties in the device tree
- Parsing `pinctrl-single,pins` entries in pinctrl child nodes
- Writing `<register offset, configuration value>` pairs into PAD registers
- Allowing consumers to switch pin configurations by state name
- Working with the GPIO subsystem through the GPIO range mechanism

On the current platform, the pinctrl controller uses the `pinctrl-single` driver model.

## Software Architecture

The following diagram illustrates the high-level call path from application or peripheral driver code down to the hardware layer:

```text
Application / peripheral drivers
├─ pinctrl_get()
├─ pinctrl_lookup_state()
└─ pinctrl_select_state()

                │
                ▼
pinctrl core
├─ components/drivers/pinctrl/core.c
├─ components/drivers/pinctrl/devicetree.c
└─ components/drivers/pinctrl/pinmux.c

                │
                ▼
SoC pinctrl driver
└─ bsp/spacemit/drivers/pinctrl/pinctrl-single.c

                │
                ▼
PAD / pinmux registers
└─ Register values are programmed through `pinctrl-single,pins`
```

## API Reference

### API Overview

- **Get the pinctrl handle for a device**

```c
struct pinctrl *pinctrl_get(struct dtb_node *dev);
```

Parameters:

- `dev`: pointer to the device tree node

Return value:

- success: returns a `struct pinctrl *` handle
- failure: returns an error pointer

- **Look up a pinctrl state by name**

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                           const char *name);
```

Parameters:

- `p`: pinctrl handle returned by `pinctrl_get()`
- `name`: state name, for example `"default"` or `"sleep"`

Return value:

- success: returns a `struct pinctrl_state *` handle
- failure: returns an error pointer

- **Select and apply a pinctrl state**

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state);
```

Parameters:

- `p`: pinctrl handle returned by `pinctrl_get()`
- `state`: state handle returned by `pinctrl_lookup_state()`

Return value:

- `0`: applied successfully
- negative value: apply failed

- **Apply a named pinctrl state to a node**

```c
int pinctrl_apply_state(struct dtb_node *node, const char *state);
```

Parameters:

- `node`: pointer to the device tree node
- `state`: name of the state to apply, for example `"default"` or `"sleep"`

Return value:

- `0`: applied successfully
- negative value: apply failed

- **Apply the `default` state**

```c
int pinctrl_apply_default(struct dtb_node *node);
```

Parameters:

- `node`: pointer to the device tree node

Return value:

- `0`: applied successfully
- negative value: apply failed

- **Apply the `sleep` state**

```c
int pinctrl_apply_sleep(struct dtb_node *node);
```

Parameters:

- `node`: pointer to the device tree node

Return value:

- `0`: applied successfully
- negative value: apply failed

### Device Tree Properties

- **pinctrl controller `compatible`**

```dts
compatible = "pinctrl-single";
```

Description:

- `compatible`: driver match string for the pinctrl controller

Return value:

- None

- **pinctrl state names**

```dts
pinctrl-names = "default";
```

Description:

- `pinctrl-names`: list of state names

Return value:

- None

- **pinctrl state reference**

```dts
pinctrl-0 = <&ruart0_0_cfg>;
```

Description:

- `pinctrl-0`: reference to the pinctrl configuration node for state index 0

Return value:

- None

- **`pinctrl-single` register configuration entries**

```dts
pinctrl-single,pins = <offset value offset value ...>;
```

Description:

- `offset`: offset of the PAD configuration register
- `value`: configuration value written to that register

Return value:

- None

### `K3_PADCONF` Macro Usage

- **`K3_PADCONF` macro syntax**

```dts
K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
```

Description:

- `GPIO_134`: PAD identifier
- `MUX_MODE4`: multiplexing function selection
- `EDGE_NONE`: edge detection configuration
- `PULL_UP`: pull-up/pull-down configuration
- `PAD_DS8`: drive strength configuration

Return value:

- Generates an `<offset value>` pair for use in `pinctrl-single,pins`

## Usage Examples

### Device Tree State Definition Example

```dts
ruart0_0_cfg: ruart0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_135, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
    >;
};
```

### Device Node Reference Example

```dts
&ruart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ruart0_0_cfg>;
    status = "okay";
};
```