# GPIO

介绍 K3 小核心（ESOS/RTT）平台 GPIO 驱动的实现方式、设备树写法、consumer 使用方式，以及 GPIO 与 pinctrl 的衔接关系。

## 模块介绍

K3 小核心平台上的 GPIO 子系统用于完成通用输入输出控制，包括：

- GPIO 输入读取
- GPIO 输出拉高拉低
- GPIO 中断配置与回调分发
- 设备树 `*-gpios` 属性解析
- GPIO 编号空间到 pinctrl pin range 的映射
- 通过 RT-Thread PIN 设备向上提供统一的 `rt_pin_*()` 接口

## 功能介绍

K3 小核心 GPIO 当前主要提供以下能力：

- 注册一个或多个 `gpio_chip`
- 为每个 GPIO 控制器分配全局 GPIO 编号空间
- 支持 AP GPIO、R GPIO 两类 SoC 内 GPIO 控制器
- 支持 `gpio_request()`、`gpio_direction_input()`、`gpio_direction_output()`、`gpio_get_value()`、`gpio_set_value()`
- 支持设备树 `gpios` 等 consumer 属性解析
- 支持中断触发模式配置：上升沿、下降沿、双边沿
- 支持通过 `rt_pin_attach_irq()` / `rt_pin_irq_enable()` 使用 RT-Thread 风格 GPIO 中断接口
- 支持 `gpio-ranges`，把 GPIO 控制器的编号空间映射到 pinctrl 的 pin number space

## 源码结构介绍

K3 ESOS GPIO 相关代码主要位于：

```text
~/k3-sdk/esos/
|-- bsp/spacemit/drivers/gpio/spacemit-gpio.c
|-- bsp/spacemit/drivers/gpio/gpio-test.c
|-- bsp/spacemit/drivers/rt/rt_interrupt.c
|-- components/drivers/gpio/gpiolib.c
|-- components/drivers/gpio/gpiolib-of.c
|-- components/drivers/include/drivers/gpio/gpio.h
|-- components/drivers/include/drivers/gpio/of_gpio.h
|-- components/drivers/misc/pin.c
`-- bsp/spacemit/platform/rt24/
    |-- k3.dtsi
    |-- os0_rcpu/dts/k3_deb1.dts
    `-- os1_rcpu/dts/k3_deb1.dts
```

其中：

- `bsp/spacemit/drivers/gpio/spacemit-gpio.c`：K3 AP GPIO / R GPIO 主驱动
- `components/drivers/gpio/gpiolib.c`：GPIO core，负责 `gpio_chip` 注册、request、direction、set/get
- `components/drivers/gpio/gpiolib-of.c`：GPIO OF 解析，包括 `of_get_named_gpio_flags()` 与 `gpio-ranges`
- `components/drivers/misc/pin.c`：RT-Thread PIN 设备封装层，向上提供 `rt_pin_*()` API
- `bsp/spacemit/drivers/gpio/gpio-test.c` / `rt_interrupt.c`：GPIO 输入输出与中断的实际 consumer 示例

## 驱动定位方式

先从设备树 `compatible` 反查真实驱动。

在 `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3.dtsi` 中，可以看到 K3 小核心相关 GPIO 节点：

```dts
gpio: gpio@d4019000 {
    compatible = "spacemit,k3-gpio";
    reg = <0xd4019000 0x800>;
    banks = <4>;
    interrupt-parent = <&intc>;
    interrupts = <0 93 0>;
    gpio-controller;
    #gpio-cells = <2>;
    status = "disabled";
};

r_gpio: r_gpio@c0889400 {
    compatible = "spacemit,k3-rgpio";
    reg = <0xc0889400 0x100>;
    clocks = <&ccu CLK_RCPU_GPIO>, <&ccu CLK_RST_RCPU_GPIO>;
    banks = <2>;
    interrupt-parent = <&intc>;
    interrupts = <0 49 0>;
    gpio-controller;
    #gpio-cells = <2>;
    status = "disabled";
};
```

因此可以确认：

- `spacemit,k3-gpio` / `spacemit,k3-rgpio` 对应：
  - `~/k3-sdk/esos/bsp/spacemit/drivers/gpio/spacemit-gpio.c`

这一步和 pinctrl 章节一样，必须先以 DTS `compatible` 定位驱动，而不是按文件名猜。

## GPIO 测试用例与调试使用

这部分更适合放在 GPIO 基本操作、GPIO 中断和 `gpio-ranges` 说明之后，因为测试用例本质上是在验证：

- GPIO 输入输出是否正常
- AP GPIO 与 R GPIO 的全局编号是否使用正确
- 中断触发模式是否符合预期
- RT-Thread `rt_pin_*()` 封装是否能正确走到 GPIO core / 驱动层

K3 ESOS GPIO 相关测试代码位于：

```text
~/k3-sdk/esos/bsp/spacemit/drivers/gpio/gpio-test.c
~/k3-sdk/esos/bsp/spacemit/drivers/rt/rt_interrupt.c
```

其中：

- `gpio-test.c`：功能性测试，覆盖输入、输出、上升沿、下降沿、双边沿
- `rt_interrupt.c`：时延测试，测量 R GPIO 中断响应延迟

### 1. `gpio_test_0001`：GPIO 输入测试

`gpio-test.c` 导出了：

```c
MSH_CMD_EXPORT(gpio_test_0001, BSP_RTTHREAD_GPIO_0001: GPIO Input Test);
```

直接执行：

```sh
gpio_test_0001
```

该测试会分别验证：

- `AP_GPIO_INPUT = 57`
- `R_GPIO_INPUT = 128`

测试步骤是：

1. `rt_pin_mode(pin, PIN_MODE_INPUT)`
2. `rt_pin_read(pin)`
3. 检查返回值是否是合法电平 `0/1`

它适合用来快速确认：

- AP GPIO 和 R GPIO 控制器是否都已正常注册
- 全局 GPIO 编号空间是否使用正确
- `rt_pin_read()` 能否正确读到输入值

### 2. `gpio_test_0002`：GPIO 输出测试

导出命令：

```c
MSH_CMD_EXPORT(gpio_test_0002, BSP_RTTHREAD_GPIO_0002: GPIO Output Test);
```

执行：

```sh
gpio_test_0002
```

该测试会分别对：

- `AP_GPIO_OUTPUT = 58`
- `R_GPIO_OUTPUT = 129`

执行以下动作：

1. 配成输出：`rt_pin_mode(pin, PIN_MODE_OUTPUT)`
2. 先写高：`rt_pin_write(pin, 1)`
3. 再回读检查
4. 再写低：`rt_pin_write(pin, 0)`
5. 再回读检查

这个命令适合用来确认：

- GPIO 输出方向配置是否正常
- `rt_pin_write()` / `rt_pin_read()` 回环行为是否正确
- R GPIO 逻辑编号 `128/129` 是否真的对应 `rgpio0/1`，而不是误用了 PAD 号

### 3. `gpio_test_0003/0004/0005`：GPIO 中断功能测试

`gpio-test.c` 还导出了三类中断测试：

- `gpio_test_0003`：上升沿中断
- `gpio_test_0004`：下降沿中断
- `gpio_test_0005`：双边沿中断

测试代码采用的方法很直接：

- 输入脚作为中断源
- 输出脚作为触发源
- 通过短接把输出脚连到输入脚
- 用 `rt_pin_write()` 人工制造边沿
- 检查回调是否被触发

例如上升沿测试的关键步骤是：

1. `rt_pin_mode(AP_GPIO_INPUT, PIN_MODE_INPUT)`
2. `rt_pin_attach_irq(AP_GPIO_INPUT, PIN_IRQ_MODE_RISING, ...)`
3. `rt_pin_irq_enable(AP_GPIO_INPUT, PIN_IRQ_ENABLE)`
4. 输出脚先拉低再拉高
5. 检查回调是否更新 `last_irq_gpio`

因此使用这些测试前，需要先完成硬件连接：

- AP 测试：把 `GPIO58` 输出连接到 `GPIO57` 输入
- R 测试：把 `GPIO129` 输出连接到 `GPIO128` 输入

如果连接不成立，测试自然会失败，这不是驱动问题。

### 4. `rt_interrupt`：R GPIO 中断时延测试

`rt_interrupt.c` 导出了：

```c
MSH_CMD_EXPORT(rt_interrupt, "rt interrupt delay");
```

执行：

```sh
rt_interrupt
```

它使用：

- `R_GPIO_OUTPUT = 129` 作为触发脚
- `R_GPIO_INPUT = 128` 作为中断输入脚
- `SysTimer_GetLoadValue()` 记录触发到中断回调之间的时间差

循环 `500` 次后，输出：

- 平均时延
- 最小时延
- 最大时延

这个命令适合在你调优：

- GPIO IRQ 响应路径
- RT-Thread 中断封装
- 小核心实时性

时使用。

### 5. 测试用例和正文内容如何对应

这些测试并不是孤立存在的，它们正好对应前文的几个关键点：

- `gpio_test_0001/0002` 对应 GPIO 基本输入输出能力
- `gpio_test_0003/0004/0005` 对应 GPIO 中断配置与回调分发能力
- `rt_interrupt` 对应 RT-Thread PIN/IRQ 封装在小核心上的实际响应表现

所以这里更适合作为文档后半部分的**验证与调试章节**，而不是在驱动定位之后立刻留一个空的“示例使用”。

## K3 GPIO 控制器节点说明

### AP GPIO：`spacemit,k3-gpio`

```dts
gpio: gpio@d4019000 {
    compatible = "spacemit,k3-gpio";
    reg = <0xd4019000 0x800>;
    banks = <4>;
    interrupt-parent = <&intc>;
    interrupts = <0 93 0>;
    gpio-controller;
    #gpio-cells = <2>;
    status = "disabled";
};
```

它表示：

- 主 GPIO 控制器
- 寄存器空间从 `0xd4019000` 开始
- 一共 4 个 bank
- 每个 bank 32 个 GPIO
- 全局 GPIO 编号范围为 `0 ~ 127`

这个范围来自 `spacemit-gpio.c` 里的芯片数据：

```c
static const struct spacemit_gpio_chip_data k3_ap_gpio_chip_data = {
    .regs         = &k3_regs,
    .bank_offsets = {0x0, 0x40, 0x80, 0x100},
    .gpio_base    = 0,
    .total_gpios  = 128,
};
```

### R GPIO：`spacemit,k3-rgpio`

```dts
r_gpio: r_gpio@c0889400 {
    compatible = "spacemit,k3-rgpio";
    reg = <0xc0889400 0x100>;
    clocks = <&ccu CLK_RCPU_GPIO>, <&ccu CLK_RST_RCPU_GPIO>;
    banks = <2>;
    interrupt-parent = <&intc>;
    interrupts = <0 49 0>;
    gpio-controller;
    #gpio-cells = <2>;
    status = "disabled";
};
```

它表示：

- 小核心 R GPIO 控制器
- 一共 2 个 bank
- bank0 有 32 个 GPIO，bank1 有 4 个 GPIO
- 全局 GPIO 编号范围为 `128 ~ 163`

这个范围同样来自 `spacemit-gpio.c`：

```c
static const struct spacemit_gpio_chip_data k3_r_gpio_chip_data = {
    .regs         = &k3_regs,
    .bank_offsets = {0x0, 0x40, 0x0, 0x0},
    .gpio_base    = 128,
    .total_gpios  = 36,
};
```

所以在 ESOS 里，R GPIO 不是重新从 0 编号，而是放在整个全局 GPIO 空间的 `128~163` 段。

这里必须特别说明：**上层通过 `rt_pin_*()` 使用 GPIO 时，传入的必须是逻辑引脚号（全局 GPIO 编号），不能把 SoC 的实际 PAD 编号、物理引脚号或者 bank 内局部 offset 直接传进去。**

原因是 ESOS 的 `pin` 设备是围绕全局 GPIO number space 建立的：

- AP GPIO 使用 `0~127`
- R GPIO 使用 `128~163`

例如：

```c
#define R_GPIO_INPUT    128
#define R_GPIO_OUTPUT   129
```

这里的 `128`、`129` 表示的是 **R GPIO 在全局 GPIO 空间中的逻辑编号**，而不是“实际 pad 128/129”, 实际应是rgpio0和1。

如果误把实际引脚号直接传给 `rt_pin_mode()` / `rt_pin_write()` / `rt_pin_attach_irq()`，就可能与 AP GPIO 的编号空间发生混淆，导致访问到错误的 GPIO 控制器或错误的引脚。

## GPIO 驱动初始化流程

### `spacemit_gpio_init()`

`spacemit-gpio.c` 的入口函数是：

```c
int spacemit_gpio_init(void)
```

它会按 `__compatible[]` 数组依次查找：

```c
{ .compatible = "spacemit,k1x-gpio", .chip_data = &k1_chip_data },
{ .compatible = "spacemit,k3-gpio", .chip_data = &k3_ap_gpio_chip_data },
{ .compatible = "spacemit,k3-rgpio", .chip_data = &k3_r_gpio_chip_data },
```

对每个可用节点执行：

1. 从 DTS 获取寄存器基址
2. 获取并使能时钟 / 复位时钟（R GPIO 会用到）
3. 读取 `banks`
4. 读取 `interrupts`
5. 调用 `gpio_probe_dt()` 初始化 bank 和中断寄存器
6. 填充 `gpio_chip`
7. 调用 `gpiochip_add()` 注册控制器
8. 如果有 IRQ，则安装 `spacemit_gpio_irq_handler()`
9. 所有 chip 注册完成后，再统一注册一个全局 RT-Thread PIN 设备：

```c
rt_device_pin_register("pin", &spacemit_pin_ops, RT_NULL);
```

这说明 ESOS 小核心 GPIO 子系统有两层接口：

- 一层是 Linux 风格的 `gpio_chip` / `gpio_request()` / `gpio_direction_*()`
- 一层是 RT-Thread 风格的 `rt_pin_mode()` / `rt_pin_write()` / `rt_pin_attach_irq()`

### `gpio_probe_dt()`

这个函数负责：

- 初始化 spinlock
- 分配 `spacemit_gpio_bank[]`
- 分配每个 chip 的 `pin_irq_hdr_tab[]`
- 根据 bank offset 建立每个 bank 的寄存器基址
- 关闭所有 GPIO 中断并清 pending 状态

所以对 AP GPIO / R GPIO 而言，控制器初始化是在这里完成的。

## GPIO 基本操作实现

### 输入方向：`spacemit_gpio_direction_input()`

驱动通过：

```c
writel(bit, bank->reg_bank + regs->gcdr);
```

把对应 GPIO 配置为输入。

### 输出方向：`spacemit_gpio_direction_output()`

它会先：

```c
writel(bit, bank->reg_bank + regs->gsdr);
```

设置为输出，再根据 `value` 写：

```c
writel(bit, bank->reg_bank + regs->gpsr);
```

或：

```c
writel(bit, bank->reg_bank + regs->gpcr);
```

因此输出方向配置和输出值设置是分开的。

### 输入读取：`spacemit_gpio_get()`

通过读取：

```c
gplr = readl(bank->reg_bank + regs->gplr);
```

再判断目标 bit 是否置位。

### 输出写值：`spacemit_gpio_set()`

写值前会先读 `gpdr`，确认该引脚当前确实已配置成输出：

```c
gpdr = readl(bank->reg_bank + regs->gpdr);
```

如果不是输出，驱动不会直接写 `gpsr/gpcr`。

## 设备树 GPIO consumer 模型

### `#gpio-cells = <2>` 的含义

K3 小核心这几类 GPIO 控制器都使用：

```dts
#gpio-cells = <2>;
```

对应 `spacemit_gpio_of_xlate()` 的解析方式是：

- `args[0]`：GPIO offset
- `args[1]`：GPIO flags

例如如果某个 consumer 写：

```dts
gpios = <&r_gpio 0 0>;
```

含义就是：

- 使用 `r_gpio` 控制器
- offset 为 `0`
- flags 为 `0`

最终返回的是该控制器对应的 **全局 GPIO 编号**，而不是局部 offset。

### `of_get_named_gpio_flags()`

在 `components/drivers/gpio/gpiolib-of.c` 中，consumer 侧解析 `*-gpios` 属性的核心接口是：

```c
of_get_named_gpio_flags(np, propname, index, flags)
```

它会：

1. 通过 `dtb_node_parse_phandle_with_args()` 解析某个 `*-gpios` 属性
2. 找到目标 `gpio_chip`
3. 调用对应 chip 的 `of_xlate()`
4. 返回全局 GPIO 编号

也就是说，consumer 驱动实际拿到的是 **全局 GPIO number**，不是某个 bank 内部局部编号。

## consumer 的实际使用方式

### 方式 1：通过设备树 `*-gpios` 获取 GPIO

这是一种常见方式。

consumer 驱动通常会这样做：

```c
gpio = of_get_named_gpio_flags(node, "gpios", 0, NULL);
if (gpio >= 0)
    gpio_request(gpio, NULL);
```

这条链路的关键点是：

- DTS 里写的是 phandle + offset + flags
- `of_get_named_gpio_flags()` 返回的是全局 GPIO 编号
- 后续 `gpio_request()` / `gpio_direction_*()` / `gpio_set_value()` 都是围绕全局 GPIO 编号工作

### 方式 2：直接使用 RT-Thread PIN 接口

ESOS 里还有一种非常常见的使用方式：通过全局 `pin` 设备，直接用 RT-Thread API 操作 GPIO。

这里必须再次强调：`rt_pin_*()` 接口传入的必须是 **逻辑引脚号**，也就是 GPIO 子系统分配的 **全局 GPIO 编号**，而不是 SoC 的实际 PAD 编号、封装球脚号，或者某个 bank 内的局部 offset。

原因是 ESOS 中的 `pin` 设备并不是直接按“物理引脚号”寻址，而是按整个 GPIO 框架统一维护的 global GPIO number space 寻址：

- AP GPIO：`0 ~ 127`
- R GPIO：`128 ~ 163`

如果把实际引脚号误传给 `rt_pin_mode()` / `rt_pin_write()` / `rt_pin_attach_irq()`：

- 就可能被错误解释成 AP GPIO 的编号
- 或者落到错误的 gpio_chip 上
- 从而与 AP GPIO 的引脚空间发生冲突

例如在 `bsp/spacemit/drivers/rt/rt_interrupt.c` 中：

```c
rt_pin_mode(R_GPIO_INPUT, PIN_MODE_INPUT);
rt_pin_attach_irq(R_GPIO_INPUT, PIN_IRQ_MODE_RISING,
        rt_irq_callback, (void *)(rt_ubase_t)R_GPIO_INPUT);
rt_pin_irq_enable(R_GPIO_INPUT, PIN_IRQ_ENABLE);
```

以及：

```c
rt_pin_mode(R_GPIO_OUTPUT, PIN_MODE_OUTPUT);
rt_pin_write(R_GPIO_OUTPUT, 0);
```

在小核心 ESOS 中，GPIO consumer 不一定总是先走 `of_get_named_gpio_flags()`，也可能直接使用平台预定义好的全局 GPIO 逻辑编号，通过 `rt_pin_*()` 完成输入输出和中断操作。这完全取决于consumer如何使用

### `rt_pin_*()` 最终如何落到 GPIO 驱动

在 `spacemit-gpio.c` 中，驱动注册了：

```c
static const struct rt_pin_ops spacemit_pin_ops = {
    .pin_mode = spacemit_pin_mode,
    .pin_write = spacemit_pin_write,
    .pin_read = spacemit_pin_read,
    .pin_attach_irq = spacemit_pin_attach_irq,
    .pin_detach_irq = spacemit_pin_detach_irq,
    .pin_irq_enable = spacemit_pin_irq_enable_ops,
};
```

并通过：

```c
rt_device_pin_register("pin", &spacemit_pin_ops, RT_NULL);
```

注册全局 pin 设备。

因此：

- `rt_pin_mode()` 最终会调用 `gpio_direction_input()` / `gpio_direction_output()`
- `rt_pin_write()` 最终会调用 `gpio_set_value()`
- `rt_pin_read()` 最终会调用 `gpio_get_value()`
- `rt_pin_attach_irq()` / `rt_pin_irq_enable()` 最终会落到 GPIO 控制器中断配置逻辑

## GPIO 中断实现

### 中断回调分发模型

GPIO 中断入口在：

```c
static void spacemit_gpio_irq_handler(int irq, void *param)
```

它的处理流程是：

1. 遍历该 chip 的每个 bank
2. 读取 `gedr` 查看哪些 bit 产生了 edge detect
3. 写回 `gedr` 清 pending
4. 与 `irq_mask` 求交集，筛出真正已使能的中断源
5. 遍历 pending bit
6. 调用 `pin_irq_hdr_tab[gpio_offset].hdr(args)` 执行上层注册回调

因此这里不是每个 GPIO 都独立挂一个中断，而是：

- 每个 GPIO 控制器通常有一个父中断
- 控制器内部再根据 `gedr` 分发到具体 pin

### 触发模式配置

在 `spacemit_gpio_irq_mode()` 中，驱动支持：

- `PIN_IRQ_MODE_RISING`
- `PIN_IRQ_MODE_FALLING`
- `PIN_IRQ_MODE_RISING_FALLING`

其本质是通过：

- `gsrer` / `gcrer` 控制 rising edge detect
- `gsfer` / `gcfer` 控制 falling edge detect

### 使能与屏蔽

在 `spacemit_gpio_irq_enable()` 中：

- 使能时会更新 `irq_mask`
- 打开对应 edge detect
- 并设置 `gcpmask` / `gapmask`
- 同时清掉当前 pending 状态

禁用时则反向关闭。

### consumer 实际使用示例

GPIO 中断的实际 consumer 用法，在 `gpio-test.c` 和 `rt_interrupt.c` 中都能直接看到：

```c
rt_pin_attach_irq(R_GPIO_INPUT, PIN_IRQ_MODE_RISING,
                  r_gpio_irq_callback, (void *)(rt_ubase_t)R_GPIO_INPUT);
rt_pin_irq_enable(R_GPIO_INPUT, PIN_IRQ_ENABLE);
```

所以当前 ESOS 小核心 GPIO 中断的典型用法是：

- `rt_pin_mode(pin, PIN_MODE_INPUT)`
- `rt_pin_attach_irq(pin, mode, callback, args)`
- `rt_pin_irq_enable(pin, PIN_IRQ_ENABLE)`

## GPIO 与 pinctrl 的关系

### `gpio-ranges` 的作用

在 ESOS 里，GPIO 和 pinctrl 之间通过 `gpio-ranges` 建立映射关系。

在 `components/drivers/gpio/gpiolib-of.c` 中：

```c
ret = dtb_node_parse_phandle_with_args(np, "gpio-ranges",
        "#gpio-range-cells", index, &pinspec);
```

解析每条 `gpio-ranges` 描述后，会调用：

```c
gpiochip_add_pin_range(chip,
                       pctldev->dev->name,
                       pinspec.args[0],
                       pinspec.args[1],
                       pinspec.args[2]);
```

而在 `gpiolib.c` 中，`gpiochip_add_pin_range()` 会建立：

- `range.base = chip->base + gpio_offset`
- `range.pin_base = pin_offset`
- `range.npins = npins`

因此，`gpio-ranges` 本质上是在说明：

- 某一段 GPIO 编号空间
- 对应 pinctrl 里的哪一段 pin number space

### 这一映射为什么重要

GPIO 子系统本身只知道“全局 GPIO 编号”。

但在 SoC 上，一个引脚要真正作为 GPIO 使用，往往还需要 pinctrl 子系统把这个 pin 切回 GPIO 功能。这个时候 GPIO 框架就需要知道：

- 这个 GPIO number 对应 pinctrl 里的哪个 pin

而 `gpio-ranges` 正是建立这层映射的关键。

### `gpiochip_add_pin_range()` 建立的关系

`gpiolib.c` 中这一段最关键：

```c
pin_range->range.base = chip->base + gpio_offset;
pin_range->range.pin_base = pin_offset;
pin_range->range.npins = npins;
pin_range->pctldev = pinctrl_find_and_add_gpio_range(pinctl_name, &pin_range->range);
```

可以这样理解：

- 从 GPIO 全局编号 `chip->base + gpio_offset` 开始
- 连续 `npins` 个 GPIO
- 对应到 pinctrl 从 `pin_offset` 开始的 `npins` 个 pin

这样后续 pinctrl 才能知道 GPIO 请求落到哪个 PAD。

### 与 pinctrl 章节的衔接

1. GPIO 控制器在 DTS 中写 `gpio-ranges`
2. `of_gpiochip_add()` 解析 `gpio-ranges`
3. `gpiochip_add_pin_range()` 把 GPIO range 注册到 pinctrl
4. 后续 GPIO 与 pinctrl 就建立了编号映射关系

注意：K3 当前 RT24 小核心 DTS 中暂时没有实际 `gpio-ranges = <...>;` 节点实例，但从 `gpiolib-of.c` 和 `gpiolib.c` 的实现来看，这套机制已经完整支持，因此 GPIO 文档里应该把它讲清楚。

## 板级 DTS 实例

### OS0 板级文件

`~/k3-sdk/esos/bsp/spacemit/platform/rt24/os0_rcpu/dts/k3_deb1.dts` 中可见：

```dts
&gpio {
    status = "okay";
};

&r_gpio {
    status = "okay";
};
```

说明 OS0 板级会打开：

- AP GPIO
- R GPIO

### OS1 板级文件

`~/k3-sdk/esos/bsp/spacemit/platform/rt24/os1_rcpu/dts/k3_deb1.dts` 中可见：

```dts
&gpio {
    status = "disabled";
};

&r_gpio {
    status = "disabled";
};
```

说明不同 OS 实例下，GPIO 控制器是否启用由板级 dts 决定。

## 关键特性总结

K3 小核心 ESOS GPIO 当前可以总结为以下几点：

2. **AP GPIO 和 R GPIO 由 `spacemit-gpio.c` 统一支持**
3. **上层既支持 `gpio_request()` 风格接口，也支持 `rt_pin_*()` 风格接口**
4. **`rt_pin_*()` 使用的必须是全局 GPIO 逻辑编号，不是实际 PAD 编号**
5. **GPIO 中断采用“控制器父中断 + bank/pin 分发”模型**
6. **`#gpio-cells = <2>`，consumer 侧通过 `of_get_named_gpio_flags()` 解析 `*-gpios` 属性**
7. **GPIO 与 pinctrl 通过 `gpio-ranges` / pin range 机制衔接**

## 调试建议

如果小核心 GPIO 配置后行为不符合预期，可以按以下顺序排查：

1. 确认板级 dts 中目标控制器已打开：
   - `&gpio { status = "okay"; };`
   - `&r_gpio { status = "okay"; };`
2. 确认 consumer 使用的是哪个 GPIO 控制器：AP GPIO 还是 R GPIO
3. 确认 `#gpio-cells = <2>` 对应的 phandle 参数顺序是否正确
4. 如果使用 `*-gpios` 属性，确认 consumer 驱动里确实调用了 `of_get_named_gpio_flags()`
5. 如果使用 RT-Thread PIN 接口，确认传入的是全局 GPIO 逻辑编号，而不是实际 PAD 编号或 bank 内局部编号
6. 如果是输入上拉/下拉、开漏等电气属性问题，注意这些通常应在 pinctrl 中配置，而不是 GPIO 驱动里配置
7. 如果中断不触发，检查：
   - GPIO 是否配置为输入
   - `rt_pin_attach_irq()` 的 mode 是否正确
   - 是否调用了 `rt_pin_irq_enable()`
   - 板级中断号与父中断是否打开
8. 如果 GPIO 复用失败，要继续检查 pinctrl 是否已经把该 PAD 切到了 GPIO 功能
9. 如果涉及 GPIO 与 pinctrl 映射，还要检查 `gpio-ranges` 是否正确描述 GPIO number space 到 pin number space 的对应关系

## 小结

K3 小核心 ESOS GPIO 文档不能只写几个 `rt_pin_*()` API，也不能只盯着寄存器。真正完整的理解路径应该是：

- 先从 DTS `compatible` 找到 GPIO 控制器真实驱动
- 再看 `spacemit-gpio.c` 如何实现方向、读写、中断
- 再看 `gpiolib.c` / `gpiolib-of.c` 如何完成 `gpio_chip` 注册、OF 解析和 pin range 映射
- 再看 consumer 是走 `of_get_named_gpio_flags()` 还是 `rt_pin_*()`
- 最后结合 `gpio-ranges` 理解 GPIO 和 pinctrl 是如何衔接起来的
