# GPADC

介绍 GPADC 的功能和使用方法。

## 模块介绍

IIO 是 Linux 内核中的一个子系统，专门用于处理工业控制、测量设备的数据采集和处理。IIO 子系统支持的设备类型众多，包括模数转换器（ADC）、数模转换器（DAC）、加速度计、陀螺仪、惯性测量单元、温度传感器。本章节介绍的 GPADC 是一个模数转换器，其嵌入到了进迭的 PMIC 芯片中。

### 功能介绍

![](static/gpadc.png)  

1. **IIO Core**：提供驱动程序和用户空间之间的接口，负责设备枚举、注册和管理。  
2. **IIO 设备驱动程序**：用于控制和读取特定 IIO 设备的代码。  
3. **IIO 缓冲区**：用于存储传感器和其他测量设备数据的内存区域。  
4. **IIO 事件处理**：用于处理来自传感器和其他测量设备的中断和事件。

### 源码结构介绍

1. **IIO Core**：`drivers/iio/industrialio-core.c`  
2. **IIO 设备驱动程序**：`drivers/iio/adc/k1x_adc.c`
3. **IIO 缓冲区**：`drivers/iio/industrialio-buffer.c`
4. **IIO 事件处理**：`drivers/iio/industrialio-event.c`

## 关键特性

### 特性

- 软件支持 6 路ADC 
- 12bit ADC 转换精度，100Hz~50Khz采样率

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG配置

```
Symbol: SPACEMIT_P1_ADC [=y]
Type  : tristate
Defined at drivers/iio/adc/Kconfig:1444
Prompt: Spacemit P1 adc driver
Depends on: IIO [=y] && MFD_SPACEMIT_PMIC [=y]
Location:
 -> Device Drivers
  -> Industrial I/O support (IIO [=y])
   -> Analog to digital converters
    -> Spacemit P1 adc driver (SPACEMIT_P1_ADC [=y])   
```

### DTS 配置

进迭的 GPADC 内嵌在 PMIC 中，使能 GPADC 需要配置两个 dts 节点：

1. GPADC channel pinctrl 配置

```
pmic_pinctrl: pinctrl {
    compatible = "pmic,pinctrl,spm8821";
    gpio-controller;
    #gpio-cells = <2>;
    spacemit,npins = <6>;

    /* 假如使用channel2 作为adc的输入管脚 */
    gpadc2_pins: gpadc2-pins {
            pins = "PIN2";
            function = "adcin";
    };
};
```

2. ADC 驱动使能配置

```
ext_adc: adc {
     compatible = "pmic,adc,spm8821";
};

```

## 接口介绍

### API 介绍

```
struct iio_dev *iio_device_alloc(struct device *parent, int sizeof_priv); // 申请iio_dev结构体
struct iio_dev *devm_iio_device_alloc(struct device *dev, int sizeof_priv) // 申请 iio_dev
int iio_device_register(struct iio_dev *indio_dev) // 注册 IIO 设备
void iio_device_unregister(struct iio_dev *indio_dev) // 注销 IIO 设备

```

## Debug介绍

### sysfs

```
cd /sys/bus/iio/devices/iio:device0  # IIO 框架目录  
in_voltage2_raw  # 读取 ADC 硬件寄存器的值  
in_voltage2_scale  # 读取 ADC 的精度  
```

## 测试介绍

通过动态改变外部采样电压值进行简单测试，软件读取电压值的方法如下：

```
cd /sys/bus/iio/devices/iio:device0
cat in_voltage2_raw
cat in_voltage2_scale

# 得到的两个节点值做乘法，即为采样电压值（单位为 mV）
```

## FAQ
