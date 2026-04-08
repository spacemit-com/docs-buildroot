# GPIO

介绍 **K3 GPIO** 的功能和使用方法。

## 模块介绍

GPIO 是 **管理 GPIO 模块的控制器**，用于完成 GPIO 方向配置、输入输出电平读写以及 GPIO 中断处理。

### 功能介绍

Linux GPIO 子系统驱动框架主要有三部分组成：

- **GPIO 控制器驱动**：与 GPIO 控制器硬件交互，完成寄存器初始化和具体操作
- **GPIO lib 驱动**：向内核其它模块提供标准 GPIO API，如方向设置、读写电平、GPIO 转 IRQ 等
- **GPIO 字符设备驱动**：将 GPIO 以字符设备方式导出到用户空间，用户空间可通过标准接口访问 GPIO

### 源码结构介绍

K3 GPIO 驱动代码位于内核目录 `drivers/gpio` 下：

```text
drivers/gpio/
|-- gpio-spacemit-k1.c
```

K3 GPIO 对应的 dt-binding 文件为：

```text
Documentation/devicetree/bindings/gpio/spacemit,k1-gpio.yaml
```

虽然文件名是 `k1-gpio.yaml`，但由于 K1/K3 使用的是同一套 GPIO IP，因此该 binding 同样是 K3 GPIO 的 binding 参考。

K3 GPIO 控制器节点位于：

```text
arch/riscv/boot/dts/spacemit/k3.dtsi
```

## 关键特性

| 特性 | 特性说明 |
| :----- | :---- |
| 支持方向设置 | 支持将 GPIO 设置为输入或输出 |
| 支持输入电平读取 | 支持读取 GPIO 当前输入/输出电平 |
| 支持输出高低电平设置 | 支持在输出模式下置高/置低 |
| 支持 GPIO 中断 | 支持上升沿、下降沿和双边沿触发 |
| 支持 gpio-ranges | 支持与 pinctrl 建立 GPIO 到 pin 的映射关系 |
| 支持分 bank 管理 | 4 个 bank，每个 bank 32 个 GPIO，共 128 个 GPIO |
| 支持 gpio irqchip | 通过 gpiolib irqchip 提供 GPIO 中断域和 nested irq 处理 |

## 配置介绍

主要包括驱动 **CONFIG 使能配置** 和 **dts 配置**。

### CONFIG 配置

- **CONFIG_GPIOLIB**：为 GPIO 控制器提供通用支持，默认值通常为 `Y`

```text
Device Drivers
        GPIO Support (GPIOLIB [=y])
```

- **CONFIG_GPIO_SPACEMIT_K1**：为 SpacemiT K1/K3 GPIO 控制器提供支持

```text
config GPIO_SPACEMIT_K1
    tristate "SPACEMIT K1/K3 GPIO support"
    depends on ARCH_SPACEMIT || COMPILE_TEST
    depends on OF_GPIO
    select GPIO_GENERIC
    select GPIOLIB_IRQCHIP
```

Makefile 对应关系：

```text
obj-$(CONFIG_GPIO_SPACEMIT_K1) += gpio-spacemit-k1.o
```

## 使用介绍

K3 GPIO 的使用可以分为三部分：

1. **GPIO 控制器描述**
2. **GPIO 对应 pin 的 pinctrl 配置**
3. **设备节点中引用 GPIO**

### GPIO 控制器描述

K3 GPIO 控制器节点位于 `k3.dtsi`，定义如下：

```dts
gpio: gpio@d4019000 {
    compatible = "spacemit,k3-gpio";
    reg = <0x0 0xd4019000 0x0 0x100>;
    clocks = <&syscon_apbc CLK_APBC_GPIO>,
             <&syscon_apbc CLK_APBC_GPIO_BUS>;
    clock-names = "core", "bus";
    gpio-controller;
    #gpio-cells = <3>;
    interrupts = <58 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    interrupt-controller;
    #interrupt-cells = <3>;
    syscon-gpio-regs = <&syscon_gpio>;
    syscon-gpio-edge = <&syscon_gpio_edge>;
    gpio-ranges = <&pinctrl 0 0 0 32>,
                  <&pinctrl 1 0 32 32>,
                  <&pinctrl 2 0 64 32>,
                  <&pinctrl 3 0 96 32>;
};
```

该节点说明：

- `compatible = "spacemit,k3-gpio"`：匹配 K3 GPIO 驱动
- `#gpio-cells = <3>`：GPIO 引用采用 3 个 cell
- `#interrupt-cells = <3>`：GPIO 中断引用采用 3 个 cell
- `gpio-ranges`：建立 GPIO bank/offset 与 pinctrl pin 的映射关系, 一般保持默认
- `syscon-gpio-regs`：GPIO 控制寄存器 syscon
- `syscon-gpio-edge`：GPIO edge 寄存器 syscon

### GPIO 编号说明

驱动中定义：

- `SPACEMIT_NR_BANKS = 4`
- `SPACEMIT_NR_GPIOS_PER_BANK = 32`

因此 K3 GPIO 总数为：

- **4 个 bank**
- **每个 bank 32 个 GPIO**
- **总计 128 个 GPIO**

bank 与 GPIO 全局编号关系如下：

| Bank | GPIO 范围 |
| :-- | :-- |
| bank0 | 0 ~ 31 |
| bank1 | 32 ~ 63 |
| bank2 | 64 ~ 95 |
| bank3 | 96 ~ 127 |

驱动内部为每个 bank 单独注册一个 `gpio_chip`，并设置：

- `gc->base = index * 32`
- `gc->ngpio = 32`

### GPIO pin 配置

GPIO 控制器本身只负责方向、电平和中断；**具体某个 pin 是否工作在 GPIO 模式**，以及其上下拉、驱动能力、电压等属性，仍然要由 **pinctrl** 来配置。

也就是说，K3 GPIO 的 pin 配置方式应参考 K3 pinctrl 文档：

- `02-GPIO.md` 负责 GPIO 控制器本身
- pin mux / bias / drive-strength / power-source 等由 `01-PINCTRL.md` 负责

在 K3 中，GPIO pin 一般通过 pinctrl state 中的 `K3_PADCONF(pin, func)` 选择为 GPIO 功能。通常 GPIO 功能对应 mux mode 为该 pin 的 GPIO 默认功能号。

例如某些 pin 在 `k3-pinctrl.dtsi` 中默认复用给外设；如果板级设计要改为 GPIO 使用，则应在板级 dts 中新增或重写对应 pinctrl 配置，将这些 pin 切换到 GPIO 功能。

### GPIO 使用

#### 作为普通 GPIO 输出/输入使用

在设备节点中，K3 GPIO 通常这样引用：

```dts
reset-gpios = <&gpio 3 4 GPIO_ACTIVE_LOW>;
```

其三个 cell 的含义为：

1. **bank 编号**
2. **bank 内 offset**
3. **GPIO flag**，如 `GPIO_ACTIVE_HIGH` / `GPIO_ACTIVE_LOW`

例如：

- `bank = 3`
- `offset = 4`
- 实际对应全局 GPIO 编号 = `3 * 32 + 4 = 100`

K3 板级 dts 中的实际示例如下。

1. 摄像头相关 GPIO：

```dts
pwdn-gpios = <&gpio 0 7 GPIO_ACTIVE_HIGH>;
reset-gpios = <&gpio 0 6 GPIO_ACTIVE_HIGH>;
dptc-gpios = <&gpio 0 8 GPIO_ACTIVE_HIGH>;
```

2. 网卡 PHY reset：

```dts
snps,reset-gpios = <&gpio 0 15 GPIO_ACTIVE_LOW>;
```

3. 蓝牙控制 GPIO：

```dts
device-wake-gpios = <&gpio 2 30 GPIO_ACTIVE_HIGH>;
enable-gpios = <&gpio 2 29 GPIO_ACTIVE_HIGH>;
host-wake-gpios = <&gpio 2 28 GPIO_ACTIVE_HIGH>;
```

4. SD 卡检测 GPIO：

```dts
cd-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
```

#### 作为 GPIO 中断使用

设备先通过 `*-gpios` 获取 GPIO，再由驱动调用 `gpiod_to_irq()` / `gpio_to_irq()` 转成中断

例如 SDHCI 控制器在 `k3.dtsi` 中：

```dts
sdcard: mmc@d4280000 {
    compatible = "spacemit,k3-sdhci";
    interrupt-parent = <&saplic>;
    interrupts = <99 IRQ_TYPE_LEVEL_HIGH>;
    ...
};
```

- 但设备还需要额外使用一根 GPIO 作为功能辅助中断信号，例如 **card detect (cd-gpio)**

```dts
cd-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
```

然后在驱动里：

1. 先通过 `mmc_of_parse()` 解析 `cd-gpios`
2. GPIO 框架获得对应的 GPIO descriptor
3. 再通过 `gpiod_to_irq()`（或旧接口 `gpio_to_irq()`）把该 GPIO 转换成 IRQ
4. 最终把这根 GPIO IRQ 当作插卡检测中断来使用

K3 SDHCI 驱动 `drivers/mmc/host/sdhci-of-k1.c` 中，在 `spacemit_sdhci_probe()` 里可以看到：

```c
ret = mmc_of_parse(host->mmc);
```

这说明像 `cd-gpios` 这一类标准 MMC GPIO 属性，是由 MMC 框架解析并进一步转换为 GPIO IRQ 使用的。

## dt-binding 说明

K3 GPIO 当前使用的 binding 文件为：

```text
Documentation/devicetree/bindings/gpio/spacemit,k1-gpio.yaml
```

虽然文件名带 `k1`，但对 K3 同样适用，因为 K1/K3 共享同一套 GPIO IP，当前 Linux 也共用同一个驱动。

binding 中的关键信息如下：

### compatible

binding 文件中给出的 compatible 是：

```yaml
compatible:
  const: spacemit,k1-gpio
```

但 K3 实际 dts 中使用的是：

```dts
compatible = "spacemit,k3-gpio";
```

因此在 K3 上，应以 **驱动代码 + K3 dts 实际写法** 为准。

### #gpio-cells

binding 说明：

```yaml
"#gpio-cells":
  const: 3
```

含义为：

- 第 1 个 cell：GPIO bank index
- 第 2 个 cell：bank 内 offset
- 第 3 个 cell：GPIO flags

### #interrupt-cells

binding 说明：

```yaml
"#interrupt-cells":
  const: 3
```

含义为：

- 第 1 个 cell：GPIO bank index
- 第 2 个 cell：bank 内 offset
- 第 3 个 cell：interrupt flags

并且 binding 明确说明：

- **不支持电平中断**
- 不应使用 `IRQ_TYPE_LEVEL_HIGH` / `IRQ_TYPE_LEVEL_LOW`
- 应使用边沿触发类型

这一点与驱动实现也是一致的：驱动只实现了 rising/falling edge 相关寄存器配置。

## 驱动实现说明

### K1/K3 共用新 GPIO 驱动

当前驱动的 `of_match_table` 如下：

```c
static const struct of_device_id spacemit_gpio_dt_ids[] = {
    { .compatible = "spacemit,k1-gpio", .data = &k1_data },
    { .compatible = "spacemit,k3-gpio", .data = &k3_data },
    { /* sentinel */ }
};
```

说明：

- K1 匹配 `k1_data`
- K3 匹配 `k3_data`
- 核心逻辑共用，差别主要在寄存器偏移与 bank 布局

### K3 寄存器布局

驱动中 K3 GPIO 的寄存器偏移定义如下：

```c
static const struct spacemit_gpio_reg_offsets k3_regs = {
    .gplr    = 0x0,
    .gpdr    = 0x4,
    .gpsr    = 0x8,
    .gpcr    = 0xc,
    .grer    = 0x10,
    .gfer    = 0x14,
    .gedr    = 0x18,
    .gsdr    = 0x1c,
    .gcdr    = 0x20,
    .gsrer   = 0x24,
    .gcrer   = 0x28,
    .gsfer   = 0x2c,
    .gcfer   = 0x30,
    .gapmask = 0x34,
    .gcpmask = 0x38,
};
```

K3 bank 偏移为：

```c
static const u32 k3_bank_offsets[] = { 0x0, 0x40, 0x80, 0x100 };
```

因此每个 bank 的寄存器块是分散排布的，bank0/bank1/bank2/bank3 分别从 `0x0`、`0x40`、`0x80`、`0x100` 开始。

### GPIO 基本操作实现

驱动中 GPIO 的基本操作如下：

- `spacemit_gpio_get()`：读取 `GPLR` 获取 GPIO 电平
- `spacemit_gpio_set()`：写 `GPSR` / `GPCR` 置高或置低
- `spacemit_gpio_direction_input()`：写 `GCDR` 设置输入方向
- `spacemit_gpio_direction_output()`：写 `GSDR` 设置输出方向，再设置输出值

这些实现对应 Linux GPIO 标准语义。

### GPIO 中断实现

K3 GPIO 使用 `gpiolib irqchip` 模式实现 GPIO 中断。

驱动中的关键点：

1. 每个 bank 都配置一个 `gpio_irq_chip`
2. 总控制器中断通过 `devm_request_threaded_irq()` 注册
3. 中断处理函数 `spacemit_gpio_irq_handler()`：
   - 读取 `GEDR` 获取 pending 位
   - W1C 清除 pending
   - 与 `irq_mask` 相与，得到有效中断
   - 对每一位调用 `handle_nested_irq()`
4. `irq_set_type` 只配置：
   - 上升沿
   - 下降沿
   - 双边沿

驱动并没有实现电平触发，因此 GPIO 中断只应用于 edge trigger 场景。

### 中断 mask / unmask 行为

驱动对每个 bank 都维护：

- `irq_mask`
- `irq_rising_edge`
- `irq_falling_edge`

中断屏蔽/使能时：

- `irq_mask()`：更新 `gapmask`，并清除相应 edge enable
- `irq_unmask()`：恢复 edge enable，再更新 `gapmask`

这种做法与硬件 edge 检测寄存器相对应。

## K3 与 K1 差异说明

虽然 K1 和 K3 GPIO IP 相同，并共用一个新驱动，但两者仍有以下差异需要分清：

1. **当前内核中的 GPIO 驱动是新驱动**
   - 文件：`drivers/gpio/gpio-spacemit-k1.c`
   - 同时支持 K1/K3
   - 应以此驱动为准

2. **不能照搬 K1 旧 GPIO 文档中的历史实现描述**
   - K1 旧文档中提到的旧驱动和旧用法，只能作为历史参考
   - K3 文档已经围绕当前标准化驱动来写

3. **K1/K3 的差异主要体现在寄存器布局**
   - K1 和 K3 的 `reg_offsets` 不同
   - K1 和 K3 的 `bank_offsets` 不同
   - 驱动通过 `k1_data` / `k3_data` 区分

4. **K3 dts compatible 为 `spacemit,k3-gpio`**
   - binding 文件名虽然还是 `spacemit,k1-gpio.yaml`
   - 但 K3 实际写法已切换到 `spacemit,k3-gpio`

## 接口介绍

### 内核 API 介绍

GPIO 子系统常用接口如下：

- **申请指定 GPIO（旧接口）**

```c
int gpio_request(unsigned gpio, const char *label);
```

- **释放指定 GPIO（旧接口）**

```c
void gpio_free(unsigned gpio);
```

- **设置 GPIO 为输入模式**

```c
int gpio_direction_input(unsigned gpio);
```

- **设置 GPIO 为输出模式并赋初值**

```c
int gpio_direction_output(unsigned gpio, int value);
```

- **设置 GPIO 输出值**

```c
void gpio_set_value(unsigned gpio, int value);
```

- **获取 GPIO 电平值**

```c
int gpio_get_value(unsigned gpio);
```

- **将 GPIO 转为 IRQ**

```c
int gpio_to_irq(unsigned gpio);
```

### 推荐的 descriptor 接口

在新驱动和新设备驱动中，更推荐使用 descriptor 风格接口，例如：

```c
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id,
                            enum gpiod_flags flags);
```

以及：

```c
int gpiod_direction_output(struct gpio_desc *desc, int value);
int gpiod_direction_input(struct gpio_desc *desc);
int gpiod_get_value(struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);
```

在 dts 中对应 `reset-gpios`、`enable-gpios`、`cd-gpios` 等属性。

## Debug 介绍

### 字符设备 / 用户空间接口

现代 Linux GPIO 更推荐通过字符设备接口使用。系统中通常可见：

```text
/dev/gpiochipX
```

用户空间可配合 `libgpiod` 工具进行查看和操作。

### sysfs

传统 sysfs GPIO 接口路径如下：

```text
/sys/class/gpio
```

常见节点：

- `export`
- `unexport`
- `gpiochipX`
- `gpioX/direction`
- `gpioX/value`

不过 sysfs GPIO 已属于旧接口，调试时建议优先使用字符设备方式。

### debugfs

查看系统中 GPIO 控制器及其 GPIO 使用信息：

```text
/sys/kernel/debug/gpio
```

可用于查看：

- gpiochip 编号
- GPIO 使用者
- 方向
- 当前高低电平
- ACTIVE LOW 等标记

### 寄存器调试

也可以通过 `devmem` 查看 GPIO 控制器寄存器值，辅助判断方向、电平和中断配置是否生效：

```bash
devmem reg_addr
```

重点可查看的寄存器包括：

- `GPLR`：电平寄存器
- `GPDR`：方向寄存器
- `GPSR` / `GPCR`：置位/清位寄存器
- `GRER` / `GFER`：上升沿/下降沿检测寄存器
- `GEDR`：中断状态寄存器
- `GAPMASK`：中断 mask 寄存器

## 测试介绍

### 普通 GPIO 测试

1. 确认对应 pin 已在 pinctrl 中切换为 GPIO 功能
2. 在驱动或用户空间中将 GPIO 配置为输入/输出
3. 读取或设置 GPIO 电平
4. 结合寄存器与 debugfs 查看是否生效

### GPIO 中断测试

1. 在 dts 中将设备中断源指定为 `interrupt-parent = <&gpio>`
2. `interrupts = <bank offset IRQ_TYPE_EDGE_*>`
3. 触发对应 GPIO 电平边沿变化
4. 检查设备中断处理是否进入
5. 如有必要，检查 `GEDR` / `GRER` / `GFER` / `GAPMASK` 寄存器

## FAQ

### 1. 为什么 K3 GPIO 文档不能直接照抄 K1 旧 GPIO 文档？

因为当前内核中的 GPIO 驱动已经重构为新的标准化驱动：`gpio-spacemit-k1.c`，同时支持 K1/K3。K1 旧文档描述的是历史实现，不应直接套用到 K3。

### 2. 为什么 dt-binding 文件名是 `spacemit,k1-gpio.yaml`，但 K3 dts 使用的是 `spacemit,k3-gpio`？

因为 K1/K3 共享同一 GPIO IP，而 binding 文件未单独拆出 K3 文件。对 K3 而言，应以 **当前驱动代码和 K3 dts 实际写法** 为准。

### 3. K3 GPIO 中断支持 level trigger 吗？

不支持。binding 和驱动实现都表明 GPIO 中断只支持边沿触发，不应使用 `IRQ_TYPE_LEVEL_HIGH` 或 `IRQ_TYPE_LEVEL_LOW`。

### 4. 为什么配置 GPIO 后不生效？

最常见原因不是 GPIO 控制器本身，而是对应 pin 还没有被 pinctrl 切换到 GPIO 功能。需要先检查 pinctrl 配置是否正确。
