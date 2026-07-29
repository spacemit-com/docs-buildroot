# GPIO

This document describes the GPIO module on the K3 secondary-core platform (ESOS/RTT), including its role, software architecture, device tree usage, and application programming model.

## Module Overview

GPIO (General-Purpose Input/Output) provides standard digital input and output capabilities. On the K3 ESOS secondary-core platform, the GPIO module is responsible for the following:

- Reading GPIO inputs and controlling GPIO outputs
- Configuring GPIO interrupts and dispatching callbacks
- Parsing `*-gpios` properties in the device tree
- Exposing a unified `rt_pin_*()` interface through the RT-Thread PIN device framework
- Working with the pinctrl subsystem to map the GPIO numbering space to the PAD/pin space

The current platform includes two main types of GPIO controllers:

- AP GPIO: the primary GPIO controller
- R GPIO: the GPIO controller on the secondary-core side

## Software Architecture

The following diagram illustrates the high-level call path from application code down to the hardware layer:

```text
Application / component drivers
├─ RT-Thread PIN interface
  ├─ rt_pin_mode()
  ├─ rt_pin_read()
  ├─ rt_pin_write()
  ├─ rt_pin_attach_irq()
  └─ rt_pin_irq_enable()

                │
                ▼
GPIO core / OF parsing layer
├─ components/drivers/gpio/gpiolib.c
└─ components/drivers/gpio/gpiolib-of.c

                │
                ▼
SoC GPIO driver layer
└─ bsp/spacemit/drivers/gpio/spacemit-gpio.c

                │
                ▼
GPIO controller hardware
├─ AP GPIO
└─ R GPIO

                │
                ▼
pinctrl / PAD configuration
└─ Mapped to pinctrl through mechanisms such as `gpio-ranges`
```

## API Reference

When using the RT-Thread PIN interface, the `pin` parameter is a logical pin number, that is, a global GPIO number.

- AP GPIO number range: `0 ~ 127`
- R GPIO number range: `128 ~ 163`

For example:

- `pin = 57` represents AP GPIO 57
- `pin = 128` represents R GPIO 0
- `pin = 129` represents R GPIO 1


**Set pin mode**

```c
void rt_pin_mode(rt_base_t pin, rt_base_t mode);
```

Parameters:

- `pin`: logical pin number, that is, the global GPIO number
- `mode`: pin mode. Supported values are:
  - `PIN_MODE_OUTPUT`: output mode
  - `PIN_MODE_INPUT`: input mode

Return value:

- None

**Write pin output value**

```c
void rt_pin_write(rt_base_t pin, rt_base_t value);
```

Parameters:

- `pin`: logical pin number, that is, the global GPIO number
- `value`: output level
  - `PIN_LOW`: low level
  - `PIN_HIGH`: high level

Return value:

- None

**Read pin level**

```c
int rt_pin_read(rt_base_t pin);
```

Parameters:

- `pin`: logical pin number, that is, the global GPIO number

Return value:

- `0`: low level
- `1`: high level
- negative value: read operation failed

**Attach a pin interrupt callback**

```c
rt_err_t rt_pin_attach_irq(rt_int32_t pin,
                           rt_uint32_t mode,
                           void (*hdr)(void *args),
                           void *args);
```

Parameters:

- `pin`: logical pin number, that is, the global GPIO number
- `mode`: interrupt trigger mode. Supported values are:
  - `PIN_IRQ_MODE_RISING`: rising-edge trigger
  - `PIN_IRQ_MODE_FALLING`: falling-edge trigger
  - `PIN_IRQ_MODE_RISING_FALLING`: both-edge trigger
- `hdr`: interrupt callback function
- `args`: private argument passed to the callback

Return value:

- `RT_EOK`: attached successfully
- other error codes: attachment failed

**Enable or disable a pin interrupt**

```c
rt_err_t rt_pin_irq_enable(rt_base_t pin, rt_uint32_t enabled);
```

Parameters:

- `pin`: logical pin number, that is, the global GPIO number
- `enabled`: interrupt control
  - `PIN_IRQ_ENABLE`: enable interrupt
  - `PIN_IRQ_DISABLE`: disable interrupt

Return value:

- `RT_EOK`: operation succeeded
- other error codes: operation failed

**Get a pin number by name**

```c
rt_base_t rt_pin_get(const char *name);
```

Parameters:

- `name`: pin name string, for example `PA.0` or `P0.12`

Return value:

- success: returns the pin number
- failure: returns an invalid pin number or an error code

## Usage Examples

### RT-Thread PIN Interface Example

```c
#define TEST_PIN 128

rt_pin_mode(TEST_PIN, PIN_MODE_OUTPUT);
rt_pin_write(TEST_PIN, PIN_HIGH);
rt_pin_write(TEST_PIN, PIN_LOW);
```

### GPIO Interrupt Example

```c
static void gpio_irq_handler(void *args)
{
    rt_kprintf("gpio irq trigger\n");
}

rt_pin_mode(128, PIN_MODE_INPUT);
rt_pin_attach_irq(128, PIN_IRQ_MODE_RISING, gpio_irq_handler, RT_NULL);
rt_pin_irq_enable(128, PIN_IRQ_ENABLE);
```

### GPIO Test Commands

The GPIO test code is located at:

```text
bsp/spacemit/drivers/gpio/gpio-test.c
```

This file is used to verify GPIO input, output, and interrupt functionality for both AP GPIO and R GPIO.

**`gpio_test_0001`: GPIO input test**

Description:

- Verifies AP GPIO input on AP GPIO `57`
- Verifies R GPIO input on logical pin `128`, which maps to R GPIO `0`
- Configures the pins as inputs with `rt_pin_mode()`
- Reads the input level with `rt_pin_read()`

Usage:

```sh
gpio_test_0001
```

**`gpio_test_0002`: GPIO output test**

Description:

- Verifies AP GPIO output on AP GPIO `58`
- Verifies R GPIO output on logical pin `129`, which maps to R GPIO `1`
- Drives both high and low levels with `rt_pin_write()`
- Reads back the output state with `rt_pin_read()`

Usage:

```sh
gpio_test_0002
```

**`gpio_test_0003`: GPIO rising-edge interrupt test**

Description:

- Verifies AP GPIO rising-edge interrupts by using AP GPIO `58` as the output to trigger an interrupt on AP GPIO `57`
- Verifies R GPIO rising-edge interrupts by using logical pin `129` (R GPIO `1`) as the output to trigger an interrupt on logical pin `128` (R GPIO `0`)
- Triggers rising-edge interrupts by toggling the output pin
- Each test group runs 3 times by default

Usage:

```sh
gpio_test_0003
```

Before running the test:

- Connect the AP GPIO output test pin to the AP GPIO input test pin
- Connect the R GPIO output test pin to the R GPIO input test pin

The test pins are currently defined as follows:

- AP GPIO input: `57`
- AP GPIO output: `58`
- R GPIO input: logical pin `128`, which maps to R GPIO `0`
- R GPIO output: logical pin `129`, which maps to R GPIO `1`

**`gpio_test_0004`: GPIO falling-edge interrupt test**

Description:

- Verifies AP GPIO falling-edge interrupts by using AP GPIO `58` as the output to trigger an interrupt on AP GPIO `57`
- Verifies R GPIO falling-edge interrupts by using logical pin `129` (R GPIO `1`) as the output to trigger an interrupt on logical pin `128` (R GPIO `0`)
- Triggers falling-edge interrupts by toggling the output pin
- Each test group runs 3 times by default

Usage:

```sh
gpio_test_0004
```

Before running the test:

- Connect the AP GPIO output test pin to the AP GPIO input test pin
- Connect the R GPIO output test pin to the R GPIO input test pin

**`gpio_test_0005`: GPIO both-edge interrupt test**

Description:

- Verifies AP GPIO both-edge interrupts by using AP GPIO `58` as the output to trigger an interrupt on AP GPIO `57`
- Verifies R GPIO both-edge interrupts by using logical pin `129` (R GPIO `1`) as the output to trigger an interrupt on logical pin `128` (R GPIO `0`)
- Verifies both rising-edge and falling-edge triggers in each test round
- Each test group runs 3 times by default, and 6 total interrupt events are expected

Usage:

```sh
gpio_test_0005
```

Before running the test:

- Connect the AP GPIO output test pin to the AP GPIO input test pin
- Connect the R GPIO output test pin to the R GPIO input test pin

**`gpio_test_all`: Run all GPIO tests**

Description:

- Runs `gpio_test_0001` through `gpio_test_0005` in sequence

Usage:

```sh
gpio_test_all
```
