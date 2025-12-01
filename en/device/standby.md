---  
sidebar_position: 7  
---

# Standby  

**Linux Standby** mode is a power-saving state where the system enters sleep to reduce power consumption. In this mode:  

- **Most hardware devices are automatically powered down** or **put into low-power states**  
- **DDR enters self-refresh mode**  

## Module Overview  

### Functional Description  

The system sleep/wakeup architecture is shown below:  
![](static/standby.png)  

The sleep/wakeup process involves four layers:  

1. **Userspace Layer:** Initiates system suspension  
2. **Kernel Layer:** Handles:  
   - Freezing user processes and kernel threads  
   - Device suspend/resume operations  
3. **OPENSBI Layer:** Interfaces with PMU to enter sleep state  
4. **Hardware Layer:** PMU handles low-level sleep/wake logic  

### Source Code Structure  

**Controller driver code** (located at `drivers/soc/spacemit/pm/`):  
```
drivers/soc/spacemit/pm/
├── Makefile
├── platform_hibernation_pm.c
├── platform_hibernation_pm_ops.c
├── platform_pm.c
├── platform_pm_ops.c
```

**Core sleep/wake layer** (located at `kernel/power`):  
```
kernel/power/
├── Kconfig
├── main.c
├── suspend.c
└── wakelock.c
```

## Key Features  

### Features  

N/A  

### Performance Parameters  

| Suspend Time | Wakeup Time | Sleep Power |  
|-------------|------------|------------|  
| 3s         | 1s        | 32.3 mW   |  

## Configuration Guide  

Includes **driver enablement** and **DTS configuration**  

### CONFIG Settings  

```
CONFIG_SUSPEND:
Allow the system to enter sleep states where main memory remains powered
(e.g., suspend-to-RAM like ACPI S3 state).

Symbol: SUSPEND [=y]
Type  : bool
Defined at kernel/power/Kconfig:2
 Prompt: Suspend to RAM and standby
 Depends on: ARCH_SUSPEND_POSSIBLE [=y]
 Location:
  -> Power management options
  -> Suspend to RAM and standby (SUSPEND [=y])
```

### Typical Wakeup Source Configurations  

#### Power Key  

Uses PMIC's ONEKEY functionality:  

##### DTS  

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

    ...
    
                pwr_key: key {
                        compatible = "pmic,pwrkey,spm8821";
                };
        };
};
```  

##### Driver Source  

```
drivers/input/misc/
├── spacemit-pwrkey.c

Wakeup enable location:
vi drivers/soc/spacemit/pm_domain/k1x-pm_domain.c +825
```  

#### Pinctrl Edge-Detect Wakeup  

Used for scenarios like hall sensor lid detection or SDIO-WiFi wakeup. Example shows hall sensor implementation using GPIO interrupt during normal operation and pinctrl edge-triggering for wakeup.  

##### DTS  

```
spacemit_lid:spacemit_lid {
        compatible = "spacemit,k1x-lid";
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_hall_wakeup>;
        lid-gpios = <&gpio 74 0>;
        interrupts-extended = <&gpio 74 1
                        &pinctrl 300>;
};
```

##### Source Structure  

```
drivers/soc/spacemit/
├── spacemit_lid.c
```

Implementation:  
```
static int spacemit_lid_probe(struct platform_device *pdev)
{
  ....
        hall->normal_irq = platform_get_irq(pdev, 0);
        hall->wakeup_irq = platform_get_irq(pdev, 1);

        /* Normal operation uses GPIO interrupt */
        error = devm_request_irq(&pdev->dev, hall->normal_irq,
                        hall_wakeup_detect,
                        IRQF_NO_SUSPEND | IRQF_TRIGGER_FALLING | IRQF_TRIGGER_RISING,
                        "hall-detect", (void *)hall);

        /* Wakeup uses pinctrl interrupt */
        dev_pm_set_dedicated_wake_irq_spacemit(&pdev->dev, hall->wakeup_irq, IRQ_TYPE_EDGE_RISING);
        device_init_wakeup(&pdev->dev, true);
}
```

## API Reference  

### Testing Procedure  

Example RTC wakeup test script:  
```
#!/bin/sh
echo 9 > /proc/sys/kernel/printk
echo N > /sys/module/printk/parameters/console_suspend
counter=1

while [ $counter -le 5000 ]
do
        echo +120 > /sys/class/rtc/rtc0/wakealarm
        echo mem > /sys/power/state
        counter=$(($counter+1))
        echo "----->counter:$counter"
        sleep 20
done
``` 

## Debugging  

### Sysfs Interface  

Follows standard Linux PM sysfs conventions. Reference:  
``` 
Documentation/admin-guide/pm/sleep-states.rst
```  

## FAQ