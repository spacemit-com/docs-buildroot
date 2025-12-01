# PMIC

Regulator Functionality and Usage Guide.

## Overview

The term 'regulator' refers to a device that controls voltage and current output. SpacemiT P1 chip is a PMIC (Power Management Integrated Circuit) with this functionality. In the Linux kernel, the regulator framework provides a standardized interface for voltage and current control.

### Function

![](static/regulator.png)  

1. **Regulator Consumer:** Devices powered by regulators, which consume the electricity provided by the regulators.
2. **Regulator Framework:** Provides standard kernel interfaces to control the system's voltage/current regulators and 3. offers mechanisms for switching, voltage, and current settings.
3. **Regulator Driver:** The driver code for regulators, responsible for registering devices with the framework and communicating with the underlying hardware.
3. Machine: Configures regulator properties for the target hardware.

### Source Code Structure

```
drivers/regulator/
├── core.c
├── devres.c
├── dummy.c
├── dummy.h
├── fixed.c
├── fixed-helper.c
├── gpio-regulator.c
├── helpers.c
├── internal.h
├── irq_helpers.c
├── Kconfig
├── Makefile
├── of_regulator.c
├── spacemit-regulator.c
```

## Key Features

| Feature | Description |
| :-----| :----|
| 6-Channel DCDC Support  | Supports dynamic voltage adjustment and enable/disable |
| 5-Channel ALDO Support | Supports voltage adjustment and enable/disable |
| 7-Channel DLDO Support | Supports voltage adjustment and enable/disable |

## Configuration Introduction

It mainly includes driver enablement configuration and DTS configuration.

### CONFIG Configuration

```
CONFIG_REGULATOR_SPACEMIT:

 This driver provides support for the voltage regulators on the
 spacemit pmic.

 Symbol: REGULATOR_SPACEMIT [=y]
 Type  : tristate
 Defined at drivers/regulator/Kconfig:1666
  Prompt: Spacemit regulator support
  Depends on: REGULATOR [=y] && MFD_SPACEMIT_PMIC [=y]
  Location: 
   -> Device Drivers
    -> Voltage and Current Regulator Support (REGULATOR [=y])
     -> Spacemit regulator support (REGULATOR_SPACEMIT [=y])
  Selects: REGULATOR_FIXED_VOLTAGE [=y] 
```

### DTS Configuration

```
&i2c8 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_i2c8>;
        status = "okay";

        spm8821@41 {
                compatible = "spacemit,spm8821";
                reg = <0x41>;
                interrupt-parent = <&intc>;
                interrupts = <64>;
                status = "okay";

                vcc_sys-supply = <&vcc4v0_baseboard>;
                dcdc5-supply = <&dcdc_5>;

                regulators {
                        compatible = "pmic,regulator,spm8821";

                        /* buck */
                        dcdc_1: DCDC_REG1 {
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-always-on;
                        };

                        dcdc_2: DCDC_REG2 {
                                regulator-name = "dcdc2";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-always-on;
                        };

                        dcdc_3: DCDC_REG3 {
                                regulator-name = "dcdc3";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <1800000>;
                                regulator-always-on;
                        };

                        dcdc_4: DCDC_REG4 {
                                regulator-name = "dcdc4";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3300000>;
                                regulator-always-on;
                        };

                        dcdc_5: DCDC_REG5 {
                                regulator-name = "dcdc5";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-always-on;
                        };

                        dcdc_6: DCDC_REG6 {
                                regulator-name = "dcdc6";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-always-on;
                        };

                        /* aldo */
                        ldo_1: LDO_REG1 {
                                regulator-name = "ldo1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                                regulator-boot-on;
                        };

                        ldo_2: LDO_REG2 {
                                regulator-name = "ldo2";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_3: LDO_REG3 {
                                regulator-name = "ldo3";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_4: LDO_REG4 {
                                regulator-name = "ldo4";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        /* dldo */
                        ldo_5: LDO_REG5 {
                                regulator-name = "ldo5";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                                regulator-boot-on;
                        };

                        ldo_6: LDO_REG6 {
                                regulator-name = "ldo6";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_7: LDO_REG7 {
                                regulator-name = "ldo7";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_8: LDO_REG8 {
                                regulator-name = "ldo8";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                                regulator-always-on;
                        };

                        ldo_9: LDO_REG9 {
                                regulator-name = "ldo9";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_10: LDO_REG10 {
                                regulator-name = "ldo10";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                                regulator-always-on;
                        };

                        ldo_11: LDO_REG11 {
                                regulator-name = "ldo11";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        sw_1: SWITCH_REG1 {
                                regulator-name = "switch1";
                        };
                };
        };
};
```

## Interface

### API

Please refer to the kernel documentation:
```
- Documentation/power/regulator/consumer.rst
- Documentation/power/regulator/machine.rst
- Documentation/power/regulator/regulator.rst

```

### Demo Example

```
1. Configure the dts to reference the regulator you want to use:
    &cpu_0 {
        clst0-supply = <&dcdc_1>;
        vin-supply-names = "clst0";
    };

Obtain the corresponding handle in the code:
        const char *strings;
        struct regulator *regulator;
        err = of_property_read_string_array(cpu_dev->of_node, "vin-supply-names",
                        &strings, 1);
        regulator = devm_regulator_get(cpu_dev, strings);  --> The passed-in struct device * must have a corresponding entity
        
 Enable the corresponding regulator in the code:
         regulator_enable(regulator);
 
 Set the voltage of the corresponding regulator in the code:
         regulator_set_voltage(regulator, 95000000, 95000000);
```

## Debugging

## FAQ

## Appendix

### SPL/UBOOT Usage Method


```
uboot-2022.10$ vi arch/riscv/dts/k1-x_spm8821.dtsi

&i2c8 {
        clock-frequency = <100000>;
        u-boot,dm-spl;
        status = "okay";

        spm8821: pmic@41 {
                compatible = "spacemit,spm8821";
                reg = <0x41>;
                bus = <8>;
                u-boot,dm-spl;

                regulators {
                        /* buck */
                        dcdc_6: DCDC_REG1 {
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-init-microvolt = <950000>;
                                regulator-boot-on;
                                u-boot,dm-spl;
                                regulator-state-mem {
                                        regulator-off-in-suspend;
                                };
                        };

                        dcdc_7: DCDC_REG2 {
                                regulator-name = "dcdc2";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                        };

                        dcdc_8: DCDC_REG3 {
                                regulator-name = "dcdc3";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-boot-on;
                                u-boot,dm-spl;
                        };

                        dcdc_9: DCDC_REG4 {
                                regulator-name = "dcdc4";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                        };

                        dcdc_10: DCDC_REG5 {
                                regulator-name = "dcdc5";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                        };

                        dcdc_11: DCDC_REG6 {
                                regulator-name = "dcdc6";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                        };

                        /* aldo */
                        ldo_23: LDO_REG1 {
                                regulator-name = "ldo1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                                regulator-init-microvolt = <3300000>;
                                regulator-boot-on;
                                u-boot,dm-spl;
                        };

                        ldo_24: LDO_REG2 {
                                regulator-name = "ldo2";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_25: LDO_REG3 {
                                regulator-name = "ldo3";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_26: LDO_REG4 {
                                regulator-name = "ldo4";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        /* dldo */
                        ldo_27: LDO_REG5 {
                                regulator-name = "ldo5";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_28: LDO_REG6 {
                                regulator-name = "ldo6";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_29: LDO_REG7 {
                                regulator-name = "ldo7";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_30: LDO_REG8 {
                                regulator-name = "ldo8";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_31: LDO_REG9 {
                                regulator-name = "ldo9";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_32: LDO_REG10 {
                                regulator-name = "ldo10";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        ldo_33: LDO_REG11 {
                                regulator-name = "ldo11";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3400000>;
                        };

                        sw_2: SWITCH_REG1 {
                                regulator-name = "switch1";
                        };
                };
        };
};
```

#### SPL Stage Power-On and Voltage Setting Method

```c
                        dcdc_6: DCDC_REG1 {
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-init-microvolt = <950000>; ---> Adding this field will automatically set the power supply voltage to 0.95v
                                regulator-boot-on; ---> Adding this field will automatically turn on the power supply during the SPL stage
                                u-boot,dm-spl;   ---> This field is required for SPL to recognize the DTS node
                                regulator-state-mem {
                                        regulator-off-in-suspend;
                                };
                        };
```

#### UBOOT Stage Power-On and Voltage Setting Method

There are two ways to set or enable a power supply during the UBOOT stage. The first method is to configure it directly in the DTS:

```c
                        dcdc_6: DCDC_REG1 { --> The name parameter passed to regulator_get_by_devname
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-init-microvolt = <950000>; ---> Adding this field will automatically set the power supply voltage to 0.95v
                                regulator-boot-on; ---> Adding this field will automatically turn on the power supply during the UBOOT stage
                                regulator-state-mem {
                                        regulator-off-in-suspend;
                                };
                        };
```

The other method is to set it directly in the code:

```c
1. First, obtain the regulator handle for the voltage you want to set or enable
    struct udevice *rdev = NULL;
    char *regulator_name = "DCDC_REG1"  --> This field is the name of the DTS node specified in the DTS
    ret = regulator_get_by_devname(regulator_name, &rdev);
    
2. Enable a specific power supply
    regulator_set_enable(&rdev, true);
    
3. Set the voltage of a specific power supply
    regulator_set_value(&rdev, 1800000);
```
