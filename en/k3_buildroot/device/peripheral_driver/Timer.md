# K3 Timer

This document describes the configuration and usage of the hardware timer for the K3 big core.

## Overview

The K3 big core runs the Linux kernel, and its timer system is managed by the Linux clocksource/clockevent framework. The K3 platform provides a dedicated SoC hardware timer (`timer-k1x`) as the system clock event device, providing clock event support for kernel scheduling, high-precision timing, and other functions.

Additionally, RISC-V architecture provides a generic RISC-V timer based on  `timebase-frequency` as the system timebase.

### Functionality

The K3 big core timer system includes two layers:

**SoC Hardware Timer (`spacemit,soc-timer`)**:
- Acts as a Linux clock event device and provides timer interrupts to the kernel
- Supports broadcast timer mode, providing unified timer events for all CPUs
- Supports local timer mode, providing independent timer events for a single CPU
- Supports power management (suspend/resume)

**RISC-V Generic Timer**:
- Based on the `time` CSR register defined by the RISC-V specification
- Timebase frequency: 24 MHz
- Accessed through the SBI interface for system timekeeping

### Source Code Structure

The relevant code directories for the big core timer are as follows:

```
linux-6.18/
├── drivers/clocksource/
│   ├── timer-k1x.c              # K3 SoC hardware timer driver 
│   ├── timer-riscv.c            # RISC-V generic timer driver
│   └── Kconfig                  # Timer driver configuration option
└── arch/riscv/boot/dts/spacemit/
    ├── k3.dtsi                  # K3 SoC main device tree (includes timer0 node)
    └── k3-cpus.dtsi             # CPU Configuration (includes riscv-timer node)
```

## Key Features

- Supports up to 3 timer modules (`SPACEMIT_MAX_TIMER = 3`), with 3 counters in each module
- Supports two modes: broadcast timer and per-CPU local timer
- Supports switching between fastclk (1 MHz) and 32K clock source
- Clock event only supports ONESHOT mode (`CLOCK_EVT_FEAT_ONESHOT`)
- Rating is 200
- Supports power management (suspend/resume)
- RISC-V timer timebase frequency: 24 MHz

## Configuration

### Kconfig Configuration

```
CONFIG_SPACEMIT_K1X_TIMER=y

Symbol: SPACEMIT_K1X_TIMER
Prompt: Spacemit k1x/k3 timer driver
Depends on: SOC_SPACEMIT_K1 || SOC_SPACEMIT_K3 || COMPILE_TEST
Selects: CLKSRC_MMIO, TIMER_OF
Location:
 -> Device Drivers
  -> Clock Source drivers
   -> Spacemit k1x/k3 timer driver (SPACEMIT_K1X_TIMER [=y])
```

Enabled by default in the K3 defconfig (`arch/riscv/configs/k3_defconfig`):

```
CONFIG_SPACEMIT_K1X_TIMER=y
CONFIG_HIGH_RES_TIMERS=y
```

### DTS Configuration

#### SoC Hardware Timer Nodes

Defined in `arch/riscv/boot/dts/spacemit/k3.dtsi`:

```dts
timer0: timer@d4016000 {
    compatible = "spacemit,soc-timer";
    reg = <0x0 0xd4016000 0x0 0xc8>;
    spacemit,timer-id = <0>;
    spacemit,timer-fastclk-frequency = <1000000>;   /* fastclk: 1 MHz */
    spacemit,timer-apb-frequency = <102400000>;      /* APB: 102.4 MHz */
    spacemit,timer-frequency = <1000000>;            /* timer frequency: 1 MHz */
    clocks = <&syscon_apbc CLK_APBC_TIMERS1>,
             <&syscon_apbc CLK_APBC_TIMERS1_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_apbc RESET_APBC_TIMERS1>;
    status = "okay";

    counter0 {
        compatible = "spacemit,timer-match";
        interrupts = <26 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&saplic>;
        spacemit,timer-broadcast;       /* broadcast mode, serving all CPUs */
        spacemit,timer-counter-id = <0>;
        status = "okay";
    };
};
```

Key Properties of the counter Subnode:

| Property | Description |
|-----|------|
| `spacemit,timer-counter-id` | Counter ID (0~2) |
| `spacemit,timer-broadcast` | Configures the counter as a broadcast timer, serving all CPUs |
| `spacemit,timer-counter-cpu` | Configures the counter as a local timer, bound to a specified CPU (mutually exclusive with broadcast)|

#### RISC-V Generic Timer Nodes

Defined in `arch/riscv/boot/dts/spacemit/k3-cpus.dtsi`:

```dts
/ {
    cpus {
        timebase-frequency = <24000000>;   /* timebase frequency: 24 MHz */
    };

    riscv-timer {
        compatible = "riscv,timer";
        riscv,timer-cannot-wake-cpu;       /* timer interrupt cannot wake up CPU */
    };
};
```



## Debugging


### View System Timer Information

```bash
# View the current clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# View available clocksource
cat /sys/devices/system/clocksource/clocksource0/available_clocksource

# View clock event device information
cat /proc/timer_list
```

### View RISC-V Timer Timebase

```bash
# View `timebase-frequency` (unit: Hz)
cat /sys/firmware/devicetree/base/cpus/timebase-frequency | od -An -tu4
# The output should be 24000000
```

## FAQ

### Q1: What is the difference between a broadcast timer and a local timer?

The **broadcast timer** is bound to `cpu_possible_mask` and provides a unified timer interrupt for all CPUs. It is suitable for scenarios where the local timer stops when a CPU enters a low-power state.

The **local timer** is bound to a single CPU, and interrupts are delivered only to that CPU.


### Q2: Why does the clock event only support ONESHOT mode?

The K3 timer hardware requires the match value to be reconfigured after each trigger, as it does not support auto-reload for periodic triggering. Therefore, this driver only registers `CLOCK_EVT_FEAT_ONESHOT`.

### Q3: What are the respective roles of the RISC-V timer and the SoC timer?

The **RISC-V timer** (based on the `time` CSR) provides the system timebase with a fixed frequency of 24 MHz and is accessed through the SBI interface.

The **SoC timer** (`spacemit,soc-timer`) acts as the clock event device and is responsible for generating timer interrupts to drive kernel scheduling.

### Q4: Does the watchdog share the same clock with the timer?

The watchdog (`watchdog@d4014000`) uses `CLK_APBC_TIMERS0`, while the SoC timer (`timer@d4016000`) uses `CLK_APBC_TIMERS1`. Therefore, they use independent clocks and do not conflict with each other.
