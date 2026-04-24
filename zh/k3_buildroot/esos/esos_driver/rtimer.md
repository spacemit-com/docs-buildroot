# K3 小核（RT24）Timer 介绍

介绍 K3 小核 RT24 的硬件定时器配置和使用方式

## 模块介绍

RT24 是 K3 平台的实时小核，基于 RISC-V 64 位架构，运行 RT-Thread 实时操作系统（ESOS）。RT24 拥有独立的时钟控制单元（CCU）和硬件定时器系统，与大核（X100/A100）完全隔离。

### 功能介绍

RT24 硬件定时器（hwtimer）提供以下功能：
- 提供高精度硬件计时，计数分辨率 1 µs
- 支持单次（ONESHOT）触发模式
- 支持中断回调，定时到期后自动触发用户回调函数
- 支持运行时动态启停
- 通过 RT-Thread 设备框架统一管理，应用层通过标准设备接口访问

### 源码结构介绍

小核定时器相关代码目录如下：

```
esos/bsp/spacemit/
├── drivers/
│   ├── hwtimer/
│   │   ├── k3-hwtimer.c          # 硬件定时器驱动
│   │   └── hwtimer-test.c        # 定时器测试代码
│   ├── clk/
│   │   ├── ccu-spacemit-k3.c     # RCPU CCU 时钟驱动
│   │   └── ccu-spacemit-k3.h     # 时钟 ID 定义（含 CLK_RCPU_TIMERx）
│   └── rpmi/
│       ├── spacemit-clk.c        # RPMI 时钟服务通用层
│       └── k3/k3-os0_clk.c       # OS0 RCPU 频率档位配置
└── platform/
    └── rt24/
        ├── k3.dtsi               # 设备树（rtimer0~3 节点定义）
        └── ccu-spacemit-k3.h     # 时钟宏定义
```

## 关键特性

- 共 4 个硬件定时器模块（rtimer0 ~ rtimer3），每模块 3 个独立计数器，合计 **12 个计数器通道**
- 计数器工作频率 1 MHz，计时分辨率 1 µs
- 支持 fastclk（3.25 MHz）和 32K 时钟两种时钟源，可按计数器独立切换
- 计数器为向上计数（CNTMODE_UP），最大计数值 0xFFFFFFFF（约 4295 秒）
- 通过 RT-Thread hwtimer 框架注册，设备名格式：`timer:<timer_id>:<counter_id>`

## 配置介绍

### CONFIG 配置

在 menuconfig 中使能硬件定时器：

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

对应 defconfig 配置项：

```
CONFIG_RT_USING_HWTIMER=y
CONFIG_BSP_USING_HWTIMER=y
# CONFIG_BSP_HWTIMER_TEST is not set   # 开启测试时改为 y
```

> 注意：默认 defconfig（`rt24_os0_rcpu_defconfig`）中 hwtimer 为关闭状态，需手动开启。

### DTS 配置

定时器节点定义在 `esos/bsp/spacemit/platform/rt24/k3.dtsi`，共 4 个 rtimer 节点，每个节点包含 3 个 counter 子节点。

以 rtimer0 为例：

```dts
rtimer0: rtimer0@c0889000 {
    compatible = "spacemit,rtimer0";
    reg = <0xc0889000 0xc8>;
    num_counter = <3>;
    spacemit,timer-fastclk-frequency = <3250000>;   /* fastclk: 3.25 MHz */
    spacemit,timer-apb-frequency = <52000000>;       /* APB: 52 MHz */
    spacemit,timer-frequency = <1000000>;            /* 工作频率: 1 MHz */
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

使能某个定时器时，将 rtimer 节点及对应 counter 子节点的 `status` 改为 `"okay"`：

```dts
&rtimer0 {
    status = "okay";
    count0 { status = "okay"; };
    count1 { status = "okay"; };
    count2 { status = "okay"; };
};
```




## API 介绍

RT24 hwtimer 驱动实现了以下 RT-Thread 标准接口：

```c
/* 初始化定时器（框架调用，应用层无需直接调用） */
static void spacemit_hwtimer_init(struct rt_hwtimer_device *timer, rt_uint32_t state);

/* 启动定时器，cnt 为计数值，mode 为 ONESHOT */
static rt_err_t spacemit_hwtimer_start(struct rt_hwtimer_device *timer,
                                        rt_uint32_t cnt, rt_hwtimer_mode_t mode);

/* 停止定时器 */
static void spacemit_hwtimer_stop(struct rt_hwtimer_device *timer);

/* 读取当前计数值 */
static rt_uint32_t spacemit_hwtimer_count_get(struct rt_hwtimer_device *timer);

/* 控制接口，支持 HWTIMER_CTRL_FREQ_SET / HWTIMER_CTRL_STOP */
static rt_err_t spacemit_hwtimer_control(struct rt_hwtimer_device *timer,
                                          rt_uint32_t cmd, void *args);
```

定时器信息（`hw_info`）：

| 参数 | 值 | 说明 |
|-----|---|------|
| `maxfreq` | 1000000 | 最大工作频率 1 MHz |
| `minfreq` | 1000000 | 最小工作频率 1 MHz（固定） |
| `maxcnt` | 0xFFFFFFFF | 最大计数值 |
| `cntmode` | `HWTIMER_CNTMODE_UP` | 向上计数模式 |

## 测试介绍

### 1. 使能 hwtimer 驱动

在 menuconfig 中开启 `BSP_USING_HWTIMER` 和 `BSP_HWTIMER_TEST`，或直接修改 defconfig：

```
CONFIG_RT_USING_HWTIMER=y
CONFIG_BSP_USING_HWTIMER=y
CONFIG_BSP_HWTIMER_TEST=y
```

同时在 DTS 中将 rtimer0 及其 count0 节点 status 改为 `"okay"`。

### 2. 运行测试

系统启动后，在 RT-Thread shell 中执行：

```
msh > hwtimer_test
```

测试程序实现思路（`hwtimer-test.c`）：
1. 通过 `rt_device_find("timer:0:0")` 查找定时器设备，确认驱动加载成功
2. 打开设备，注册超时回调函数 `timer_timeout_cb`
3. 设置为单次模式（`HWTIMER_MODE_ONESHOT`）
4. 设置超时时间为 5 秒，启动定时器
5. 等待回调触发，打印 `enter hardware timer isr`
6. 停止并关闭设备

预期输出：

```
SetTime: Sec 5, Usec 0
enter hardware timer isr
Close timer:0:0
```

## FAQ

### Q1: 如何确认 hwtimer 驱动加载成功？

在 RT-Thread shell 中执行 `list_device`，若看到 `timer:0:0`、`timer:0:1` 等设备，说明驱动加载成功。

### Q2: 定时器精度是多少？

工作频率固定为 1 MHz，计时分辨率为 1 µs。

### Q3: 最多可以同时使用多少个定时器？

4 个 rtimer 模块 × 3 个 counter = 12 个独立计数器通道，可同时使用。

### Q4: 看门狗定时器与 hwtimer 是否冲突？

看门狗（`rwdt`）复用 `CLK_RCPU_TIMER1`（与 rtimer0 相同时钟），但地址独立，不冲突。使用时注意不要同时开启 rtimer0 和 wdt 的同一时钟资源。


