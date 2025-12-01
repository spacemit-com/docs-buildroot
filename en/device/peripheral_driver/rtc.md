# RTC

RTC (Real-Time Clock) Functionality and Usage Guide.

## Overview

RTC (Real-Time Clock) is primarily used for timekeeping, generating alarms, and system time maintenance. Typically, an RTC is equipped with a backup battery, which allows it to continue functioning normally even when the system is powered off, ensuring that the time is accurately maintained.

### Functional Description

![](static/rtc.png)  

1. `dev/sysfs/proc` Layer: Interface layer responsible for providing operation nodes and related interfaces to user space.
2. `rtc-core` Layer: Provides a set of APIs for RTC drivers, completing device and driver registration, among other tasks.
3. `RTC Driver` Layer: Implements the core functions of the RTC, such as setting the time and setting alarms.

### Source Code Structure

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

## Key Features

### Features

- Supports calendar, alarm, and second-level timekeeping

## Configuration

The primary configurations are **driver enablement** and **DTS (Device Tree Source) configuration**.

### CONFIG Configuration

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
    ....

                ext_rtc: rtc {
                        compatible = "pmic,rtc,spm8821";
                };
        };
};
```

## Interface

After the RTC driver is registered, it will create a character device node `/dev/rtcN`. The application layer can access it simply by following the standard RTC programming method in the Linux system.

### API

```
#define devm_rtc_register_device(device) \
        __devm_rtc_register_device(THIS_MODULE, device)
int __devm_rtc_register_device(struct module *owner, struct rtc_device *rtc)  -- RTC device registration

```

## Testing

For details, see the kernel documentation: `Documentation/ABI/testing/rtc-cdev`

Test procedure (pseudocode)：
```
1. fd = open("/dev/rtcN", xxx)  // Open the RTC device
2. ioctl(fd, RTC_SET_TIME, ...) // Set the RTC time
3. ioctl(fd, RTC_RD_TIME, ...)  // Read the RTC time
4. ioctl(fd, RTC_ALM_SET, ...)  // Set the RTC alarm
5. ioctl(fd, RTC_AIE_ON, ...)   // Enable the RTC alarm interrupt
6. ioctl(fd, RTC_ALM_READ, ...) // Read the RTC alarm 
```

## FAQ
