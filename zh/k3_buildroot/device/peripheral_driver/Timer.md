# K3 Timer

介绍 K3 大核的硬件定时器配置和使用方式

## 模块介绍

K3 大核运行 Linux 内核，其定时器系统由 Linux clocksource/clockevent 框架管理。K3 平台提供了一个专用的 SoC 硬件定时器（`timer-k1x`），作为系统的 clock event 设备，为内核调度、高精度定时等提供时钟事件支持。

此外，RISC-V 架构本身提供了基于 `timebase-frequency` 的通用 RISC-V timer，用于系统时间基准。

### 功能介绍

K3 大核定时器系统包含两个层次：

**SoC 硬件定时器（spacemit,soc-timer）**：
- 作为 Linux clock event device，为内核提供定时中断
- 支持 broadcast timer 模式，为所有 CPU 提供统一的定时事件
- 支持 local timer 模式，为单个 CPU 提供独立定时事件
- 支持电源管理（suspend/resume）

**RISC-V 通用 Timer**：
- 基于 RISC-V 规范的 `time` CSR 寄存器
- 时基频率（timebase-frequency）：24 MHz
- 通过 SBI 接口操作，用于系统时间计量

### 源码结构介绍

大核定时器相关代码目录如下：

```
linux-6.18/
├── drivers/clocksource/
│   ├── timer-k1x.c              # K3 SoC 硬件定时器驱动
│   ├── timer-riscv.c            # RISC-V 通用 timer 驱动
│   └── Kconfig                  # Timer 驱动配置选项
└── arch/riscv/boot/dts/spacemit/
    ├── k3.dtsi                  # K3 SoC 主设备树（含 timer0 节点）
    └── k3-cpus.dtsi             # CPU 配置（含 riscv-timer 节点）
```

## 关键特性

- 支持最多 3 个 timer 模块（`SPACEMIT_MAX_TIMER = 3`），每个模块 3 个计数器
- 支持 broadcast timer 和 per-CPU local timer 两种模式
- 支持 fastclk（1 MHz）和 32K 时钟源切换
- clock event 仅支持 ONESHOT 模式（`CLOCK_EVT_FEAT_ONESHOT`）
- rating 为 200
- 支持 suspend/resume 电源管理
- RISC-V timer 时基频率 24 MHz

## 配置介绍

### CONFIG 配置

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

K3 defconfig（`arch/riscv/configs/k3_defconfig`）中已默认开启：

```
CONFIG_SPACEMIT_K1X_TIMER=y
CONFIG_HIGH_RES_TIMERS=y
```

### DTS 配置

#### SoC 硬件定时器节点

定义于 `arch/riscv/boot/dts/spacemit/k3.dtsi`：

```dts
timer0: timer@d4016000 {
    compatible = "spacemit,soc-timer";
    reg = <0x0 0xd4016000 0x0 0xc8>;
    spacemit,timer-id = <0>;
    spacemit,timer-fastclk-frequency = <1000000>;   /* fastclk: 1 MHz */
    spacemit,timer-apb-frequency = <102400000>;      /* APB: 102.4 MHz */
    spacemit,timer-frequency = <1000000>;            /* 工作频率: 1 MHz */
    clocks = <&syscon_apbc CLK_APBC_TIMERS1>,
             <&syscon_apbc CLK_APBC_TIMERS1_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_apbc RESET_APBC_TIMERS1>;
    status = "okay";

    counter0 {
        compatible = "spacemit,timer-match";
        interrupts = <26 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&saplic>;
        spacemit,timer-broadcast;       /* broadcast 模式，服务所有 CPU */
        spacemit,timer-counter-id = <0>;
        status = "okay";
    };
};
```

counter 子节点关键属性说明：

| 属性 | 说明 |
|-----|------|
| `spacemit,timer-counter-id` | 计数器编号（0~2） |
| `spacemit,timer-broadcast` | 配置为 broadcast timer，服务所有 CPU |
| `spacemit,timer-counter-cpu` | 配置为 local timer，绑定到指定 CPU（与 broadcast 二选一） |

#### RISC-V 通用 Timer 节点

定义于 `arch/riscv/boot/dts/spacemit/k3-cpus.dtsi`：

```dts
/ {
    cpus {
        timebase-frequency = <24000000>;   /* 时基频率 24 MHz */
    };

    riscv-timer {
        compatible = "riscv,timer";
        riscv,timer-cannot-wake-cpu;       /* timer 中断不能唤醒 CPU */
    };
};
```



## Debug 介绍


### 查看系统 timer 信息

```bash
# 查看当前 clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# 查看可用 clocksource
cat /sys/devices/system/clocksource/clocksource0/available_clocksource

# 查看 clock event 设备信息
cat /proc/timer_list
```

### 查看 RISC-V timer 时基

```bash
# 查看 timebase-frequency（单位 Hz）
cat /sys/firmware/devicetree/base/cpus/timebase-frequency | od -An -tu4
# 输出应为 24000000
```

## FAQ

### Q1: broadcast timer 和 local timer 有什么区别？

broadcast timer 绑定到 `cpu_possible_mask`，为所有 CPU 提供统一的定时中断，适用于 CPU 进入低功耗状态时本地 timer 停止的场景。local timer 绑定到单个 CPU，中断只发给该 CPU。

### Q2: 为什么 clock event 只支持 ONESHOT 模式？

K3 timer 硬件每次触发后需要重新设置匹配值，不支持自动重载周期触发，因此驱动只注册 `CLOCK_EVT_FEAT_ONESHOT`。

### Q3: RISC-V timer 和 SoC timer 各自的作用是什么？

RISC-V timer（基于 `time` CSR）提供系统时间基准（timebase），频率固定为 24 MHz，通过 SBI 接口访问。SoC timer（`spacemit,soc-timer`）作为 clock event device，负责产生定时中断驱动内核调度。

### Q4: 看门狗和 timer 是否共用时钟？

看门狗（`watchdog@d4014000`）使用 `CLK_APBC_TIMERS0`，SoC timer（`timer@d4016000`）使用 `CLK_APBC_TIMERS1`，两者时钟独立，不冲突。
