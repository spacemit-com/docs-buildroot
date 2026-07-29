# WDT

This document introduces the basic principles, configuration methods, and debugging procedures of the WDT (Watchdog Timer).

## Overview

**WDT (Watchdog Timer) controller** is a hardware component used to monitor whether the system is operating normally.

After configuring a timeout value and enabling the watchdog, the system must periodically “feed” the watchdog within the specified time window.
If the watchdog is not fed within the timeout period (for example, when the system hangs or becomes unresponsive), the watchdog will trigger a reset signal to reset the K3 chip.

### Functionality

The Linux kernel registers the watchdog driver with the **WDT framework** and the **application layer** via the **WDT framework interfaces**, and creates the device node `/dev/watchdog0`.

User-space applications can use `open()` and `ioctl()` operations on this device node to implement the following functions:

- Enable / Disable the watchdog  
- Set the timeout value
- Feed the watchdog

### Source Code Structure

The WDT controller driver source code is located in the kernel directory `drivers/watchdog`:

```
drivers/watchdog
|--watchdog_core.c        # Kernel WDT framework interface code
|--watchdog_dev.c         # Kernel WDT framework code for registering a character device for user space
|--spacemit-k1-wdt.c      # K3 WDT driver  
```

### Hardware Features

The K3 watchdog uses a 24-bit hardware timer:
- Counter width: 24 bit
- Maximum count value: 16777215 (0xFFFFFF)
- Clock Divider: 256 (DEFAULT_SHIFT = 8)
- Maximum timeout value: 65535 seconds (about 18.2 hours)

## Key Features

- Timeout-triggered system reset
- Uses 24-bit timer
- Maximum configurable timeout: **65535s** (about 18.2 hours)
- Supports acting as a system restart handler

## Configuration

This mainly includes **Kconfig configuration** and **DTS configuration**.

### Kconfig Configuration

#### Enable the Watchdog Framework

`CONFIG_WATCHDOG` provides support for the kernel watchdog framework.
When using the K3 WDT driver, this option must be set to `y`.
```
Symbol: WATCHDOG [=y]
Device Drivers
      -> Watchdog Timer Support (WATCHDOG [=y])
```

#### Enable the SpacemiT K3 WDT Driver

After enabling the WDT framework, set `CONFIG_SPACEMIT_K1_WATCHDOG` to `y` to enable the K3 WDT driver.
```
Symbol: SPACEMIT_K1_WATCHDOG [=y]
      -> Spacemit K1/K3 SoC Watchdog (SPACEMIT_K1_WATCHDOG [=y])
```

### DTS Configuration

#### DTSI Configuration Example

In the `dtsi` file, the WDT controller's register base address, clock, and reset resources are defined.
In most cases, **no modification is required**.

**Main Domain Watchdog (k3.dtsi)**

```dts
watchdog: watchdog@d4014000 {
    compatible = "spacemit-k1,wdt";
    clocks = <&syscon_apbc CLK_APBC_TIMERS0>,
             <&syscon_apbc CLK_APBC_TIMERS0_BUS>;
    clock-names = "clk", "clk-bus";
    resets = <&syscon_apbc RESET_APBC_TIMERS0>;
    reg = <0x0 0xd4014000 0x0 0xff>,
          <0x0 0xd4050000 0x0 0x1024>;
    interrupts = <35 4>;
    interrupt-parent = <&saplic>;
    spa,wdt-disabled;
    spa,wdt-enable-restart-handler;
    status = "okay";
};
```

**Property Description:**

- `spa,wdt-disabled`: Disables automatic watchdog startup. If this property is not specified, the WDT driver automatically enables the watchdog when loaded and periodically feeds it using `hrtimer`.
- `spa,wdt-enable-restart-handler`: Enables the watchdog as a system restart handler.

#### DTS Configuration Example

The complete DTS configuration is as follows: 

```dts
&watchdog {
    status = "okay";
};
```

To enable automatic watchdog startup when the driver loads, remove the `spa,wdt-disabled` property:

```dts
&watchdog {
    /delete-property/ spa,wdt-disabled;
    status = "okay";
};
```

## Interface

### API

The Linux kernel mainly registers the watchdog as a character device and provides the device node to user space.
Commonly used `file_operations` interfaces include:

- This interface allows user space to open the watchdog node:

   ```c
   static int watchdog_open(struct inode *inode, struct file *file)
   ```

- This interface implements the following functions through different `cmd` values:
  - Enable / Disable the watchdog
  - Set / get the timeout value
  - Feed the watchdog

   ```c
   static long watchdog_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
   ```

### User-space Interface

User space interacts with the watchdog through the `/dev/watchdog0` device node.

**Basic implementation:**

```c
#include <linux/watchdog.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

int fd;
int flags;
int timeout;

// Open the WDT node
fd = open("/dev/watchdog0", O_WRONLY);

// Enable the watchdog device
flags = WDIOS_ENABLECARD;
ioctl(fd, WDIOC_SETOPTIONS, &flags);

// Set the watchdog timeout value (seconds)
timeout = 30;
ioctl(fd, WDIOC_SETTIMEOUT, &timeout);

// Get the current watchdog timeout value
ioctl(fd, WDIOC_GETTIMEOUT, &timeout);

// Feed the watchdog
ioctl(fd, WDIOC_KEEPALIVE, 0);

// Disable the watchdog device
flags = WDIOS_DISABLECARD;
ioctl(fd, WDIOC_SETOPTIONS, &flags);

close(fd);
```

## Debugging

Because the kernel watchdog framework registers the watchdog as a character device and provides it to the application layer, testing requires the user to implement a watchdog application in user space.

### Test Program Example

The following example is verified on a **Buildroot system**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/watchdog.h>

int main(int argc, char *argv[])
{
    int fd;
    int flags;
    int timeout = 10;  // 10-seconds timeout
    int ret;

    // Open the watchdog device
    fd = open("/dev/watchdog0", O_WRONLY);
    if (fd < 0) {
        perror("open /dev/watchdog0");
        return -1;
    }

    // Set the timeout value
    ret = ioctl(fd, WDIOC_SETTIMEOUT, &timeout);
    if (ret) {
        perror("WDIOC_SETTIMEOUT");
        close(fd);
        return -1;
    }

    printf("Watchdog timeout set to %d seconds\n", timeout);

    // Enable the watchdog
    flags = WDIOS_ENABLECARD;
    ret = ioctl(fd, WDIOC_SETOPTIONS, &flags);
    if (ret) {
        perror("WDIOC_SETOPTIONS enable");
        close(fd);
        return -1;
    }

    printf("Watchdog enabled, feeding every 5 seconds...\n");

    // Feed the watchdog periodically
    while (1) {
        sleep(5);
        ret = ioctl(fd, WDIOC_KEEPALIVE, 0);
        if (ret) {
            perror("WDIOC_KEEPALIVE");
            break;
        }
        printf("Watchdog fed\n");
    }

    close(fd);
    return 0;
}
```

### Command-Line Testing

You can also test it using simple command-line commands:

```bash
# Enable the watchdog (automatically enabled when the device node is opened)
cat /dev/watchdog0 &

# View the watchdog status
cat /sys/class/watchdog/watchdog0/status
cat /sys/class/watchdog/watchdog0/timeout

# Stop feeding the watchdog (the system will reset after timeout)
killall cat
```

## Testing

The WDT functionality can be verified as follows:

### Normal Feeding Test

1. Compile and run the test program above
2. Observe that the program periodically outputs `Watchdog fed`
3. The system continues running without resetting

### Stop Feeding Test

1. Run the test program
2. Force-terminate the program (using Ctrl+C or kill)
3. After the timeout expires, the system should trigger a reset
4. After rebooting, use `dmesg` to check the reason for the reboot

### Timeout Value Test

```bash
# Check the current timeout value
cat /sys/class/watchdog/watchdog0/timeout

# Check supported timeout range
cat /sys/class/watchdog/watchdog0/max_timeout
cat /sys/class/watchdog/watchdog0/min_timeout
```

## FAQ

### Watchdog fails to start

Check the following items:

1. Ensure the kernel configurations `CONFIG_WATCHDOG` and `CONFIG_SPACEMIT_K1_WATCHDOG` are enabled
2. Ensure the watchdog node status in the DTS is set to "okay"
3. Check whether the device node exists: `ls -l /dev/watchdog*`
4. Check the kernel log: `dmesg | grep -i watchdog`

### Watchdog automatic startup

If automatic watchdog startup on load is not required, ensure the `spa,wdt-disabled` property is configured in the DTS:

```dts
&watchdog {
    spa,wdt-disabled;
    status = "okay";
};
```

### Unexpected system reboots

If the system reboots unexpectedly and frequently, it may be caused by watchdog timeout:

1. Check if any program has opened `/dev/watchdog0` but is not feeding the watchdog normally
2. Increase the timeout value
3. Check the system load, and make sure the watchdog feeding process can run in a timely manner
4. Check `/var/log/messages` or `dmesg` to confirm the reason for the reboot

### How to disable the watchdog

**Temporary Disable (restored after reboot):**

```bash
# Method 1: Disable via ioctl
echo V > /dev/watchdog0

# Method 2: Unload the driver module (If compiled as a module)
rmmod spacemit_k1_watchdog
```

**Permanent Disable:**

 Set the status of the watchdog node to `disabled` in the DTS:

```dts
&watchdog {
    status = "disabled";
};
```

### Watchdog as Restart Handler

The K3 watchdog supports acting as a system restart handler. When the `spa,wdt-enable-restart-handler` property is configured, executing the `reboot` command triggers a system reset through the watchdog.

To disable this feature, remove the property:   

```dts
&watchdog {
    /delete-property/ spa,wdt-enable-restart-handler;
    status = "okay";
};
```
