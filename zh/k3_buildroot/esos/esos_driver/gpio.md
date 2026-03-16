# GPIO

介绍 K3 小核心（ESOS/RTT）平台 GPIO 模块的定位、软件层次、设备树写法以及应用侧使用方式。

## 模块介绍

GPIO（General Purpose Input/Output）用于提供通用数字输入输出能力。在 K3 小核心 ESOS 中，GPIO 模块主要承担以下职责：

- 提供 GPIO 输入读取与输出控制能力
- 提供 GPIO 中断配置与回调分发能力
- 解析设备树中的 `*-gpios` 属性
- 通过 RT-Thread PIN 设备向上提供统一的 `rt_pin_*()` 接口
- 与 pinctrl 子系统配合，完成 GPIO 编号空间与 PAD/pin 空间的映射

当前平台上主要有两类 GPIO 控制器：

- AP GPIO：主 GPIO 控制器
- R GPIO：小核心侧 GPIO 控制器

## 软件结构

从应用到硬件的大致调用关系如下：

```text
应用/组件驱动
├─ RT-Thread PIN 接口
  ├─ rt_pin_mode()
  ├─ rt_pin_read()
  ├─ rt_pin_write()
  ├─ rt_pin_attach_irq()
  └─ rt_pin_irq_enable()

                │
                ▼
GPIO core / OF 解析层
├─ components/drivers/gpio/gpiolib.c
└─ components/drivers/gpio/gpiolib-of.c

                │
                ▼
SoC GPIO 驱动层
└─ bsp/spacemit/drivers/gpio/spacemit-gpio.c

                │
                ▼
GPIO 控制器硬件
├─ AP GPIO
└─ R GPIO

                │
                ▼
pinctrl / PAD 配置
└─ 通过 gpio-ranges 等机制与 pinctrl 建立映射
```

## 接口介绍

在使用 RT-Thread PIN 接口时，`pin` 参数传入的是逻辑引脚号，即全局 GPIO 编号。

- AP GPIO 编号范围：`0 ~ 127`
- R GPIO 编号范围：`128 ~ 163`

例如：

- `pin = 57` 表示 AP GPIO 57
- `pin = 128` 表示 R GPIO 0
- `pin = 129` 表示 R GPIO 1


**设置引脚模式**

```c
void rt_pin_mode(rt_base_t pin, rt_base_t mode);
```

参数说明：

- `pin`：逻辑引脚号，即全局 GPIO 编号
- `mode`：引脚模式，支持以下取值：
  - `PIN_MODE_OUTPUT`：输出模式
  - `PIN_MODE_INPUT`：输入模式

返回值说明：

- 无返回值

**写引脚输出值**

```c
void rt_pin_write(rt_base_t pin, rt_base_t value);
```

参数说明：

- `pin`：逻辑引脚号，即全局 GPIO 编号
- `value`：输出电平
  - `PIN_LOW`：低电平
  - `PIN_HIGH`：高电平

返回值说明：

- 无返回值

**读取引脚电平值**

```c
int rt_pin_read(rt_base_t pin);
```

参数说明：

- `pin`：逻辑引脚号，即全局 GPIO 编号

返回值说明：

- `0`：低电平
- `1`：高电平
- 负值：读取失败

**绑定引脚中断回调**

```c
rt_err_t rt_pin_attach_irq(rt_int32_t pin,
                           rt_uint32_t mode,
                           void (*hdr)(void *args),
                           void *args);
```

参数说明：

- `pin`：逻辑引脚号，即全局 GPIO 编号
- `mode`：中断触发方式，支持以下取值：
  - `PIN_IRQ_MODE_RISING`：上升沿触发
  - `PIN_IRQ_MODE_FALLING`：下降沿触发
  - `PIN_IRQ_MODE_RISING_FALLING`：双边沿触发
- `hdr`：中断回调函数
- `args`：传递给回调函数的私有参数

返回值说明：

- `RT_EOK`：绑定成功
- 其他错误码：绑定失败

**使能或关闭引脚中断**

```c
rt_err_t rt_pin_irq_enable(rt_base_t pin, rt_uint32_t enabled);
```

参数说明：

- `pin`：逻辑引脚号，即全局 GPIO 编号
- `enabled`：中断开关
  - `PIN_IRQ_ENABLE`：使能中断
  - `PIN_IRQ_DISABLE`：关闭中断

返回值说明：

- `RT_EOK`：操作成功
- 其他错误码：操作失败

**按名称获取引脚编号**

```c
rt_base_t rt_pin_get(const char *name);
```

参数说明：

- `name`：引脚名称字符串，例如 `PA.0`、`P0.12`

返回值说明：

- 成功：返回引脚编号
- 失败：返回无效编号或错误值

## 使用示例

### RT-Thread PIN 接口示例

```c
#define TEST_PIN 128

rt_pin_mode(TEST_PIN, PIN_MODE_OUTPUT);
rt_pin_write(TEST_PIN, PIN_HIGH);
rt_pin_write(TEST_PIN, PIN_LOW);
```

### GPIO 中断示例

```c
static void gpio_irq_handler(void *args)
{
    rt_kprintf("gpio irq trigger\n");
}

rt_pin_mode(128, PIN_MODE_INPUT);
rt_pin_attach_irq(128, PIN_IRQ_MODE_RISING, gpio_irq_handler, RT_NULL);
rt_pin_irq_enable(128, PIN_IRQ_ENABLE);
```

### GPIO 测试命令说明

GPIO 测试代码位于：

```text
bsp/spacemit/drivers/gpio/gpio-test.c
```

该文件主要用于验证 GPIO 输入、输出以及中断功能，覆盖 AP GPIO 和 R GPIO 两类引脚。

**`gpio_test_0001`：GPIO 输入测试**

功能说明：

- 测试 AP GPIO 输入功能，测试引脚为 AP GPIO `57`
- 测试 R GPIO 输入功能，测试引脚为逻辑号 `128`，对应 R GPIO `0`
- 通过 `rt_pin_mode()` 配置输入模式
- 通过 `rt_pin_read()` 读取输入电平

使用方法：

```sh
gpio_test_0001
```

**`gpio_test_0002`：GPIO 输出测试**

功能说明：

- 测试 AP GPIO 输出功能，测试引脚为 AP GPIO `58`
- 测试 R GPIO 输出功能，测试引脚为逻辑号 `129`，对应 R GPIO `1`
- 通过 `rt_pin_write()` 分别输出高低电平
- 通过 `rt_pin_read()` 回读输出结果

使用方法：

```sh
gpio_test_0002
```

**`gpio_test_0003`：GPIO 上升沿中断测试**

功能说明：

- 测试 AP GPIO 上升沿中断功能，使用 AP GPIO `58` 输出触发 AP GPIO `57` 输入中断
- 测试 R GPIO 上升沿中断功能，使用逻辑号 `129`（R GPIO `1`）输出触发逻辑号 `128`（R GPIO `0`）输入中断
- 通过输出脚翻转触发输入脚上升沿中断
- 每组测试默认执行 3 次

使用方法：

```sh
gpio_test_0003
```

使用前说明：

- 需要将 AP GPIO 输出测试引脚与 AP GPIO 输入测试引脚连接
- 需要将 R GPIO 输出测试引脚与 R GPIO 输入测试引脚连接

当前测试引脚定义为：

- AP GPIO 输入：`57`
- AP GPIO 输出：`58`
- R GPIO 输入：逻辑号 `128`，对应 R GPIO `0`
- R GPIO 输出：逻辑号 `129`，对应 R GPIO `1`

**`gpio_test_0004`：GPIO 下降沿中断测试**

功能说明：

- 测试 AP GPIO 下降沿中断功能，使用 AP GPIO `58` 输出触发 AP GPIO `57` 输入中断
- 测试 R GPIO 下降沿中断功能，使用逻辑号 `129`（R GPIO `1`）输出触发逻辑号 `128`（R GPIO `0`）输入中断
- 通过输出脚翻转触发输入脚下降沿中断
- 每组测试默认执行 3 次

使用方法：

```sh
gpio_test_0004
```

使用前说明：

- 需要将 AP GPIO 输出测试引脚与 AP GPIO 输入测试引脚连接
- 需要将 R GPIO 输出测试引脚与 R GPIO 输入测试引脚连接

**`gpio_test_0005`：GPIO 双边沿中断测试**

功能说明：

- 测试 AP GPIO 双边沿中断功能，使用 AP GPIO `58` 输出触发 AP GPIO `57` 输入中断
- 测试 R GPIO 双边沿中断功能，使用逻辑号 `129`（R GPIO `1`）输出触发逻辑号 `128`（R GPIO `0`）输入中断
- 每轮测试同时验证上升沿和下降沿触发
- 每组测试默认执行 3 次，预期共触发 6 次中断

使用方法：

```sh
gpio_test_0005
```

使用前说明：

- 需要将 AP GPIO 输出测试引脚与 AP GPIO 输入测试引脚连接
- 需要将 R GPIO 输出测试引脚与 R GPIO 输入测试引脚连接

**`gpio_test_all`：运行全部 GPIO 测试**

功能说明：

- 依次执行 `gpio_test_0001` ~ `gpio_test_0005`

使用方法：

```sh
gpio_test_all
```
