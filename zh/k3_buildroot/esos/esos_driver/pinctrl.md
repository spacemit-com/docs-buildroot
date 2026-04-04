# PINCTRL

介绍 ESOS on K3 小核心平台 pinctrl 驱动的实现方式、设备树写法以及各类 consumer 的使用方式。

## 模块介绍

在 K3 小核心（ESOS/RTT）系统中，pinctrl 用于完成 SoC PAD 的复用选择和电气配置。外设驱动本身通常只关心“我要使用哪一路 UART/I2C/SPI/PWM/GMAC”，真正把某组引脚切到对应功能、并把这些配置写入 PAD 寄存器的是 pinctrl 子系统。

和 Linux 大核心文档不同，这里的 pinctrl 运行在 ESOS（基于 RTT 二开）环境中，驱动框架、consumer 调用方式和 DTS 解析路径都不是 Linux 内核原生实现，而是 ESOS 自己的 pinctrl/gpio/of 子系统实现。

## 功能介绍

K3 小核心 pinctrl 的职责主要包括：

- 根据设备树中的 `pinctrl-*` 属性找到设备各个 pinctrl state
- 解析 pinctrl 子节点中的 `pinctrl-single,pins`
- 将 `<寄存器偏移, 配置值>` 形式的数据写入 pinmux 寄存器
- 为每个 pinctrl 子节点生成 pin group / function / map
- 支持 consumer 通过 pinctrl API 按 state 名称切换引脚配置
- 支持 pinconf 框架，但 K3 当前 DTS 实际上主要使用 `pinctrl-single,pins` 一次性给出完整寄存器值

## 源码结构介绍

K3 ESOS pinctrl 相关代码主要位于：

```text
~/k3-sdk/esos/
|-- bsp/spacemit/drivers/pinctrl/pinctrl-single.c
|-- components/drivers/pinctrl/core.c
|-- components/drivers/pinctrl/pinmux.c
|-- components/drivers/pinctrl/pinctrl-core.h
|-- components/drivers/include/drivers/pinctrl/pinctrl.h
|-- components/drivers/include/drivers/pinctrl/pinctrl-consumer.h
|-- components/drivers/gpio/gpiolib-of.c
`-- bsp/spacemit/platform/rt24/
    |-- k3.dtsi
    |-- k3-pinctrl.dtsi
    |-- k3-pinctrl.h
    |-- os0_rcpu/dts/k3_deb1.dts
    `-- os1_rcpu/dts/k3_deb1.dts
```


其中：

- `bsp/spacemit/drivers/pinctrl/pinctrl-single.c`：K3 小核心当前实际 pinctrl 驱动
- `components/drivers/pinctrl/core.c`：pinctrl core，负责 state 查找、map 应用、consumer API
- `bsp/spacemit/platform/rt24/k3.dtsi`：定义 pinctrl 控制器节点
- `bsp/spacemit/platform/rt24/k3-pinctrl.dtsi`：定义各外设的 pinctrl 状态节点
- `bsp/spacemit/platform/rt24/k3-pinctrl.h`：定义 K3 pad 编码宏、pin ID 和各 bit 域宏

## 驱动定位方式

可以通过从 DTS 的 `compatible` 反查驱动。
>>>>>>> 54da859 (esos: esos_driver: pinctrl: initial version)

在 `~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3.dtsi` 中，pinctrl 节点定义为：

```dts
pinctrl: pinctrl@d401e000 {
    compatible = "pinctrl-single";
    reg = <0xd401e000 0x250>;
    #gpio-range-cells = <3>;
    pinctrl-single,bit-per-mux;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0xff77>;
    status = "disabled";

## 测试用例与调试使用


这部分更适合放在文档靠后位置，原因是：

- 前文先解释清楚 pinctrl 控制器、状态模型、consumer API 和 DTS 写法
- 后文再说明“如何验证这些配置真的生效”

也就是说，测试用例章节应作为**验证与调试章节**，而不是在驱动定位之后立刻插入一个空的“示例使用”。

```

因此可以确认：

- K3 小核心 pinctrl 控制器的 `compatible` 是 **`pinctrl-single`**
- 对应驱动文件是：

```text
~/k3-sdk/esos/bsp/spacemit/drivers/pinctrl/pinctrl-single.c
```

## 测试用例与调试使用

本章放在 pinctrl 驱动原理和 consumer API 之后更合适，因为测试用例本质上是在验证：

- 某个 pinctrl state 是否真的生效
- 对应 PAD 寄存器的 mux / pull / drive-strength 是否已按预期写入
- GPIO 唤醒相关配置是否能配合外部电平变化工作

K3 ESOS pinctrl 相关测试代码位于：

```text
~/k3-sdk/esos/bsp/spacemit/drivers/pinctrl/pinctrl-test.c
~/k3-sdk/esos/bsp/spacemit/drivers/pinctrl/pinctrl-test-wakeup.c
```

这两个文件不是 pinctrl 控制器驱动本身，而是基于当前 K3 pinctrl 寄存器模型写的调试/验证命令。

### 1. `pmux_test`：检查 PAD 当前寄存器配置

`pinctrl-test.c` 导出了 shell 命令：

```c
MSH_CMD_EXPORT(pmux_test, pinctrl driver test. e.g: pmux_test help);
```

#### 用法 1：不带参数，快速检查 UART0 默认 pinmux

```sh
pmux_test
```

当不带参数时，测试代码会走 `pmux_check_uart0()`，直接读取：

- `0xd401e1e8`
- `0xd401e1ec`

也就是 K3 当前 `uart0` 那组 PAD 的寄存器值，并检查：

- pull 是否为 `PULL_UP`
- mux mode 是否为 `4`
- drive strength 是否为 `8`

如果检查通过，会打印：

```text
ok
```

这适合做一个最直接的冒烟检查：

- pinctrl default state 是否已应用
- UART0 对应 PAD 是否真的被配成了预期功能

#### 用法 2：带参数检查任意 pin 的当前配置

```sh
pmux_test <pin> <pull> <mode> <strength>
```

例如：

```sh
pmux_test 134 2 4 8
```

表示检查：

- pin = `134`
- pull = `2`，也就是 `PULL_UP`
- mux mode = `4`
- drive strength = `8`

测试代码会：

1. 根据 pin 号计算寄存器偏移
2. 读取 `PMUX_BASE_ADDR + _pin2offset(pin)`
3. 分别校验：
   - pull 类型
   - mux mode
   - drive strength
4. 全部匹配则输出 `ok`

其中参数含义在测试代码里已经写死：

- `pull: 0=DIS, 1=DOWN, 2=UP`
- `mode: 0-7`
- `strength: 0-15`

这个命令适合在你修改了某个 `pinctrl-single,pins` 节点后，直接验证寄存器实际值是否与 DTS 预期一致。

### 2. `pmux_test_wakeup`：验证 GPIO 电平翻转触发路径

`pinctrl-test-wakeup.c` 导出了另一个命令：

```c
MSH_CMD_EXPORT_ALIAS(pmux_test_wakeup, pmux_test_wakeup,
                     pinctrl driver test. e.g: pmux_test_wakeup());
```

其用法为：

```sh
pmux_test_wakeup <gpio_num> <rising|failling|all>
```

例如：

```sh
pmux_test_wakeup 57 rising
pmux_test_wakeup 57 failling
pmux_test_wakeup 57 all
```

这个测试命令会：

1. `gpio_request(pin, "pmux wakeup test")`
2. 把指定 GPIO 配成输出：
   - `gpio_direction_output(pin, 0)`
3. 按不同模式自动输出 0/1 翻转序列：
   - `rising`：适合验证上升沿唤醒
   - `failling`：适合验证下降沿唤醒
   - `all`：适合验证双边沿唤醒

它的目的不是验证普通 GPIO 收发，而是帮助检查：

- pinctrl 对唤醒相关边沿位的配置是否正确
- GPIO 翻转序列是否能与预期唤醒模式对应起来

### 3. 什么时候用这些测试命令

建议放在以下场景使用：

#### 场景 A：你刚改了 `k3-pinctrl.dtsi`

比如你修改了：

- `ruart0_0_cfg`
- `ri2c0_3_cfg`
- `rpwm0_0_cfg`

这时可以先启动系统，再用 `pmux_test` 去读实际 PAD 寄存器，确认 DTS 写进去的模式确实落到了硬件。

#### 场景 B：你怀疑 consumer 没有真正 apply state

如果驱动 probe 里已经调用了：

- `pinctrl_apply_default()`
- `pinctrl_apply_state()`

但外设仍然不工作，那么 `pmux_test` 可以帮助区分：

- 是 pinctrl state 根本没应用
- 还是 state 已应用，但外设链路本身有问题

#### 场景 C：你在调 wakeup/edge detect 路径

如果你在 DTS 里改了边沿检测或唤醒相关配置，就可以用 `pmux_test_wakeup` 做一个可重复的翻转输入序列验证，而不用每次手工短接或手动拨高低电平。


## K3 pinctrl 控制器节点说明

K3 小核心 pinctrl 控制器节点的关键属性如下：

```dts
pinctrl: pinctrl@d401e000 {
    compatible = "pinctrl-single";
    reg = <0xd401e000 0x250>;
    #gpio-range-cells = <3>;
    pinctrl-single,bit-per-mux;
    pinctrl-single,register-width = <32>;
    pinctrl-single,function-mask = <0xff77>;
    status = "disabled";

    range: gpio-range {
        #pinctrl-single,gpio-range-cells = <3>;
    };
};
```

各字段含义如下：

### `reg = <0xd401e000 0x250>;`

表示 pinctrl 控制器寄存器基地址和大小。

在 `spacemit_pcs_init()` 中，驱动通过解析 `reg`：

- 取第一项作为 `pcs->base[0]`
- 取第二项作为 `pcs->size`

后续 pinctrl-single 驱动就是围绕这段寄存器空间为每个 pin 建表、读写寄存器。

### `pinctrl-single,register-width = <32>;`

表示 pinmux 寄存器宽度为 32 bit。

驱动在 `spacemit_pcs_init()` 中会读取该属性，并据此选择：

- `pcs_readb` / `pcs_writeb`
- `pcs_readw` / `pcs_writew`
- `pcs_readl` / `pcs_writel`

K3 当前配置为 32 位，因此实际使用的是 32 位寄存器读写。

### `pinctrl-single,bit-per-mux;`

该布尔属性表示：**一个寄存器里可能包含多个 pin 的 mux bit 域**，因此驱动不是简单按“一个 pin 对应一个完整寄存器值”处理，而是按 bit-field 方式管理 pin/function。

在 `pcs_allocate_pin_table()` 中可见：

- 如果设置了 `bits_per_mux`
- 驱动会根据 `function-mask` 计算 `bits_per_pin`
- 再推导整个寄存器空间一共能描述多少个 pin

说明 K3 当前 ESOS pinctrl 驱动是按 bit-field 方式建立 pin table 的。

### `pinctrl-single,function-mask = <0xff77>;`

该属性是 pinctrl-single 驱动的关键。

在 `spacemit_pcs_init()` 中，驱动会读取它并计算：

- `pcs->fmask`
- `pcs->fshift`
- `pcs->fmax`

后续在 `pcs_enable()` 里，驱动真正写寄存器时会做：

```c
val &= ~mask;
val |= (vals->val & mask);
pcs->write(val, vals->reg);
```

也就是说：

- `function-mask` 定义了“哪些 bit 属于 pinctrl 需要管理的 mux/function 位域”
- 当应用某个 state 时，驱动只更新这些位，不会粗暴覆盖掉整个寄存器所有 bit
- K3 当前的 `0xff77` 其实就是在告诉驱动：该 PAD 配置寄存器中，mux/配置相关位分布在这个 mask 对应的 bit 位上

### `#gpio-range-cells = <3>` 与子节点 `range`

这部分用于和 GPIO 子系统配合。

- pinctrl 节点本身声明 `#gpio-range-cells = <3>`
- 其子节点 `range: gpio-range` 声明 `#pinctrl-single,gpio-range-cells = <3>`

驱动 `pcs_add_gpio_func()` 会去解析：

```c
pinctrl-single,gpio-range
```

每一项 3 个 cell，分别表示：

- `offset`
- `npins`
- `gpiofunc`

当 GPIO 子系统请求把某个 pin 切换为 GPIO 功能时，`pcs_request_gpio()` 会：

1. 根据 pin 所属 range 找到对应的 GPIO 功能值
2. 读取该 pin 的 PAD 寄存器
3. 清掉 `function-mask` 覆盖的位
4. 写入该 range 对应的 `gpiofunc`

因此，**pinctrl 不只是做外设复用，还负责在 GPIO 申请时把 pin 切回 GPIO mux 功能。**

## K3 pad 编码方式

K3 小核心 DTS 并不是写 Linux 大核心那种 `pinmux = <K3_PADCONF(pin, func)>;`，而是采用 `pinctrl-single,pins = <offset value>` 数组写法。

相关宏定义在：

```text
~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3-pinctrl.h
```

其中可见：

- pin ID 定义：`GPIO_00` ~ `GPIO_154`
- mux 定义：`MUX_MODE0` ~ `MUX_MODE7`
- 边沿检测：`EDGE_NONE` / `EDGE_RISE` / `EDGE_FALL` / `EDGE_BOTH`
- 上下拉：`PULL_DIS` / `PULL_UP` / `PULL_DOWN`
- 驱动能力：`PAD_DS0` ~ `PAD_DS15`
- 施密特输入阈值：`ST00` ~ `ST03`
- 强上拉：`SPU_EN`
- 输出沿速率：`SLE_EN`

并且提供了编码宏：

```c
#define K3_PADCONF(pinid, conf)	((pinid) * 4) (conf)
```

从 `k3-pinctrl.dtsi` 实际用法可以看出，这个宏的设计意图是把：

- pin 的寄存器偏移：`pinid * 4`
- 以及该 pin 的配置值：`conf`

成对写入 `pinctrl-single,pins` 数组。

例如：

```dts
ruart0_0_cfg: ruart0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_135, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
    >;
};
```

其本质含义就是：

- `GPIO_134` 这个 pad 的寄存器偏移是 `134 * 4`
- 将这个 pad 配置为 `MUX_MODE4 + EDGE_NONE + PULL_UP + PAD_DS8`
- `GPIO_135` 同理

在驱动 `pcs_parse_one_pinctrl_entry()` 中，`pinctrl-single,pins` 被当作 `<offset, value>` 二元组数组解析：

```c
offset = fdt32_to_cpu(*(mux++));
val = fdt32_to_cpu(*(mux++));
vals[found].reg = (void *)(pcs->base[0] + offset);
vals[found].val = val;
```

所以要理解 K3 ESOS pinctrl 的 dts，核心就是：

- **前一个数是寄存器偏移**
- **后一个数是要写入该寄存器 function-mask 对应位域的配置值**

## consumer API 与设备树状态模型

K3 小核心 pinctrl 在 consumer 侧的使用，不能只理解成“调用 `pinctrl_apply_default()` 就结束了”，还要把 **设备树状态模型** 和 **core 提供的 API** 一起看。

### 设备树中的 `pinctrl-*` 属性

在 consumer 设备节点里，pinctrl 使用的是一组通用属性，而不是只限于 `pinctrl-0`。

典型写法如下：

```dts
&rspi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rssp0_0_cfg>;
    status = "okay";
};
```

这里要这样理解：

- `pinctrl-names`：给每个 state 起名字
- `pinctrl-0`、`pinctrl-1`、`pinctrl-2` ...：分别引用对应 state 的 pinctrl 配置节点

也就是说，真正完整的模型是：

- `pinctrl-names = "default", "sleep", ...;`
- `pinctrl-0 = <...>;`
- `pinctrl-1 = <...>;`
- `pinctrl-2 = <...>;`

当前 K3 小核心平台的实际 dts 里主要使用的是：

- `pinctrl-names = "default";`
- `pinctrl-0 = <&xxx_cfg>;`

但从 pinctrl core 的设计来看，它支持的并不只是 default，而是按 **state name** 查找和切换。

### `pinctrl_get()`

`pinctrl_get(struct dtb_node *dev)` 用于为某个设备节点获取 pinctrl handle。

在 `components/drivers/pinctrl/core.c` 中，它会：

- 先查找该设备是否已有 pinctrl handle
- 如果没有，就根据该设备节点的 `pinctrl-*` 属性创建一个新的 pinctrl 对象

因此这个函数适合用于：

- 驱动希望手动管理 pinctrl 生命周期
- 后续还要多次查找 state 并主动切换

### `pinctrl_lookup_state()`

`pinctrl_lookup_state(struct pinctrl *p, const char *name)` 用于从 pinctrl handle 中查找某个名字对应的 state。

例如：

```c
pinctrl_lookup_state(p, "default");
pinctrl_lookup_state(p, "sleep");
```

它的适用场景是：

- 一个设备在运行时存在多个状态
- 驱动希望显式区分 `default`、`sleep` 等状态
- 先拿到 state handle，后续再决定什么时候切换

### `pinctrl_select_state()`

`pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)` 用于真正把某个 state 应用到硬件。

在 core 中，这个函数会：

1. 检查当前 state 是否已生效
2. 如果旧 state 中存在而新 state 中不存在的 mux group，则先执行 disable
3. 遍历新 state 的所有 setting
4. 分别执行 mux 和 config 的下发

也就是说，这个函数适合用于：

- 设备需要在不同场景之间切换 pinctrl 状态
- 例如运行态 / 休眠态 / 低功耗态切换

### `pinctrl_apply_state()`

`pinctrl_apply_state(struct dtb_node *node, const char *state)` 是一个更方便的封装接口。

它内部等价于：

1. `pinctrl_get(node)`
2. `pinctrl_lookup_state(p, state)`
3. `pinctrl_select_state(p, s)`

所以它适合用于：

- 驱动只想按名字快速切换某个 state
- 不想手动拆成 get / lookup / select 三步

例如：

```c
pinctrl_apply_state(node, "default");
pinctrl_apply_state(node, "sleep");
```

### `pinctrl_apply_default()`

`pinctrl_apply_default(struct dtb_node *node)` 是当前 K3 小核心驱动里最常见的调用方式。

它本质上就是：

```c
pinctrl_apply_state(node, "default");
```

适用于：

- 驱动 probe/init 阶段
- 设备只需要一个默认引脚状态
- 驱动不需要自己保存 pinctrl handle

当前实际 consumer 驱动里大量使用这个接口，例如：

- UART
- I2C
- SPI
- PWM
- GMAC

所以它是 **最常见** 的 API，但不是 **唯一** 的 API。

### `pinctrl_apply_sleep()`

`pinctrl_apply_sleep(struct dtb_node *node)` 本质上是：

```c
pinctrl_apply_state(node, "sleep");
```

它适用于：

- 设备在低功耗 / suspend / 待机场景下
- 需要把 PAD 切换到另一组更安全或更省电的状态

虽然当前 K3 小核心板级 dts 里主要看到的是 `default`，没有大量看到 `sleep` 的实际节点，但从 core API 设计看，它显然已经预留了这类用法。

因此在文档中应该这样理解：

- `pinctrl_apply_default()`：最常用于 probe 时应用默认状态
- `pinctrl_apply_sleep()`：适合低功耗或休眠切换
- `pinctrl_apply_state()`：适合按任意 state name 做通用切换
- `pinctrl_get()` + `pinctrl_lookup_state()` + `pinctrl_select_state()`：适合驱动自己精细管理多个 state

### 当前 K3 小核心的实际使用习惯

结合当前 dts 和 consumer 驱动代码，K3 小核心平台现在至少有两种常见使用模式。

#### 模式 1：设备节点在 pinctrl 控制器外部，由设备驱动主动应用

这是当前板级 dts 里最常见的写法，例如：

```dts
&rspi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rssp0_0_cfg>;
    status = "okay";
};
```

此时设备节点本身是一个普通 consumer，驱动 probe 时通常主动调用：

```c
pinctrl_apply_default(node);
```

像 UART、I2C、SPI、PWM、GMAC 等驱动现在基本就是这一路。

#### 模式 2：设备节点直接写在 pinctrl 节点下面，作为 hog/default 配置自动生效

除了“由 consumer 驱动主动调用”这种模式，pinctrl core 本身还支持另一种模式：

- 把一个设备节点或配置节点直接放在 pinctrl 控制器节点下面
- 让它作为 pinctrl 控制器自己的 default state
- 在 pinctrl 控制器注册完成后，由 core 自动选择并下发该 default state

这一点在 `components/drivers/pinctrl/core.c` 里可以直接看到：

```c
pctldev->p = pinctrl_get(pctldev->dev);

if (!IS_ERR(pctldev->p)) {
    pctldev->hog_default =
        pinctrl_lookup_state(pctldev->p, PINCTRL_STATE_DEFAULT);
    if (!IS_ERR(pctldev->hog_default))
        pinctrl_select_state(pctldev->p, pctldev->hog_default);
}
```

也就是说：

- 如果 default state 是挂在 pinctrl 控制器自己下面的
- 那么 pinctrl 控制器注册时，core 就会自动查找 `default` 状态并立即应用
- 这种情况下，不需要外围设备驱动再额外调用 `pinctrl_apply_*()`

因此这部分文档应该明确区分：

- **外部 consumer 模式**：设备驱动主动 `pinctrl_apply_default()` / `pinctrl_apply_state()`
- **pinctrl hog/default 模式**：配置挂在 pinctrl 节点下面，由 pinctrl core 自动生效

当前 K3 小核心板级 dts 中，常见的还是第一种；但从 core 机制上讲，第二种也是完整支持的。

## DTS 中 pinctrl 状态定义方式

K3 小核心平台的 pinctrl state 主要定义在：

```text
~/k3-sdk/esos/bsp/spacemit/platform/rt24/k3-pinctrl.dtsi
```

典型写法如下：

```dts
ruart1_0_cfg: ruart1_0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_140, (MUX_MODE2 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_141, (MUX_MODE2 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_142, (MUX_MODE2 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_143, (MUX_MODE2 | EDGE_NONE | PULL_UP | PAD_DS8))
    >;
};
```

这表示为 `ruart1` 定义了一组可直接复用的默认 pinctrl 状态。

从 `k3-pinctrl.dtsi` 可以看到，K3 小核心已预定义了大量状态组，例如：

- UART/ruart：`ruart0_0_cfg`、`ruart1_0_cfg`、`ruart5_0_cfg`
- I2C/ri2c：`ri2c0_3_cfg`、`ri2c1_1_cfg`、`ri2c2_cfg`
- PWM/rpwm：`rpwm0_0_cfg` ~ `rpwm9_*_cfg`
- GMAC：`rgmac0_cfg`
- CAN：`rcan0_0_cfg` 等
- IR：`rir0_0_cfg` 等
- SSP：`rsspa0_0_cfg` 等

这说明 K3 小核心 pinctrl.dtsi 的作用主要是：**把 SoC 侧所有可选引脚复用方案抽象成可复用的 state 组，供各 consumer 设备节点在板级 dts 中引用。**

## consumer 如何使用 pinctrl

### 板级 DTS 的使用方式

consumer 设备节点在板级 dts 中通常按以下方式引用 pinctrl：

```dts
&rspi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rssp0_0_cfg>;
    status = "okay";
};
```

或者：

```dts
&ri2c0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ri2c0_3_cfg>;
    status = "disabled";
};
```

或者：

```dts
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rgmac0_cfg>;
    status = "okay";
};
```

可以看出小核心平台当前主要使用：

- `pinctrl-names = "default";`
- `pinctrl-0 = <&xxx_cfg>;`

也就是默认状态模型。

### 驱动侧的使用方式

consumer 驱动并不是自己手写寄存器，而是在 probe 初始化时调用：

```c
pinctrl_apply_default(compatible_node);
```

这个调用在多个驱动中都能看到，例如：

- `bsp/spacemit/drivers/uart/board_uart.c`
- `bsp/spacemit/drivers/i2c/i2c-k1.c`
- `bsp/spacemit/drivers/spi/k1x_spi.c`
- `bsp/spacemit/drivers/pwm/pwm-pxa.c`
- `bsp/spacemit/drivers/gmac/dwc_eth_qos.c`

其调用时机通常是：

1. 找到自己的 compatible 节点
2. 确认节点 `status = "okay"`
3. 调用 `pinctrl_apply_default(node)`
4. 再去做时钟、复位、寄存器、中断等初始化

例如 UART 驱动 `board_uart.c` 中就是：

```c
/* use the default pinmux */
pinctrl_apply_default(compatible_node);
```

说明小核心下 pinctrl 的典型用法是：**在设备驱动 probe 时主动把 default state 应用到硬件。**

## pinctrl core 的工作流程

ESOS 的 pinctrl core 主要在：

```text
~/k3-sdk/esos/components/drivers/pinctrl/core.c
```

典型流程如下。

### 1. consumer 获取 pinctrl handle

驱动调用：

```c
pinctrl_get(node)
```

core 会为该设备创建/查找对应 pinctrl handle。

### 2. 查找 state

随后调用：

```c
pinctrl_lookup_state(p, "default")
```

根据 `pinctrl-names` 和 `pinctrl-0` 把这个设备关联到的 state 查出来。

### 3. 应用 state

再调用：

```c
pinctrl_select_state(p, state)
```

在这个函数里，core 会遍历当前 state 的所有 setting：

- `PIN_MAP_TYPE_MUX_GROUP`
- `PIN_MAP_TYPE_CONFIGS_GROUP`

分别调用：

- `pinmux_enable_setting()`
- `pinconf_apply_setting()`

最终把 pinmux 和 pinconf 都下发到硬件。

### 4. 快捷接口

为了简化 consumer 驱动使用，core 又提供了：

```c
pinctrl_apply_state(node, "default")
pinctrl_apply_default(node)
pinctrl_apply_sleep(node)
```

目前 K3 小核心驱动最常见的是 `pinctrl_apply_default(node)`。

## pinctrl-single 驱动如何解析 DTS

### 解析 `pinctrl-single,pins`

驱动 `pcs_parse_one_pinctrl_entry()` 会把每个 pinctrl state 节点中的：

```dts
pinctrl-single,pins = <offset value offset value ...>;
```

解析为：

- `pcs_function`
- `pingroup`
- `pinctrl_map`

并建立：

- 该组里有哪些 pin
- 每个 pin 对应哪个寄存器地址
- 要写什么配置值

### 应用 mux 配置

当某个 state 被选中时，最终会进入 `pcs_enable()`：

```c
val = pcs->read(vals->reg);
val &= ~mask;
val |= (vals->val & mask);
pcs->write(val, vals->reg);
```

### pinconf 支持

`pinctrl-single.c` 同时支持 pinconf 属性解析，例如：

- `pinctrl-single,drive-strength`
- `pinctrl-single,slew-rate`
- `pinctrl-single,input-schmitt`
- `pinctrl-single,bias-pullup`
- `pinctrl-single,bias-pulldown`
- `pinctrl-single,input-schmitt-enable`

对应会映射到：

- `PIN_CONFIG_DRIVE_STRENGTH`
- `PIN_CONFIG_SLEW_RATE`
- `PIN_CONFIG_INPUT_SCHMITT`
- `PIN_CONFIG_BIAS_PULL_UP`
- `PIN_CONFIG_BIAS_PULL_DOWN`
- `PIN_CONFIG_INPUT_SCHMITT_ENABLE`

K3 当前 `k3-pinctrl.dtsi` 的实际写法主要使用的是：

```dts
pinctrl-single,pins = <...>;
```

也就是把 mux、edge、pull、drive-strength 等信息直接编码进寄存器值里，而不是拆成独立 pinconf 属性。

因此当前文档是以 **实际 DTS 写法** 为主，而不是只写驱动“理论上支持什么”。

## 板级 DTS 实例

### OS1 RCPU 板级文件

`~/k3-sdk/esos/bsp/spacemit/platform/rt24/os1_rcpu/dts/k3_deb1.dts` 中可见：

```dts
&pinctrl {
    status = "okay";
};

&rspi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rssp0_0_cfg>;
    status = "okay";
};

&ri2c0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ri2c0_3_cfg>;
    status = "disabled";
};

&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rgmac0_cfg>;
    status = "okay";
};
```

说明：

- 板级 dts 先把 pinctrl 控制器本身打开
- 再给具体外设节点选择自己的 `default` 状态

### OS0 RCPU 板级文件

`~/k3-sdk/esos/bsp/spacemit/platform/rt24/os0_rcpu/dts/k3_deb1.dts` 中可见：

```dts
&pinctrl {
    status = "okay";
};

&ri2c2 {
    pinctrl-names = "default";
    pinctrl-0 = <&ri2c2_cfg>;
    status = "okay";
};
```

说明 OS0 / OS1 虽然使用场景不同，但 pinctrl 使用模型是一致的：

- SoC dtsi 提供 pinctrl 控制器与状态组
- 板级 dts 决定哪些外设启用、用哪组 pinmux
- 驱动 probe 时调用 `pinctrl_apply_default()` 真正将配置写入硬件

## 关键特性总结

K3 小核心 ESOS pinctrl 当前可以总结为以下几点：

1. **通过 DTS compatible 反查，实际驱动是 `pinctrl-single.c`**
2. **consumer 典型使用方式是 `pinctrl-names` + `pinctrl-0` + `pinctrl_apply_default()`**
3. **DTS 采用 `pinctrl-single,pins = <offset value ...>` 的寄存器直写模型**
4. **pad 配置值通过 `k3-pinctrl.h` 的宏组合得到**
5. **驱动应用配置时只修改 `function-mask` 覆盖的位域**
6. **支持 GPIO range 机制，在 GPIO 请求时将 pin 切回 GPIO 功能**
7. **虽然驱动支持独立 pinconf 属性，但 K3 当前 DTS 主要使用编码后的 `pinctrl-single,pins` 写法**

## 调试建议

如果小核心 pinctrl 配置后外设仍不工作，可以按以下顺序排查：

1. 确认 `&pinctrl { status = "okay"; };` 已在板级 dts 中打开
2. 确认目标外设节点存在完整的 `pinctrl-*` 配置，例如：
   - `pinctrl-names = "default";`
   - `pinctrl-0 = <&xxx_cfg>;`
3. 如果驱动涉及多状态切换，确认 `pinctrl-names` 与 `pinctrl-0` / `pinctrl-1` / `pinctrl-2` 的顺序和名字一致
4. 确认引用的 `xxx_cfg` 在 `k3-pinctrl.dtsi` 中确实存在
5. 确认 `k3-pinctrl.h` 中该组 PAD 的 `MUX_MODE`、`PULL_*`、`PAD_DS*` 组合正确
6. 确认外设驱动 probe 或状态切换路径中确实调用了合适的 pinctrl API：
   - `pinctrl_apply_default()`
   - `pinctrl_apply_sleep()`
   - 或 `pinctrl_apply_state()` / `pinctrl_select_state()`
7. 如果是同一外设存在多组 pinmux，可先对照 `k3-pinctrl.dtsi` 注释确认当前板子实际走的是哪一组

## 小结

K3 小核心 ESOS 的 pinctrl 文档不能简单照搬 Linux 大核心，也不能只看一篇旧使用文档。真正应当依据的是：

- pinctrl 控制器 dts compatible：`pinctrl-single`
- 实际驱动：`bsp/spacemit/drivers/pinctrl/pinctrl-single.c`
- pinctrl core：`components/drivers/pinctrl/core.c`
- 实际状态定义：`bsp/spacemit/platform/rt24/k3-pinctrl.dtsi`
- 实际 consumer 用法：UART / I2C / SPI / PWM / GMAC 等驱动中的 pinctrl API 调用

因此理解 K3 小核心 pinctrl 的关键是：

- **先看 dts compatible 定位驱动**
- **再看 `pinctrl-single,pins` 如何编码寄存器值**
- **再看 consumer 如何通过 `pinctrl-*` 属性和 pinctrl API 使用它**
- **最后结合 core 的 state 模型理解 default / sleep / 其他命名状态如何切换**
