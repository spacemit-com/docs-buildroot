# PMIC

本文介绍 Linux 中 Regulator 子系统的功能及使用方法。

## 模块介绍

Regulator（电源调节器）子系统主要用于控制能够输出电压和电流的硬件模块，我司P1芯片就是一款包含该功能的PMIC。对于 Linux 内核来说，regulator 是一套标准的软件框架，提供统一的接口用于控制各类电压/电流输出设备。

### 功能介绍

![](static/regulator.png)  

1. **regulator consumer：** 有调节器供电的设备，他们消耗调节器提供的电力
2. **regulator framework：** 提供标准的内核接口，控制系统的voltage/current regulators，并提供相应的开关、电压/电流设置的机制
3. **regulator diver：** regulator 的驱动代码，负责向 framework 注册设备，并且与底层硬件通讯
4. **machine：** 主要是配置各个regulator的属性，如电压范围、初始状态等。

### 源码结构介绍

Regulator 模块在内核源码中的路径为 `drivers/regulator/`，目录结构如下：

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

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 6路 DCDC  | 支持动态调压：enable/disable |
| 支持 5路 ALDO | 支持调压：enable/disable |
| 支持 7路 DLDO | 支持调压：enable/disable |

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

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

### DTS 配置

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

## 接口描述

### API 介绍

```
请参考内核文档：
Documentation/power/regulator/consumer.rst
Documentation/power/regulator/machine.rst
Documentation/power/regulator/regulator.rst

```

### Demo 示例

1. 配置dts，引用想要使用的regulator

```
    &cpu_0 {
        clst0-supply = <&dcdc_1>;
        vin-supply-names = "clst0";
    };
```

2. 代码中获得相应的句柄：

```
        const char *strings;
        struct regulator *regulator;
        err = of_property_read_string_array(cpu_dev->of_node, "vin-supply-names",
                        &strings, 1);
        regulator = devm_regulator_get(cpu_dev, strings);  --> 传入的struct device *必须有实体对应
```

3. 代码中使能相应的regulator:

```
         regulator_enable(regulator);
```

4. 代码中设置相应的regulator的电压：

```
         regulator_set_voltage(regulator, 95000000, 95000000);
```

## Debug介绍
待补充

## FAQ
待补充

## 附录

### SPL/U-Boot 配置说明

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

#### SPL 阶段电源开启及电压设置方法

```c
                        dcdc_6: DCDC_REG1 {
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-init-microvolt = <950000>; ---> 加上该字段，会自动设置该电源电压为0.95v
                                regulator-boot-on; ---> 加上该字段，在SPL阶段会自动打开该电源
                                u-boot,dm-spl;   ---> 需要加该字段，SPL才可识别该dts node
                                regulator-state-mem {
                                        regulator-off-in-suspend;
                                };
                        };
```

#### U-Boot 阶段电源开启及电压设置方法

U-Boot 阶段有两种方式设置或者开启电源

方式一：直接在 DTS 中配置
```c
                        dcdc_6: DCDC_REG1 { --> regulator_get_by_devname传入的名字参数
                                regulator-name = "dcdc1";
                                regulator-min-microvolt = <500000>;
                                regulator-max-microvolt = <3450000>;
                                regulator-init-microvolt = <950000>; ---> 加上该字段，会自动设置该电源电压为0.95v
                                regulator-boot-on; ---> 加上该字段，在UBOOT阶段会自动打开该电源
                                regulator-state-mem {
                                        regulator-off-in-suspend;
                                };
                        };
```

方式二：通过代码动态设置

```c
1. 首先要获得想要设置或者开启电压的regulator句柄
    struct udevice *rdev = NULL;
    char *regulator_name = "DCDC_REG1"  --> 该字段传入的是dts中标识的dts node的名字
    ret = regulator_get_by_devname(regulator_name, &rdev);
    
2. 开启某一路电
    regulator_set_enable(&rdev, true);
    
3. 设置某一路电的电压
    regulator_set_value(&rdev, 1800000);
```
