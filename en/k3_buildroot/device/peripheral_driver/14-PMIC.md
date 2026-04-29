# PMIC

This document describes the PMIC (Power Management IC) features and usage on the K3 platform based on the RPMI protocol.

## Module Overview

The **PMIC (Power Management IC)** is an integrated circuit responsible for system power management. It controls power-rail enablement and voltage adjustment through the Regulator subsystem.

The K3 platform implements PMIC functionality through **RPMI (RISC-V Platform Management Interface)**. 
The RPMI PMIC communicates with firmware through a mailbox mechanism to provide dynamic power management.

### Functional Overview

K3 RPMI PMIC architecture:

```
User space / drivers
    ↓
Regulator framework
    ↓
RPMI regulator driver
    ↓
Mailbox subsystem
    ↓
MPXY Mailbox
    ↓
esos (Regulator)
    ↓
Hardware PMIC
```

The Regulator framework consists of the following components:

1. **Regulator Consumer**: a device that requires power from a regulator.
2. **Regulator Framework**: provides standard kernel interfaces for controlling system voltage and current regulators.
3. **Regulator Driver**: registers the device with the framework and communicates with the underlying hardware.
4. **Machine**: defines regulator properties such as voltage range and initial state.

### Source Tree Overview

The Regulator module is located under `drivers/regulator/` in the kernel source tree:

```
drivers/regulator/
├── core.c                  # Regulator framework core code
├── of_regulator.c          # Device Tree parsing
├── helpers.c               # Helper functions
├── regulator-rpmi.c        # K3 RPMI regulator driver
└── ...
```

### RPMI Regulator Types

RPMI supports two types of regulators:

1. **`RPMI_REGULATOR_DISCRETE`**: discrete-voltage type with fixed voltage levels.
2. **`RPMI_REGULATOR_LINEAR`**: linear-voltage type with a continuously adjustable voltage range.

### K3 Power Domains

Main power domains on the K3 platform include:

- **`edcdc1`** (or **`adcdc1`**): supplies power to the X100 core
- **`edcdc2`** (or **`adcdc2`**): supplies power to the A100 core
- **`dcdc1-6`**, **`aldo1-4`**, and **`dldo1-7`**: other DCDC and LDO rails
- **`pvin`**: main power input

> **Note:** In different board-level DTS files, `edcdc` may be named `adcdc`. The function is the same.

## Key Features

- Supports multiple DCDC and LDO power rails
- Supports dynamic voltage adjustment
- Supports power enable and disable control
- Uses RPMI-based communication
- Communicates with firmware through mailbox
- Supports linear voltage ranges and discrete voltage levels
- Supports power dependency management
- Supports dynamic voltage adjustment for CPU cores through DVFS

## Configuration

Configuration mainly consists of **kernel CONFIG options** and **DTS configuration**.

### Kernel CONFIG Options

#### Enable the Regulator Framework

`CONFIG_REGULATOR` enables support for the kernel Regulator framework.

```
Symbol: REGULATOR [=y]
Device Drivers
      -> Voltage and Current Regulator Support (REGULATOR [=y])
```

#### Enable the RPMI Regulator Driver

Enable `CONFIG_REGULATOR_RPMI` to support the K3 RPMI regulator driver.

```
Symbol: REGULATOR_RPMI [=y]
Device Drivers
      -> Voltage and Current Regulator Support
            -> RISC-V RPMI Regulator (REGULATOR_RPMI [=y])
```

### DTS Configuration

#### DTSI Configuration Example

Define the mailbox channel for the RPMI regulator in the `dtsi` file.

In general, this section **does not need to be modified**.

**RPMI regulator node (`k3.dtsi`)**

```dts
rpmi_regulator: rpmi_regulator@0 {
    compatible = "riscv,rpmi-regulator";
    mboxes = <&mpxy_mbox 0x0007 0x0>;
    status = "okay";
};
```

**Property descriptions:**

- `compatible`: compatibility string that identifies the device as an RPMI regulator
- `mboxes`: mailbox channel configuration used for firmware communication
    - `&mpxy_mbox`: mailbox controller
    - `0x0007`: regulator service ID
    - `0x0`: channel parameter

#### Board-Level DTS Configuration Example

Configure rail properties in the board-level DTS. The following example uses `k3_evb.dts`:

```dts
&rpmi_regulator {
    status = "okay";

    pvin: pvin {
        regulator-min-microvolt = <5000000>;
        regulator-max-microvolt = <5000000>;
        regulator-always-on;
        regulator-boot-on;
    };

    edcdc2: edcdc2 {
        /* A100 core supply */
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    edcdc1: edcdc1 {
        /* X100 core supply */
        regulator-min-microvolt = <534000>;
        regulator-max-microvolt = <10000000>;
        regulator-always-on;
        regulator-boot-on;
    };

    p3v3: p3v3 {
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    p1v8: p1v8 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc1: dcdc1 {
        regulator-min-microvolt = <1050000>;
        regulator-max-microvolt = <1050000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc3: dcdc3 {
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc4: dcdc4 {
        regulator-min-microvolt = <2100000>;
        regulator-max-microvolt = <2100000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc5: dcdc5 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc6: dcdc6 {
        regulator-min-microvolt = <500000>;
        regulator-max-microvolt = <600000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo1: aldo1 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo2: aldo2 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo3: aldo3 {
        regulator-min-microvolt = <500000>;
        regulator-max-microvolt = <3400000>;
    };

    aldo4: aldo4 {
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo1: dldo1 {
        regulator-min-microvolt = <1200000>;
        regulator-max-microvolt = <1200000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo2: dldo2 {
        regulator-min-microvolt = <900000>;
        regulator-max-microvolt = <900000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo3: dldo3 {
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo4: dldo4 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo5: dldo5 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo6: dldo6 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo7: dldo7 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };
};
```

**Common properties:**

| Property | Description |
| :-- | :-- |
| `regulator-name` | Power rail name. Optional; defaults to the node name. |
| `regulator-min-microvolt` | Minimum voltage in microvolts |
| `regulator-max-microvolt` | Maximum voltage in microvolts |
| `regulator-always-on` | Keeps the rail permanently enabled |
| `regulator-boot-on` | Enables the rail during boot |
| `regulator-ramp-delay` | Voltage transition delay in microseconds per volt |

## Interface Overview

### RPMI Service IDs

The RPMI regulator supports the following services:

| Service name | Description |
| :-- | :-- |
| `GET_NUM_DOMAINS` | Gets the number of power domains |
| `GET_ATTRIBUTES` | Gets power domain attributes |
| `GET_SUPPORTED_LEVELS` | Gets supported voltage levels |
| `SET_CONFIG` | Sets power configuration (enable / disable) |
| `GET_CONFIG` | Gets power configuration |
| `SET_LEVEL` | Sets the voltage |
| `GET_LEVEL` | Gets the voltage |

### Kernel API

The driver implements the standard `regulator_ops` interface:

```c
static const struct regulator_ops regulator_rpmi_ops = {
    .list_voltage = regulator_list_voltage_linear_range,
    .map_voltage = regulator_map_voltage_linear_range,
    .set_voltage_sel = regulator_rpmi_set_voltage_sel,
    .get_voltage_sel = regulator_rpmi_get_voltage_sel,
    .enable = regulator_rpmi_enable,
    .disable = regulator_rpmi_disable,
    .is_enabled = regulator_rpmi_is_enabled,
};
```

### User-Space Interface

#### `sysfs` Interface

Use `sysfs` to inspect and interact with regulators:

```bash
# View all regulators
ls /sys/class/regulator/

# View information for a specific regulator
cat /sys/class/regulator/regulator.0/name
cat /sys/class/regulator/regulator.0/type
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/state

# View the voltage range
cat /sys/class/regulator/regulator.0/min_microvolts
cat /sys/class/regulator/regulator.0/max_microvolts

# View the devices using this regulator
cat /sys/class/regulator/regulator.0/num_users
```

```bash
# View summary information for all regulators
cat /sys/kernel/debug/regulator/regulator_summary
```

#### Kernel Driver Usage Example

Example of regulator usage in a driver:

```c
#include <linux/regulator/consumer.h>

struct regulator *reg;
int ret;

/* Get the regulator */
reg = regulator_get(dev, "vdd");
if (IS_ERR(reg)) {
    dev_err(dev, "Failed to get regulator\n");
    return PTR_ERR(reg);
}

/* Set the voltage */
ret = regulator_set_voltage(reg, 1800000, 1800000);
if (ret) {
    dev_err(dev, "Failed to set voltage\n");
    goto err;
}

/* Enable the regulator */
ret = regulator_enable(reg);
if (ret) {
    dev_err(dev, "Failed to enable regulator\n");
    goto err;
}

/* Get the current voltage */
int uV = regulator_get_voltage(reg);
dev_info(dev, "Current voltage: %d uV\n", uV);

/* Disable the regulator */
regulator_disable(reg);

/* Release the regulator */
regulator_put(reg);

err:
    regulator_put(reg);
    return ret;
```

#### Reference a Regulator in Device Tree

Reference a regulator in a device node:

```dts
&i2c0 {
    sensor@48 {
        compatible = "example,sensor";
        reg = <0x48>;
        vdd-supply = <&dcdc1>;      /* References dcdc1 */
        vddio-supply = <&aldo1>;    /* References aldo1 */
    };
};
```

## Debugging

### Basic Checks

1. Check whether regulator devices are registered.

   ```bash
   ls /sys/class/regulator/
   ```

2. Check driver loading status.

   ```bash
   dmesg | grep -i regulator
   dmesg | grep -i rpmi
   ```

3. View all regulator information.

   ```bash
   cat /sys/kernel/debug/regulator/regulator_summary
   ```

### View Detailed Regulator Information

```bash
# Traverse all regulators
for reg in /sys/class/regulator/regulator.*; do
    echo "=== $(basename $reg) ==="
    echo "Name: $(cat $reg/name 2>/dev/null)"
    echo "Type: $(cat $reg/type 2>/dev/null)"
    echo "State: $(cat $reg/state 2>/dev/null)"
    echo "Voltage: $(cat $reg/microvolts 2>/dev/null) uV"
    echo "Min: $(cat $reg/min_microvolts 2>/dev/null) uV"
    echo "Max: $(cat $reg/max_microvolts 2>/dev/null) uV"
    echo "Users: $(cat $reg/num_users 2>/dev/null)"
    echo ""
done
```

### Test Program Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>

#define REGULATOR_PATH "/sys/class/regulator"

void print_regulator_info(const char *reg_name)
{
    char path[256];
    char buf[256];
    FILE *fp;

    printf("=== %s ===\n", reg_name);

    // Read name
    snprintf(path, sizeof(path), "%s/%s/name", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("Name: %s", buf);
        fclose(fp);
    }

    // Read state
    snprintf(path, sizeof(path), "%s/%s/state", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("State: %s", buf);
        fclose(fp);
    }

    // Read voltage
    snprintf(path, sizeof(path), "%s/%s/microvolts", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("Voltage: %s uV\n", buf);
        fclose(fp);
    }

    printf("\n");
}

int main(void)
{
    DIR *dir;
    struct dirent *entry;

    dir = opendir(REGULATOR_PATH);
    if (!dir) {
        perror("opendir");
        return -1;
    }

    while ((entry = readdir(dir)) != NULL) {
        if (strncmp(entry->d_name, "regulator.", 10) == 0) {
            print_regulator_info(entry->d_name);
        }
    }

    closedir(dir);
    return 0;
}
```

## Testing

### Voltage Setting Test

Test voltage setting through the kernel driver interface. A test driver is required.

```c
struct regulator *reg;

reg = regulator_get(dev, "dcdc1");
if (!IS_ERR(reg)) {
    /* Set voltage to 1.2 V */
    regulator_set_voltage(reg, 1200000, 1200000);
    regulator_enable(reg);
    
    /* Read the actual voltage */
    int uV = regulator_get_voltage(reg);
    pr_info("dcdc1 voltage: %d uV\n", uV);
    
    regulator_put(reg);
}
```

### Power Enable and Disable Test

Test power enable and disable behavior:

```bash
# View the power state
cat /sys/class/regulator/regulator.0/state

# Test enable and disable through driver code
# (requires calls to regulator_enable and regulator_disable in the driver)
```

## FAQ

### Regulator Device Does Not Exist

Check the following items:

1. Confirm that `CONFIG_REGULATOR` and `CONFIG_REGULATOR_RPMI` are enabled in the kernel configuration.
2. Confirm that the `rpmi_regulator` node in DTS has `status = "okay"`.
3. Check whether the mailbox driver has loaded correctly: `dmesg | grep -i mbox`
4. Check kernel logs: `dmesg | grep -i regulator`

### Voltage Setting Fails

Possible causes include:

1. The requested voltage is outside the range defined in DTS.
2. RPMI communication failed.
3. The firmware does not support the requested voltage value.

Recommended checks:

```bash
# Check the voltage range
cat /sys/class/regulator/regulator.X/min_microvolts
cat /sys/class/regulator/regulator.X/max_microvolts

# Check error logs
dmesg | grep -i "regulator\|rpmi"
```

### How to View Regulator Consumers

```bash
# View the number of devices using this regulator
cat /sys/class/regulator/regulator.X/num_users

# View the detailed regulator tree
cat /sys/kernel/debug/regulator/regulator_summary
```

### RPMI Communication Fails

If RPMI communication errors occur, check the following:

1. Check whether the mailbox driver is operating normally: `dmesg | grep mpxy`
2. Confirm that the firmware version supports the RPMI regulator.
3. Review detailed error messages: `dmesg | grep -i "rpmi\|regulator"`

### Power Dependency Configuration

Some power rails may depend on other rails and must be described in DTS:

```dts
dcdc1: dcdc1 {
    regulator-min-microvolt = <1050000>;
    regulator-max-microvolt = <1050000>;
    vin-supply = <&pvin>;  /* dcdc1 depends on pvin */
};
```

### How to Set Voltage During Boot

Configure `regulator-boot-on` and the voltage range in DTS:

```dts
dcdc1: dcdc1 {
    regulator-min-microvolt = <1200000>;
    regulator-max-microvolt = <1200000>;
    regulator-boot-on;      /* Automatically enabled during boot */
    regulator-always-on;    /* Always remains enabled */
};
```

## Notes

1. **RPMI dependency**: the RPMI regulator depends on firmware support. Make sure the firmware version is correct.
2. **Mailbox channel**: the regulator uses mailbox channel `0x0007`. Avoid conflicts with other devices.
3. **Voltage range**: configured voltages must stay within the `min` and `max` range defined in DTS.
4. **Power dependencies**: pay attention to dependencies between power rails and avoid circular dependencies.
5. **`always-on` property**: rails marked with `regulator-always-on` cannot be disabled.
6. **Voltage accuracy**: the actual voltage may differ slightly from the configured value, depending on the voltage levels supported by the hardware.
7. **Concurrent access**: concurrent access is handled by the Regulator framework, so no additional locking is required in the driver.
8. **CPU core supplies**:
   - `edcdc1` (or `adcdc1`) supplies the X100 core. Voltage changes require caution.
   - `edcdc2` (or `adcdc2`) supplies the A100 core. Voltage changes require caution.
   - The naming may differ across board-level DTS files (`edcdc` or `adcdc`), but the function is the same.
   - CPU voltage scaling is usually managed automatically by DVFS (Dynamic Voltage and Frequency Scaling).

## References

- Linux Regulator Framework documentation: `Documentation/power/regulator/`
- RPMI specification: *RISC-V Platform Management Interface Specification*
- Device Tree binding documentation: `Documentation/devicetree/bindings/regulator/`