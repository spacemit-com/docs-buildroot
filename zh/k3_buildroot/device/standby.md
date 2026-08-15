---
sidebar_position: 6
---

# 休眠唤醒

介绍 K3 平台 Suspend to RAM 和 Hibernate 的功能与实现。

## 模块介绍

K3 平台支持两种系统休眠模式：Suspend to RAM（内存挂起，STR）和 Suspend to Disk（休眠到磁盘，Hibernate）。两种模式均通过标准 Linux PM 框架触发，经由 SBI → OpenSBI → RPMI → ESOS 多层协作完成。

### 功能介绍

K3 休眠唤醒的软件栈如下：

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
│   platform/generic/spacemit/k3_corepm.c │
└──────────────────────────────────────────┘
              ↓ RPMI 协议
┌──────────────────────────────────────────┐
│   ESOS / RT-Thread (RCPU0、RCPU1)        │
│   bsp/spacemit/drivers/pm/k3_pm.c        │
│   bsp/spacemit/drivers/rpmi/             │
└──────────────────────────────────────────┘
              ↓ memcpy 到 SRAM 0x0（仅 STR）
┌──────────────────────────────────────────┐
│   ESOS-Lite（运行于 SRAM 0x0）           │
│   components/esos-lite/                  │
└──────────────────────────────────────────┘
```

**ESOS-Lite** 是一个极简 RT-Thread 实例，以 `esos_lite.bin` 的形式通过 `.incbin` 内嵌在 ESOS 主体的 `.data` 段。STR 休眠时，RCPU0 hart0 将其拷贝到 SRAM（0x0）执行，负责 DDR LP2 控制与系统掉电，运行在 SRAM 因此不依赖 AP DRAM。

### 源码结构介绍

```
linux-6.18/
├── arch/riscv/kernel/suspend.c          # STR：SBI SUSP 调用、cpu_suspend、CSR 保存恢复
├── arch/riscv/kernel/hibernate.c        # Hibernate：CPU 上下文保存、内存镜像恢复
├── kernel/power/hibernate.c             # Hibernate 主流程（snapshot、syscore、关机）
└── drivers/remoteproc/k3-rproc.c        # Hibernate：RCPU 快照保存/恢复（syscore_ops）

opensbi/
├── lib/utils/suspend/fdt_suspend_rpmi.c # RPMI SYSSUSP 消息封装与收发
└── platform/generic/spacemit/
    ├── k3_corepm.c                      # AP 核心/集群下电投票、__rpmi_hsm_suspend
    └── spacemit_k3.c                    # 平台初始化、_start_warm 热启动入口

esos/
├── bsp/spacemit/drivers/pm/k3_pm.c      # STR/Hibernate PM 主逻辑、__do_hibernation
├── bsp/spacemit/drivers/rpmi/
│   ├── spacemit-hsm.c                   # RPMI HSM/SYSSUSP 消息处理
│   └── k3/k3-os0_hsm.c                  # syssusp_prepare/ready/finalize/resume
└── components/esos-lite/
    └── bsp/spacemit/drivers/pm/
        ├── k3_pm.c                      # ESOS-Lite：DDR LP2、WFI 掉电、唤醒恢复
        └── k3_ddr_sr.c                  # DDR 进入/退出 LP2 自刷新
```

## 关键特性

| 特性 | 说明 |
| :--- | :--- |
| Suspend to RAM（STR） | 系统挂起，内存掉电（DDR 进入 LP2 自刷新），可由 RTC、PMIC、GPIO、USB 等唤醒 |
| Suspend to Disk（Hibernate） | 内存镜像保存到 swap，系统完全关机，上电后恢复到休眠前状态（功能尚未正式发布） |
| 多核协调 | RCPU0 hart0 负责掉电主流程，RCPU1 独立掉电；唤醒时按 RCPU1 → Cluster2/3 → AP 顺序上电 |
| ESOS-Lite SRAM 执行 | STR 掉电前将 ESOS-Lite 拷贝到 SRAM（0x0），保证 AP DRAM 断电后仍能执行低功耗操作 |
| RCPU 快照（Hibernate） | Hibernate 保存时对 RCPU0、RCPU1、OpenSBI、SRAM、RPMI 五块内存做快照并随 swap 镜像持久化 |
| 标准 Linux PM 接口 | 通过 `/sys/power/state` 触发，符合 Linux PM 框架规范 |

## 配置介绍

### CONFIG 配置

STR 所需配置：

```
Power management options
    Suspend to RAM and standby (SUSPEND [=y])

CPU Power Management
    CPU Idle
        RISC-V SBI CPU Idle Driver (CPU_IDLE_RISCV_SBI [=y])
```

Hibernate 所需配置：

```
Power management options
    Hibernation (aka 'suspend to disk') (HIBERNATION [=y])

Device Drivers
    Remoteproc drivers
        SpaceMIT K3 remoteproc driver (SPACEMIT_K3_RPROC [=y])
            K3 RCPU hibernation snapshot support (CONFIG_HIBERNATION [=y])
```

### DTS 配置

#### 唤醒源配置

STR 的硬件唤醒源在 ESOS-Lite 的 `__suspend_hw_process()` 中通过 AWUCRM 寄存器直接配置，无需内核侧 DTS 额外设置。当前固定使能的唤醒源包括：

| 唤醒源 | 说明 |
| :--- | :--- |
| RTC Alarm | RTC 定时唤醒 |
| PMIC | 电源键唤醒 |
| USB | USB 插入唤醒 |
| GPIO 边沿检测 | GPIO 中断唤醒 |
| PCIe | PCIe 设备唤醒 |

#### Hibernate swap 分区配置

Hibernate 需要配置 swap 分区，并在内核命令行中指定 resume 设备：

```
# uboot bootargs 中添加（以 /dev/mmcblk0p3 为 swap 分区为例）
resume=/dev/mmcblk0p3
```

#### Hibernate 保留内存配置（DTS）

k3-rproc 驱动需要两块 no-map 保留内存用于存放 RCPU 快照，在 DTS 中配置如下：

```c
reserved-memory {
    /* RCPU0(5M) + RCPU1(5M) + OpenSBI(2M) 快照 */
    hibernation_snap_rcpu: hibernation_snap@100f00000 {
        reg = <0x1 0x00f00000 0x0 0xc00000>;
        no-map;
    };

    /* AP misc + SRAM(512K) + RPMI(16K) 快照 */
    hibernation_nomap: hibernation_nomap@100700000 {
        reg = <0x1 0x00700000 0x0 0x85400>;
        no-map;
    };
};

&k3_rproc0 {
    hibernation_snap = <&hibernation_nomap>, <&hibernation_snap_rcpu>;
};
```

## 使用介绍

### Suspend to RAM

```sh
# 进入 STR
echo mem > /sys/power/state

# 查看支持的休眠状态
cat /sys/power/state
```

### Hibernate

> **注意**：Hibernate 完整功能尚未正式发布。

```sh
# 确认 swap 已启用
swapon -s

# 进入 Hibernate
echo disk > /sys/power/state

# 查看当前 hibernate 模式（shutdown / platform / reboot）
cat /sys/power/disk
```

## 实现原理

### Suspend to RAM 组件说明

K3 的 STR 由五个组件协同完成，各组件职责如下：

| 组件 | 运行域 | 职责 |
| :--- | :--- | :--- |
| Linux 内核 | S-mode（AP） | PM 策略入口；保存和恢复 AP S-mode 寄存器上下文；通过 SBI 接口触发下层休眠 |
| OpenSBI | M-mode（AP） | 对 AP 各核心和集群执行下电投票；通过 RPMI 协议将挂起请求转发给 ESOS；唤醒时从掉电状态重新上电完成 M-mode 初始化 |
| ESOS/RCPU0 主体 | M-mode（RCPU0） | 电源管理主控；协调 RCPU1 进入低功耗；保存中断控制器配置；将 ESOS-Lite 部署到 SRAM 后移交控制权；唤醒后依次恢复各组件并拉起 AP 各 Cluster |
| ESOS-Lite | M-mode（RCPU0，SRAM） | 运行于 SRAM，不依赖 AP DRAM；负责将 DDR 拉入自刷新（LP2）、配置 PMU 唤醒源、执行掉电；唤醒后负责将 DDR 从 LP2 恢复，再将控制权交回 ESOS 主体 |
| RCPU1 | M-mode（RCPU1） | 收到 RCPU0 通知后独立进入低功耗掉电；唤醒时由 RCPU0 触发，从预设的启动入口重新上电 |

**ESOS-Lite 的必要性**

STR 期间 AP DRAM 需要进入低功耗自刷新状态，ESOS 主体代码本身驻留在 DRAM 中无法继续执行。ESOS-Lite 以二进制形式内嵌在 ESOS 主体内，休眠时被整体拷贝到片内 SRAM（0x0）运行，从而在 DRAM 断电期间持续接管 RCPU0 的执行，完成 DDR 控制和系统掉电。

**唤醒路径**

外部事件（RTC、PMIC、GPIO、USB 等）触发 PMU 上电。RCPU0 从预先写入的启动入口地址重新上电，ESOS-Lite 完成 DDR 恢复后将控制权交回 ESOS 主体；ESOS 主体恢复中断控制器配置，依次拉起 RCPU1、AP Cluster2/3、AP Cluster0；OpenSBI 重新上电完成 M-mode 初始化；Linux 从挂起点恢复执行。

---

### Hibernate 组件说明

> **注意**：Hibernate 完整功能尚未正式发布。

K3 的 Hibernate 在标准 Linux Hibernate 框架基础上，增加了 RCPU 侧固件的状态保存与恢复机制，各组件职责如下：

| 组件 | 运行域 | 职责 |
| :--- | :--- | :--- |
| Linux hibernate 框架 | S-mode（AP） | 将 AP 内存镜像序列化写入 swap 分区；关机前调用 `kernel_power_off()`；恢复时将 swap 镜像还原到内存，切换页表，恢复 CPU 上下文 |
| Linux k3-rproc 驱动 | S-mode（AP） | 在 Linux 休眠/恢复关键节点触发 RCPU 固件的快照保存与恢复；将快照副本（snap_backup）纳入 swap 镜像一并持久化，确保断电后能完整恢复 RCPU 运行状态 |
| OpenSBI | M-mode（AP） | 在休眠/恢复两个方向接收 RPMI 消息并转发给 ESOS；操作完成后由 RCPU0 触发 AP 上电，从掉电状态重新上电完成 M-mode 初始化 |
| ESOS/RCPU0 | M-mode（RCPU0） | 休眠时将 RCPU0、RCPU1、OpenSBI、SRAM、RPMI 五块运行时内存保存到 no-map 快照区，完成后触发 AP 上电返回；恢复时将快照区内容还原到各运行时地址，再触发 AP 上电并跳转到恢复入口 |

**快照保存与恢复的设计要点**

RCPU 侧固件（RCPU0、RCPU1、OpenSBI）的运行时内存位于 no-map 区域，不在 Linux 内存管理范围内，因此标准 hibernate 框架不会自动保存这部分状态。k3-rproc 驱动在 Linux 将内存镜像写入 swap **之前**，主动触发 RCPU0 完成五块内存的快照，并将快照副本保存到 vmalloc 内存（snap_backup）。由于 snap_backup 在 Linux 地址空间内，它会随内存镜像一并写入 swap，从而在断电后得以持久化。

恢复时，新内核将 swap 镜像还原到内存后，snap_backup 也随之恢复。k3-rproc 驱动在 Linux 切换回休眠镜像页表**之前**，将 snap_backup 中的内容写回 no-map 区域，再触发 RCPU0 将快照还原到各运行时地址，使 RCPU 侧固件恢复到休眠前的状态。

---

### Hibernate 内存快照布局

| 区域 | 运行时地址 | 大小 | 快照位置 |
| :--- | :--- | :--- | :--- |
| RCPU0 | 0x100200000 | 5 MB | Snapshot0 起始 |
| RCPU1 | 0x100800000 | 5 MB | Snapshot0 +5 MB |
| OpenSBI | 0x100000000 | 2 MB | Snapshot0 +10 MB |
| SRAM | 0x0 | 512 KB | Snapshot1 起始 |
| RPMI | 0x100e00000 | 16 KB | Snapshot1 +512 KB |

快照存储区（no-map，不被 Linux 覆盖）：

| 区域 | 地址 | 大小 | 说明 |
| :--- | :--- | :--- | :--- |
| Snapshot0（snap_rcop） | 0x100f00000 | 12 MB | RCPU0 + RCPU1 + OpenSBI |
| Snapshot1（snap_srrpi） | 0x100701400 | ~528 KB | SRAM + RPMI |
| 专用栈 | 0x100700400 | 4 KB | `__hibernation_enter()` 切栈用 |
| AP misc（hiber_apuse） | 0x100700400 | 4 KB | magic 标志位（REST / DONE） |
| snap_backup | Linux vmalloc | 同上两块之和 | 两块快照副本，随 swap 镜像持久化 |

## Debug 介绍

### 查看 PM 状态

```sh
# 查看支持的休眠状态
cat /sys/power/state

# 查看当前唤醒锁
cat /sys/power/wake_lock

# 查看 wakeup 事件统计
cat /sys/kernel/debug/wakeup_sources
```

### STR 调试

```sh
# 模拟 RTC 唤醒（N 秒后唤醒）
echo +N > /sys/class/rtc/rtc0/wakealarm
echo mem > /sys/power/state
```

### Hibernate 调试

```sh
# 查看 swap 使用情况
swapon -s
free -h

# 确认保留内存区域
cat /proc/iomem | grep -i "hiber\|snapshot"

# 查看 syscore suspend/resume 日志
dmesg | grep -i "k3-rproc\|rcpu\|snapshot\|hiber"
```

## FAQ

### 1. STR 后系统无法唤醒

开启 PM 调试日志后根据打印排查：

```sh
# 开启 PM 调试日志
echo 1 > /sys/power/pm_debug_messages
echo 1 > /sys/power/pm_print_times

# 查看唤醒相关日志
dmesg | grep -i "suspend\|resume"
```

其他排查项：

1. 确认唤醒源已正确配置（AWUCRM 寄存器中对应 bit 已置位）；
2. 确认 RTC 闹钟已正确设置：`cat /sys/class/rtc/rtc0/wakealarm`。

### 2. STR 唤醒后系统异常

开启内核 PM 调试日志后根据打印排查具体硬件问题：

```sh
# 开启 PM 调试日志
echo 1 > /sys/power/pm_debug_messages
echo 1 > /sys/power/pm_print_times

# 查看唤醒相关日志
dmesg | grep -i "suspend\|resume"
```

### 3. Hibernate 镜像保存失败

排查步骤：

1. 确认 swap 分区已启用且空间充足：`swapon -s`；
2. 确认 DTS 中 no-map 保留内存区域配置正确；
3. 查看 dmesg 中 `k3-rproc: rcpu snapshot` 相关日志；
4. 确认 `CONFIG_HIBERNATION` 和 `CONFIG_SPACEMIT_K3_RPROC` 均已开启。

### 4. Hibernate 恢复后 RCPU 状态异常

排查步骤：

1. 检查 `hiber_apuse_va` 中的 magic 值（正常流程应为 REST → DONE → 0）；
2. 确认 snap_backup 大小与快照区大小一致；
3. 查看 `spacemit_rproc_syscore_resume` 的返回值是否为 0；
4. 注意：若上次 Hibernate 中途崩溃，magic 可能停留在 REST，需手动清除后重试。


