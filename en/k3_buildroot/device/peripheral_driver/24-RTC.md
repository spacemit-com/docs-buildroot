# RTC

This document introduces the basic principles, configuration methods, and debugging procedures of the RTC (Real-Time Clock). 

## Overview

**RTC (Real-Time Clock)** is a hardware module used to record and maintain system time.

The RTC can continue operating when the system is powered off or in a sleep state (usually powered by a backup battery), maintaining time continuity. After the system boots, it reads the current time from the RTC to ensure system time accuracy. 

### Functionality

The K3 platform uses the **RPMI (RISC-V Platform Management Interface)** protocol to implement RTC functionality.
The RPMI RTC communicates with firmware through the mailbox mechanism to implement time reading, time setting, and alarm functions.

The kernel registers the RTC driver with the system via the **RTC framework interface**, creating the device nodes `/dev/rtc0`.

User-space applications can operate the RTC in the following ways:

- Read the current time
- Set the system time
- Set and manage alarms
- Check RTC status via the sysfs interface

### Source Code Structure

The RTC driver code is located in the kernel directory `drivers/rtc`:

```
drivers/rtc
|--rtc-core.c           # RTC framework core code 
|--rtc-dev.c            # RTC character device interface 
|--rtc-sysfs.c          # RTC sysfs interface 
|--rtc-rpmi.c           # K3 RPMI RTC driver  
```

### RPMI RTC Architecture

The RPMI RTC is based on the mailbox communication mechanism:

```
User-space
    ↓
RTC framework
    ↓
RPMI RTC driver
    ↓
Mailbox subsystem
    ↓
MPXY Mailbox
    ↓
Hardware RTC
```

## Key Features 

- Supports reading and setting system time
- Supports alarm functionality
- Supports interrupt notifications
- Communicates based on the RPMI protocol
- Interacts with firmware via mailbox
- Supports system sleep and wake-up

## Configuration

This mainly includes **Kconfig configuration** and **DTS configuration**.

### Kconfig Configuration

#### Enable the RTC Framework

`CONFIG_RTC_CLASS` provides support for the kernel RTC framework, enabling the kernel RTC subsystem.
When using the K3 RTC driver, this option must be set to `y`.

```
Symbol: RTC_CLASS [=y]
Device Drivers
      -> Real Time Clock (RTC_CLASS [=y])
```

#### Enable the RPMI RTC Driver

After enabling the RTC framework, set `CONFIG_RTC_DRV_RPMI` to `y` to support the K3 RPMI RTC driver.

```
Symbol: RTC_DRV_RPMI [=y]
Device Drivers
      -> Real Time Clock
            -> RISC-V RPMI RTC (RTC_DRV_RPMI [=y])
```

### DTS Configuration

#### DTSI Configuration Example

Define the mailbox channel and interrupt resources for the RPMI RTC in the `dtsi` file.
This part usually **does not need to be modified**.

**RPMI RTC Node (k3.dtsi)**

```dts
rpmi_rtc: rpmi_rtc@0 {
    compatible = "riscv,rpmi-rtc";
    mboxes = <&mpxy_mbox 0xe 0x0>;
    interrupt-parent = <&saplic>;
    interrupts = <64 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "rpmi rtc";
    status = "okay";
};
```

**Property Description:**

- `compatible`: Compatible string identifying the device as an RPMI RTC device
- `mboxes`: Mailbox channel configuration used for communication with firmware
    - `&mpxy_mbox`: Mailbox controller
    - `0xe`: RTC service ID
    - `0x0`: Channel parameter
- `interrupts`: Interrupt number configuration, used for alarm interrupts
- `interrupt-names`: Interrupt name

#### DTS Configuration Example

Usually, the RPMI RTC is enabled by default in the `dtsi` (`status = "okay"`), so no additional configuration is required in the board-level DTS.

To disable the RTC, set the following configuration in the board-level DTS:

```dts
&rpmi_rtc {
    status = "disabled";
};
```

## Interface

### Kernel API

The RTC driver implements the standard `rtc_class_ops` interface:

```c
static const struct rtc_class_ops rpmi_rtc_ops = {
    .read_time = rpmi_read_time,      // Read time
    .set_time = rpmi_set_time,        // Set time
    .read_alarm = rpmi_read_alarm,    // Read alarm
    .set_alarm = rpmi_set_alarm,      // Set alarm
    .alarm_irq_enable = rpmi_alarm_irq_enable,  // Enable alarm interrupt
};
```

### RPMI Service ID

The RPMI RTC supports the following services:

| Service ID | Service Name | Function |
|---------|---------|---------|
| 0x01 | ENABLE_NOTIFICATION | Enable notifications |
| 0x02 | SET_TIME | Set time |
| 0x03 | GET_TIME | Get time |
| 0x04 | SET_ALARM | Set alarm |
| 0x05 | GET_ALARM | Get alarm |
| 0x06 | ALARM_GET_EN | Get alarm enable state |
| 0x07 | ALARM_SET_EN | Set alarm enable |
| 0x08 | QUERY_PENDING | Query pending events |
| 0x09 | CLR_PENDING | Clear pending events |

### User-Space Interface

#### Command-Line Tools

Use the `hwclock` command to operate the RTC:

```bash
# Read RTC time
hwclock -r

# Write system time to RTC
hwclock -w

# Synchronize RTC time to system time
hwclock -s

# Set RTC time
hwclock --set --date="2025-04-22 18:30:00"
```

Use the `date` command to operate the system time:

```bash
# View system time
date

# Set system time
date -s "2025-04-22 18:30:00"

# Set the system time and then synchronize it to the RTC
date -s "2025-04-22 18:30:00" && hwclock -w
```

#### sysfs Interface

View and operate the RTC through sysfs:

```bash
# View RTC time
cat /sys/class/rtc/rtc0/time
cat /sys/class/rtc/rtc0/date

# View RTC device information
cat /sys/class/rtc/rtc0/name

# View alarm settings
cat /sys/class/rtc/rtc0/wakealarm

# Set alarm (trigger after 10 seconds) 
echo +10 > /sys/class/rtc/rtc0/wakealarm
```

#### Programming Interface

Use ioctl to operate RTC:

```c
#include <linux/rtc.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

int fd;
struct rtc_time rtc_tm;

// Open RTC device
fd = open("/dev/rtc0", O_RDONLY);

// Read RTC time
ioctl(fd, RTC_RD_TIME, &rtc_tm);

// Set RTC time
ioctl(fd, RTC_SET_TIME, &rtc_tm);

// Read alarm
struct rtc_wkalrm alarm;
ioctl(fd, RTC_WKALM_RD, &alarm);

// Set alarm
ioctl(fd, RTC_WKALM_SET, &alarm);

close(fd);
```

## Debugging

### Basic Testing

1. Check whether the RTC device exists

   ```bash
   ls -l /dev/rtc*
   ```

2. View RTC driver loading state

   ```bash
   dmesg | grep -i rtc
   ```

3. View RTC device information

   ```bash
   cat /sys/class/rtc/rtc0/name
   cat /sys/class/rtc/rtc0/date
   cat /sys/class/rtc/rtc0/time
   ```

### Time Read and Write Test

```bash
# Read current RTC time
hwclock -r

# Set system time
date -s "2025-04-22 18:30:00"

# Write system time to RTC
hwclock -w

# Verify RTC time
hwclock -r
```

### Alarm Testing

```bash
# Set alarm to trigger after 10 seconds
echo +10 > /sys/class/rtc/rtc0/wakealarm

# View alarm settings
cat /sys/class/rtc/rtc0/wakealarm

# Wait for the alarm to trigger and check the kernel logs
dmesg | tail
```

### Test Program Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/rtc.h>
#include <time.h>

int main(void)
{
    int fd, ret;
    struct rtc_time rtc_tm;

    // Open RTC device
    fd = open("/dev/rtc0", O_RDONLY);
    if (fd < 0) {
        perror("open /dev/rtc0");
        return -1;
    }

    // Read RTC time
    ret = ioctl(fd, RTC_RD_TIME, &rtc_tm);
    if (ret < 0) {
        perror("RTC_RD_TIME");
        close(fd);
        return -1;
    }

    printf("RTC Time: %04d-%02d-%02d %02d:%02d:%02d\n",
           rtc_tm.tm_year + 1900,
           rtc_tm.tm_mon + 1,
           rtc_tm.tm_mday,
           rtc_tm.tm_hour,
           rtc_tm.tm_min,
           rtc_tm.tm_sec);

    close(fd);
    return 0;
}
```

## Test Guide

### Time Retention Test

1. Set the RTC time
2. Reboot the system
3. Check if the RTC time is retained after startup

```bash
# Set the time
date -s "2025-04-22 18:30:00"
hwclock -w

# Reboot the system
reboot

# Check after startup
hwclock -r
```

### Alarm Wake-up Test

1. Set the alarm 
2. System enters sleep mode
3. The system wakes up when the alarm is triggered

```bash
# Set alarm to trigger after 60 seconds
echo +60 > /sys/class/rtc/rtc0/wakealarm

# Enter system sleep state (if supported)
echo mem > /sys/power/state

# Wait for the alarm to wake up system
```

## FAQ

### RTC Device Not Found 

Check the following items:

1. Confirm that the kernel configuration options `CONFIG_RTC_CLASS` and `CONFIG_RTC_DRV_RPMI` are enabled
2. Confirm that the status of the `rpmi_rtc` node in the DTS is set to "okay"
3. Check if the mailbox driver is loaded correctly: `dmesg | grep -i mbox`
4. View the kernel logs: `dmesg | grep -i rtc`

### RTC Time Not Accurate

Possible causes:

1. The RTC has not been initialized correctly; manual time setting is required.
2. The system time is not synchronized with the RTC time.
3. Firmware RTC implementation issues

Solution:

```bash
# Set the correct system time
date -s "2025-04-22 18:30:00"

# Synchronize to RTC
hwclock -w

# Verify
hwclock -r
```

### Alarm Not Working

Check the following items:

1. Confirm if the interrupt configuration is correct: `cat /proc/interrupts | grep rtc`
2. Confirm that the alarm is enabled: `cat /sys/class/rtc/rtc0/wakealarm`
3. Check whether there are error messages in the kernel logs

### System time not synchronized with RTC time

At system startup, the time is usually read automatically from the RTC. If they are not synchronized:

```bash
# Manually synchronize system time from RTC
hwclock -s

# Or synchronize RTC from the system time
hwclock -w
```

### RPMI Communication Failure

If an RPMI communication error occurs:

1. Check if the mailbox driver is working properly: `dmesg | grep mpxy`
2. Confirm if the firmware version supports RPMI RTC
3. View detailed error information: `dmesg | grep -i "rpmi\|rtc"`

### How to Automatically Synchronize RTC Time on Startup

Add the following to the system startup script:

```bash
# /etc/init.d/S01rtc
#!/bin/sh

case "$1" in
    start)
        echo "Syncing system time from RTC..."
        hwclock -s
        ;;
    stop)
        echo "Syncing RTC from system time..."
        hwclock -w
        ;;
esac
```

## Notes

1. **RPMI dependency**: RPMI RTC depends on firmware support, so make sure the firmware version is correct.
2. **Mailbox channel**: The RTC uses mailbox channel `0xe`. Make sure it does not conflict with other devices.
3. **Interrupt Sharing**: RPMI RTC shares interrupt 64 with RPMI PWRKEY. This is normal.
4. **Time format**: The RTC uses UTC time, so time zone conversion must be considered.
5. **Backup power**: The RTC depends on a backup battery to maintain time when the system is powered off.
