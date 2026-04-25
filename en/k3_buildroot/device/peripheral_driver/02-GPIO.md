# GPIO

This document describes the functionality and usage of **K3 GPIO**.

## Module Overview

GPIO is **the controller that manages the GPIO module**, and is responsible for GPIO direction configuration, input/output level read and write operations, and GPIO interrupt handling.

### Functional Description

The Linux GPIO subsystem driver framework mainly consists of three parts:

- **GPIO Controller Driver**: Interacts with the GPIO controller hardware and performs register initialization and specific operations
- **GPIO lib Driver**: Provides standard GPIO APIs to other kernel modules, such as direction configuration, level read/write operations, and GPIO-to-IRQ conversion
- **GPIO Character Device Driver**: Exposes GPIOs to user space as character devices, allowing user space to access GPIOs through standard file interfaces

### Source Code Structure

The K3 GPIO driver code is located in the kernel `drivers/gpio` directory:

```text
drivers/gpio/
|-- gpio-spacemit-k1.c
```

The dt-binding file corresponding to K3 GPIO is:

```text
Documentation/devicetree/bindings/gpio/spacemit,k1-gpio.yaml
```

Although the filename is `k1-gpio.yaml`, K1 and K3 use the same GPIO IP, so this binding also serves as a reference for K3 GPIO.

The K3 GPIO controller node is located at:

```text
arch/riscv/boot/dts/spacemit/k3.dtsi
```

## Key Features

| Feature | Description |
| :----- | :---- |
| Direction setting support | Supports configuring GPIO as input or output |
| Input level read support | Supports reading the current GPIO input or output level |
| Output level setting support | Supports driving GPIO high or low in output mode |
| GPIO interrupt support | Supports GPIO interrupts triggered by rising edges, falling edges, and both edges |
| gpio-ranges support | Supports establishing GPIO-to-pin mapping with pinctrl |
| Bank-based management support | 4 banks, 32 GPIOs per bank, for a total of 128 GPIOs |
| gpio irqchip support | Provides a GPIO interrupt domain and nested IRQ handling through gpiolib irqchip |

## Configuration

This section mainly covers driver **CONFIG options** and **DTS configuration**.

### CONFIG Configuration

- **CONFIG_GPIOLIB**: Provides generic support for GPIO controllers, and is typically enabled by default as `Y`

```text
Device Drivers
        GPIO Support (GPIOLIB [=y])
```

- **CONFIG_GPIO_SPACEMIT_K1**: Provides support for the SpacemiT K1/K3 GPIO controller

```text
config GPIO_SPACEMIT_K1
    tristate "SPACEMIT K1/K3 GPIO support"
    depends on ARCH_SPACEMIT || COMPILE_TEST
    depends on OF_GPIO
    select GPIO_GENERIC
    select GPIOLIB_IRQCHIP
```

Corresponding Makefile entry:

```text
obj-$(CONFIG_GPIO_SPACEMIT_K1) += gpio-spacemit-k1.o
```

## Usage Guide

Using K3 GPIO can be divided into three parts:

1. **GPIO Controller Description**
2. **pinctrl configuration for the pin used by the GPIO**
3. **GPIO Reference in Device Node**

### GPIO Controller Description

The K3 GPIO controller node is located in `k3.dtsi` and is defined as follows:

```dts
gpio: gpio@d4019000 {
    compatible = "spacemit,k3-gpio";
    reg = <0x0 0xd4019000 0x0 0x100>;
    clocks = <&syscon_apbc CLK_APBC_GPIO>,
             <&syscon_apbc CLK_APBC_GPIO_BUS>;
    clock-names = "core", "bus";
    gpio-controller;
    #gpio-cells = <3>;
    interrupts = <58 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    interrupt-controller;
    #interrupt-cells = <3>;
    syscon-gpio-regs = <&syscon_gpio>;
 syscon-gpio-edge = <&syscon_gpio_edge>;
    gpio-ranges = <&pinctrl 0 0 0 32>,
                  <&pinctrl 1 0 32 32>,
                  <&pinctrl 2 0 64 32>,
                  <&pinctrl 3 0 96 32>;
};
```

This node indicates:

- `compatible = "spacemit,k3-gpio"`: matches the K3 GPIO driver
- `#gpio-cells = <3>`: GPIO references use 3 cells
- `#interrupt-cells = <3>`: GPIO interrupt references use 3 cells
- `gpio-ranges`: establishes the mapping between GPIO bank/offset values and pinctrl pins, and usually remains unchanged
- `syscon-gpio-regs`: GPIO controller register syscon
- `syscon-gpio-edge`: GPIO edge register syscon

### GPIO Numbering Description

The driver defines:

- `SPACEMIT_NR_BANKS = 4`
- `SPACEMIT_NR_GPIOS_PER_BANK = 32`

Therefore, the total number of K3 GPIOs is:

- **4 banks**
- **32 GPIOs per bank**
- **Totaling 128 GPIOs**

The mapping between banks and global GPIO numbers is as follows:

| Bank | GPIO Range |
| :-- | :-- |
| bank0 | 0 ~ 31 |
| bank1 | 32 ~ 63 |
| bank2 | 64 ~ 95 |
| bank3 | 96 ~ 127 |

The driver registers a separate `gpio_chip` for each bank and sets:

- `gc->base = index * 32`
- `gc->ngpio = 32`

### GPIO Pin Configuration

The GPIO controller itself is only responsible for direction, level, and interrupts. Attributes such as **whether a specific pin operates in GPIO mode**, along with pull-up/pull-down, drive strength, and voltage, must still be configured through **pinctrl**.

In other words, K3 GPIO pin configuration should refer to the K3 pinctrl documentation:

- `02-GPIO.md` describes the GPIO controller itself
- `01-PINCTRL.md` describes pin mux, bias, drive-strength, and power-source`

On K3, GPIO pins are generally configured for GPIO functionality through `K3_PADCONF(pin, func)` in the pinctrl state. Usually, the GPIO function corresponds to the default GPIO mux mode number of that pin.

For example, some pins are multiplexed to peripheral functions by default in `k3-pinctrl.dtsi`. If a board design requires them to be used as GPIOs, the corresponding pinctrl configuration should be added or overridden in the board DTS to switch those pins to GPIO functionality.

### GPIO Usage

#### Using GPIO as a Normal Output/Input

In a device node, K3 GPIOs are typically referenced as follows:

```dts
reset-gpios = <&gpio 3 4 GPIO_ACTIVE_LOW>;
```

The meanings of the three cells are:

1. **bank number**
2. **offset within the bank**
3. **GPIO flag**, such as `GPIO_ACTIVE_HIGH` / `GPIO_ACTIVE_LOW`

For example:

- `bank = 3`
- `offset = 4`
- Actual global GPIO number = `3 * 32 + 4 = 100`

Actual examples from the K3 board DTS are shown below:

1. Camera-related GPIO:

```dts
pwdn-gpios = <&gpio 0 7 GPIO_ACTIVE_HIGH>;
reset-gpios = <&gpio 0 6 GPIO_ACTIVE_HIGH>;
dptc-gpios = <&gpio 0 8 GPIO_ACTIVE_HIGH>;
```

2. Network PHY reset:

```dts
snps,reset-gpios = <&gpio 0 15 GPIO_ACTIVE_LOW>;
```

3. Bluetooth-controlled GPIO:

```dts
device-wake-gpios = <&gpio 2 30 GPIO_ACTIVE_HIGH>;
enable-gpios = <&gpio 2 29 GPIO_ACTIVE_HIGH>;
host-wake-gpios = <&gpio 2 28 GPIO_ACTIVE_HIGH>;
```

4. SD card detection GPIO:

```dts
cd-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
```

#### Using GPIO as an Interrupt

Devices can obtain a GPIO through a `*-gpios` property, and the driver can then call `gpiod_to_irq()` to convert it to an IRQ number, allowing the GPIO to be used as an interrupt source.

Example DTS:

```dts
my_device: my-device@0 {
   compatible = "vendor,my-device";
   event-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
};
```

Typical driver flow:

```c
/* 1. Get the GPIO descriptor */
struct gpio_desc *event_gpio;
event_gpio = devm_gpiod_get(&pdev->dev, "event", GPIOD_IN);
if (IS_ERR(event_gpio))
   return PTR_ERR(event_gpio);

/* 2. Convert the GPIO to an IRQ number */
int irq = gpiod_to_irq(event_gpio);
if (irq < 0)
   return irq;

/* 3. Request the interrupt */
ret = devm_request_threaded_irq(&pdev->dev, irq, NULL, my_irq_handler,
                        IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                        "my-event-irq", priv);
if (ret)
   return ret;
```

Notes:

1. The second parameter of `devm_gpiod_get()`, `"event"`, corresponds to the `event-gpios` property in DTS
2. `gpiod_to_irq()` converts the GPIO descriptor to a Linux IRQ number, and the mapping is handled by the GPIO irqchip layer
3. K3 GPIO interrupts support only edge-triggered interrupts, so trigger flags should use `IRQF_TRIGGER_RISING`, `IRQF_TRIGGER_FALLING`, or both

## dt-binding Description

The binding file currently used by K3 GPIO is:

```text
Documentation/devicetree/bindings/gpio/spacemit,k1-gpio.yaml
```

Although the filename contains `k1`, it also applies to K3, because K1 and K3 share the same GPIO IP and the current Linux kernel uses a single shared driver.

The key information in the binding is as follows:

### compatible

The compatible string in the binding file is:

```yaml
compatible:
  const: spacemit,k1-gpio
```

However, the actual DTS used for K3 is:

```dts
compatible = "spacemit,k3-gpio";
```

Therefore, on K3, refer to **the driver code and the actual K3 DTS implementation**.

### #gpio-cells

Binding description:

```yaml
"#gpio-cells":
  const: 3
```

Meaning:

- 1st cell: GPIO bank index
- 2nd cell: offset within the bank
- 3rd cell: GPIO flags

### #interrupt-cells

Binding description:

```yaml
"#interrupt-cells":
  const: 3
```

Meaning:

- 1st cell: GPIO bank index
- 2nd cell: offset within the bank
- 3rd cell: interrupt flags

The binding explicitly states:

- **Level-triggered interrupts are not supported**
- `IRQ_TYPE_LEVEL_HIGH` / `IRQ_TYPE_LEVEL_LOW` must not be used
- Edge-triggered interrupt types should be used instead

This is consistent with the driver implementation, which only provides register configuration for rising-edge and falling-edge triggers.

## Driver Implementation

### New GPIO Driver Shared by K1/K3

`of_match_table` of the current driver is as follows:

```c
static const struct of_device_id spacemit_gpio_dt_ids[] = {
    { .compatible = "spacemit,k1-gpio", .data = &k1_data },
    { .compatible = "spacemit,k3-gpio", .data = &k3_data },
    { /* sentinel */ }
};
```

This indicates:

- K1 matches `k1_data`
- K3 matches `k3_data`
- The core logic is shared, and the main differences are register offsets and bank layout

### K3 Register Layout

The register offset definitions for the K3 GPIO in the driver are as follows:

```c
static const struct spacemit_gpio_reg_offsets k3_regs = {
    .gplr    = 0x0,
    .gpdr    = 0x4,
    .gpsr    = 0x8,
    .gpcr    = 0xc,
    .grer    = 0x10,
    .gfer    = 0x14,
    .gedr    = 0x18,
    .gsdr    = 0x1c,
    .gcdr    = 0x20,
    .gsrer   = 0x24,
    .gcrer   = 0x28,
    .gsfer   = 0x2c,
    .gcfer   = 0x30,
    .gapmask = 0x34,
    .gcpmask = 0x38,
};
```

K3 bank offsets:

```c
static const u32 k3_bank_offsets[] = { 0x0, 0x40, 0x80, 0x100 };
```

Therefore, the register blocks for each bank are arranged separately: bank0, bank1, bank2, and bank3 start at `0x0`, `0x40`, `0x80`, and `0x100` respectively.

### Basic Operation Implementation

The basic GPIO operations in the driver are as follows:

- `spacemit_gpio_get()`: reads `GPLR` to obtain the GPIO level
- `spacemit_gpio_set()`: Writes to `GPSR`/`GPCR` to set the pin high or low
- `spacemit_gpio_direction_input()`: Writes to `GCDR` to set the input direction
- `spacemit_gpio_direction_output()`: Writes to `GSDR` to set the output direction, then sets the output value

These implementations follow standard Linux GPIO semantics.

### GPIO Interrupt Implementation

K3 GPIO uses `gpiolib irqchip` mode to implement GPIO interrupts.

Key points in the driver:

1. Configure a `gpio_irq_chip` for each bank
2. The main controller interrupt is registered through `devm_request_threaded_irq()`
3. Interrupt handler `spacemit_gpio_irq_handler()`:
   - Reads `GEDR` to get pending bits
   - Clears pending bits with W1C
   - Performs a bitwise AND with `irq_mask` to obtain valid interrupts
   - Calls `handle_nested_irq()` for each set bit
4. `irq_set_type` only configures:
   - Rising edge
   - Falling edge
   - Both edges

The driver does not implement level-triggered interrupts. Therefore, GPIO interrupts are only applicable to edge-triggered scenarios.

### Interrupt Mask / Unmask Behavior

The driver maintains the following state for each bank:

- `irq_mask`
- `irq_rising_edge`
- `irq_falling_edge`

When interrupts are masked or unmasked:

- `irq_mask()`: updates `gapmask` and clears the corresponding edge enable bits
- `irq_unmask()`: restores edge enable bits and then updates `gapmask`

This behavior matches the hardware edge-detection register design.

## K3 vs K1 Differences

Although K1 and K3 share the same GPIO IP and use a common new driver, the following differences should still be noted:

1. **The GPIO driver in the current kernel is the new driver**
   - File: `drivers/gpio/gpio-spacemit-k1.c`
   - Simultaneously supports K1/K3
   - This driver should be treated as the authoritative implementation

2. **Do not directly copy historical implementation descriptions from legacy K1 GPIO documentation**
   - The legacy drivers and usage patterns described in K1 legacy documentation are for historical reference only
   - The K3 documentation is written around the current standardized driver

3. **The differences between K1/K3 are mainly reflected in the register layout**
   - The `reg_offsets` of K1 and K3 are different
   - The `bank_offsets` of K1 and K3 are different
   - The driver distinguishes them via `k1_data` / `k3_data`

4. **K3 dts compatible is `spacemit,k3-gpio`**
   - Although the binding file name is still `spacemit,k1-gpio.yaml`
   - The actual writing for K3 has switched to `spacemit,k3-gpio`

## Interface Description

### Kernel API

Common GPIO subsystem interfaces are as follows:

- **Request a specified GPIO (legacy interface)**

```c
int gpio_request(unsigned gpio, const char *label);
```

- **Release a specified GPIO (legacy interface)**

```c
void gpio_free(unsigned gpio);
```

- **Set GPIO to input mode**

```c
int gpio_direction_input(unsigned gpio);
```

- **Set GPIO to output mode and assign an initial value**

```c
int gpio_direction_output(unsigned gpio, int value);
```

- **Set the output value of GPIO**

```c
void gpio_set_value(unsigned gpio, int value);
```

- **Get GPIO level value**

```c
int gpio_get_value(unsigned gpio);
```

- **Convert GPIO to IRQ**

```c
int gpio_to_irq(unsigned gpio);
```

### Recommended Descriptor Interfaces

In new drivers and new device drivers, descriptor-style interfaces are strongly recommended. For example:

```c
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id,
                            enum gpiod_flags flags);
```

and:

```c
int gpiod_direction_output(struct gpio_desc *desc, int value);
int gpiod_direction_input(struct gpio_desc *desc);
int gpiod_get_value(struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);
```

In DTS, these correspond to properties such as `reset-gpios`, `enable-gpios`, and `cd-gpios`.

## Debugging

### Character Device / Userspace Interface

For modern Linux GPIO, character device interfaces are recommended. They are typically visible in the system as:

```text
/dev/gpiochipX
```

User space can use `libgpiod` tools for inspection and operation.

### sysfs

The path of the traditional sysfs GPIO interface is as follows:

```text
/sys/class/gpio
```

Common nodes:

- `export`
- `unexport`
- `gpiochipX`
- `gpioX/direction`
- `gpioX/value`

However, sysfs GPIO is a legacy interface, and the character device method is recommended for debugging.

### debugfs

To view GPIO controllers and GPIO usage information in the system:

```text
/sys/kernel/debug/gpio
```

This can be used to inspect:

- gpiochip number
- GPIO users
- direction
- current high/low level
- flags such as ACTIVE LOW

### Register Debugging

You can also use `devmem` to inspect GPIO controller register values and help verify whether direction, level, and interrupt configuration have taken effect:

```bash
devmem reg_addr
```

Key registers that can be viewed include:

- `GPLR`: Level register
- `GPDR`: Direction register
- `GPSR` / `GPCR`: Set/Clear Register
- `GRER` / `GFER`: Rising/Falling Edge Detect Register
- `GEDR`: Interrupt Status Register
- `GAPMASK`: Interrupt Mask Register

## Testing

### Normal GPIO Test

1. Confirm that the corresponding pin has been switched to GPIO function in pinctrl
2. Configure the GPIO as input/output in the driver or user space
3. Read or set the GPIO level
4. Verify that the configuration has taken effect by checking both registers and debugfs

### GPIO Interrupt Test

1. In the DTS, specify the device interrupt source as `interrupt-parent = <&gpio>`
2. `interrupts = <bank offset IRQ_TYPE_EDGE_*>`
3. Trigger a level edge change of the corresponding GPIO
4. Check whether the device interrupt handler is entered
5. If necessary, check the `GEDR`, `GRER`, `GFER`, and `GAPMASK` registers

## FAQ

### 1. Why can't K3 GPIO documentation directly copy K1 legacy GPIO documentation?

Because the GPIO driver in the current kernel has been reconstructed into a new standardized driver: `gpio-spacemit-k1.c`, which supports both K1 and K3. The K1 legacy documentation describes historical implementations and should not be directly applied to K3.

### 2. Why is the dt-binding filename `spacemit,k1-gpio.yaml`, but K3 DTS uses `spacemit,k3-gpio`?

Because K1 and K3 share the same GPIO IP, and the binding file has not been split into a separate K3 file. For K3, the **current driver code and actual K3 DTS implementation** should be treated as authoritative.

### 3. Does K3 GPIO interrupt support level trigger?

No. Both the binding and driver implementation indicate that GPIO interrupts only support edge triggering, and `IRQ_TYPE_LEVEL_HIGH` or `IRQ_TYPE_LEVEL_LOW` should not be used.

### 4. Why does GPIO configuration not take effect?

The most common cause is not the GPIO controller itself, but that the corresponding pin has not been switched to GPIO function via pinctrl. You need to first check whether the pinctrl configuration is correct.
