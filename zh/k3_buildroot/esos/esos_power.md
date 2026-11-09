---
sidebar_position: 3
---

# ESOS 电源管理

**适用平台：** Spacemit K3 · **更新时间：** 2026-11-09

---

## 目录

1. [概述](#概述)
2. [系统架构](#系统架构)
3. [Regulator（电压管理）](#regulator电压管理)
4. [PM Domain 管理](#pm-domain-管理)
5. [系统休眠（SysSuspend）](#系统休眠syssuspend)
6. [RTC 服务](#rtc-服务)
7. [Power Key 服务](#power-key-服务)
8. [DTS 配置](#dts-配置)
9. [Kconfig 配置](#kconfig-配置)
10. [使用示例](#使用示例)
11. [调试方法](#调试方法)

---

## 概述

ESOS（Embedded Small OS）是 Spacemit 在 K3 SoC 上基于 RT-Thread 构建的小核实时操作系统，运行于 RCPU（RISC-V 管理核）之上。K3 芯片采用 AMP（非对称多处理）架构，同时运行两个 RCPU 实例（rcpu0 和 rcpu1）。

ESOS 通过 RPMI（RISC-V Platform Management Interface）共享内存通道为 AP 侧（Linux）提供以下五类平台服务：

| 服务 | 职责 |
|------|------|
| **Regulator** | 通过 I²C 控制多种 PMIC，响应 AP 侧调频时的电压调节请求 |
| **PM Domain** | 响应 Linux 各外设 power-switch 的电源状态开关请求，供 Linux genpd 框架调用 |
| **SysSuspend** | 协调 rcpu0/rcpu1 多 OS 有序断电，驱动系统进入 STR/STD 休眠并完成唤醒恢复 |
| **RTC** | 提供实时时钟读写与定时唤醒，支持作为系统休眠的唤醒源 |
| **Power Key** | 检测硬件按键事件，短按上报 Linux；长按或 Linux 发起关机时执行 PMIC 断电（shutdown） |

> ESOS 不参与 CPU 频率切换，仅通过 Regulator 服务执行电压调节；PM Domain 服务响应 Linux 各外设 power-switch 的电源状态开关请求；SysSuspend 服务协调系统级休眠/唤醒；RTC 服务提供定时唤醒；Power Key 服务处理按键事件并驱动关机（shutdown）流程。

---

## 系统架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Linux / AP 核                               │
│  genpd（PM Domain 请求）  sysfs suspend  RTC class  input（Power Key）│
│                              RPMI 驱动                               │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ RPMI 共享内存通道
                           │ Regulator · PMDomain · SysSuspend · RTC · PowerKey
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    ESOS（RCPU RT-Thread）                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │  Regulator  │  │  PM Domain  │  │  SysSuspend │                  │
│  │  多 PMIC    │  │  时钟/电源域│  │  多OS协调   │                  │
│  │  I²C 调压   │  │  开关管理   │  │  mailbox    │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
│  ┌─────────────┐  ┌─────────────┐                                    │
│  │     RTC     │  │  Power Key  │                                    │
│  │  定时唤醒   │  │  按键事件   │                                    │
│  └─────────────┘  └─────────────┘                                    │
└──────────────────────────┬───────────────────────────────────────────┘
                           ▼
              硬件：PMIC I²C · 时钟/电源域寄存器 · RTC · RCPU
```

---

## Regulator（电压管理）

源文件：`esos/bsp/spacemit/drivers/regulator/k3-regulator.c`

ESOS 通过通用调压器框架经 I²C 总线控制多种 PMIC，为 AP 核心供电。AP 侧在调频时通过 RPMI Regulator 服务请求 ESOS 调整电压；ESOS 不负责频率切换本身。

**调压时序规则：** 升频时 AP 侧先请求 ESOS 升压，待电压稳定后再切换 PLL；降频时 AP 侧先切 PLL 降频，再请求 ESOS 降压。违反此顺序可能导致 CPU 在低压下运行高频，引发不稳定。

### 支持的 PMIC

| 芯片 | DT compatible | 类型 | 电压范围 | 步长 |
|------|---------------|------|----------|------|
| P1 内部 DCDC | `p1-regulator` | DCDC | 500–1375 mV | 5 mV |
| P1 内部 LDO | `p1-regulator` | LDO | 500–3200 mV | 25 mV |
| IS6608 | `regulator-is6608` | DCDC | 534–1072 mV | 2 mV |
| TDA38740 | `regulator-tda38740-1/2` | DCDC | 531–1000 mV | ≈3.906 mV |
| IS6615A | `regulator-is6615a` | DCDC | 531–1000 mV | ≈1.953 mV |
| AU4562 | `regulator-au4562` | DCDC | 540–1080 mV | 5 mV |
| MP2973 | `regulator-mp2973` | DCDC | 540–1000 mV | 5 / 10 mV |


---

## PM Domain 管理

源文件：`esos/bsp/spacemit/drivers/rpmi/spacemit-pmdomain.c`

ESOS 通过 RPMI PMDomain 服务响应 Linux 各外设 power-switch 的电源状态开关请求。AP 侧 Linux 通过通用 PM Domain 框架（`genpd`）发起 domain 开关请求，经 RPMI 通道转发给 ESOS 执行实际的寄存器操作。

PM Domain 管理的主要场景：

- 外设驱动 suspend/resume 或者runtime suspend/resume时，关闭/恢复对应外设的 power-switch；
- 系统休眠时 ESOS 在多 OS 协调完成后，按顺序关闭非必要外设的 power-switch，降低待机功耗；
- 唤醒时按序恢复各外设 power-switch，确保外设驱动 resume 前硬件已就绪。

---

## 系统休眠（SysSuspend）

源文件：`esos/bsp/spacemit/drivers/rpmi/spacemit-hsm.c`

系统级休眠由 `spacemit_multiple_os_poll()` 线程协调，负责 rcpu0 与 rcpu1 之间的有序断电与恢复。

| 类型 | 恢复速度 | 说明 |
|------|----------|------|
| STR（Suspend-to-RAM） | 快（毫秒级） | RAM 保持供电，上下文驻留内存 |

### STR 挂起流程

1. Linux 触发系统挂起，AP 核依次完成自身断电序列并进入挂起状态。
2. ESOS `multiple_os_poll` 线程收到下电事件，通过 mailbox 发送低功耗进入消息给 rcpu1，等待 rcpu1 确认信号量。
3. rcpu1 完成自身断电后，ESOS 进入深度睡眠（RCPU 执行 WFI）。

### STR 唤醒恢复流程

1. 唤醒源（RTC alarm、Power Key 等）触发 ESOS 退出睡眠。
2. ESOS 通过 mailbox 通知 rcpu1 恢复，等待 rcpu1 就绪信号量。
3. 唤醒 C2 集群：写入 C2 启动向量（`C2_RVBADDR_LO/HI`），调用 `spacemit_wakeup_c2()`，等待 C2 上电完成。
4. 唤醒 C3 集群：同 C2 流程，调用 `spacemit_wakeup_c3()`。
5. 通过信号量释放已挂起的 AP 核，Linux 从挂起点继续执行。


---

## RTC 服务

源文件：`esos/bsp/spacemit/drivers/rpmi/spacemit-rtc.c`

ESOS 通过 RPMI RTC 服务将硬件实时时钟能力暴露给 AP 侧 Linux。Linux 通过标准 `/dev/rtc` 接口操作 RTC，RPMI 请求经 ESOS 转换为 RTC 硬件寄存器操作。

**主要功能：**

- **时间读写：** 读取/设置当前时间（年/月/日/时/分/秒）；
- **定时唤醒（alarm）：** 设置 alarm 时间点，系统休眠期间由 RTC 硬件在到期时触发唤醒中断；
- **唤醒源：** ESOS 将 RTC alarm 中断注册为 PM 唤醒源，收到中断后退出 RCPU 睡眠，启动 STR 恢复流程。

---

## Power Key 服务

源文件：`esos/bsp/spacemit/drivers/rpmi/spacemit-powerkey.c`

ESOS 通过 RPMI PowerKey 服务将开关机按键事件传递给 AP 侧 Linux。

**主要功能：**

- **短按：** 上报 `KEY_POWER` input 事件给 Linux，由用户空间决策（锁屏、休眠等）；
- **长按：** 触发关机流程，ESOS 执行 PMIC 断电序列；
- **开机/唤醒：** Power Key 长按可作为系统休眠的唤醒源，ESOS 退出睡眠后启动 STR 恢复流程；
- **关机通知：** Linux 调用关机 syscall 后，经 RPMI 通知 ESOS 执行最终的 PMIC 断电操作。

---

## DTS 配置

### Regulator 节点（IS6608 示例）

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

### TDA38740 节点（含休眠挂起配置）

```dts
vdd_cpu_a100: regulator@5c {
    compatible = "regulator-tda38740-1";
    reg = <0x5c>;
    regulator-name = "vdd-cpu-a100";
    regulator-min-microvolt = <531250>;
    regulator-max-microvolt = <1000000>;
    /* 休眠时通过此寄存器发送挂起信号 */
    spacemit,pm-suspend-reg = <0x5c>;
};
```

### SysSuspend 多 OS Mailbox 节点

```dts
lpm_mailbox: lpm-mailbox {
    compatible = "spacemit,k3-lpm-mailbox";
    mboxes = <&mailbox0 0>, <&mailbox0 1>;
    mbox-names = "tx", "rx";
    memory-region = <&lpm_shared_mem>;
    status = "okay";
};
```

### RTC 节点

```dts
rtc: rtc@d4010000 {
    compatible = "spacemit,k3-rtc";
    reg = <0x0 0xd4010000 0x0 0x1000>;
    interrupts = <GIC_SPI 46 IRQ_TYPE_LEVEL_HIGH>;
    status = "okay";
};
```

### Power Key 节点

```dts
powerkey {
    compatible = "spacemit,k3-powerkey";
    /* 关联 PMIC 的 ONKEY 引脚中断 */
    interrupt-parent = <&pmic>;
    interrupts = <0>;
    debounce-interval = <100>;   /* ms */
    status = "okay";
};
```

---

## Kconfig 配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `BSP_USING_K3_REGULATOR` | y | 启用调压器驱动（k3-regulator），RPMI Regulator 服务的后端 |
| `BSP_USING_RPMI_REGULATOR` | y | 启用 RPMI Regulator 服务，响应 AP 侧调压请求 |
| `BSP_USING_RPMI_PMDOMAIN` | y | 启用 RPMI PMDomain 服务，管理时钟域/电源域开关 |
| `BSP_USING_RPMI_SYSSUSP` | y | 启用 RPMI SysSuspend 服务，处理系统 STR/STD 休眠 |
| `BSP_USING_RPMI_RTC` | y | 启用 RPMI RTC 服务，提供时钟读写与定时唤醒 |
| `BSP_USING_RPMI_POWERKEY` | y | 启用 RPMI PowerKey 服务，检测开关机按键事件 |

---

## 使用示例

### 示例 1：AP 侧触发系统休眠（STR）

AP 侧 Linux 通过标准接口触发，ESOS 自动完成多 OS 协调：

```bash
# 触发 Suspend-to-RAM
echo mem > /sys/power/state

# 设置 RTC alarm 在 30 秒后唤醒，再触发休眠
rtcwake -m mem -s 30
```

### 示例 2：AP 侧读写 RTC

```bash
# 读取当前 RTC 时间
hwclock -r

# 将系统时间同步到 RTC
hwclock -w

# 设置 RTC alarm（使用 timerfd 或 /dev/rtc ioctl）
rtcwake -m no -s 60   # 60 秒后触发 alarm，不立即休眠
```

### 示例 3：ESOS 侧注册调压器（驱动开发参考）

```c
/* esos/bsp/spacemit/drivers/regulator/k3-regulator.c（简化） */
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

## 调试方法

### 查看 RPMI 服务状态（ESOS MSH）

```
msh> rpmi_dump
```

输出各 RPMI 服务组的注册状态、消息队列深度和最近一次请求的参数与结果。

### 查看调压器状态（ESOS MSH）

```
msh> regulator_dump
```

输出所有已注册调压器的当前电压、使能状态和 I²C 地址。

### 常见问题排查

**问题：系统休眠后无法唤醒**

检查 rcpu1 mailbox 通道配置（DTS 中 `lpm_mailbox` 节点）；确认 C2/C3 集群启动向量寄存器（`C2_RVBADDR_LO/HI`、`C3_RVBADDR_LO/HI`）在休眠前已正确设置；检查 ESOS 串口日志是否有 `wakeup_c2/c3` 相关错误。

**问题：RTC alarm 不触发唤醒**

确认 RTC 节点在 DTS 中正确配置，中断号与硬件手册一致；使用 `hwclock -r` 验证 ESOS RTC 服务是否正常响应；检查 alarm 时间是否晚于当前 RTC 时间。

**问题：电压调节失败**

确认 ESOS 侧调压器驱动加载正常（ESOS 串口日志中有 `regulator init` 信息）；检查 DTS 中调压器节点 I²C 地址和电压范围是否与实际 PMIC 型号匹配；调压路径使用 polling 方式，不依赖中断，若仍失败请检查 I²C 总线是否被其他设备占用。

**问题：Power Key 事件未上报 Linux**

检查 ESOS 串口日志是否有 `powerkey` 中断收到的打印；确认 DTS 中 `interrupt-parent` 和中断号与 PMIC/GPIO 硬件一致；检查 Linux 侧 `dmesg` 是否有 RPMI PowerKey 驱动注册成功的日志。
