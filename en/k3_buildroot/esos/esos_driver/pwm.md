# PWM

This document describes the PWM module on the K3 secondary-core platform (ESOS/RTT), including its role, software architecture, device tree usage, and application programming model.

## Module Overview

PWM (Pulse Width Modulation) is used to generate pulse waveforms with configurable period and duty cycle. On the K3 ESOS secondary-core platform, the PWM module provides the following functions:

- Enabling and disabling PWM output
- Configuring the period and pulse width
- Automatically probing and registering PWM devices through the DTS `compatible` property
- Providing a unified PWM interface through the RT-Thread PWM device model
- Working with the pinctrl subsystem to multiplex the PWM function onto the target PAD

On the current platform, PWM controllers use compatible strings from `spacemit,k1x-rpwm0` through `spacemit,k1x-rpwm9`.

## Software Architecture

The following diagram illustrates the high-level call path from application or component driver code down to the hardware layer:

```text
Application / component drivers
├─ rt_device_find("rpwm0")
├─ rt_pwm_set()
├─ rt_pwm_enable()
├─ rt_pwm_disable()
└─ rt_pwm_get()

                │
                ▼
RT-Thread PWM framework
├─ components/drivers/misc/rt_drv_pwm.c
└─ components/drivers/include/drivers/rt_drv_pwm.h

                │
                ▼
SoC PWM driver layer
└─ bsp/spacemit/drivers/pwm/pwm-pxa.c

                │
                ▼
PWM controller hardware
└─ rpwm0 ~ rpwm9

                │
                ▼
pinctrl / PAD multiplexing
└─ The PWM output PAD is selected through `pinctrl-0`
```

## API Reference

### RT-Thread PWM API Overview

When using the RT-Thread PWM interface, the PWM device handle is typically obtained by device name first:

- `rpwm0` ~ `rpwm9`: PWM device names on the K3 secondary-core platform

**Register a PWM device**

```c
rt_err_t rt_device_pwm_register(struct rt_device_pwm *device,
                                const char *name,
                                const struct rt_pwm_ops *ops,
                                const void *user_data);
```

Parameters:

- `device`: pointer to the PWM device object
- `name`: device name, for example `"rpwm0"`
- `ops`: PWM operation table
- `user_data`: pointer to private user data

Return value:

- `RT_EOK`: registration succeeded
- other error codes: registration failed

**Enable PWM output**

```c
rt_err_t rt_pwm_enable(struct rt_device_pwm *device, int channel);
```

Parameters:

- `device`: PWM device handle
- `channel`: PWM channel number

Return value:

- `RT_EOK`: operation succeeded
- other error codes: operation failed

**Disable PWM output**

```c
rt_err_t rt_pwm_disable(struct rt_device_pwm *device, int channel);
```

Parameters:

- `device`: PWM device handle
- `channel`: PWM channel number

Return value:

- `RT_EOK`: operation succeeded
- other error codes: operation failed

**Set the PWM period and pulse width**

```c
rt_err_t rt_pwm_set(struct rt_device_pwm *device,
                    int channel,
                    rt_uint32_t period,
                    rt_uint32_t pulse);
```

Parameters:

- `device`: PWM device handle
- `channel`: PWM channel number
- `period`: PWM period (ns)
- `pulse`: PWM pulse width (ns) and must satisfy `pulse <= period`

Return value:

- `RT_EOK`: configuration succeeded
- other error codes: configuration failed

### PWM Control Commands

**Enable command**

```c
#define PWM_CMD_ENABLE      (128 + 0)
```

Description:

- Enables PWM output

Return value:

- None

**Disable command**

```c
#define PWM_CMD_DISABLE     (128 + 1)
```

Description:

- Disables PWM output

Return value:

- None

**Set command**

```c
#define PWM_CMD_SET         (128 + 2)
```

Description:

- Sets the PWM period and pulse width

Return value:

- None

**Get command**

```c
#define PWM_CMD_GET         (128 + 3)
```

Description:

- Retrieves the PWM configuration

Return value:

- None

## Usage Examples

### Device Tree Node Example

```dts
&rpwm0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rpwm0_0_cfg>;
    status = "okay";
};
```

### Driver Usage Example

```c
struct rt_device_pwm *pwm_dev;

pwm_dev = (struct rt_device_pwm *)rt_device_find("rpwm0");
if (pwm_dev == RT_NULL)
    return -RT_ERROR;

rt_pwm_set(pwm_dev, 1, 100000, 50000);
rt_pwm_enable(pwm_dev, 1);
```

### Disable Output Example

```c
rt_pwm_disable(pwm_dev, 1);
```

### Shell Command Example

```sh
pwm_set rpwm0 1 100000 50000
pwm_enable rpwm0 1
```