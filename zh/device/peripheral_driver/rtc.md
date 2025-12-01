# RTC

本文介绍 RTC（实时时钟）的功能和使用方法。

## 模块介绍

RTC（Real-Time Clock，实时时钟）主要用于计时、产生闹钟等功能，并维护系统时间。RTC 通常配有一颗备用电池，因此即使系统掉电，RTC 也能在备用电池的供电下继续正常工作。

### 功能介绍

![](static/rtc.png)  


1. `dev/sysfs/proc` 层：接口层，负责向用户空间提供操作节点及相关接口
2. `rtc-core` 层：为 RTC 驱动提供一套 API，完成设备与驱动的注册等
3. `RTC 驱动层`：具体实现 RTC 的核心功能，如设置时间、设置闹钟等

### 源码结构介绍

```
drivers/rtc/
├── class.c
├── dev.c
├── interface.c
├── Kconfig
├── lib.c
├── Makefile
├── proc.c
├── rtc-core.h
├── rtc-spt-pmic.c
├── sysfs.c
```

## 关键特性

### 特性

- 支持日历、闹钟、秒计数

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG配置

```
 CONFIG_RTC_DRV_SPT_PMIC:

 If you say yes here you will get support for the
 RTC of Spacemit spm8xxx PMIC.

 Symbol: RTC_DRV_SPT_PMIC [=y]
 Type  : tristate
 Defined at drivers/rtc/Kconfig:721
 Prompt: Spacemit spm8xxx RTC
 Depends on: RTC_CLASS [=y] && I2C [=y] && MFD_SPACEMIT_PMIC [=y]
 Location:
  -> Device Drivers
   -> Real Time Clock (RTC_CLASS [=y])
    -> Spacemit spm8xxx RTC (RTC_DRV_SPT_PMIC [=y])      
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
    ....

                ext_rtc: rtc {
                        compatible = "pmic,rtc,spm8821";
                };
        };
};
```

## 接口描述

RTC 驱动注册后会生成字符设备节点 `/dev/rtcN`，应用层只需按照 Linux 系统中标准的 RTC 编程方式进行访问即可。

### API介绍

```
#define devm_rtc_register_device(device) \
        __devm_rtc_register_device(THIS_MODULE, device)
int __devm_rtc_register_device(struct module *owner, struct rtc_device *rtc)  -- rtc设备注册

```

## 测试介绍

详见内核文档：
`Documentation/ABI/testing/rtc-cdev`

下面用伪代码的方式阐述测试方法：
```
1. fd = open("/dev/rtcN", xxx)  // 打开 RTC 设备
2. ioctl(fd, RTC_SET_TIME, ...) // 设置 RTC 时间
3. ioctl(fd, RTC_RD_TIME, ...)  // 读取 RTC 时间
4. ioctl(fd, RTC_ALM_SET, ...)  // 设置 RTC 闹钟
5. ioctl(fd, RTC_AIE_ON, ...)   // 使能 RTC
6. ioctl(fd, RTC_ALM_READ, ...)  // 读取 RTC 闹钟 
```

## FAQ
