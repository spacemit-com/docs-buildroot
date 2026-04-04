# PWM

介绍 K3 小核心（ESOS/RTT）平台 PWM 驱动的实现方式、设备树写法、consumer 使用方式，以及 PWM 与 pinctrl 的配合关系。

## 模块介绍

K3 小核心平台上的 PWM 子系统用于输出可调周期、可调占空比的脉冲波形，常见用途包括：

- 背光亮度调节
- 蜂鸣器驱动
- 时钟/方波输出
- 其他需要脉宽调制信号的外设控制

## 功能介绍

K3 小核心 PWM 当前主要提供以下能力：

- 注册多个独立 PWM 设备
- 支持通过 DTS `compatible` 自动探测并注册 PWM 控制器
- 支持设置 period / pulse
- 支持 enable / disable
- 支持 RT-Thread PWM 设备模型
- 支持 shell 命令 `pwm_enable` / `pwm_disable` / `pwm_set` / `pwm_get`
- 支持板级 pinctrl 默认状态自动下发

## 源码结构介绍

K3 ESOS PWM 相关代码主要位于：

```text
~/k3-sdk/esos/
|-- bsp/spacemit/drivers/pwm/pwm-pxa.c
|-- bsp/spacemit/drivers/pwm/pwm-test.c
|-- components/drivers/misc/rt_drv_pwm.c
|-- components/drivers/include/drivers/rt_drv_pwm.h
`-- bsp/spacemit/platform/rt24/
    |-- k3.dtsi
    |-- k3-pinctrl.dtsi
    |-- os0_rcpu/dts/k3_deb1.dts
    `-- os1_rcpu/dts/k3_deb1.dts
```

其中：

- `bsp/spacemit/drivers/pwm/pwm-pxa.c`：K3 小核心 PWM 主驱动
- `bsp/spacemit/drivers/pwm/pwm-test.c`：PWM 实际测试与 consumer 示例
- `components/drivers/misc/rt_drv_pwm.c`：RT-Thread PWM 设备封装层
- `components/drivers/include/drivers/rt_drv_pwm.h`：PWM command、配置结构、接口定义
- `k3.dtsi`：PWM 控制器节点
- `k3-pinctrl.dtsi`：PWM 输出 PAD 的 pinctrl 复用配置

## 驱动定位方式

先从设备树 `compatible` 反查真实驱动。

在 `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3.dtsi` 中，可以看到小核心 PWM 控制器节点，例如：

```dts
rpwm0: pwm@c088d100 {
    compatible = "spacemit,k1x-rpwm0";
    reg = <0xc088d100 0x10>;
    clocks = <&ccu CLK_RCPU_PWM0>, <&ccu CLK_RST_RCPU_PWM0>;
    clock-rate = <24576000>;
    k1x,pwm-disable-fd;
    status = "disabled";
};
```

同类节点一直到：

- `spacemit,k1x-rpwm0`
- `spacemit,k1x-rpwm1`
- `spacemit,k1x-rpwm2`
- `spacemit,k1x-rpwm3`
- `spacemit,k1x-rpwm4`
- `spacemit,k1x-rpwm5`
- `spacemit,k1x-rpwm6`
- `spacemit,k1x-rpwm7`
- `spacemit,k1x-rpwm8`
- `spacemit,k1x-rpwm9`

在 `~/k3-sdk/esos/bsp/spacemit/drivers/pwm/pwm-pxa.c` 中，驱动通过：

```c
static struct dtb_compatible_array __compatible[] = {
    { .compatible = "spacemit,k1x-rpwm0", .data = "rpwm0" },
    { .compatible = "spacemit,k1x-rpwm1", .data = "rpwm1" },
    ...
    { .compatible = "spacemit,k1x-rpwm9", .data = "rpwm9" },
};
```

逐个匹配这些 compatible，并把设备名注册成：

- `rpwm0`
- `rpwm1`
- ...
- `rpwm9`

因此可以确认：

- K3 小核心 RT24 当前 PWM 文档应该基于：
  - `~/k3-sdk/esos/bsp/spacemit/drivers/pwm/pwm-pxa.c`
  - `~/k3-sdk/esos/components/drivers/misc/rt_drv_pwm.c`
  - `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3.dtsi`
  - `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3-pinctrl.dtsi`

## PWM 测试用例与调试使用

这部分更适合放在 PWM 驱动实现、consumer 获取方式和 pinctrl 配合关系之后，因为测试代码本质上是在验证：

- PWM 设备是否真的注册成功
- consumer 是否能正确拿到 `struct rt_device_pwm *`
- `period/pulse` 设置是否能走通驱动和 RT-Thread PWM 封装层
- 输出波形是否随着参数变化而变化

K3 ESOS PWM 相关测试代码位于：

```text
~/k3-sdk/esos/bsp/spacemit/drivers/pwm/pwm-test.c
```

这个文件既是调试命令集合，也是当前最直接的 PWM consumer 使用参考。

### 1. `pwm_dev_enum`：先枚举系统里可用的 PWM 设备

`pwm-test.c` 里首先提供了设备枚举命令：

```c
static int pwm_dev_enum(int argc, char *argv[])
```

它会依次尝试：

- `rpwm0 ~ rpwm9`
- `pwm0 ~ pwm9`

并通过：

```c
rt_device_find(pwm_names[i])
```

检查设备是否存在。

这个测试命令的意义非常直接：

- 如果枚举不到 `rpwmX`
- 先不要急着怀疑 `rt_pwm_set()`
- 应先检查：
  - 对应 `&rpwmX` 节点是否在板级 DTS 打开
  - `pwm_probe()` 是否成功执行
  - `pinctrl_apply_default()` 是否因 pinctrl 节点不可用而提前失败

也就是说，**设备枚举是 PWM 测试的第一步**。

### 2. `get_pwm_device()`：测试代码本身如何拿到 consumer 设备句柄

文档前面已经说明，普通 consumer 需要先通过：

```c
pwm_dev = (struct rt_device_pwm *)rt_device_find("rpwm0");
```

拿到 `struct rt_device_pwm *`。

而 `pwm-test.c` 中的测试代码也正是这样做的，只不过它又包了一层 `get_pwm_device()`：

- 如果用户命令行指定了设备名，就优先找指定设备
- 如果没指定，就自动遍历 `rpwm0 ~ rpwm9` / `pwm0 ~ pwm9`
- 找到第一个可用设备后继续测试

这说明：

- 测试代码并没有绕开正常 consumer 路径
- 它本身就是标准 consumer 用法的实际例子

### 3. `pwm_test_0001`：周期设置测试

执行方式：

```sh
pwm_test_0001 rpwm0
```

或者不带设备名，让它自动探测：

```sh
pwm_test_0001
```

这个测试会：

1. 获取 PWM 设备
2. 固定 `pulse = 25000ns`
3. 依次切换多个周期值，例如：
   - `50000`
   - `100000`
   - `150000`
   - `200000`
4. 每次调用：
   - `rt_pwm_set()`
   - `rt_pwm_enable()`
5. 最后归零并 disable

它适合验证：

- `period` 配置路径是否正常
- 驱动对 `prescale/pv` 的换算是否能覆盖常见周期
- 输出频率是否随设置变化

如果你把 PWM 输出接到 LED、示波器或逻辑分析仪上，这个测试最容易观察到“频率变化”。

### 4. `pwm_test_0002`：占空比设置测试

执行方式：

```sh
pwm_test_0002 rpwm0
```

它会固定：

- `period = 100000ns`

然后依次设置多个 `pulse`：

- `0`
- `25000`
- `50000`
- `75000`
- `100000`

也就是让 duty 从：

- `0%`
- `25%`
- `50%`
- `75%`
- `100%`

逐级变化。

这个测试最适合验证：

- `rt_pwm_set()` 的 duty 配置链路
- `k1x,pwm-disable-fd` 对 100% duty 的边界行为
- 输出占空比是否随设置变化

如果负载是 LED，通常能直接观察亮度变化；如果接示波器，则能直接看到高电平宽度变化。

### 5. watch 模式

`pwm-test.c` 还支持一个很实用的观察模式：

```sh
pwm_test_0001 rpwm0 watch
pwm_test_0002 rpwm0 watch
```

在这个模式下，测试间隔会从默认 `100ms` 延长到 `1000ms`，便于人工观察：

- LED 亮灭/亮度变化
- 示波器波形变化
- 逻辑分析仪采样结果

所以如果是硬件联调，建议优先用 `watch` 模式。

### 6. 测试前的准备条件

PWM 测试成功与否，不只取决于 `pwm-test.c`，还取决于前面的 DTS 和 pinctrl 是否已经准备好。

在执行测试前，至少应确认：

1. 对应 PWM 节点已启用，例如：

```dts
&rpwm0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rpwm0_0_cfg>;
    status = "okay";
};
```

2. `pinctrl-0` 选择的确实是当前板子实际走线的 PWM PAD
3. 系统启动后 `pwm_probe()` 已成功把设备注册成 `rpwm0`
4. 通过 `pwm_dev_enum` 或 `rt_device_find("rpwm0")` 能找到设备

否则即使 `pwm_test_0001/0002` 命令本身存在，也可能只是“软件接口可调用”，而外部 PAD 上没有任何波形输出。

### 7. 测试章节为什么应该放在后面

这里不适合只留一个“示例使用”的空标题，更合适的做法是放在文档后半部分，作为**验证与调试**章节。

原因是：

- 前文先解释清楚 PWM 控制器、寄存器模型、consumer 获取方式和 pinctrl 配合关系
- 后文再说明如何用测试命令去验证这些配置是否真的生效

这样读者先理解“驱动怎么工作”，再理解“怎么证明它工作了”。

## K3 PWM 控制器节点说明

K3 RT24 小核心当前在 `k3.dtsi` 中定义了 `rpwm0 ~ rpwm9` 共 10 个 PWM 节点。

以 `rpwm0` 为例：

```dts
rpwm0: pwm@c088d100 {
    compatible = "spacemit,k1x-rpwm0";
    reg = <0xc088d100 0x10>;
    clocks = <&ccu CLK_RCPU_PWM0>, <&ccu CLK_RST_RCPU_PWM0>;
    clock-rate = <24576000>;
    k1x,pwm-disable-fd;
    status = "disabled";
};
```

它表示：

- PWM 控制器实例名是 `rpwm0`
- 寄存器基址是 `0xc088d100`
- 寄存器空间大小是 `0x10`
- 使用两个 clock：功能时钟和复位时钟
- 默认 `clock-rate = <24576000>`
- 通过 `k1x,pwm-disable-fd` 指示 FD 特性处理方式
- 默认状态为 `disabled`，需要在板级 dts 中显式打开

其余 `rpwm1 ~ rpwm9` 只是：

- compatible 不同
- 寄存器地址不同
- clock ID 不同

驱动模型是一致的。

## PWM 驱动初始化流程

### `pwm_probe()`

`pwm-pxa.c` 的入口函数是：

```c
int pwm_probe(void)
```

并通过：

```c
INIT_DEVICE_EXPORT(pwm_probe);
```

在系统初始化阶段执行。

它的处理流程是：

1. 遍历 `__compatible[]` 中列出的 `spacemit,k1x-rpwm0 ~ spacemit,k1x-rpwm9`
2. 用 `dtb_node_find_compatible_node()` 查找对应 dts 节点
3. 用 `dtb_node_device_is_available()` 判断节点是否 `status = "okay"`
4. 分配 `struct pxa_pwm_chip`
5. 读取 `reg` 获取 `mmio_base`
6. 调用 `pinctrl_apply_default(compatible_node)` 自动下发该 PWM 节点的默认 pinctrl 状态
7. 获取时钟与复位时钟：
   - `pc->clk = of_clk_get(compatible_node, 0)`
   - `pc->rst = of_clk_get(compatible_node, 1)`
8. 使能 reset 相关时钟：
   - `clk_prepare_enable(pc->rst)`
9. 解析可选属性：
   - `k1x,pwm-disable-fd`
   - `clock-rate`
10. 调用：

```c
rt_device_pwm_register(dev, pc->name, &pxa_pwm_ops, NULL);
```

把 PWM 设备注册成 RT-Thread 设备，例如 `rpwm0`。

这说明 K3 小核心 PWM 子系统也是两层结构：

- 一层是 SoC 驱动 `pwm-pxa.c`
- 一层是 RT-Thread PWM 框架 `rt_drv_pwm.c`

## PWM 寄存器与基本配置模型

驱动里定义的核心寄存器为：

```c
#define PWMCR   (0x00)
#define PWMDCR  (0x04)
#define PWMPCR  (0x08)
```

以及关键位：

```c
#define PWMCR_SD   (1 << 6)
#define PWMDCR_FD  (1 << 10)
```

驱动注释已经给出换算公式：

```c
period_ns = 10^9 * (PRESCALE + 1) * (PV + 1) / PWM_CLK_RATE
duty_ns   = 10^9 * (PRESCALE + 1) * DC / PWM_CLK_RATE
```

因此 `period` / `pulse` 并不是直接写入寄存器，而是先根据时钟频率换算出：

- `prescale`
- `pv`
- `dc`

再写到寄存器中。

## PWM 参数设置实现

### `pxa_pwm_config()`

真正把 `period_ns` / `duty_ns` 配置到硬件的函数是：

```c
int pxa_pwm_config(struct rt_device_pwm *dev,
                   rt_uint64_t duty_ns, rt_uint64_t period_ns)
```

它的关键逻辑包括：

#### 1. 先校验 duty 不能大于 period

```c
if (duty_ns > period_ns)
    return -RT_EINVAL;
```

这也是 `pwm-test.c` 里专门测试的异常场景之一。

#### 2. 根据时钟频率换算周期总 cycle 数

```c
c = clk_get_rate(pc->clk);
c = c * period_ns;
do_div(c, 1000000000);
period_cycles = c;
```

#### 3. 计算 prescale 和 pv

```c
prescale = (period_cycles - 1) / 1024;
pv = period_cycles / (prescale + 1) - 1;
```

并要求：

```c
if (prescale > 63)
    return -RT_EINVAL;
```

这说明当前硬件 prescale 最大是 63。

#### 4. 计算占空比 DC

如果 `duty_ns != period_ns`：

```c
dc = ((pv + 1) * duty_ns / period_ns);
```

如果 `duty_ns == period_ns`，则走满占空比处理逻辑。

#### 5. 写寄存器

```c
writel(prescale | PWMCR_SD, pc->mmio_base + PWMCR);
writel(dc, pc->mmio_base + PWMDCR);
writel(pv, pc->mmio_base + PWMPCR);
```

也就是说，`PWM_CMD_SET` 最终对应的是一次寄存器重配置。

## `k1x,pwm-disable-fd` 的作用

在 DTS 节点里可见：

```dts
k1x,pwm-disable-fd;
```

驱动里会检测这个属性：

```c
property_status = dtb_node_get_dtb_node_property(compatible_node,
                "k1x,pwm-disable-fd", RT_NULL);
if (property_status)
    pc->dcr_fd = 1;
else
    pc->dcr_fd = 0;
```

这个属性真正影响的，是 **`duty_ns == period_ns`，也就是请求 100% 占空比时，驱动到底怎么写 `PWMDCR` 和 `PWMPCR`**。

### 先看普通情况

当 `duty_ns != period_ns` 时，驱动直接按比例计算：

```c
dc = ((pv + 1) * duty_ns / period_ns);
```

然后写入：

```c
writel(prescale | PWMCR_SD, pc->mmio_base + PWMCR);
writel(dc, pc->mmio_base + PWMDCR);
writel(pv, pc->mmio_base + PWMPCR);
```

所以普通占空比下，`k1x,pwm-disable-fd` 不起作用。

### 100% 占空比时的分支

真正有差异的是：

```c
if (duty_ns == period_ns)
```

这时驱动分成两种行为。

#### 情况 A：声明了 `k1x,pwm-disable-fd`

驱动会走：

```c
if (pc->dcr_fd)
    dc = PWMDCR_FD;
```

也就是把 `PWMDCR` 直接写成 `PWMDCR_FD`。

这里的含义是：

- 不再按普通比例去计算 `dc`
- 而是使用控制器定义的 FD 特殊编码来表示“满占空比”

也就是说，**100% duty 由硬件 FD 语义表达**。

#### 情况 B：没有声明 `k1x,pwm-disable-fd`

驱动会走普通数值计算路径：

```c
dc = (pv + 1) * duty_ns / period_ns;
if (dc >= PWMDCR_FD) {
    dc = PWMDCR_FD - 1;
    pv = dc - 1;
}
```

因为此时 `duty_ns == period_ns`，理论上：

```c
dc = pv + 1
```

但如果这个值已经碰到 `PWMDCR_FD` 这个特殊阈值，驱动会主动退一格：

- `dc = PWMDCR_FD - 1`
- `pv = dc - 1`

也就是说：

- **不使用 FD 特殊编码**
- 而是尽量构造一个“接近 100% duty 的普通数值配置”
- 避免把 `PWMDCR_FD` 当成特殊功能位误触发

### 这个属性为什么重要

所以 `k1x,pwm-disable-fd` 本质上是在告诉驱动：

- 当用户请求 100% duty 时
- 是要不要使用硬件的 FD 特殊表达方式

换句话说，它决定的是 **100% duty 的编码策略**，不是普通 duty 的编码策略。

可以把它理解成：

- 带 `k1x,pwm-disable-fd`：100% duty 直接走 FD 特殊值
- 不带 `k1x,pwm-disable-fd`：100% duty 尽量退化成普通数值范围内的最大占空比配置

### 对代码行为的实际影响

这个属性会直接影响以下几个变量最终写入寄存器时的值：

- `dc`
- 在某些分支下还会影响 `pv`

因此它影响的不是抽象概念，而是：

- `PWMDCR` 最终写什么
- `PWMPCR` 是否会被一并调整
- 100% duty 最终是“真正的 FD 语义”，还是“普通数值方式下尽量逼近 100%”

也就是说，这不是一个普通 consumer 属性，而是 PWM 控制器驱动自身在 **满占空比边界条件** 下的行为开关。

## RT-Thread PWM 设备模型

### `rt_device_pwm_register()`

`rt_drv_pwm.c` 提供了标准 RT-Thread PWM 设备包装层。

PWM 驱动完成底层寄存器控制后，通过：

```c
rt_device_pwm_register(dev, pc->name, &pxa_pwm_ops, NULL);
```

把设备注册到 RT-Thread 设备模型。

### 控制命令

`rt_drv_pwm.h` 定义了：

```c
#define PWM_CMD_ENABLE      (128 + 0)
#define PWM_CMD_DISABLE     (128 + 1)
#define PWM_CMD_SET         (128 + 2)
#define PWM_CMD_GET         (128 + 3)
```

以及配置结构：

```c
struct rt_pwm_configuration
{
    int channel;
    rt_uint32_t period;
    rt_uint32_t pulse;
    rt_bool_t complementary;
};
```

当前 `pwm-pxa.c` 里真正实现的是：

- `PWM_CMD_ENABLE`
- `PWM_CMD_DISABLE`
- `PWM_CMD_SET`

而 `PWM_CMD_GET` / `PWMN_CMD_ENABLE` / `PWMN_CMD_DISABLE` 在这个驱动里没有真正展开实现逻辑。

## consumer 的实际使用方式

### 方式 1：直接使用 RT-Thread PWM API

K3 小核心当前 PWM 的典型 consumer 使用方式，是直接通过 RT-Thread PWM API。

在 `rt_drv_pwm.c` 中，提供了：

- `rt_pwm_enable()`
- `rt_pwm_disable()`
- `rt_pwm_set()`
- `rt_pwm_get()`

但这里要先说清楚一件事：**consumer 先要拿到 `struct rt_device_pwm *pwm_dev`，然后才能调用这些接口。**

当前代码里最直接的获取方式就是按设备名从 RT-Thread device model 里查找：

```c
struct rt_device_pwm *pwm_dev;

pwm_dev = (struct rt_device_pwm *)rt_device_find("rpwm0");
if (pwm_dev == RT_NULL) {
    /* 设备不存在，通常说明对应 PWM 节点未启用或驱动未注册成功 */
    return -RT_ERROR;
}
```

这个设备名来自 `pwm-pxa.c` 的注册逻辑：

```c
rt_device_pwm_register(dev, pc->name, &pxa_pwm_ops, NULL);
```

而 `pc->name` 则来自 `__compatible[]` 映射表，例如：

- `spacemit,k1x-rpwm0` -> `rpwm0`
- `spacemit,k1x-rpwm1` -> `rpwm1`
- ...
- `spacemit,k1x-rpwm9` -> `rpwm9`

所以从 consumer 角度看，获取 PWM 设备的基本路径是：

1. 板级 DTS 把目标 `&rpwmX` 节点打开
2. `pwm_probe()` 匹配到对应 compatible
3. 驱动把它注册成名为 `rpwmX` 的 RT-Thread PWM 设备
4. consumer 再通过 `rt_device_find("rpwmX")` 拿到 `struct rt_device_pwm *`

拿到 `pwm_dev` 后，才可以这样使用：

```c
rt_pwm_set(pwm_dev, 1, 100000, 50000);
rt_pwm_enable(pwm_dev, 1);
```

含义是：

- 选择 `channel = 1`
- 周期 `period = 100000 ns`
- 脉宽 `pulse = 50000 ns`
- 即 100us 周期、50us 高电平，对应 50% duty

### 方式 2：通过 shell 命令调试

`rt_drv_pwm.c` 还导出了几个常见命令：

- `pwm_enable <pwm_dev> <channel/-channel>`
- `pwm_disable <pwm_dev> <channel/-channel>`
- `pwm_set <pwm_dev> <channel> <period> <pulse>`
- `pwm_get <pwm_dev> <channel>`

例如：

```sh
pwm_set rpwm0 1 100000 50000
pwm_enable rpwm0 1
```

这条命令链的意义和上面的 `rt_pwm_set()` / `rt_pwm_enable()` 是一致的。

## `channel` 的含义

在当前 K3 小核心 PWM 驱动里：

- 设备名如 `rpwm0`、`rpwm1` 表示 **PWM 控制器实例**
- `channel` 是 RT-Thread PWM 框架接口里的通道参数

但从当前 `pwm-pxa.c` 实现看，底层寄存器计算和写入并没有根据 `channel` 去切换不同硬件子通道，当前驱动本质上是按“一个设备实例对应一路 PWM 输出”来工作的。

所以在实际使用上：

- `rpwm0` 就是一个 PWM 实例
- `rpwm1` 是另一个 PWM 实例
- 测试代码里统一使用 `channel = 1`

不要把 `channel` 理解成 pinctrl pad 号，也不要把它理解成 SoC 物理引脚号。

## pinctrl 与 PWM 的关系

PWM 能否真正输出到外部 PAD，不只取决于 PWM 控制器节点是否启用，还取决于 pinctrl 是否把目标 PAD 切到了对应的 PWM 复用功能。

### 驱动会自动下发 default pinctrl 状态

在 `pwm_probe()` 中可以看到：

```c
pinctrl_apply_default(compatible_node);
```

这表示：

- 如果 PWM 控制器节点在 DTS 中声明了默认 pinctrl 状态
- 驱动 probe 时会主动应用它
- 所以 PWM PAD 复用通常不需要 consumer 再单独手动切换

这和前面 ESOS pinctrl 文档中讲的 `pinctrl_apply_default()` 模型是一致的。

### `k3-pinctrl.dtsi` 中的 PWM 复用组

在 `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3-pinctrl.dtsi` 中，可以看到大量 `rpwm*_cfg`：

例如：

```dts
rpwm0_0_cfg: rpwm0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_15, (MUX_MODE3 | EDGE_NONE | PULL_UP | PAD_DS8))
    >;
};
```

同一个 PWM 往往有多个可选输出位置，例如：

- `rpwm0_0_cfg`
- `rpwm0_1_cfg`
- `rpwm0_2_cfg`

这表示：

- 同一 PWM 实例可以复用到不同 PAD
- 板级 dts 需要根据实际走线选择其中一组
- 最终只能让板子实际接出的那个 pad 输出 PWM

所以 PWM 文档里必须把 pinctrl 一并讲清楚，否则只看 PWM 驱动本身，是解释不通“为什么有波形却板子上没输出”的。

## 板级 DTS 状态模型

当前 `os0_rcpu/dts/k3_deb1.dts` 和 `os1_rcpu/dts/k3_deb1.dts` 中暂时没有搜到实际打开 `&rpwm0`、`&rpwm1` 之类的板级实例。

这意味着：

- 当前 RT24 平台已经把 PWM 控制器节点、驱动和 pinctrl 组准备好了
- 但具体某个板子是否启用哪个 `rpwmX`，仍取决于板级 DTS 是否显式 `status = "okay"` 并绑定对应 `pinctrl-*`

也就是说，PWM 这套能力当前是“驱动 ready、dtsi ready、pinctrl ready”，但板级启用策略还是按产品需求决定。

## consumer 测试代码说明

`bsp/spacemit/drivers/pwm/pwm-test.c` 是当前最直接的 consumer 示例。

它说明了几件很重要的事：

### 1. 当前系统预期的设备名

测试代码会枚举：

```c
"rpwm0", "rpwm1", ..., "rpwm9",
"pwm0", "pwm1", ..., "pwm9"
```

但对 RT24/K3 小核心当前 DTS + 驱动来说，真正能从本文这条链路注册出来的是 `rpwm0 ~ rpwm9`。

### 2. 当前 period 有明确硬件上限

测试代码里写了：

```c
/* period_max ≈ 266829 ns */
```

这是根据：

- `clk = 245.76MHz`
- `prescale <= 63`
- `pv <= 1023`

换算得到的。

因此如果 period 配太大，驱动侧会因为 `prescale > 63` 返回错误。

### 3. `duty > period` 是非法配置

测试代码专门覆盖了这个异常场景，而驱动中也明确返回 `-RT_EINVAL`。

所以 consumer 使用时必须满足：

```text
0 <= pulse <= period
```

## 关键特性总结

K3 小核心 ESOS PWM 当前可以总结为以下几点：

1. **必须从 DTS `compatible` 反查真实驱动**
2. **当前 RT24 PWM 由 `pwm-pxa.c` 负责，匹配 `spacemit,k1x-rpwm0 ~ spacemit,k1x-rpwm9`**
3. **每个 `rpwmX` 都会被注册成一个独立 RT-Thread PWM 设备**
4. **上层主要通过 `rt_pwm_set()` / `rt_pwm_enable()` / `rt_pwm_disable()` 使用 PWM**
5. **PWM 驱动 probe 时会主动 `pinctrl_apply_default()`，因此 pinctrl 配置是输出成功的关键前提**
6. **同一个 PWM 实例可能有多个可选 PAD，需要在 `k3-pinctrl.dtsi` 和板级 dts 中选定**
7. **当前 period / pulse 是纳秒单位，且必须满足硬件范围约束**

## 调试建议

如果小核心 PWM 配置后行为不符合预期，可以按以下顺序排查：

1. 确认目标 PWM 节点已在板级 dts 中打开，例如：
   - `&rpwm0 { status = "okay"; ... };`
2. 确认目标 PWM 节点配置了正确的 `pinctrl-*` 状态
3. 确认 `pwm_probe()` 已匹配到对应 compatible，并完成设备注册
4. 用 `rt_device_find("rpwm0")` 或 `pwm_dev_enum` 确认设备是否真的存在
5. 确认 `period` 没超过硬件能表达的最大范围
6. 确认 `pulse <= period`
7. 如果看到设备注册成功但外部引脚没波形，优先检查 pinctrl 复用是否切到了正确 PAD
8. 如果时序不符合预期，检查以下换算关系：
   - `clock-rate`
   - `prescale`
   - `pv`
   - `dc`

## 小结

K3 小核心 ESOS PWM 文档不能只写 `rt_pwm_set()` 这几个接口，也不能只看寄存器。真正完整的理解路径应该是：

- 先从 DTS `compatible` 找到 PWM 控制器真实驱动
- 再看 `pwm-pxa.c` 如何把 `period/pulse` 换算成寄存器值
- 再看 `rt_drv_pwm.c` 如何把它包装成 RT-Thread PWM 设备
- 再看 `pwm-test.c` 这样的 consumer/测试代码怎样实际使用
- 最后结合 `k3-pinctrl.dtsi` 理解 PWM 输出为什么依赖 pinctrl 复用状态
