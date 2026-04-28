# RT24 Timer on the K3 Secondary Core

This document describes the hardware timer configuration and usage model for the K3 RT24 secondary core.

## Module Overview

RT24 is the real-time secondary core on the K3 platform. It is based on a 64-bit RISC-V architecture and runs the RT-Thread real-time operating system (ESOS). RT24 has an independent clock control unit (CCU) and hardware timer subsystem, fully isolated from the primary cores (X100/A100).

### Functional Overview

The RT24 hardware timer (`hwtimer`) provides the following capabilities:

- High-precision hardware timing with 1 µs resolution
- One-shot (ONESHOT) trigger mode
- Interrupt callback support, with automatic callback invocation when the timer expires
- Dynamic runtime start and stop control
- Unified device management through the RT-Thread device framework, with access through the standard device interface

### Source Layout

The timer-related code for the secondary core is located in the following directories:

```
esos/bsp/spacemit/
├── drivers/
│   ├── hwtimer/
│   │   ├── k3-hwtimer.c          # hardware timer driver
│   │   └── hwtimer-test.c        # timer test code
│   ├── clk/
│   │   ├── ccu-spacemit-k3.c     # RCPU CCU clock driver
│   │   └── ccu-spacemit-k3.h     # clock ID definitions, including CLK_RCPU_TIMERx
│   └── rpmi/
│       ├── spacemit-clk.c        # common RPMI clock service layer
│       └── k3/k3-os0_clk.c       # OS0 RCPU frequency profile configuration
└── platform/
    └── rt24/
        ├── k3.dtsi               # device tree definitions for rtimer0 - rtimer3
        └── ccu-spacemit-k3.h     # clock macro definitions
```

## Key Features

- 4 hardware timer modules (`rtimer0` - `rtimer3`), each with 3 independent counters, for a total of **12 counter channels**
- Counter operating frequency of 1 MHz, with a timing resolution of 1 µs
- Two selectable clock sources: fastclk (3.25 MHz) and 32 KHz, selectable independently for each counter
- Up-counting mode (CNTMODE_UP) with a maximum count value of `0xFFFFFFFF` (approximately 4295 seconds)
- Registration through the RT-Thread hwtimer framework, using the device name format `timer:<timer_id>:<counter_id>`

## Configuration

### Kconfig Settings

Enable the hardware timer in `menuconfig`:

```
BSP_USING_HWTIMER:

Symbol: BSP_USING_HWTIMER [=n]
Prompt: Enable HWtimer
Selects: RT_USING_HWTIMER
Location:
 -> Drivers Config
  -> Enable HWtimer (BSP_USING_HWTIMER)

  config BSP_HWTIMER_TEST
      bool "Enable hwtimer test driver"
      default n
```

The corresponding `defconfig` entries are as follows:

```
CONFIG_RT_USING_HWTIMER=y
CONFIG_BSP_USING_HWTIMER=y
# CONFIG_BSP_HWTIMER_TEST is not set   # change to y when enabling the test
```

> Note: In the default defconfig (`rt24_os0_rcpu_defconfig`), hwtimer is disabled and must be enabled manually.

### Device Tree Settings

The timer nodes are defined in `esos/bsp/spacemit/platform/rt24/k3.dtsi`. There are 4 `rtimer` nodes, and each node contains 3 counter child nodes.

The following example shows `rtimer0`:

```dts
rtimer0: rtimer0@c0889000 {
    compatible = "spacemit,rtimer0";
    reg = <0xc0889000 0xc8>;
    num_counter = <3>;
    spacemit,timer-fastclk-frequency = <3250000>;   /* fastclk: 3.25 MHz */
    spacemit,timer-apb-frequency = <52000000>;       /* APB: 52 MHz */
    spacemit,timer-frequency = <1000000>;            /* operating frequency: 1 MHz */
    clocks = <&ccu CLK_RCPU_TIMER1>, <&ccu CLK_RST_RCPU_TIMER1>;
    status = "disabled";

    count0 {
        compatible = "timer0-counter0";
        interrupt-parent = <&intc>;
        interrupts = <0 8 0>;
        status = "disabled";
    };
    count1 {
        compatible = "timer0-counter1";
        interrupt-parent = <&intc>;
        interrupts = <0 9 0>;
        status = "disabled";
    };
    count2 {
        compatible = "timer0-counter2";
        interrupt-parent = <&intc>;
        interrupts = <0 10 0>;
        status = "disabled";
    };
};
```

To enable a timer, set the `status` of the corresponding `rtimer` node and its counter child nodes to `"okay"`:

```dts
&rtimer0 {
    status = "okay";
    count0 { status = "okay"; };
    count1 { status = "okay"; };
    count2 { status = "okay"; };
};
```

## API Reference

The RT24 hwtimer driver implements the following RT-Thread standard interfaces:

```c
/* Initialize the timer (called by the framework; not typically called directly by application code) */
static void spacemit_hwtimer_init(struct rt_hwtimer_device *timer, rt_uint32_t state);

/* Start the timer; cnt is the count value and mode is ONESHOT */
static rt_err_t spacemit_hwtimer_start(struct rt_hwtimer_device *timer,
                                        rt_uint32_t cnt, rt_hwtimer_mode_t mode);

/* Stop the timer */
static void spacemit_hwtimer_stop(struct rt_hwtimer_device *timer);

/* Read the current count value */
static rt_uint32_t spacemit_hwtimer_count_get(struct rt_hwtimer_device *timer);

/* Control interface; supports HWTIMER_CTRL_FREQ_SET / HWTIMER_CTRL_STOP */
static rt_err_t spacemit_hwtimer_control(struct rt_hwtimer_device *timer,
                                          rt_uint32_t cmd, void *args);
```



## Testing

### 1. Enable the hwtimer Driver

Enable `BSP_USING_HWTIMER` and `BSP_HWTIMER_TEST` in `menuconfig`, or update `defconfig` directly:

```
CONFIG_RT_USING_HWTIMER=y
CONFIG_BSP_USING_HWTIMER=y
CONFIG_BSP_HWTIMER_TEST=y
```

Also set the `status` of `rtimer0` and its `count0` node to `"okay"` in the device tree.

### 2. Run the Test

After the system boots, run the following command in the RT-Thread shell:

```
msh > hwtimer_test
```

Test flow in `hwtimer-test.c`:
1. Call `rt_device_find("timer:0:0")` to locate the timer device and confirm that the driver has loaded successfully.
2. Open the device and register the timeout callback function `timer_timeout_cb`.
3. Set the timer to one-shot mode (`HWTIMER_MODE_ONESHOT`).
4. Set the timeout to 5 seconds and start the timer.
5. Wait for the callback to trigger and print `enter hardware timer isr`.
6. Stop and close the device.

Expected output:

```
SetTime: Sec 5, Usec 0
enter hardware timer isr
Close timer:0:0
```

## FAQ

### Q1: How can hwtimer driver initialization be verified?

Run `list_device` in the RT-Thread shell. If devices such as `timer:0:0` and `timer:0:1` are listed, the driver has loaded successfully.

### Q2: What is the timer resolution?

The operating frequency is fixed at 1 MHz, which provides a timing resolution of 1 µs.

### Q3: How many timers can be used at the same time?

There are 4 rtimer modules with 3 counters each, for a total of 12 independent counter channels that can be used simultaneously.

### Q4: Does the watchdog timer conflict with hwtimer?

The watchdog (`rwdt`) reuses `CLK_RCPU_TIMER1`, which is also used by `rtimer0`. However, the register address spaces are independent, so there is no direct conflict. Even so, avoid enabling `rtimer0` and the watchdog on the same clock resource at the same time.

