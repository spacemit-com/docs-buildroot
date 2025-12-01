# GPIO

介绍GPIO的功能和使用方法。

## 模块介绍

GPIO是 **管理 GPIO模块的控制器**

### 功能介绍

![](static/linux_gpio.png)  

Linux gpio子系统驱动框架主要有三部分组成:  

- **GPIO 控制器驱动**: 与GPIO控制器进行交互，对控制器进行初始化和操作  
- **GPIO lib驱动**: 提供标准的API给其它模块使用，如设置GPIO 方向、读写GPIO电平状态  
- **GPIO 字符设备驱动**: 将GPIO以字符设备的上报给用户空间，用户空间可以通过标准的文件接口访问GPIO

### 源码结构介绍

控制器驱动代码位于 `drivers/gpio` 目录下：  

```
|-- gpio-k1x.c  
```

## 关键特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持方向设置 | 支持将GPIO的设置成输入或输出 |
| 支持输出设置高低电平 | 支持在GPIO输出模式下，设置GPIO的电平高低 |
| 支持gpio中断功能 | 支持上升沿和下降沿触发gpio中断 |

## 配置介绍

主要包括驱动 **CONFIG 使能配置** 和 **dts配置**

### CONFIG配置

- **CONFIG_GPIOLIB**: 为 GPIO控制器提供支持，默认值为 `Y`

```
Device Drivers
        GPIO Support (GPIOLIB [=y])
```

- **CONFIG_GPIO_K1X**: 为 K1 GPIO 控制器提供支持，默认值为 `Y`

```
Device Drivers  
        GPIO Support (GPIOLIB [=y])
                Virtual GPIO drivers  
                        SPACEMIT-K1X GPIO support (GPIO_K1X [=y])
```

## 使用介绍

GPIO 的使用分三步：

1. GPIO 描述

2. GPIO Pin 配置

3. GPIO 使用

说明:

1. pin 编号在内核目录 `include\dt-bindings\pinctrl\k1-x-pinctrl.h` 中定义。
2. pin 配置相同表示一组 pin 设置成 gpio 功能时配置相同，即 mux mode、上下拉、边沿检测、驱动能力配置相同。

### GPIO 描述

描述方案中使用的所有 GPIO。

采用Linux GPIO框架 `gpio-ranges` 属性进行定义。如果某段 GPIO 对应的 pin编号也连续，则定义为一组。

上述例子方案 dts文件 GPIO控制器定义

```c
&gpio{
        gpio-ranges = <
                &pinctrl 49  GPIO_49  2
                &pinctrl 58  GPIO_58  1
                &pinctrl 63  GPIO_63  5
                &pinctrl 70  PRI_TDI  4
                &pinctrl 74  GPIO_74  1
                &pinctrl 80  GPIO_80  4
                &pinctrl 90  GPIO_90  3
                &pinctrl 96  DVL0     2
                &pinctrl 110 GPIO_110 1
                &pinctrl 114 GPIO_114 3
                &pinctrl 123 GPIO_123 5
        >;
};
```

### GPIO pin 配置

将方案使用的 GPIO 对应的pin设置成 GPIO功能，并进行配置(边沿检测/上下拉/驱动能力)。

采用 `pinctrl-single,gpio-range` 属性设置。如果存在某段pin编号连续且配置相同，则配置为一组。

pin配置参数参考[pin配置参数](01-PINCTRL.md)。

例如：

```c
&pinctrl {
        pinctrl-single,gpio-range = <
                &range GPIO_49  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range GPIO_58  1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_63  2 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_65  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_66  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range PRI_TDI  2 (MUX_MODE1 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range PRI_TCK  1 (MUX_MODE1 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range PRI_TDO  1 (MUX_MODE1 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_74  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_80  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range GPIO_81  3 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_90  1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_91  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range DVL0     2 (MUX_MODE1 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_110 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_114 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_115 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_116 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_123 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_124 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_125 3 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
        >;
};
```

### GPIO 使用

如方案 eth0 使用 gpio 110 作为 PHY Reset 信号：

```c
&eth0 {
    emac,reset-gpio = <&gpio 110 0>;
};
```

## 接口介绍

### API介绍

- **申请指定的GPIO**

```
int gpio_request(unsigned gpio, const char *label);
```

- **释放已申请的指定GPIO**

```
void gpio_free(unsigned gpio);
```

- **设置指定的GPIO为输入模式**

```
int gpio_direction_input(unsigned gpio)
```

- **设置指定的GPIO为输出模式并赋初值**

```
int gpio_direction_output(unsigned gpio, int value)
```

- **设置指定GPIO的输出值**

```
void gpio_set_value(unsigned gpio, int value)
```

- **获取指定GPIO的信号值**

```
int gpio_get_value(unsigned gpio)
```

- **获取指定GPIO对应的中断编号**

```
int gpio_to_irq(unsigned gpio)
```

## Debug 介绍

### sysfs

读取和操作GPIO端口

```
/sys/class/gpio
|-- export
|-- gpiochip0 -> ../../devices/gpiochip0/gpio/gpiochip0
|-- gpiochip512 -> ../../devices/platform/soc/d401d800.i2c/i2c-8/8-0041/spacemit-pinctrl@spm8821/gpio/gpiochip512
`-- unexport
```

- **export**: 用于通知系统需要导出控制的GPIO引脚编号
- **unexport**: 用于通知系统取消导出
- **gpiochipX**: GPIO X 的详细信息
- **gpioX/direction**: GPIO X 的方向
- **gpioX/value**: GPIO X的端口值

### debugfs

查看系统的 GPIO 控制器及其管理的 GPIO 信息。

```
/sys/kernel/debug/gpio
# cat gpio
gpiochip0: GPIOs 0-127, k1x-gpio:
 gpio-63  (                    |reset               ) out lo
 gpio-67  (                    |pwr                 ) out hi
 gpio-80  (                    |cd                  ) in  hi ACTIVE LOW
 gpio-96  (                    |sys-led             ) out lo
 gpio-97  (                    |vbus                ) out hi
 gpio-110 (                    |mdio-reset          ) out hi
 gpio-111 (                    |reset               ) out lo
 gpio-113 (                    |pwdn                ) out lo
 gpio-115 (                    |mdio-reset          ) out hi
 gpio-116 (                    |regon               ) out hi
 gpio-123 (                    |hub                 ) out hi
 gpio-124 (                    |hub                 ) out hi
 gpio-127 (                    |?                   ) out lo

gpiochip1: GPIOs 512-517, parent: platform/spacemit-pinctrl@spm8821, spacemit-pinctrl@spm8821, can sleep:

```

## 测试介绍

可通过 `devmem` 查看寄存器值判断 GPIO 配置是否生效：

```
devmem reg_addr
```

## FAQ
