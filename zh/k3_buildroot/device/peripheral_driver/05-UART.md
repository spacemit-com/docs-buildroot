# UART

介绍 UART 的配置和使用方法。

## 模块介绍

UART 是一种通用异步串行通信接口，可用于控制台输出、调试日志、蓝牙模块通信、MCU 通信等场景。

### 功能介绍

Linux 下 UART 主要由以下两部分组成：

- **UART 控制器驱动**：负责访问串口控制器寄存器，完成波特率、中断、FIFO、DMA 等配置
- **串口核心与 TTY 框架**：向系统注册串口设备节点，例如 `/dev/ttyS*`

K3 平台当前 UART 使用的是 **8250 OF 平台驱动体系**，并结合 SpacemiT 在 8250 OF 中增加的 SoC 适配逻辑实现。

### 源码结构介绍

K3 UART 相关代码与描述文件主要位于：

```text
Linux-6.18/
|-- drivers/tty/serial/8250/8250_of.c
|-- drivers/tty/serial/8250/Kconfig
|-- drivers/tty/serial/Kconfig
|-- Documentation/devicetree/bindings/serial/8250.yaml
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3-pinctrl.dtsi
```

其中：

- `drivers/tty/serial/8250/8250_of.c` 是 K3 当前 dts compatible 实际落到的驱动代码
- `Documentation/devicetree/bindings/serial/8250.yaml` 是 K3 当前 UART 使用的 dt-binding
- `k3.dtsi` 定义 AP 域 UART 控制器节点
- `k3-rdomain.dtsi` 定义 RCPU 域 UART 控制器节点
- `k3-pinctrl.dtsi` 定义 UART / RUART 的 pinmux 方案

## K1 与 K3 驱动关系

这一点需要先分清，否则很容易判断错驱动文件。

### K1 旧内核中的情况

在 `k1-sdk/linux-6.6` 中，K1 常见 UART compatible 包括：

- `spacemit,pxa-uart`
- `spacemit,rcpu-pxa-uart0`
- `spacemit,rcpu-pxa-uart1`

对应驱动文件是：

```text
drivers/tty/serial/pxa_k1x.c
```

也就是说，K1 旧内核阶段使用的是一套早期的 SpacemiT PXA UART 专用驱动。

### K3 当前内核中的情况

在 `k3-sdk/linux-6.18` 中，K3 UART 节点实际写法是：

```dts
compatible = "spacemit,k1-uart",
             "intel,xscale-uart";
```

RCPU UART 节点同样也是：

```dts
compatible = "spacemit,k1-uart",
             "intel,xscale-uart";
```

所以 K3 UART 文档时，是以：

- `drivers/tty/serial/8250/8250_of.c`
- `Documentation/devicetree/bindings/serial/8250.yaml`
- `arch/riscv/boot/dts/spacemit/k3.dtsi`
- `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`
- `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi`

为准。

## 关键特性

| 特性 | 特性说明 |
| :----- | :----- |
| 支持多路 UART | K3 在 AP 域和 RCPU 域都提供多路 UART |
| 基于 8250 OF 平台驱动 | 当前 compatible 走 `8250_of.c` 路径 |
| 支持 FIFO 配置 | DTS 中可配置 `fifo-size`、`tx-threshold` |
| 支持时钟动态适配 | 8250 OF 中包含 SpacemiT 的时钟切换逻辑 |
| 支持 RCPU UART 区分 | 可通过 `spacemit,rcpu-uart` 属性区分 |
| 支持 pinctrl 多组复用 | 同一 UART 通常对应多组复用管脚 |
| 支持标准控制台链路 | 通过 `aliases`、`stdout-path`、`ttyS*` 组织 |

## 配置介绍

主要包括驱动 **CONFIG 使能配置** 和 **dts 配置**。

### CONFIG 配置

K3 当前 UART 走 8250 OF 体系，因此通常需要关注：

- `CONFIG_SERIAL_8250`
- `CONFIG_SERIAL_OF_PLATFORM`
- 以及平台对 SpacemiT SoC 的使能配置

在 `drivers/tty/serial/8250/Kconfig` 中可以看到：

```text
config SERIAL_OF_PLATFORM
	depends on SERIAL_8250 && OF
```

### DTS 配置

#### 1. AP 域 UART 节点

K3 的 AP 域 UART 定义在 `arch/riscv/boot/dts/spacemit/k3.dtsi` 中，例如：

```dts
uart5: serial@d4017400 {
    compatible = "spacemit,k1-uart",
                 "intel,xscale-uart";
    reg = <0x0 0xd4017400 0x0 0x100>;
    clocks = <&syscon_apbc CLK_APBC_UART5>,
             <&syscon_apbc CLK_APBC_UART5_BUS>,
             <&syscon_mpmu CLK_MPMU_SLOW_UART>;
    clock-names = "core", "bus", "gate";
    resets = <&syscon_apbc RESET_APBC_UART5>;
    reg-shift = <2>;
    reg-io-width = <4>;
    fifo-size = <256>;
    tx-threshold = <32>;
    interrupt-parent = <&saplic>;
    interrupts = <47 IRQ_TYPE_LEVEL_HIGH>;
    status = "disabled";
};
```

从当前 dtsi 可以看出，AP 域 UART 常用属性包括：

- `compatible`
- `reg`
- `clocks`
- `clock-names`
- `resets`
- `reg-shift`
- `reg-io-width`
- `fifo-size`
- `tx-threshold`
- `interrupt-parent`
- `interrupts`
- `status`

K3 当前 `k3.dtsi` 中定义了：

- `uart0` ~ `uart10`

共 11 路 AP 域 UART。

#### 2. RCPU 域 UART 节点

K3 的 RCPU 域 UART 定义在 `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi` 中，例如：

```dts
r_uart0: serial@c0881000 {
    compatible = "spacemit,k1-uart",
                 "intel,xscale-uart";
    reg = <0x0 0xc0881000 0x0 0x100>;
    clocks = <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0>,
             <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0_BUS>,
             <&syscon_mpmu CLK_MPMU_SLOW_UART>;
    clock-names = "core", "bus", "gate";
    spacemit,rcpu-uart;
    resets = <&syscon_rcpu_uartctrl RESET_RCPU_UARTCTRL_RUART0>;
    reg-shift = <2>;
    reg-io-width = <4>;
    fifo-size = <256>;
    tx-threshold = <32>;
    interrupt-parent = <&saplic>;
    interrupts = <251 IRQ_TYPE_LEVEL_HIGH>;
    status = "disabled";
};
```

和 AP 域 UART 相比，RCPU UART 多了：

```dts
spacemit,rcpu-uart;
```

该属性用于让驱动区分这是 RCPU 域串口。在 `8250_of.c` 中，SpacemiT 的 `spacemit_8250_set_termios()` 会读取这个属性，用不同方式为 ACPU / RCPU UART 选择时钟频率。

K3 当前 `k3-rdomain.dtsi` 中定义了：

- `r_uart0` ~ `r_uart5`

共 6 路 RCPU 域 UART。

#### 3. dt-binding 说明

K3 UART 当前使用的 binding 是：

```text
Documentation/devicetree/bindings/serial/8250.yaml
```

其中与 SpacemiT 相关的 compatible 定义为：

```yaml
- items:
    - enum:
        - mrvl,mmp-uart
        - spacemit,k1-uart
    - const: intel,xscale-uart
```

同时 binding 还规定：

- 当 compatible 包含 `spacemit,k1-uart` 时
- `clock-names` 需要包含 `core`、`bus`

而 K3 当前 dtsi 中实际写法为：

```dts
clock-names = "core", "bus", "gate";
```

也就是说：

- `core`：核心功能时钟
- `bus`：总线时钟
- `gate`：慢时钟 / 门控相关时钟

在 `drivers/tty/serial/8250/8250_of.c` 中，也可以看到对应获取逻辑：

```c
info->gate_clk = devm_clk_get_optional_enabled(dev, "gate");
info->bus_clk  = devm_clk_get_optional_enabled(dev, "bus");
info->clk      = devm_clk_get_enabled(dev, info->bus_clk ? "core" : NULL);
```

#### 4. serial aliases 与 stdout-path

在 `k3.dtsi` 中可以看到：

```dts
aliases {
    serial0 = &uart0;
    serial1 = &uart1;
    ...
    serial10 = &uart10;
    serial11 = &r_uart0;
    ...
    serial16 = &r_uart5;
};
```

例如在 `k3_evb.dts` 中：

```dts
chosen {
    bootargs = "earlycon=sbi console=ttyS0,115200 ...";
    stdout-path = "serial0:115200";
};
```

这里表示：

- `serial0` 指向 `&uart0`
- 系统控制台对应 `ttyS0`

这也是开发板默认调试串口的来源。

## 管脚配置

UART 真正从外部引脚收发数据时，还必须在 pinctrl 中将对应管脚切换到 UART 功能。

K3 的 UART / RUART pinmux 定义在：

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

例如 `uart0_0_cfg`：

```dts
uart0_0_cfg: uart0-0-cfg {
    uart0-0-pins {
        pinmux = <K3_PADCONF(149, 2)>, /* uart0 tx */
                 <K3_PADCONF(150, 2)>; /* uart0 rx */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

同一个 `uart0` 在 K3 上还有多组可选 pinmux：

- `uart0_1_cfg`
- `uart0_2_cfg`
- `uart0_3_cfg`
- `uart0_4_cfg`

说明同一 UART 控制器可以映射到不同的外部引脚组合。

再例如 `uart2_0_cfg`：

```dts
uart2_0_cfg: uart2-0-cfg {
    uart2-0-pins {
        pinmux = <K3_PADCONF(134, 2)>, /* uart2 tx */
                 <K3_PADCONF(135, 2)>, /* uart2 rx */
                 <K3_PADCONF(136, 2)>, /* uart2 cts */
                 <K3_PADCONF(137, 2)>; /* uart2 rts */

        bias-pull-up;
        drive-strength = <25>;
    };
};
```

说明 K3 UART 支持 TX/RX，也支持部分 UART 的 CTS/RTS 硬件流控管脚复用。

RCPU 域 UART 也有对应配置，例如：

- `ruart0_0_cfg`
- `ruart1_0_cfg`
- `ruart2_0_cfg`
- `ruart5_0_cfg`

因此实际使用 UART 时，通常需要同时完成：

1. 使能对应 UART / RUART 节点
2. 选择正确的 pinctrl 配置

## 使用说明

### dtsi 中的 UART 资源配置

SoC dtsi 中已经给出了 UART 控制器的基地址、时钟、复位和中断资源。正常情况下这些内容不需要板级用户修改，主要由 SoC 平台统一维护。

### dts 中使能 UART

板级 dts 通常通过覆盖节点的方式启用串口。例如 `k3_evb.dts` 中：

```dts
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_0_cfg>;
    status = "okay";
};
```

这表示：

- 使能 `uart0`
- 选择 `uart0_0_cfg` 这一组引脚复用
- 将节点状态改为 `okay`

### 控制台串口

如果某一路 UART 作为控制台输出口，还需要同时配合：

- `aliases`
- `chosen/stdout-path`
- `bootargs` 中的 `console=ttySx,baudrate`

例如：

```dts
chosen {
    bootargs = "earlycon=sbi console=ttyS0,115200 ...";
    stdout-path = "serial0:115200";
};
```

### 蓝牙等外设复用示例

在 `k3_evb.dts` 中，`uart2` 还保留了蓝牙子节点示例：

```dts
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_0_cfg>;
    status = "disabled";

    bluetooth {
        compatible = "realtek,rtl8852bs-bt";
        device-wake-gpios = <&gpio 2 30 GPIO_ACTIVE_HIGH>;
        enable-gpios = <&gpio 2 29 GPIO_ACTIVE_HIGH>;
        host-wake-gpios = <&gpio 2 28 GPIO_ACTIVE_HIGH>;
    };
};
```

## 驱动实现说明

K3 当前 UART 使用的核心驱动路径是：

```text
drivers/tty/serial/8250/8250_of.c
```

### 1. 通过 compatible 匹配到 8250 OF

K3 dtsi 中当前 compatible 是：

```dts
compatible = "spacemit,k1-uart", "intel,xscale-uart";
```

而 `Documentation/devicetree/bindings/serial/8250.yaml` 正好定义了这一组 compatible，因此当前 K3 UART 走的是 8250 OF 平台串口驱动路径。

### 2. SpacemiT 的时钟适配逻辑

`8250_of.c` 中在 `CONFIG_SOC_SPACEMIT` 条件下增加了专门的逻辑，例如：

- `spacemit_acpu_match_clk_rate()`
- `spacemit_8250_set_termios()`
- `spacemit_8250_of_serial_clk_work_cb()`

其中 `spacemit_8250_set_termios()` 会根据：

- 当前波特率
- 是否为 `spacemit,rcpu-uart`

来选择不同的串口时钟策略：

- **AP 域 UART**：按固定查表方式为常用波特率选择合适的时钟源
- **RCPU UART**：通过 `clk_round_rate()` / `clk_set_rate()` 让时钟框架选择最合适频率

这也是 K3 UART 文档里必须强调 `spacemit,rcpu-uart` 的原因。

### 3. FIFO / 阈值参数

K3 当前 dtsi 中常见写法如下：

```dts
fifo-size = <256>;
tx-threshold = <32>;
```

这两个参数不能只从字面理解成“FIFO 大小”和“阈值”，还要结合 8250 OF 驱动的实际处理路径来看。

#### `fifo-size` 的含义

`fifo-size` 会被串口公共属性解析逻辑读出，并写入：

```c
port->fifosize
```

在 `drivers/tty/serial/8250/8250_of.c` 中，后续会根据它判断该串口是否具备 FIFO 能力：

```c
if (port8250.port.fifosize)
    port8250.capabilities = UART_CAP_FIFO;
```

也就是说：

- `fifo-size` 描述的是 **该 UART 控制器硬件发送/接收 FIFO 的深度**
- 对 K3 来说，`fifo-size = <256>` 表示该串口控制器按 **256 字节 FIFO** 能力参与 8250 框架配置
- 这个值会影响 8250 核心对该端口是否按 FIFO 方式工作，以及后续发送装载策略

在 8250 核心发送路径 `serial8250_tx_chars()` 中，可以看到驱动每次会尝试向 TX FIFO 填充 `up->tx_loadsz` 个字节：

```c
count = up->tx_loadsz;
do {
    ...
    serial_out(up, UART_TX, c);
    ...
} while (--count > 0);
```

而如果端口没有显式设置 `fifosize` / `tx_loadsz`，8250 核心还会按端口类型默认值回填：

```c
if (!port->fifosize)
    port->fifosize = uart_config[port->type].fifo_size;
if (!up->tx_loadsz)
    up->tx_loadsz = uart_config[port->type].tx_loadsz;
```

K3 这里在 dts 中显式写出 `fifo-size = <256>`，本质上就是告诉 8250 框架：**该端口的硬件 FIFO 深度不是走某个保守默认值，而是按 K3 实际硬件能力配置为 256。**

#### `tx-threshold` 的含义

`tx-threshold` 并不是“FIFO 总大小”，而是 **TX FIFO 的低水位阈值**。它在 `8250_of.c` 中的处理非常直接：

```c
if ((of_property_read_u32(ofdev->dev.of_node, "tx-threshold",
              &tx_threshold) == 0) &&
    (tx_threshold < port8250.port.fifosize))
    port8250.tx_loadsz = port8250.port.fifosize - tx_threshold;
```

这段代码说明：

- 驱动先读取设备树里的 `tx-threshold`
- 如果该值合法，并且 **小于 `fifo-size`**
- 则把：

```c
tx_loadsz = fifosize - tx_threshold
```

对 K3 当前配置：

- `fifo-size = 256`
- `tx-threshold = 32`

那么得到：

```text
tx_loadsz = 256 - 32 = 224
```

这意味着什么？

可以理解为：

- 当发送 FIFO 剩余空间达到一定程度后，8250 发送路径会按 `tx_loadsz` 的规模去装填数据
- 对当前 K3 配置而言，驱动倾向于一次最多向 TX FIFO 装入 **224 字节**
- 预留的 **32 字节** 则相当于保留为“阈值空间”

因此，`tx-threshold` 在这里更准确的理解是：

- **发送 FIFO 的触发/保留阈值参数**
- 它不直接表示“发多少”，而是通过 `fifosize - tx_threshold` 间接决定 8250 核心的 `tx_loadsz`

#### 两个参数配合后的效果

把两者放在一起看，就很清楚了：

- `fifo-size = <256>`：声明 K3 UART 硬件 FIFO 深度为 256
- `tx-threshold = <32>`：指定 TX FIFO 的阈值为 32
- 驱动据此计算出：
  - `tx_loadsz = 224`

也就是说，在 8250 框架里：

- FIFO 能力按 256 字节对待
- 单次批量写 FIFO 的装载规模按 224 字节对待

这样做的目的，是让驱动既能利用大 FIFO 提升吞吐，又保留一定阈值空间，避免把 FIFO 每次都写到过满。

#### 对文档使用者的实际意义

对客户或板级开发者来说，理解这两个参数时要注意：

1. `fifo-size` 反映的是 **硬件 FIFO 深度**，不建议随意改小或改大
2. `tx-threshold` 反映的是 **发送阈值策略**，它会直接影响 8250 的 `tx_loadsz`
3. `tx-threshold` 必须小于 `fifo-size`，否则 `8250_of.c` 不会按该公式生效
4. 如果错误配置这两个参数，可能会影响串口发送性能、FIFO 利用率，严重时还可能带来吞吐下降或行为异常

## K3 UART 资源概览

从 `k3.dtsi` 和 `k3-rdomain.dtsi` 当前内容可以整理出：

- **AP 域 UART**：`uart0` ~ `uart10`
- **RCPU 域 UART**：`r_uart0` ~ `r_uart5`
- **aliases**：`serial0` ~ `serial16`

因此在调试串口、设备节点名和控制台输出口之间建立对应关系时，要始终结合 alias 来理解。

## 调试建议

如果 UART 工作异常，建议按以下顺序排查：

6. 确认 pinctrl 已切换到正确的 UART / RUART 引脚组
7. 如果作为控制台，确认 `stdout-path`、`aliases`、`console=ttySx` 三者一致
8. 如果是 RCPU UART，确认 `spacemit,rcpu-uart` 属性存在
9. 如果是外挂蓝牙等设备，确认相关 GPIO 也已经正确配置

## 小结

这次 K3 UART 文档的关键点在于：**不能只看名字像 UART 的驱动文件，就直接认定是当前驱动。**

对 K3 当前内核来说，真正应当依据的是：

- dtsi 中实际 compatible：
  - `spacemit,k1-uart`
  - `intel,xscale-uart`
- binding：
  - `Documentation/devicetree/bindings/serial/8250.yaml`
- 驱动路径：
  - `drivers/tty/serial/8250/8250_of.c`

因此写和使用 K3 UART 时，重点要关注：

- **AP 域和 RCPU 域 UART 要区分**
- **`spacemit,rcpu-uart` 会影响时钟选择策略**
- **时钟 / 复位 / 中断 / pinctrl 必须成套配置**
- **控制台串口要结合 `aliases` 和 `stdout-path` 一起理解**
