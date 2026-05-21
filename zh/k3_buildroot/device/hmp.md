---
sidebar_position: 6
---

# K3 HMP (Heterogeneous Multi-Processing) 使用说明

## 1. 概述

K3 SoC 采用异构多核架构，包含两种不同类型的 CPU 核心：

| 核心类型 | 型号 | CPU 编号 | VLEN | ISA 特性 | 用途 |
|-|-|-|-|-|-|
| Regular (常规核) | X100 | cpu0 \~ cpu7 | 256bit (vlenb=32) | rv64imafdcv**h** | 通用计算、KVM 虚拟化 |
| AI 核 | A100 | cpu8 \~ cpu15 | 1024bit (vlenb=128) | rv64imafdcv (无 H 扩展) | AI/向量计算专用 |

**核心设计动机：**

X100 和 A100 核心的 Vector 寄存器宽度不同（256bit vs 1024bit），向量上下文无法兼容。如果一个使用了 Vector 指令的线程在两种核心之间迁移，会导致向量寄存器状态损坏、计算结果错误甚至系统崩溃。**HMP 的核心作用就是从调度层面彻底隔离两类核心，防止线程在不兼容的核心之间迁移。**

HMP 功能通过内核配置 `CONFIG_SPACEMIT_HMP` 启用，实现：

- 常规线程只运行在 Regular 核心（X100, VLEN=256bit）上
- AI 线程只运行在 AI 核心（A100, VLEN=1024bit）上
- 禁止任何线程跨核心类型迁移，保证向量上下文安全
- 中断默认路由到 Regular 核心，避免干扰 AI 计算
- KVM 虚拟化仅在支持 H 扩展的 Regular 核心上启用

## 2. 内核配置

在内核 menuconfig 中启用：

```Plain Text
CONFIG_SPACEMIT_HMP=y

```

依赖条件：`SMP && SOC_SPACEMIT`

## 3. HMP 管理策略详解

### 3.1 CPU 分组初始化

系统启动时（`start_kernel` 阶段），通过 DTS 中 CPU 节点的 `cpu-ai` 属性划分核心类型：

- 有 `cpu-ai = "true"` 属性的核心 → 归入 `ai_cpu_mask`
- 无此属性的核心 → 归入 `regular_cpu_mask`

K3 默认配置：

- Regular: cpu0-cpu7 (X100 核心)
- AI: cpu8-cpu15 (A100 核心)

### 3.2 线程调度策略

**默认行为：**

- 所有新创建的用户线程和内核线程默认为 `HMP_REGULAR` 类型，只能运行在 Regular 核心上
- Per-CPU 内核线程（如 ksoftirqd、migration 等）保持其原始 CPU 绑定，不受 HMP 限制

**AI 线程：**

- 通过 `/proc/set_ai_thread` 接口将线程标记为 AI 类型
- 标记后线程的 cpumask 被限制为 `ai_cpu_mask`，只能运行在 AI 核心上
- 如果当前没有在线的 AI 核心，系统会自动上线一个 AI 核心

**亲和性限制：**

- `sched_setaffinity()` 系统调用受 HMP 约束
- Regular 线程只能设置 Regular 核心范围内的亲和性
- AI 线程只能设置 AI 核心范围内的亲和性
- 跨类型设置亲和性会返回 `-EINVAL`

### 3.3 CPU 热插拔策略

- 禁止下线最后一个 Regular 核心（系统必须保留至少一个常规核心）
- 如果存在 AI 线程，禁止下线最后一个 AI 核心
- 系统挂起（suspend）期间，所有 CPU 下线请求被允许（用户任务已冻结）

### 3.4 中断路由策略

- IRQ 默认亲和性（`irq_default_affinity`）被限制为 Regular 核心子集
- 当用户尝试将中断绑定到 AI 核心时，系统会自动将其重映射到对应的 Regular 核心
- IMSIC 中断域分配优先使用在线的 Regular 核心

### 3.5 向量寄存器（Vector）隔离

这是 HMP 最关键的设计目标。X100 和 A100 的向量寄存器宽度不同：

| 核心 | VLEN | vlenb | 向量上下文大小 (32个寄存器) |
|-|-|-|-|
| X100 (Regular) | 256bit | 32 字节 | 1024 字节 |
| A100 (AI) | 1024bit | 128 字节 | 4096 字节 |

**为什么必须隔离：**

- 向量寄存器的宽度是硬件固有属性，两种核心的向量上下文格式和大小完全不同
- 如果线程在 X100 上保存了 256bit 宽的向量状态，迁移到 A100 后恢复时会导致数据错位
- 反之，A100 上保存的 1024bit 向量状态在 X100 上根本无法完整恢复
- 这不是性能问题，而是**正确性问题**——会导致计算结果错误或内核崩溃

**内核实现：**

- `riscv_v_vsize` 固定为 4096 字节（取最大值，确保内存分配足够）
- 信号处理（signal）和 ptrace 使用每个线程实际的 `vstate.vlenb * 32` 计算向量上下文大小，确保保存/恢复正确的字节数
- AI 线程切换到 AI 核心前如果已初始化向量上下文，内核会打印警告（表明可能存在状态不一致风险）
- HMP 调度约束从根本上防止线程跨核心迁移，避免向量上下文损坏

### 3.6 KVM 虚拟化限制

- H 扩展（Hypervisor）仅在 X100 (Regular) 核心上可用
- KVM 虚拟化初始化跳过 AI 核心
- VMID 刷新操作仅在 Regular 核心上执行
- `elf_hwcap` 和 `riscv_isa` 位图保留 H 扩展位（即使 A100 不支持）

### 3.7 电源管理

- 系统挂起前设置 `system_suspending` 标志
- 挂起期间放宽 HMP 限制（允许任意 CPU 下线、允许跨类型亲和性设置）
- 恢复后清除标志，恢复正常 HMP 约束

## 4. 用户空间使用指南

### 4.1 将线程设置为 AI 线程

通过 `/proc/set_ai_thread` 接口：

```Bash
# 将指定 PID 的线程设置为 AI 线程
echo <pid> > /proc/set_ai_thread

# 将当前线程设置为 AI 线程
echo 0 > /proc/set_ai_thread

```

**注意事项：**

- 写入的值为目标线程的 PID（0 表示当前线程）
- 设置成功后，线程将被迁移到 AI 核心上运行
- 如果所有 AI 核心都处于离线状态，系统会自动上线一个 AI 核心
- 建议在线程初始化阶段、向量指令使用前设置，避免向量上下文冲突

### 4.2 查看 CPU 拓扑

```Bash
# 查看各 CPU 在线状态
cat /sys/devices/system/cpu/online

# 查看 CPU 型号信息
cat /proc/cpuinfo

```

### 4.3 手动管理 AI 核心上下线

```Bash
# 上线 AI 核心 (以 cpu8 为例)
echo 1 > /sys/devices/system/cpu/cpu8/online

# 下线 AI 核心
echo 0 > /sys/devices/system/cpu/cpu8/online

```

**限制：** 如果有 AI 线程正在运行，最后一个 AI 核心无法下线。

### 4.4 查看线程运行的 CPU

```Bash
# 查看进程当前运行在哪个 CPU
taskset -p <pid>

# 查看进程允许的 CPU 集合
cat /proc/<pid>/status | grep Cpus_allowed

```

### 4.5 编程接口示例（C 语言）

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

    /* 写入 "0" 表示设置当前线程 */
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

## 5. 设计约束与注意事项

1. **向量寄存器不兼容**：X100 (VLEN=256bit) 和 A100 (VLEN=1024bit) 的向量上下文无法互相兼容，这是 HMP 隔离的根本原因。任何使用 Vector 指令的线程都必须始终运行在同一类核心上
2. **设置时机**：务必在线程使用 Vector 指令之前设置其类型为 AI，否则已有的 256bit 向量上下文迁移到 1024bit 核心后会产生不可预期的行为
3. **不可逆操作**：线程一旦被设置为 AI 类型，当前没有接口将其恢复为 Regular 类型
4. **亲和性限制**：AI 线程无法通过 `taskset` 或 `sched_setaffinity` 绑定到 Regular 核心，反之亦然。这是保证向量上下文安全的硬性约束
5. **中断隔离**：AI 核心默认不处理外部中断，减少 AI 计算被打断的概率，提升向量计算吞吐
6. **KVM 不可用**：AI 核心不支持 H 扩展，不要在 AI 线程中使用 KVM 相关功能
7. **挂起恢复**：系统挂起/恢复过程中 HMP 约束暂时放宽（此时用户任务已冻结，不存在向量指令执行），恢复后自动恢复约束

## 6. 代码结构

| 文件路径 | 功能 |
|-|-|
| `drivers/soc/spacemit/hmp.c` | HMP 核心实现（cpumask 管理、线程类型设置、proc 接口） |
| `include/linux/soc/spacemit/spacemit-hmp.h` | HMP 头文件（类型定义、API 声明） |
| `drivers/soc/spacemit/Kconfig` | HMP 配置选项定义 |
| `init/main.c` | 启动时调用 `hmp_cpumask_init()` |
| `kernel/sched/core.c` | 新线程创建时设置默认 cpumask |
| `kernel/sched/syscalls.c` | `sched_setaffinity` 亲和性限制 |
| `kernel/cpu.c` | CPU 热插拔限制 |
| `kernel/irq/irqdesc.c` | IRQ 默认亲和性限制 |
| `kernel/irq/manage.c` | IRQ 亲和性设置重映射 |
| `arch/riscv/kernel/vector.c` | 向量上下文大小固定为 4096 |
| `arch/riscv/kernel/signal.c` | 信号处理中向量上下文按实际 vlenb 计算 |
| `arch/riscv/kernel/ptrace.c` | ptrace 向量寄存器读写按实际 vlenb 计算 |
| `arch/riscv/kernel/cpufeature.c` | ISA 特性检测保留 H 扩展 |
| `arch/riscv/kvm/main.c` | KVM 初始化跳过 AI 核心 |
| `arch/riscv/kvm/vmid.c` | VMID 刷新仅在 Regular 核心执行 |
| `drivers/irqchip/irq-riscv-imsic-platform.c` | IMSIC 中断分配优先 Regular 核心 |
| `arch/riscv/boot/dts/spacemit/k3-cpus.dtsi` | DTS 中 cpu-ai 属性定义 |