# CPUFREQ

CPUFREQ Functionality and Usage Guide.

## Overview

The CPUFREQ subsystem is responsible for adjusting the CPU frequency and voltage during runtime. Its goal is to minimize CPU power consumption while ensuring that performance requirements are met.

### Functionality Description

![](static/cpufreq.png)

1. cpufreq core is the core module of the cpufreq framework, and it mainly implements three types of functions:
   - Abstracts shared logic for frequency/voltage scaling.
   - Exposes a unified sysfs interface to userspace and broadcasts frequency changes via notifiers.
   - Implements the driver framework for CPU frequency/voltage control.
2. cpufreq governor is responsible for the strategies of frequency and voltage scaling.
3. cpufreq driver is responsible for the implementation of platform-specific frequency and voltage scaling mechanisms.
4. cpufreq stats is responsible for the statistics of frequency scaling information and the runtime events of each frequency point, providing statistical information related to cpufreq for each CPU.

### Source Code Structure

The directory structure for the CPU frequency scaling platform driver is as follows:

```
drivers/cpufreq/
├── cpufreq.c
├── cpufreq_conservative.c
├── cpufreq-dt.c
├── cpufreq-dt.h
├── cpufreq-dt-platdev.c
├── cpufreq_governor_attr_set.c
├── cpufreq_governor.c
├── cpufreq_governor.h
├── cpufreq_ondemand.c
├── cpufreq_ondemand.h
├── cpufreq_performance.c
├── cpufreq_powersave.c
├── cpufreq_stats.c
├── cpufreq_userspace.c
├── freq_table.c
├── spacemit-cpufreq.c --> Platform Driver  
```

## Key Features

| Feature | Description |
| :-----| :----|
| Dynamic Frequency/Voltage Scaling | Supports dynamic adjustment of CPU frequency and voltage to optimize performance and power consumption |
| Turbo Boost up to 1.8 GHz | Supports boosting the CPU frequency to 1.8 GHz for improved performance under high load conditions |

### Performance Parameters

| Frequency Steps (Hz) | Voltage (V) |  
| :-----| :----:|  
| 1600000000HZ | 1.05V |  
| 1228800000HZ | 0.95V |  
| 1000000000HZ | 0.95V |  
| 819000000HZ | 0.95V |  
| 614400000HZ | 0.95V |  

**Testing Method**

1. Set the governor to userspace mode:
   ```
   echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
   ```

2. Check the list of supported frequencies:
   ```
   cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
   Expected output: 614400 819000 1000000 1228800 1600000
   ```

3. Set the CPU frequency to a specific value (e.g., 1228800 Hz):
   ```
   echo 1228800 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
   ```

4. Verify if the frequency has been set successfully:
   ```
   cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
   ```

## Configuration

The configuration mainly includes driver enabling and dts (Device Tree Source) configuration.

### CONFIG Configuration

The CPUFREQ configuration is as follows:

```
CONFIG_SPACEMIT_K1X_CPUFREQ:

 This adds the CPUFreq driver support for Freescale QorIQ SoCs
 which are capable of changing the CPU's frequency dynamically.
 
 Symbol: SPACEMIT_K1X_CPUFREQ [=y]
 Type  : tristate
 Defined at drivers/cpufreq/Kconfig:315
 Prompt: CPU frequency scaling driver for Spacemit K1X
 Depends on: CPU_FREQ [=y] && OF [=y] && COMMON_CLK [=y]
 Location:
  -> CPU Power Management
   -> CPU Frequency scaling
    -> CPU Frequency scaling (CPU_FREQ [=y])
     -> CPU frequency scaling driver for Spacemit K1X (SPACEMIT_K1X_CPUFREQ [=y])
 Selects: CPUFREQ_DT [=y] && CPUFREQ_DT_PLATDEV [=y]    
```

### DTS Configuration

Due to the large size of the DTS configuration, only the relevant file is provided here:
```
arch/riscv/boot/dts/spacemit/k1-x_opp_table.dtsi
```

## Debugging

### sysfs

```
All sysfs nodes related to CPUFREQ can be found in the following directory:
/sys/devices/system/cpu/cpufreq/policy0/

affected_cpus

related_cpus

Check the policies supported by the system:
scaling_governor

The node for supporting boost:
boost

To view the frequencies supported by the system:
scaling_available_frequencies

To view the maximum frequency supported by the software:
scaling_max_freq        

To view the current running frequency of the CPU:
cpuinfo_cur_freq

To view the scaling governors supported by the system:
scaling_available_governors   

View the minimum frequency supported by the software:
scaling_min_freq

The maximum frequency supported by the CPU hardware:
cpuinfo_max_freq

CPU frequencies supported in boost mode:
scaling_boost_frequencies

The userspace mode is the interface for setting the CPU frequency:
scaling_setspeed

The minimum frequency supported by the CPU hardware:
cpuinfo_min_freq

To view the current running frequency of the CPU:
scaling_cur_freq
cpuinfo_transition_latency

The name of the current CPU frequency scaling driver:
scaling_driver

```

## Testing

To test frequency and voltage scaling, you can follow the testing method described in the previous section **Testing Method** and repeatedly test to verify the correctness of setting different frequencies.

## FAQ
