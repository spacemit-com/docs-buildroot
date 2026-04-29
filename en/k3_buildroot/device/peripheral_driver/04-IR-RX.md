# IR-RX

This document describes IR-RX (infrared receiver) configuration and debugging.

## Module Overview

The IR-RX (infrared receiver) module handles infrared signal reception.

### Feature Overview

On the K3 platform:

- an external infrared receiver head (demodulator) captures the demodulated electrical signal
- the driver and the kernel IR framework decode the signal
- the decoded event is then reported to the upper layer

**IR-RX workflow:**

```
Infrared remote control
    ↓ (infrared signal)
Infrared receiver head (demodulator)
    ↓ (electrical signal)
IR-RX controller
    ↓
IR-RX driver
    ↓
Kernel RC framework
    ↓
IR decoder (NEC/RC5/RC6, etc.)
    ↓
Input subsystem
    ↓
User-space application
```

### Source Tree

The IR-RX controller driver code is located under `drivers/media/rc`:

```
drivers/media/rc
|--rc-core.c              # RC framework core code
|--rc-ir-raw.c            # kernel IR framework interface code
|--ir-nec-decoder.c       # NEC protocol decoder
|--ir-rc5-decoder.c       # RC5 protocol decoder
|--ir-rc6-decoder.c       # RC6 protocol decoder
|--ir-spacemit-k1.c       # K3 IR-RX driver
```

### Hardware Features

The K3 IR-RX controller provides the following features:

- Supports 4 IR-RX controllers:
    - 2 in the main domain (`ircrx0`, `ircrx1`)
    - 2 in the R-domain (`r_ircrx0`, `r_ircrx1`)
- 32-byte RX FIFO
- Configurable noise threshold
- Operating frequency: 100 kHz
- Base clock frequency: 102.4 MHz
- Supports multiple infrared protocols, including NEC, RC5, and RC6

## Key Features

- Supports 4 infrared receive channels:
    - 2 in the main domain
    - 2 in the R-domain
- 32-byte RX FIFO
- Configurable noise threshold
- Supports decoding of multiple infrared protocols
- Interrupt-driven operation
- Integrated with the Linux RC framework

## Configuration

Configuration mainly includes:

- **Kernel CONFIG settings**
- **DTS settings**

### Kernel CONFIG Settings

#### Enable the RC Framework

`CONFIG_RC_CORE` enables support for the kernel RC (Remote Controller) framework.

```
Symbol: RC_CORE [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
```

#### Enable IR Decoders

Enable the required IR protocol decoders. For example, the NEC decoder:

```
Symbol: IR_NEC_DECODER [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
            -> Enable IR raw decoder for the NEC protocol (IR_NEC_DECODER [=y])
```

#### Enable the K3 IR-RX Driver

Enable `CONFIG_IR_SPACEMIT_K1` to support the K3 IR-RX driver.

```
Symbol: IR_SPACEMIT_K1 [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
            -> Remote Controller devices (RC_DEVICES [=y])
                  -> Spacemit K1/K3 IR remote receiver control (IR_SPACEMIT_K1 [=y])
```

### DTS Settings

#### DTSI Configuration Example

The `dtsi` file defines the IR-RX controller register address, clock source, and reset resource.
Generally, this section does **not** need to be modified.

**IR-RX controller nodes (`k3.dtsi`)**

```dts
ircrx0: irc-rx0@d4017e00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017e00 0x0 0x100>;
    interrupts = <69 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR0>;
    resets = <&syscon_apbc RESET_APBC_IR0>;
    clock-frequency = <102400000>;
    status = "disabled";
};

ircrx1: irc-rx1@d4017f00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017f00 0x0 0x100>;
    interrupts = <20 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR1>;
    resets = <&syscon_apbc RESET_APBC_IR1>;
    clock-frequency = <102400000>;
    status = "disabled";
};
```

**R-domain IR-RX controller nodes (`k3-rdomain.dtsi`)**

The K3 platform also provides 2 additional IR-RX controllers in the R-domain (RCPU domain). They are defined in `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`:

```dts
r_ircrx0: r-irc-rx0@c0887000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc0887000 0x0 0x100>;
    interrupts = <274 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC0>;
    status = "disabled";
};

r_ircrx1: r-irc-rx1@c088e000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc088e000 0x0 0x100>;
    interrupts = <275 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC1>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC1>;
    status = "disabled";
};
```

Main differences from the main-domain nodes:

- The register base address is located in the R-domain address space (`0xc08xxxxx`)
- Clock and reset resources come from `syscon_rcpu_sysctrl` instead of `syscon_apbc`
- No `clock-frequency` property is present
- Interrupt numbers are different (`274`, `275`)

**Property description:**

- `compatible`: compatibility string that identifies the SpacemiT K1/K3 IR controller
- `reg`: register base address and size
- `interrupts`: interrupt number
- `clocks`: clock resource
- `resets`: reset resource
- `clock-frequency`: base clock frequency (`102.4 MHz`, main-domain nodes only)

#### Board-Level DTS Configuration Example

Enable IR-RX and configure `pinctrl` in the board DTS.

**Main-domain IR-RX:**

```dts
&ircrx0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ir0_0_cfg>;  /* use a predefined pinctrl configuration */
    linux,rc-map-name = "rc-empty";  /* or specify a specific remote-control map */
    status = "okay";
};
```

**R-domain IR-RX:**

```dts
&r_ircrx0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rir0_0_cfg>;  /* use a predefined R-domain pinctrl configuration */
    linux,rc-map-name = "rc-empty";
    status = "okay";
};
```

**Optional property:**

- `linux,rc-map-name`: specifies the remote-control keymap
    - `rc-empty`: empty map (default)
    - `rc-nec-terratec-cinergy-xs`: specific remote-control map
    - Supported keymaps can be found in `drivers/media/rc/keymaps/`

#### `pinctrl` Configuration

The K3 platform provides multiple predefined IR-RX `pinctrl` groups in `k3-pinctrl.dtsi`.

**Optional `ircrx0` configurations:**

```dts
/* Configuration 1: GPIO 41, function 5 */
ir0_0_cfg: ir0-0-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(41, 5)>;  /* ir0 rx */
    };
};

/* Configuration 2: GPIO 108, function 4 */
ir0_1_cfg: ir0-1-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(108, 4)>;  /* ir0 rx */
    };
};
```

**Optional `ircrx1` configurations:**

```dts
/* Configuration 1: GPIO 0, function 4 */
ir1_0_cfg: ir1-0-cfg {
    ir1-0-pins {
        pinmux = <K3_PADCONF(0, 4)>;  /* ir1 rx */
    };
};

/* Configuration 2: GPIO 70, function 4 */
ir1_1_cfg: ir1-1-cfg {
    ir1-0-pins {
        pinmux = <K3_PADCONF(70, 4)>;  /* ir1 rx */
    };
};
```

**Optional R-domain `ircrx` configurations:**

```dts
/* rir0 configuration 1: GPIO 40, function 5 */
rir0_0_cfg: rir0-0-cfg {
    rir0-0-pins {
        pinmux = <K3_PADCONF(40, 5)>;  /* rir0 rx */
    };
};

/* rir0 configuration 2: GPIO 71, function 4 */
rir0_1_cfg: rir0-1-cfg {
    rir0-0-pins {
        pinmux = <K3_PADCONF(71, 4)>;  /* rir0 rx */
    };
};
```

**`K3_PADCONF` macro definition:**

```c
#define K3_PADCONF(pin, func) (((pin) << 16) | (func))
```

- `pin`: GPIO pin number
- `func`: multiplexing function number

**Optional `pinctrl` properties:**

```dts
ir0_0_cfg: ir0-0-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(41, 5)>;
        bias-pull-up;           /* pull-up */
        drive-strength = <25>;  /* drive strength DS8 */
        power-source = <1800>;  /* power voltage 1.8V */
    };
};
```

> **Note**: Select the appropriate `pinctrl` group based on the hardware schematic, such as `ir0_0_cfg`, `ir0_1_cfg`, `ir1_0_cfg`, or `ir1_1_cfg`.

## Interface Description

### Kernel API

The IR-RX driver uses the interfaces provided by the Linux RC framework:

#### Device Registration

```c
struct rc_dev *rc_allocate_device(enum rc_driver_type type);
int rc_register_device(struct rc_dev *dev);
void rc_unregister_device(struct rc_dev *dev);
void rc_free_device(struct rc_dev *dev);
```

#### Event Reporting

```c
/* Store and filter raw IR events */
int ir_raw_event_store_with_filter(struct rc_dev *dev, struct ir_raw_event *ev);

/* Handle raw IR events and trigger decoding */
void ir_raw_event_handle(struct rc_dev *dev);
```

#### Raw IR Event Structure

```c
struct ir_raw_event {
    unsigned pulse:1;      /* 1 = pulse, 0 = idle */
    unsigned duration:31;  /* duration in microseconds */
};
```

### User-Space Interface

#### Input Device Node

The IR-RX driver is registered as an input device and can be accessed through the standard input subsystem:

```bash
# View input devices
cat /proc/bus/input/devices

# View IR device events
evtest /dev/input/eventX
```

#### `sysfs` Interface

`sysfs` can be used to inspect and configure RC devices:

```bash
# View RC devices
ls /sys/class/rc/

# View supported protocols
cat /sys/class/rc/rc0/protocols

# Enable a specific protocol
echo nec > /sys/class/rc/rc0/protocols

# View the current keymap
cat /sys/class/rc/rc0/keymap
```

## Debugging

### Basic Checks

1. Check whether the IR-RX device is registered

   ```bash
   ls /sys/class/rc/
   cat /sys/class/rc/rc0/name
   ```

2. Check driver load status

   ```bash
   dmesg | grep -i "ir\|spacemit"
   ```

3. Check the input device

   ```bash
   cat /proc/bus/input/devices | grep -A 10 "spacemit-ir"
   ```

### Infrared Signal Test

Use `ir-keytable` to verify infrared reception:

```bash
# Install ir-keytable if it is not available
# apt-get install ir-keytable

# View RC device information
ir-keytable

# Test infrared reception while pressing remote-control buttons
ir-keytable -t

# Clear the keymap
ir-keytable -c

# Load a keymap
ir-keytable -w /etc/rc_keymaps/nec.toml
```

### Test with `evtest`

```bash
# Install evtest
# apt-get install evtest

# List input devices
evtest

# Test the IR device by selecting the corresponding event node
evtest /dev/input/event0

# Press remote-control buttons and observe the output
```

### Test Program Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <linux/input.h>

int main(void)
{
    int fd;
    struct input_event ev;

    fd = open("/dev/input/event0", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    printf("Waiting for IR events...\n");

    while (1) {
        if (read(fd, &ev, sizeof(ev)) == sizeof(ev)) {
            if (ev.type == EV_KEY) {
                printf("Key %s: code=%d, value=%d\n",
                       ev.value ? "pressed" : "released",
                       ev.code, ev.value);
            }
        }
    }

    close(fd);
    return 0;
}
```

## Test Procedure

### Hardware Connection

1. Prepare an infrared receiver head, such as VS1838B or IRM-3638T.
2. Connect the infrared receiver head to the K3 development board:
   - VCC → 3.3V
   - GND → GND
    - OUT → IR-RX GPIO pin
3. Prepare an infrared remote control that uses the NEC protocol.

### Functional Test

1. **Basic reception test**

    ```bash
    # Enable the NEC protocol
    echo nec > /sys/class/rc/rc0/protocols
   
    # Test reception
    ir-keytable -t
   
    # Press remote-control buttons and observe the output
    ```

2. **Keymap verification**

    ```bash
    # Test with evtest
    evtest /dev/input/event0
   
    # Press remote-control buttons and inspect key values
    ```

3. **Protocol switching test**

    ```bash
    # View supported protocols
    cat /sys/class/rc/rc0/protocols
   
    # Switch to the RC5 protocol
    echo rc5 > /sys/class/rc/rc0/protocols
   
    # Test an RC5 remote control
    ir-keytable -t
    ```

## FAQ

### IR-RX Device Does Not Exist

Check the following items:

1. Confirm that `CONFIG_RC_CORE`, `CONFIG_IR_NEC_DECODER`, and `CONFIG_IR_SPACEMIT_K1` are enabled in the kernel configuration.
2. Confirm that the `ircrx` node status is set to `"okay"` in DTS.
3. Check whether the `pinctrl` configuration is correct.
4. Check the kernel log: `dmesg | grep -i "ir\|spacemit"`

### No Infrared Signal Is Received

Possible causes:

1. The infrared receiver head is connected incorrectly or is faulty.
2. The GPIO pin configuration is incorrect.
3. The protocol does not match. The remote control uses a different protocol from the one enabled in the driver.
4. Power supplied to the infrared receiver head is insufficient.

Recommended checks:

```bash
# Check protocol configuration
cat /sys/class/rc/rc0/protocols

# Enable all protocols
echo "nec rc5 rc6" > /sys/class/rc/rc0/protocols

# Check interrupt count
cat /proc/interrupts | grep ir

# Check hardware connections
```

### Incorrect Key Values Are Received

Possible causes:

1. The keymap does not match.
2. Protocol decoding is incorrect.

Recommended checks:

```bash
# Clear the existing keymap
ir-keytable -c

# Use ir-keytable in learning mode
ir-keytable -t

# Create a custom keymap based on the reported scancodes
```

### How to Create a Custom Keymap

1. Use `ir-keytable -t` to capture remote-control scancodes.
2. Create a mapping file in TOML format:

   ```toml
   [[protocols]]
   name = "my_remote"
   protocol = "nec"
   
   [protocols.scancodes]
   0x00 = "KEY_POWER"
   0x01 = "KEY_VOLUMEUP"
   0x02 = "KEY_VOLUMEDOWN"
   0x03 = "KEY_MUTE"
   ```

3. Load the mapping:

   ```bash
   ir-keytable -w /path/to/my_remote.toml
   ```

### Supported Infrared Protocols

The K3 IR-RX driver supports all protocols provided by the Linux RC framework. Common protocols include:

- NEC
- RC5
- RC6
- Sony SIRC
- JVC
- Sanyo
- Sharp

To view supported protocols:

```bash
cat /sys/class/rc/rc0/protocols
```

### How to Debug Infrared Signals

Enable kernel debug output:

```bash
# Enable RC framework debugging
echo 8 > /proc/sys/kernel/printk
echo "file drivers/media/rc/* +p" > /sys/kernel/debug/dynamic_debug/control

# View debug output
dmesg -w
```

## Notes

1. **Hardware connection**: Ensure that the infrared receiver head is connected correctly, especially the `VCC`, `GND`, and `OUT` pins.
2. **Protocol matching**: The protocol used by the remote control must match the protocol enabled in the driver.
3. **GPIO configuration**: The IR-RX GPIO pin must be configured correctly in `pinctrl`.
4. **Interrupt conflicts**: Ensure that the IR-RX interrupt numbers do not conflict with other devices.
5. **Power interference**: Infrared receiver heads are sensitive to power noise. A filter capacitor is recommended.
6. **Distance and angle**: Infrared reception range and angle are limited. Device placement should be considered during testing.
7. **Ambient light interference**: Strong light may affect infrared reception. Avoid testing under strong light when possible.

## References

- Linux RC framework documentation: `Documentation/media/rc-core.rst`
- IR protocol documentation: `Documentation/media/rc-protocols.rst`
- Keymap files: `drivers/media/rc/keymaps/`