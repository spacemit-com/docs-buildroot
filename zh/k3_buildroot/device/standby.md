# SpaceMIT K3 平台系统休眠唤醒实现

## 1. 系统架构概述

SpaceMIT K3 是一个 RISC-V 异构多核平台，包含以下主要组件：

- **AP (Application Processor)**: 运行 Linux 的应用处理器，包含多个 RISC-V 核心（hart），组织为 4 个集群（Cluster 0-3），共 16 个核心
- **RCPU0**: 运行 RT-Thread (ESOS) 的实时处理器，负责管理 Cluster 0 的电源
- **RCPU1**: 运行 RT-Thread 的第二个实时处理器
- **OpenSBI**: 运行在 M-mode 的固件，作为 SBI (Supervisor Binary Interface) 实现
- **RPMI (RISC-V Platform Management Interface)**: OpenSBI 与 RCPU 之间的通信协议
- **ESOS-Lite**: 极简 RT-Thread 实例，以二进制 blob 内嵌于 ESOS，在 STR 休眠时由 RCPU0 拷贝到 SRAM(0x0) 执行，负责 DDR LP2 控制和执行 WFI 使系统掉电

### 软件栈层次

```
┌─────────────────────────────────────┐
│   Linux Kernel (S-mode)             │
│   - arch/riscv/kernel/suspend.c     │
│   - arch/riscv/kernel/hibernate.c   │
└─────────────────────────────────────┘
              ↓ SBI 调用
┌─────────────────────────────────────┐
│   OpenSBI (M-mode)                  │
│   - fdt_suspend_rpmi.c              │
│   - k3_corepm.c                     │
└─────────────────────────────────────┘
              ↓ RPMI 协议
┌─────────────────────────────────────┐
│   ESOS/RT-Thread (RCPU0/RCPU1)      │
│   - k3_pm.c  /  spacemit-hsm.c      │
└─────────────────────────────────────┘
              ↓ memcpy 到 SRAM 0x0 后跳转
┌─────────────────────────────────────┐
│   ESOS-Lite (运行于 SRAM 0x0)       │
│   - esos_lite.bin (内嵌于 ESOS)     │
│   - k3_pm.c  /  k3_ddr_sr.c        │
└─────────────────────────────────────┘
```

### ESOS-Lite 简介

ESOS-Lite 是一个极简 RT-Thread 实例，以二进制 blob（`esos_lite.bin`）的形式内嵌在 ESOS 主体的 `.data` 段（通过 `builtin.S` 中的 `.incbin` 指令）。它的唯一职责是在系统完全断电前接管 RCPU0 hart0，完成以下硬件级操作：

- 将 DDR 控制器拉入 LP2（自刷新）模式
- 配置 PMU 唤醒源及系统低功耗寄存器
- 执行 WFI 使系统掉电
- 唤醒后将 DDR 从 LP2 恢复
- 跳回 ESOS 主体（`__pre_cpu_resume_enter` → `__cpu_resume_enter`）

ESOS-Lite 运行在 SRAM（`0x0`），因此即使 AP DRAM 完全断电，它也能正常执行。

## 2. 两种休眠模式

### 2.1 Suspend to RAM (STR / S3)

**目标**: 将系统置于低功耗状态，内存保持供电，可快速唤醒

**触发方式**:
```bash
echo mem > /sys/power/state
```

### 2.2 Suspend to Disk (STD / S4 / Hibernate)

**目标**: 将系统完全断电，内存内容保存到磁盘，唤醒后恢复到休眠前的状态

**触发方式**:
```bash
echo disk > /sys/power/state
```

## 3. Suspend to RAM (STR) 详细流程

**触发方式**: `echo mem > /sys/power/state`

### 3.1 休眠流程

```
  Linux (S-mode)            OpenSBI (M-mode)          ESOS/RCPU0 主体              ESOS-Lite (SRAM 0x0)       RCPU1
       │                         │                          │                              │                      │
  用户写 /sys/power/state        │                          │                              │                      │
       │                         │                          │                              │                      │
  suspend_enter()                │                          │                              │                      │
       │                         │                          │                              │                      │
  保存 S-mode CSR                │                          │                              │                      │
  (TVEC/IE/SATP等)               │                          │                              │                      │
       │                         │                          │                              │                      │
  __cpu_suspend_enter()          │                          │                              │                      │
  保存 CPU 上下文到栈            │                          │                              │                      │
       │                         │                          │                              │                      │
  sbi_ecall(SBI_EXT_SUSP) ──────>│                          │                              │                      │
  (sleep_type=0, resume_addr)    │                          │                              │                      │
                            接收 SBI SUSP 调用             │                              │                      │
                                 │                          │                              │                      │
                            投票 AP 集群下电                │                              │                      │
                            (spacemit_vote_powrdown)        │                              │                      │
                                 │                          │                              │                      │
                            发送 RPMI 消息 ────────────────>│                              │                      │
                            (SYSSUSP_SUSPEND)               │                              │                      │
                                 │                     接收 RPMI 消息                      │                      │
                                 │                     syssusp_finalize()                  │                      │
                                 │                          │                              │                      │
                                 │                     通知 RCPU1 进入低功耗 ─────────────────────────────────────>│
                                 │                          │                              │              设置唤醒入口地址
                                 │                          │                              │              配置 Core1 idle 寄存器
                                 │                          │                              │              wfi() → 掉电
                                 │                          │                              │                      │
                                 │                     触发 PM 框架                        │                      │
                                 │                     sleep(PM_SLEEP_MODE_DEEP)           │                      │
                                 │                          │                              │                      │
                                 │                     保存 PLIC/Timer 配置               │                      │
                                 │                          │                              │                      │
                                 │                     将 esos_lite.bin                    │                      │
                                 │                     拷贝到 SRAM (0x0)                   │                      │
                                 │                          │                              │                      │
                                 │                     跳转到 SRAM 0x0 ──────────────────>│                      │
                                 │                          │                         _start 初始化              │
                                 │                          │                         保存唤醒回跳地址            │
                                 │                          │                         RT-Thread 启动             │
                                 │                          │                              │                      │
                                 │                          │                         DDR 进入 LP2 自刷新         │
                                 │                          │                              │                      │
                                 │                          │                         配置 PMU 唤醒源             │
                                 │                          │                         (RTC/PMIC/USB/GPIO/PCIe)   │
                                 │                          │                              │                      │
                                 │                          │                         投票 RCPU Core0 下电        │
                                 │                          │                         等待 SoC Top 进入 D2        │
                                 │                          │                              │                      │
                            wfi() → 掉电                    │                         wfi() → 掉电               │
```

---

### 3.2 唤醒流程

```
  Linux (S-mode)            OpenSBI (M-mode)          ESOS/RCPU0 主体              ESOS-Lite (SRAM 0x0)       RCPU1
       │                         │                          │                              │                      │
       │                         │                          │                    外部事件触发系统重新上电          │
       │                         │                          │                    (RTC/PMIC/GPIO等)                │
       │                         │                          │                              │                      │
       │                         │                          │                    硬件从 RCPU_CORE0_BOOT_ENTRY     │
       │                         │                          │                    跳入 __pre_cpu_resume_enter      │
       │                         │                          │                              │                      │
       │                         │                          │                    清零所有寄存器                   │
       │                         │                          │                    切换到临时栈                     │
       │                         │                          │                    调用 __spacemit_wakeup_asm()     │
       │                         │                          │                              │                      │
       │                         │                          │                    清除 D2 唤醒状态                 │
       │                         │                          │                    取消下电投票                     │
       │                         │                          │                              │                      │
       │                         │                          │                    DDR 退出 LP2 自刷新              │
       │                         │                          │                              │                      │
       │                         │                          │                    __cpu_resume_enter(0)            │
       │                         │                          │ <────────────────────────────┤                      │
       │                         │                     恢复 CPU 通用寄存器                 │                      │
       │                         │                     恢复 M-mode CSR                    │                      │
       │                         │                     恢复 PLIC/Timer 配置               │                      │
       │                         │                          │                              │                      │
       │                         │                     唤醒 RCPU1 ─────────────────────────────────────────────>│
       │                         │                          │                              │              硬件从 Core1 BOOT_ENTRY
       │                         │                          │                              │              重新上电运行
       │                         │                          │                              │                      │
       │                         │                     依次唤醒 Cluster2/3                │                      │
       │                         │                          │                              │                      │
       │                         │                     解除 AP Cluster0 复位              │                      │
       │                         │                     (spacemit_deassert_corex)           │                      │
       │                         │                          │                              │                      │
       │                  _start_warm 重新执行              │                              │                      │
       │                  恢复 OpenSBI M-mode 上下文        │                              │                      │
       │                         │                          │                              │                      │
  cpu_suspend() 返回 <───────────┤                          │                              │                      │
       │                         │                          │                              │                      │
  恢复 S-mode CSR                │                          │                              │                      │
  (TVEC/IE/SATP等)               │                          │                              │                      │
       │                         │                          │                              │                      │
  系统恢复正常运行               │                          │                              │                      │
```

## 4. Suspend to Disk (Hibernate) 详细流程

> **注意**: Hibernation 完整功能尚未发布，以下流程为当前实现的设计分析。

**触发方式**: `echo disk > /sys/power/state`

### 4.1 休眠保存流程

```
  Linux (S-mode)                  Linux k3-rproc 驱动            OpenSBI (M-mode)         ESOS/RCPU0
       │                                  │                              │                      │
  hibernate()                            │                              │                      │
       │                                  │                              │                      │
  PM_HIBERNATION_PREPARE 通知 ──────────>│                              │                      │
                                  spacemit_rproc_hibernating=true        │                      │
       │                                  │                              │                      │
  设备挂起 / 关中断 / 禁用从核            │                              │                      │
       │                                  │                              │                      │
  syscore_suspend() ───────────────────>│                              │                      │
                                  spacemit_rproc_syscore_suspend():      │                      │
                                    cpu_suspend(SUSPEND_TO_DISK,  ──────>│                      │
                                      rproc_system_suspend)        发送 RPMI SYSSUSP            │
                                                                    (type=SUSPEND_TO_DISK) ────>│
                                                                         │               RPMI 消息接收
                                                                         │               hibernate_pending=1
                                                                         │               触发 PM_SLEEP_MODE_SHUTDOWN
                                                                         │                      │
                                                                         │               cpu_suspend(SHUTDOWN,
                                                                         │                 __suspend_asm_finish)
                                                                         │                      │
                                                                         │               __hibernation_enter()
                                                                         │               切换到专用栈(0x100700400)
                                                                         │                      │
                                                                         │               __do_hibernation():
                                                                         │               BOOT_ENTRY_LO != 0
                                                                         │               → 保存5块内存快照到 no-map 区:
                                                                         │                  RCPU0(5M) → Snapshot0
                                                                         │                  RCPU1(5M) → Snapshot0
                                                                         │                  OpenSBI(2M) → Snapshot0
                                                                         │                  SRAM(512K) → Snapshot1
                                                                         │                  RPMI(16K)  → Snapshot1
                                                                         │               spacemit_wakeup_c0()
                                                                         │               → 触发 AP Cluster0 上电
                                                                         │               __do_hibernation() 返回
                                                                         │                      │
                                                                  _start_warm 重新执行           │
                                                                  恢复 OpenSBI M-mode 上下文     │
                                                                  从掉电状态重新上电启动          │
                                    cpu_suspend() 返回 <──────────┤                      │
                                    dcache invalidate 两块快照区        │                      │
                                    memcpy 快照 → snap_backup           │                      │
                                    (snap_backup 随 swap 镜像持久化)    │                      │
       │ <──────────────────────────────┤                              │                      │
  syscore_suspend() 返回                 │                              │                      │
       │                                  │                              │                      │
  swsusp_arch_suspend()                  │                              │                      │
    save_processor_state()               │                              │                      │
    保存 CPU 上下文 / S-mode CSR          │                              │                      │
    swsusp_save() → 将内存镜像写入 swap  │                              │                      │
    (snap_backup 此时已在内存中，         │                              │                      │
     会随镜像一起写入 swap)              │                              │                      │
       │                                  │                              │                      │
  kernel_power_off() → 系统关机          │                              │                      │
```

---

### 4.2 休眠恢复流程

系统上电后，Linux 加载器从 swap 读取镜像并恢复内存，过程如下：

```
  Linux (S-mode)                  Linux k3-rproc 驱动            OpenSBI (M-mode)         ESOS/RCPU0
       │                                  │                              │                      │
  系统上电，正常 boot 到 Linux             │                              │                      │
       │                                  │                              │                      │
  检测到 swap 中有有效休眠镜像            │                              │                      │
       │                                  │                              │                      │
  PM_RESTORE_PREPARE 通知 ─────────────>│                              │                      │
                                  写 HIBERRESTORE_MAGIC                  │                      │
                                  到 no-map 区 (hiber_apuse_va)          │                      │
       │                                  │                              │                      │
  swsusp_arch_resume():                   │                              │                      │
    从 swap 恢复 Linux 内存镜像           │                              │                      │
       │                                  │                              │                      │
  syscore_resume() ───────────────────>│                              │                      │
                                  spacemit_rproc_syscore_resume():       │                      │
                                    读 hiber_apuse_va:                   │                      │
                                                                          │                      │
                                    ┌─ magic == DONE ─────────────────────────────────────────>│
                                    │  OpenSBI 上轮已恢复 RCPU         │              已在上轮 boot 时
                                    │  清除标志，直接跳过              │              由 OpenSBI 完成恢复
                                    │                                  │                      │
                                    └─ magic == REST ─────────────────┤                      │
                                       清零 RCPU0_BOOT_ENTRY           │                      │
                                       memcpy snap_backup →            │                      │
                                         snap_rcop_va (RCPU0/RCPU1/OpenSBI)                   │
                                         snap_srrpi_va (SRAM/RPMI)    │                      │
                                       cpu_suspend(SUSPEND_TO_DISK, ──>│                      │
                                         rproc_system_suspend)   发送 RPMI SYSSUSP            │
                                                                  (type=SUSPEND_TO_DISK) ────>│
                                                                         │               BOOT_ENTRY_LO == 0
                                                                         │               → 写入恢复入口地址
                                                                         │               → 从 no-map 恢复5块内存:
                                                                         │                  Snapshot0 → RCPU0/RCPU1/OpenSBI
                                                                         │                  Snapshot1 → SRAM/RPMI
                                                                         │               spacemit_wakeup_c0()
                                                                         │               jr entry (跳到恢复入口)
                                    cpu_suspend() 返回                  │                      │
                                    写 HIBERSNAP_DONE_MAGIC             │                      │
                                    到 hiber_apuse_va                   │                      │
       │ <──────────────────────────────┤                              │                      │
  syscore_resume() 返回                  │                              │                      │
       │                                  │                              │                      │
  hibernate_restore_image():             │                              │                      │
    切换到恢复页表                        │                              │                      │
    跳转到 __hibernate_cpu_resume()      │                              │                      │
    恢复 CPU 上下文和 S-mode CSR         │                              │                      │
       │                                  │                              │                      │
  系统恢复到休眠前状态                   │                              │                      │
```

---

### 4.3 内存快照区域布局

| 区域 | 运行时地址 | 大小 | 快照存储地址 |
|------|-----------|------|-------------|
| RCPU0 | 0x100200000 | 5 MB | Snapshot0 起始 |
| RCPU1 | 0x100800000 | 5 MB | Snapshot0 +5MB |
| OpenSBI | 0x100000000 | 2 MB | Snapshot0 +10MB |
| SRAM | 0x0 | 512 KB | Snapshot1 起始 |
| RPMI | 0x100e00000 | 16 KB | Snapshot1 +512KB |

**快照存储区（no-map，不被 Linux swap 覆盖）:**
- `Snapshot0`（snap_rcop）: `0x100f00000`，12 MB，存 RCPU0+RCPU1+OpenSBI
- `Snapshot1`（snap_srrpi）: `0x100701400`，~528 KB，存 SRAM+RPMI
- 专用栈: `0x100700400`，4 KB，`__hibernation_enter()` 切栈用
- AP 杂项（hiber_apuse）: `0x100700400`，4 KB，存 magic 标志位

**snap_backup**: Linux vmalloc 内存，保存两块快照区域的副本，随 swap 镜像一起持久化。

## 5. 调试方法

### 7.1 Linux 侧调试

```bash
# 使能 PM 调试
echo 1 > /sys/power/pm_debug_messages
echo core > /sys/power/pm_test  # 测试到 suspend_enter

# 查看唤醒源
cat /sys/kernel/debug/wakeup_sources

# 查看休眠统计
cat /sys/power/suspend_stats
```

### 7.2 ESOS 侧调试

在 `k3_pm.c` 中添加 `rt_kprintf` 调试输出:

```c
rt_kprintf("[PM] Entering suspend, mode=%d\n", mode);
rt_kprintf("[PM] Boot entry: 0x%lx\n", readl(RCPU_CORE0_BOOT_ENTRY_LO));
```

### 7.3 OpenSBI 侧调试

启用 OpenSBI 日志 (`CONFIG_ENABLE_LOGGING=y`):

```c
sbi_printf("Suspend type: %d, resume_addr: 0x%lx\n", sleep_type, resume_addr);
```

## 6. 常见问题

### 8.1 唤醒失败

**症状**: 系统进入休眠后无法唤醒

**可能原因**:
1. 唤醒源未正确配置
2. RCPU 未正确处理 M2 exit 中断
3. Boot entry 地址未设置或错误

**调试方法**:
```bash
# 检查中断配置
cat /proc/interrupts | grep -i wakeup

# 检查 RCPU 状态
# 需要通过 JTAG 或串口查看 ESOS 日志
```

### 8.2 Hibernate 恢复失败

**症状**: 从 disk 唤醒后系统无法正常启动

**可能原因**:
1. Swap 分区不足或损坏
2. 快照内存区域被覆盖
3. RCPU0 Boot entry 寄存器未正确清零

**检查**:
```bash
# 确认 swap 分区足够大
swapon --show
free -h

# 检查休眠镜像
cat /sys/power/image_size
cat /sys/power/reserved_size
```

### 8.3 RCPU1 无法唤醒

**症状**: RCPU0 唤醒后，RCPU1 仍处于休眠状态

**可能原因**:
1. Mailbox 通信失败
2. RCPU1 idle 配置未清除
3. 信号量同步问题

**解决方法**: 检查 `rt_lowpwr_poll()` 中的 mailbox 发送和 `RT24_CORE1_IDLE_CFG_REG` 清除逻辑

## 7. 总结

SpaceMIT K3 平台的休眠唤醒机制是一个多层次、多组件协作的复杂系统:

1. **Linux**: 负责高层 PM 策略和用户接口
2. **OpenSBI**: 作为固件层，处理 SBI 调用并协调硬件
3. **ESOS/RCPU**: 管理底层电源状态，控制集群和核心的上下电

两种休眠模式各有特点:
- **STR**: 快速唤醒，但需要保持内存供电
- **STD**: 完全断电，但需要更长的唤醒时间和额外的存储空间

整个流程涉及:
- CPU 上下文保存/恢复
- CSR 寄存器管理
- 内存镜像序列化
- 多核协调和同步
- 电源投票机制
- 中断处理和唤醒源管理

理解这些机制对于调试休眠唤醒问题、优化功耗和开发相关功能至关重要。


