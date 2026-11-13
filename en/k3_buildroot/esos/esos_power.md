---
sidebar_position: 3
---

# ESOS Power Management

**Platform:** SpacemiT K3 · **Updated:** 2026-11-09

---

## Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Regulator (Voltage Management)](#regulator-voltage-management)
4. [PM Domain Management](#pm-domain-management)
5. [System Suspend (SysSuspend)](#system-suspend-syssuspend)
6. [RTC Service](#rtc-service)
7. [Power Key Service](#power-key-service)
8. [DTS Configuration](#dts-configuration)
9. [Kconfig Configuration](#kconfig-configuration)
10. [Usage Examples](#usage-examples)
11. [Debugging](#debugging)

---

## Overview

ESOS (Embedded Small OS) is an RT-Thread-based real-time operating system developed by SpacemiT for the K3 SoC's RCPUs (RISC-V management cores). K3 uses an AMP (asymmetric multiprocessing) architecture and runs two RCPU instances, rcpu0 and rcpu1, concurrently.

ESOS provides five platform services to Linux on the AP through an RPMI (RISC-V Platform Management Interface) shared-memory channel:

| Service | Responsibility |
|------|------|
| **Regulator** | Controls multiple PMICs over I²C and handles voltage adjustment requests during AP frequency scaling |
| **PM Domain** | Handles Linux peripheral power-switch requests forwarded from the genpd framework |
| **SysSuspend** | Coordinates orderly power-down across rcpu0 and rcpu1, drives the system into STR/STD suspend, and handles wake-up and recovery |
| **RTC** | Provides real-time clock read/write and timed wake-up, and can serve as a system-suspend wake-up source |
| **Power Key** | Detects hardware key events and reports short presses to Linux; initiates PMIC power-off on a long press or when Linux shuts down |

> ESOS does not perform CPU frequency scaling; it only handles voltage adjustments through the Regulator service. PM Domain handles Linux peripheral power-switch requests, SysSuspend coordinates system suspend and wake-up, RTC provides timed wake-up, and Power Key handles key events and initiates shutdown.

---

## System Architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│                         Linux / AP cores                             │
│  genpd (PM Domain requests)  sysfs suspend  RTC class  input (key)   │
│                              RPMI driver                             │
└──────────────────────────┬───────────────────────────────────────────┘
						   │ RPMI shared-memory channel
						   │ Regulator · PMDomain · SysSuspend · RTC · PowerKey
						   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    ESOS (RCPU RT-Thread)                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│  │  Regulator  │  │  PM Domain  │  │  SysSuspend │                   │
│  │  Multi-PMIC │  │ Clock/power │  │ Multi-OS    │                   │
│  │ I²C voltage │  │ switching   │  │ mailbox     │                   │
│  └─────────────┘  └─────────────┘  └─────────────┘                   │
│  ┌─────────────┐  ┌─────────────┐                                    │
│  │     RTC     │  │  Power Key  │                                    │
│  │ Timed wake  │  │ Key events  │                                    │
│  └─────────────┘  └─────────────┘                                    │
└──────────────────────────┬───────────────────────────────────────────┘
						   ▼
	   Hardware: PMIC I²C · clock/power-domain registers · RTC · RCPU
```

---

## Regulator (Voltage Management)

Source: `esos/bsp/spacemit/drivers/regulator/k3-regulator.c`

ESOS uses the generic regulator framework to control multiple PMICs over I²C and provide power to the AP cores. During frequency scaling, the AP requests voltage adjustments through the RPMI Regulator service; ESOS does not perform the frequency switch itself.

**Voltage sequencing:** When increasing frequency, the AP first requests a voltage increase from ESOS and waits for the voltage to stabilize before switching the PLL. When decreasing frequency, the AP first switches the PLL to lower the frequency, then requests a voltage reduction. Reversing this order can cause the CPU to run at high frequency with insufficient voltage, resulting in instability.

### Supported PMICs

| Chip | DT compatible | Type | Voltage range | Step |
|------|---------------|------|---------------|------|
| P1 internal DCDC | `p1-regulator` | DCDC | 500–1375 mV | 5 mV |
| P1 internal LDO | `p1-regulator` | LDO | 500–3200 mV | 25 mV |
| IS6608 | `regulator-is6608` | DCDC | 534–1072 mV | 2 mV |
| TDA38740 | `regulator-tda38740-1/2` | DCDC | 531–1000 mV | ≈3.906 mV |
| IS6615A | `regulator-is6615a` | DCDC | 531–1000 mV | ≈1.953 mV |
| AU4562 | `regulator-au4562` | DCDC | 540–1080 mV | 5 mV |
| MP2973 | `regulator-mp2973` | DCDC | 540–1000 mV | 5 / 10 mV |

---

## PM Domain Management

Source: `esos/bsp/spacemit/drivers/rpmi/spacemit-pmdomain.c`

ESOS handles power-switch requests for Linux peripherals through the RPMI PMDomain service. Linux on the AP issues power-domain on/off requests through the generic PM domain framework (`genpd`). RPMI forwards the requests to ESOS, which performs the corresponding register operations.

The main PM Domain use cases are:

- Turn off or restore a peripheral's power switch during driver suspend/resume or runtime suspend/resume.
- During system suspend, turn off nonessential peripheral power switches in sequence after ESOS completes multi-OS coordination, reducing standby power consumption.
- On wake-up, restore peripheral power switches in sequence so hardware is ready before peripheral drivers resume.

---

## System Suspend (SysSuspend)

Source: `esos/bsp/spacemit/drivers/rpmi/spacemit-hsm.c`

System-level suspend is coordinated by the `spacemit_multiple_os_poll()` thread, which manages the orderly power-down and recovery of rcpu0 and rcpu1.

| Type | Recovery speed | Description |
|------|----------------|-------------|
| STR (Suspend-to-RAM) | Fast (milliseconds) | RAM remains powered and context stays in memory |

### STR Suspend Sequence

1. Linux triggers system suspend. The AP cores complete their power-down sequence in order and enter suspend.
2. The ESOS `multiple_os_poll` thread receives the power-down notification, sends a low-power entry message to rcpu1 through the mailbox, and waits for the acknowledgement semaphore from rcpu1.
3. After rcpu1 completes its power-down, ESOS enters deep sleep (the RCPU executes WFI).

### STR Wake-up and Recovery Sequence

1. A wake-up source, such as an RTC alarm or Power Key, causes ESOS to exit sleep.
2. ESOS notifies rcpu1 through the mailbox to resume and waits for its ready semaphore.
3. Wake the C2 cluster: write the C2 boot vector (`C2_RVBADDR_LO/HI`), call `spacemit_wakeup_c2()`, and wait for C2 to finish powering on.
4. Wake the C3 cluster using the same sequence as C2, calling `spacemit_wakeup_c3()`.
5. Release the suspended AP cores using a semaphore; Linux resumes from the suspend point.

---

## RTC Service

Source: `esos/bsp/spacemit/drivers/rpmi/spacemit-rtc.c`

ESOS exposes the hardware real-time clock to Linux on the AP through the RPMI RTC service. Linux accesses the RTC through the standard `/dev/rtc` interface; ESOS translates RPMI requests into RTC hardware register operations.

**Main functions:**

- **Time read/write:** Read or set the current year, month, day, hour, minute, and second.
- **Timed wake-up (alarm):** Set an alarm time; during system suspend, the RTC hardware generates a wake-up interrupt when it expires.
- **Wake-up source:** ESOS registers the RTC alarm interrupt as a PM wake-up source. When the interrupt is received, ESOS exits RCPU sleep and initiates the STR recovery sequence.

---

## Power Key Service

Source: `esos/bsp/spacemit/drivers/rpmi/spacemit-powerkey.c`

ESOS forwards power-key events to Linux on the AP through the RPMI PowerKey service.

**Main functions:**

- **Short press:** Report a `KEY_POWER` input event to Linux; user space decides what to do, such as lock the screen or suspend.
- **Long press:** Initiate shutdown; ESOS executes the PMIC power-off sequence.
- **Power-on/wake-up:** A long press can also serve as a wake-up source when the system is suspended. After ESOS exits sleep, it starts STR recovery.
- **Shutdown notification:** When Linux invokes the shutdown syscall, it notifies ESOS through RPMI to perform the final PMIC power-off operation.

---

## DTS Configuration

### Regulator Node (IS6608 Example)

```dts
&i2c0 {
	status = "okay";

	vdd_cpu_x100: regulator@40 {
		compatible = "regulator-is6608";
		reg = <0x40>;
		regulator-name = "vdd-cpu-x100";
		regulator-min-microvolt = <534000>;
		regulator-max-microvolt = <1072000>;
		regulator-always-on;
	};
};
```

### TDA38740 Node (Including Suspend Configuration)

```dts
vdd_cpu_a100: regulator@5c {
	compatible = "regulator-tda38740-1";
	reg = <0x5c>;
	regulator-name = "vdd-cpu-a100";
	regulator-min-microvolt = <531250>;
	regulator-max-microvolt = <1000000>;
	/* Send the suspend signal through this register during suspend */
	spacemit,pm-suspend-reg = <0x5c>;
};
```

### SysSuspend Multi-OS Mailbox Node

```dts
lpm_mailbox: lpm-mailbox {
	compatible = "spacemit,k3-lpm-mailbox";
	mboxes = <&mailbox0 0>, <&mailbox0 1>;
	mbox-names = "tx", "rx";
	memory-region = <&lpm_shared_mem>;
	status = "okay";
};
```

### RTC Node

```dts
rtc: rtc@d4010000 {
	compatible = "spacemit,k3-rtc";
	reg = <0x0 0xd4010000 0x0 0x1000>;
	interrupts = <GIC_SPI 46 IRQ_TYPE_LEVEL_HIGH>;
	status = "okay";
};
```

### Power Key Node

```dts
powerkey {
	compatible = "spacemit,k3-powerkey";
	/* Connect to the PMIC ONKEY pin interrupt */
	interrupt-parent = <&pmic>;
	interrupts = <0>;
	debounce-interval = <100>;   /* ms */
	status = "okay";
};
```

---

## Kconfig Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `BSP_USING_K3_REGULATOR` | y | Enable the k3-regulator driver, the backend for the RPMI Regulator service |
| `BSP_USING_RPMI_REGULATOR` | y | Enable the RPMI Regulator service to handle AP voltage requests |
| `BSP_USING_RPMI_PMDOMAIN` | y | Enable the RPMI PMDomain service to manage clock and power domain switches |
| `BSP_USING_RPMI_SYSSUSP` | y | Enable the RPMI SysSuspend service for system STR/STD suspend |
| `BSP_USING_RPMI_RTC` | y | Enable the RPMI RTC service for clock read/write and timed wake-up |
| `BSP_USING_RPMI_POWERKEY` | y | Enable the RPMI PowerKey service to detect power-key events |

---

## Usage Examples

### Example 1: Trigger System Suspend from the AP (STR)

Linux on the AP triggers suspend through standard interfaces; ESOS coordinates the operating systems automatically:

```bash
# Trigger Suspend-to-RAM
echo mem > /sys/power/state

# Set an RTC alarm to wake after 30 seconds, then suspend
rtcwake -m mem -s 30
```

### Example 2: Read and Write the RTC from the AP

```bash
# Read the current RTC time
hwclock -r

# Synchronize system time to the RTC
hwclock -w

# Set an RTC alarm (using timerfd or /dev/rtc ioctl)
rtcwake -m no -s 60   # Alarm after 60 seconds, without suspending immediately
```

### Example 3: Register a Regulator in ESOS (Driver Development Reference)

```c
/* esos/bsp/spacemit/drivers/regulator/k3-regulator.c (simplified) */
#include "k3-regulator.h"

static const struct regulator_ops is6608_ops = {
	.set_voltage = is6608_set_voltage,
	.get_voltage = is6608_get_voltage,
	.enable      = is6608_enable,
	.disable     = is6608_disable,
};

static struct regulator_desc is6608_desc = {
	.name       = "vdd-cpu-x100",
	.ops        = &is6608_ops,
	.min_uV     = 534000,
	.max_uV     = 1072000,
	.step_uV    = 2000,
};

int is6608_regulator_init(struct i2c_device *dev)
{
	return regulator_register(&is6608_desc, dev);
}
```

---

## Debugging

### Check RPMI Service Status (ESOS MSH)

```text
msh> rpmi_dump
```

The output shows the registration status and message queue depth of each RPMI service group, as well as the parameters and result of the most recent request.

### Check Regulator Status (ESOS MSH)

```text
msh> regulator_dump
```

The output shows the current voltage, enable state, and I²C address of every registered regulator.

### Troubleshooting

**Problem: The system cannot wake after suspend**

Check the rcpu1 mailbox configuration (the `lpm_mailbox` node in DTS). Confirm that the C2/C3 cluster boot-vector registers (`C2_RVBADDR_LO/HI`, `C3_RVBADDR_LO/HI`) were set correctly before suspend. Check the ESOS serial log for `wakeup_c2/c3` errors.

**Problem: The RTC alarm does not trigger wake-up**

Confirm that the RTC node is configured correctly in DTS and that the interrupt number matches the hardware manual. Use `hwclock -r` to verify that the ESOS RTC service responds, and check that the alarm time is later than the current RTC time.

**Problem: Voltage adjustment fails**

Confirm that the ESOS regulator driver loaded successfully (look for `regulator init` in the ESOS serial log). Check that the regulator node's I²C address and voltage range match the actual PMIC model. The voltage adjustment path uses polling and does not depend on interrupts; if it still fails, check whether another device is using the I²C bus.

**Problem: Power Key events are not reported to Linux**

Check the ESOS serial log for messages indicating that the `powerkey` interrupt was received. Confirm that `interrupt-parent` and the interrupt number in DTS match the PMIC/GPIO hardware. Check Linux `dmesg` for successful registration of the RPMI PowerKey driver.