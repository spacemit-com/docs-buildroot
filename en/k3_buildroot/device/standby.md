---
sidebar_position: 7
---

# Standby

This document describes the Suspend to RAM (STR) and hibernation features and implementations on the K3 platform.

## Module Overview

The K3 platform supports two system suspend modes: Suspend to RAM (STR) and Suspend to Disk (Hibernate). Both modes are initiated through the standard Linux PM framework and implemented through coordinated operation across SBI → OpenSBI → RPMI → ESOS.

### Software Stack

The K3 suspend and resume software stack is as follows:

```
┌──────────────────────────────────────────┐
│   Linux Kernel (S-mode)                  │
│   kernel/power/  arch/riscv/kernel/      │
│   drivers/remoteproc/k3-rproc.c          │
└──────────────────────────────────────────┘
              ↓ SBI ecall (SBI_EXT_SUSP)
┌──────────────────────────────────────────┐
│   OpenSBI (M-mode)                       │
│   lib/utils/suspend/fdt_suspend_rpmi.c   │
│   platform/generic/spacemit/k3_corepm.c  │
└──────────────────────────────────────────┘
              ↓ RPMI protocol
┌──────────────────────────────────────────┐
│   ESOS / RT-Thread (RCPU0, RCPU1)        │
│   bsp/spacemit/drivers/pm/k3_pm.c        │
│   bsp/spacemit/drivers/rpmi/             │
└──────────────────────────────────────────┘
              ↓ memcpy to SRAM 0x0 (STR only)
┌──────────────────────────────────────────┐
│   ESOS-Lite (runs in SRAM 0x0)           │
│   components/esos-lite/                  │
└──────────────────────────────────────────┘
```

**ESOS-Lite** is a minimal RT-Thread instance embedded in the `.data` section of the main ESOS image as `esos_lite.bin` via `.incbin`. During STR suspend, RCPU0 hart0 copies it to SRAM (0x0) for execution. ESOS-Lite is responsible for DDR LP2 control and system power-down. Since it runs from SRAM, it does not depend on AP DRAM.

### Source Code Structure

```
linux-6.18/
├── arch/riscv/kernel/suspend.c          # STR: SBI SUSP call, cpu_suspend, CSR save/restore
├── arch/riscv/kernel/hibernate.c        # Hibernate: CPU context save, memory image restore
├── kernel/power/hibernate.c             # Hibernate main flow (snapshot, syscore, shutdown)
└── drivers/remoteproc/k3-rproc.c        # Hibernate: RCPU snapshot save/restore (syscore_ops)

opensbi/
├── lib/utils/suspend/fdt_suspend_rpmi.c # RPMI SYSSUSP message encapsulation and transmission
└── platform/generic/spacemit/
    ├── k3_corepm.c                      # AP core/cluster power-down voting, __rpmi_hsm_suspend
    └── spacemit_k3.c                    # Platform initialization, _start_warm warm boot entry

esos/
├── bsp/spacemit/drivers/pm/k3_pm.c      # STR/Hibernate PM main logic, __do_hibernation
├── bsp/spacemit/drivers/rpmi/
│   ├── spacemit-hsm.c                   # RPMI HSM/SYSSUSP message handling
│   └── k3/k3-os0_hsm.c                  # syssusp_prepare/ready/finalize/resume
└── components/esos-lite/
    └── bsp/spacemit/drivers/pm/
        ├── k3_pm.c                      # ESOS-Lite: DDR LP2, WFI power-down, wake-up restore
        └── k3_ddr_sr.c                  # DDR enter/exit LP2 self-refresh
```

## Key Features

| Feature | Description |
| :--- | :--- |
| Suspend to RAM (STR) | System suspend with memory powered down (DDR enters LP2 self-refresh); can be woken by RTC, PMIC, GPIO, USB, etc. |
| Suspend to Disk (Hibernate) | The memory image is saved to swap and the system is completely powered off. System state is restored after power-on (not yet officially released). |
| Multi-core Coordination | RCPU0 hart0 manages the primary power-down flow, while RCPU1 powers down independently. Wake-up sequence: RCPU1 → Cluster2/3 → AP. |
| ESOS-Lite SRAM Execution | Before STR power-down, ESOS-Lite is copied to SRAM (0x0), ensuring low-power operations continue after AP DRAM power-down |
| RCPU Snapshot (Hibernate) | During Hibernate save, snapshots of five memory regions (RCPU0, RCPU1, OpenSBI, SRAM, RPMI) are persisted with the swap image |
| Standard Linux PM Interface | Triggered via `/sys/power/state`, compliant with Linux PM framework specifications |

## Configuration

### CONFIG Options

STR requires the following configuration:

```
Power management options
    Suspend to RAM and standby (SUSPEND [=y])

CPU Power Management
    CPU Idle
        RISC-V SBI CPU Idle Driver (CPU_IDLE_RISCV_SBI [=y])
```

Hibernate requires the following configuration:

```
Power management options
    Hibernation (aka 'suspend to disk') (HIBERNATION [=y])

Device Drivers
    Remoteproc drivers
        SpaceMIT K3 remoteproc driver (SPACEMIT_K3_RPROC [=y])
            K3 RCPU hibernation snapshot support (CONFIG_HIBERNATION [=y])
```

### DTS Configuration

#### Wake-up Source Configuration

STR hardware wake-up sources are configured directly in ESOS-Lite's `__suspend_hw_process()` via AWUCRM registers and do not require additional kernel-side DTS settings. Currently enabled wake-up sources include:

| Wake-up Source | Description |
| :--- | :--- |
| RTC Alarm | RTC timer wake-up |
| PMIC | Power button wake-up |
| USB | USB insertion wake-up |
| GPIO Edge Detection | GPIO interrupt wake-up |
| PCIe | PCIe device wake-up |

#### Hibernate Swap Partition Configuration

Hibernate requires a configured swap partition and a resume device specified in the kernel command line:

```
# Add to U-Boot boot arguments (example using /dev/mmcblk0p3 as the swap partition)
resume=/dev/mmcblk0p3
```

#### Hibernate Reserved Memory Configuration (DTS)

The k3-rproc driver requires two no-map reserved memory regions for storing RCPU snapshots. Configure in DTS as follows:

```c
reserved-memory {
    /* RCPU0(5M) + RCPU1(5M) + OpenSBI(2M) snapshot */
    hibernation_snap_rcpu: hibernation_snap@100f00000 {
        reg = <0x1 0x00f00000 0x0 0xc00000>;
        no-map;
    };

    /* AP misc + SRAM(512K) + RPMI(16K) snapshot */
    hibernation_nomap: hibernation_nomap@100700000 {
        reg = <0x1 0x00700000 0x0 0x85400>;
        no-map;
    };
};

&k3_rproc0 {
    hibernation_snap = <&hibernation_nomap>, <&hibernation_snap_rcpu>;
};
```

## Usage

### Suspend to RAM

```sh
# Enter STR
echo mem > /sys/power/state

# View supported suspend states
cat /sys/power/state
```

### Hibernate

> **Note**: Hibernate functionality is not yet officially released.

```sh
# Confirm swap is enabled
swapon -s

# Enter Hibernate
echo disk > /sys/power/state

# View current hibernate mode (shutdown / platform / reboot)
cat /sys/power/disk
```

## Implementation Details

### Suspend to RAM Component Overview

K3 STR is accomplished through five cooperating components, each with the following responsibilities:

| Component | Execution Domain | Responsibilities |
| :--- | :--- | :--- |
| Linux Kernel | S-mode (AP) | PM policy entry point; saves and restores AP S-mode register context; triggers lower-layer suspend via SBI interface |
| OpenSBI | M-mode (AP) | Performs power-down voting for AP cores and clusters; forwards suspend requests to ESOS through RPMI; after wake-up, powers the AP back on and completes M-mode initialization |
| ESOS/RCPU0 Main | M-mode (RCPU0) | Power management controller; coordinates RCPU1 entry into the low-power state; saves interrupt controller configuration; deploys ESOS-Lite to SRAM and transfers control; after wake-up, restores components sequentially and powers on the AP clusters |
| ESOS-Lite | M-mode (RCPU0, SRAM) | Runs from SRAM, independent of AP DRAM; responsible for bringing DDR into self-refresh (LP2), configuring PMU wake-up sources, and executing power-down; upon wake-up, restores DDR from LP2 and returns control to ESOS main |
| RCPU1 | M-mode (RCPU1) | Independently enters the low-power state upon notification from RCPU0; after wake-up, is powered on by RCPU0 from the preset boot entry |

**ESOS-Lite Necessity**

During STR, AP DRAM must enter low-power self-refresh state. The main ESOS code resides in DRAM and cannot continue execution. ESOS-Lite is embedded as a binary in the main ESOS image and copied to on-chip SRAM (0x0) during suspend, allowing RCPU0 to continue execution during DRAM power-down and complete DDR control and system power-down.

**Wake-up Path**

External events (RTC, PMIC, GPIO, USB, etc.) trigger PMU power-on. RCPU0 powers on from the preconfigured boot entry address. After ESOS-Lite completes DDR restoration, it returns control to the ESOS main image. The ESOS main image restores the interrupt controller configuration and sequentially powers on RCPU1, AP Cluster2/3, and AP Cluster0. OpenSBI then powers on and completes M-mode initialization. Linux resumes execution from the suspend point.

---

### Hibernate Component Overview

> **Note**: Hibernate functionality is not yet officially released.

K3 hibernation extends the standard Linux hibernation framework with RCPU firmware state-save and restore mechanisms. Component responsibilities are as follows:

| Component | Execution Domain | Responsibilities |
| :--- | :--- | :--- |
| Linux hibernation framework | S-mode (AP) | Serializes the AP memory image to the swap partition; calls `kernel_power_off()` before shutdown; during restore, loads the swap image into memory, switches page tables, and restores CPU context |
| Linux k3-rproc driver | S-mode (AP) | Triggers RCPU firmware snapshot save and restore at key Linux suspend and resume points; includes the snapshot copy (`snap_backup`) in the swap image for persistence, ensuring complete RCPU runtime-state restoration after power-off |
| OpenSBI | M-mode (AP) | Receives and forwards RPMI messages to ESOS during both suspend and resume; after completion, RCPU0 triggers AP power-on and OpenSBI completes M-mode initialization |
| ESOS/RCPU0 | M-mode (RCPU0) | During suspend, saves five runtime memory regions (RCPU0, RCPU1, OpenSBI, SRAM, RPMI) to the no-map snapshot area, then triggers AP power-on to return; during restore, copies snapshot contents back to their runtime addresses, triggers AP power-on, and jumps to the restore entry |

**Snapshot Save and Restore Design Principles**

RCPU-side firmware (RCPU0, RCPU1, OpenSBI) runtime memory resides in no-map regions outside Linux memory management, so the standard hibernate framework does not automatically save this state. The k3-rproc driver proactively triggers RCPU0 to complete the five-region snapshot **before** Linux writes the memory image to swap, saving a snapshot copy to vmalloc memory (snap_backup). Since snap_backup is within Linux address space, it is written to swap along with the memory image and thus persisted after power-off.

During restore, after the new kernel loads the swap image back to memory, snap_backup is also restored. The k3-rproc driver writes snap_backup contents back to no-map regions **before** Linux switches back to the hibernated image page tables, then triggers RCPU0 to restore the snapshot to runtime addresses, returning RCPU-side firmware to its pre-suspend state.

---

### Hibernate Memory Snapshot Layout

| Region | Runtime Address | Size | Snapshot Location |
| :--- | :--- | :--- | :--- |
| RCPU0 | 0x100200000 | 5 MB | Snapshot0 start |
| RCPU1 | 0x100800000 | 5 MB | Snapshot0 +5 MB |
| OpenSBI | 0x100000000 | 2 MB | Snapshot0 +10 MB |
| SRAM | 0x0 | 512 KB | Snapshot1 start |
| RPMI | 0x100e00000 | 16 KB | Snapshot1 +512 KB |

Snapshot storage area (no-map, not overwritten by Linux):

| Region | Address | Size | Description |
| :--- | :--- | :--- | :--- |
| Snapshot0 (snap_rcop) | 0x100f00000 | 12 MB | RCPU0 + RCPU1 + OpenSBI |
| Snapshot1 (snap_srrpi) | 0x100701400 | ~528 KB | SRAM + RPMI |
| Dedicated stack | 0x100700400 | 4 KB | Stack switching for `__hibernation_enter()` |
| AP misc (hiber_apuse) | 0x100700400 | 4 KB | Magic flags (REST / DONE) |
| snap_backup | Linux vmalloc | Sum of above two | Snapshot copies, persisted with swap image |

## Debugging

### Viewing PM Status

```sh
# View supported suspend states
cat /sys/power/state

# View current wake locks
cat /sys/power/wake_lock

# View wakeup event statistics
cat /sys/kernel/debug/wakeup_sources
```

### STR Debugging

```sh
# Simulate RTC wake-up (wake after N seconds)
echo +N > /sys/class/rtc/rtc0/wakealarm
echo mem > /sys/power/state
```

### Hibernate Debugging

```sh
# View swap usage
swapon -s
free -h

# Confirm reserved memory regions
cat /proc/iomem | grep -i "hiber\|snapshot"

# View syscore suspend/resume logs
dmesg | grep -i "k3-rproc\|rcpu\|snapshot\|hiber"
```

## FAQ

### 1. System Fails to Wake from STR

Enable PM debug logging and investigate based on output:

```sh
# Enable PM debug logging
echo 1 > /sys/power/pm_debug_messages
echo 1 > /sys/power/pm_print_times

# View suspend/resume logs
dmesg | grep -i "suspend\|resume"
```

Additional troubleshooting steps:

1. Confirm wake-up sources are correctly configured (corresponding bits set in AWUCRM register);
2. Confirm RTC alarm is correctly set: `cat /sys/class/rtc/rtc0/wakealarm`.

### 2. Abnormal System Behavior After STR Wake-up

Enable kernel PM debug logging and investigate specific hardware issues based on output:

```sh
# Enable PM debug logging
echo 1 > /sys/power/pm_debug_messages
echo 1 > /sys/power/pm_print_times

# View suspend/resume logs
dmesg | grep -i "suspend\|resume"
```

### 3. Hibernate Image Save Failure

Troubleshooting steps:

1. Confirm swap partition is enabled and has sufficient space: `swapon -s`;
2. Confirm no-map reserved memory regions are correctly configured in DTS;
3. Review `k3-rproc: rcpu snapshot` related logs in dmesg;
4. Confirm both `CONFIG_HIBERNATION` and `CONFIG_SPACEMIT_K3_RPROC` are enabled.

### 4. RCPU State Abnormal After Hibernate Restore

Troubleshooting steps:

1. Check magic value in `hiber_apuse_va` (normal flow should be REST → DONE → 0);
2. Confirm snap_backup size matches snapshot area size;
3. Review return value of `spacemit_rproc_syscore_resume` (should be 0);
4. Note: If previous Hibernate crashed midway, magic may remain at REST; manual clearing required before retry.

