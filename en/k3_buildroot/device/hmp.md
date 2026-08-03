---
sidebar_position: 7
---

# K3 HMP (Heterogeneous Multi-Processing) User Guide

## 1. Overview

The K3 SoC uses a heterogeneous multicore architecture with two different types of CPU cores:

| Core Type | Model | CPU IDs | VLEN | ISA Features | Purpose |
| - | - | - | - | - | - |
| Regular core | X100 | cpu0 \~ cpu7 | 256bit (vlenb=32) | rv64imafdcv**h** | General-purpose computing and KVM virtualization |
| AI core | A100 | cpu8 \~ cpu15 | 1024bit (vlenb=128) | rv64imafdcv (without the H extension) | Dedicated AI and vector computing |

**Core Design Rationale:**

The X100 and A100 cores have different Vector register widths (256bit versus 1024bit), and their vector contexts are incompatible. If a thread that uses Vector instructions migrates between the two core types, Vector register state corruption, incorrect computation results, or even a system crash may occur. **The primary purpose of HMP is to completely isolate the two core types at the scheduler level and prevent threads from migrating between incompatible cores.**

The HMP feature is enabled through the `CONFIG_SPACEMIT_HMP` kernel configuration and provides the following capabilities:

- Regular threads run only on Regular cores (X100, VLEN=256bit).
- AI threads run only on AI cores (A100, VLEN=1024bit).
- Migration of any thread across core types is prohibited to ensure vector context safety.
- Interrupts are routed to Regular cores by default to avoid disrupting AI computation.
- KVM virtualization is enabled only on Regular cores that support the H extension.

## 2. Kernel Configuration

Enable the following option in kernel menuconfig:

```Plain Text
CONFIG_SPACEMIT_HMP=y

```

Dependencies: `SMP && SOC_SPACEMIT`

## 3. HMP Management Policy Details

### 3.1 CPU Group Initialization

During system startup, at the `start_kernel` stage, core types are classified according to the `cpu-ai` property in the CPU nodes of the DTS:

- Cores with the `cpu-ai = "true"` property are added to `ai_cpu_mask`.
- Cores without this property are added to `regular_cpu_mask`.

The default K3 configuration is as follows:

- Regular: cpu0-cpu7 (X100 cores)
- AI: cpu8-cpu15 (A100 cores)

### 3.2 Thread Scheduling Policy

**Default Behavior:**

- All newly created user threads and kernel threads are `HMP_REGULAR` by default and can run only on Regular cores.
- Per-CPU kernel threads, such as `ksoftirqd` and `migration`, retain their original CPU affinity and are not subject to HMP restrictions.

**AI Threads:**

- A thread can be marked as an AI thread through the `/proc/set_ai_thread` interface.
- After being marked, the thread's cpumask is limited to `ai_cpu_mask`, and the thread can run only on AI cores.
- If no AI cores are currently online, the system automatically brings one AI core online.

**Affinity Restrictions:**

- The `sched_setaffinity()` system call is subject to HMP restrictions.
- Regular threads can set affinity only within the Regular core range.
- AI threads can set affinity only within the AI core range.
- Setting affinity across core types returns `-EINVAL`.

### 3.3 CPU Hotplug Policy

- Offlining the last Regular core is prohibited; the system must retain at least one Regular core.
- If AI threads exist, offlining the last AI core is prohibited.
- During system suspend, all CPU offline requests are permitted because user tasks have been frozen.

### 3.4 Interrupt Routing Policy

- The default IRQ affinity, `irq_default_affinity`, is restricted to the Regular core subset.
- When an attempt is made to bind an interrupt to an AI core, the system automatically remaps it to the corresponding Regular core.
- IMSIC interrupt domain allocation prioritizes online Regular cores.

### 3.5 Vector Register Isolation

This is the most critical design objective of HMP. The X100 and A100 cores have different Vector register widths:

| Core | VLEN | vlenb | Vector Context Size (32 registers) |
| - | - | - | - |
| X100 (Regular) | 256bit | 32 bytes | 1024 bytes |
| A100 (AI) | 1024bit | 128 bytes | 4096 bytes |

**Why Isolation Is Required:**

- Vector register width is a hardware-inherent property. The vector context formats and sizes of the two core types are completely different.
- If a thread saves 256bit-wide Vector state on an X100 core and then migrates to an A100 core, restoring the state causes data misalignment.
- Conversely, 1024bit Vector state saved on an A100 core cannot be completely restored on an X100 core.
- This is not a performance issue, but a **correctness issue** that can result in incorrect computation results or a kernel crash.

**Kernel Implementation:**

- `riscv_v_vsize` is fixed at 4096 bytes, the maximum value, to ensure sufficient memory allocation.
- Signal handling and ptrace calculate the vector context size using each thread's actual `vstate.vlenb * 32`, ensuring that the correct number of bytes is saved and restored.
- If the Vector context has already been initialized before an AI thread switches to an AI core, the kernel prints a warning, indicating a potential risk of inconsistent state.
- HMP scheduling constraints fundamentally prevent threads from migrating across core types, thereby avoiding Vector context corruption.

### 3.6 KVM Virtualization Restrictions

- The H extension (Hypervisor) is available only on X100 (Regular) cores.
- KVM virtualization initialization skips AI cores.
- VMID refresh operations are performed only on Regular cores.
- The `elf_hwcap` and `riscv_isa` bitmaps retain the H extension bit even though A100 does not support it.

### 3.7 Power Management

- The `system_suspending` flag is set before system suspend.
- HMP restrictions are relaxed during suspend, allowing any CPU to be offlined and allowing affinity to be set across core types.
- The flag is cleared after resume, and normal HMP constraints are restored.

## 4. User-Space Usage Guide

### 4.1 Set a Thread as an AI Thread

Use the `/proc/set_ai_thread` interface:

```Bash
# Set the thread with the specified PID as an AI thread
echo <pid> > /proc/set_ai_thread

# Set the current thread as an AI thread
echo 0 > /proc/set_ai_thread

```

**Notes:**

- The written value is the PID of the target thread. `0` is the current thread.
- After successful configuration, the thread migrates to an AI core.
- If all AI cores are offline, the system automatically brings one AI core online.
- Set the thread type during thread initialization and before using Vector instructions to avoid Vector context conflicts.

### 4.2 View CPU Topology

```Bash
# View the online state of each CPU
cat /sys/devices/system/cpu/online

# View CPU model information
cat /proc/cpuinfo

```

### 4.3 Manually Manage AI Core Online and Offline States

```Bash
# Bring an AI core online, using cpu8 as an example
echo 1 > /sys/devices/system/cpu/cpu8/online

# Take an AI core offline
echo 0 > /sys/devices/system/cpu/cpu8/online

```

**Restriction:** The last AI core cannot be offlined while AI threads are running.

### 4.4 View the CPU on Which a Thread Runs

```Bash
# View the CPU on which a process is currently running
taskset -p <pid>

# View the CPU set allowed for a process
cat /proc/<pid>/status | grep Cpus_allowed

```

### 4.5 Programming Interface Example (C)

```C
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int set_current_as_ai_thread(void)
{
    int fd = open("/proc/set_ai_thread", O_WRONLY);
    if (fd < 0)
        return -1;

    /* Write "0" to set the current thread */
    const char *val = "0";
    int ret = write(fd, val, strlen(val));
    close(fd);
    return (ret > 0) ? 0 : -1;
}

int set_thread_as_ai(pid_t pid)
{
    int fd = open("/proc/set_ai_thread", O_WRONLY);
    if (fd < 0)
        return -1;

    char buf[16];
    snprintf(buf, sizeof(buf), "%d", pid);
    int ret = write(fd, buf, strlen(buf));
    close(fd);
    return (ret > 0) ? 0 : -1;
}

```

## 5. Design Constraints and Considerations

1. **Vector Register Incompatibility**: The Vector contexts of X100 (VLEN=256bit) and A100 (VLEN=1024bit) are incompatible. This is the fundamental reason for HMP isolation. Any thread using Vector instructions must always run on the same type of core.
2. **Configuration Timing**: The thread must be configured as an AI thread before it uses Vector instructions. Otherwise, migrating an existing 256bit Vector context to a 1024bit core results in unpredictable behavior.
3. **Irreversible Operation**: Once a thread is set as an AI thread, no interface is currently available to restore it to a Regular thread.
4. **Affinity Restrictions**: AI threads cannot be bound to Regular cores through `taskset` or `sched_setaffinity`, and vice versa. This is a strict constraint that ensures Vector context safety.
5. **Interrupt Isolation**: AI cores do not handle external interrupts by default, reducing the likelihood of AI computation being interrupted and improving vector computing throughput.
6. **KVM Unavailable**: AI cores do not support the H extension. Do not use KVM-related functionality in AI threads.
7. **Suspend and Resume**: HMP constraints are temporarily relaxed during system suspend and resume, when user tasks are frozen and no Vector instructions are executed. Constraints are automatically restored after resume.

## 6. Code Structure

| File Path | Function |
| - | - |
| `drivers/soc/spacemit/hmp.c` | HMP core implementation, including cpumask management, thread type configuration, and the proc interface |
| `include/linux/soc/spacemit/spacemit-hmp.h` | HMP header file, including type definitions and API declarations |
| `drivers/soc/spacemit/Kconfig` | HMP configuration option definition |
| `init/main.c` | Calls `hmp_cpumask_init()` during startup |
| `kernel/sched/core.c` | Sets the default cpumask when creating new threads |
| `kernel/sched/syscalls.c` | `sched_setaffinity` affinity restrictions |
| `kernel/cpu.c` | CPU hotplug restrictions |
| `kernel/irq/irqdesc.c` | Default IRQ affinity restrictions |
| `kernel/irq/manage.c` | IRQ affinity configuration remapping |
| `arch/riscv/kernel/vector.c` | Fixes the Vector context size at 4096 bytes |
| `arch/riscv/kernel/signal.c` | Calculates Vector context size from the actual vlenb during signal handling |
| `arch/riscv/kernel/ptrace.c` | Reads and writes ptrace Vector registers according to the actual vlenb |
| `arch/riscv/kernel/cpufeature.c` | Retains the H extension during ISA feature detection |
| `arch/riscv/kvm/main.c` | Skips AI cores during KVM initialization |
| `arch/riscv/kvm/vmid.c` | Performs VMID refresh only on Regular cores |
| `drivers/irqchip/irq-riscv-imsic-platform.c` | Prioritizes Regular cores for IMSIC interrupt allocation |
| `arch/riscv/boot/dts/spacemit/k3-cpus.dtsi` | Defines the `cpu-ai` property in the DTS |
