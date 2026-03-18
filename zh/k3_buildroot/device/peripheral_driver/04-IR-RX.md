# IR-RX

介绍 K3 平台 IR 接收控制器的配置、驱动接口和基本调试方式。

## 模块介绍

K3 上的 IR-RX 仍然走 Linux `rc-core` 原始红外解码框架，底层控制器驱动复用了 K1 兼容 IP，而不是一条全新的 K3 专用 IR 驱动。

K3 SDK 当前实际对应关系如下：

- IR 控制器驱动：`drivers/media/rc/ir-spacemit-k1.c`
- rc-core 原始事件框架：`drivers/media/rc/rc-ir-raw.c`
- 常见协议解码器：
  - `drivers/media/rc/ir-nec-decoder.c`
  - `drivers/media/rc/ir-rc5-decoder.c`
  - `drivers/media/rc/ir-rc6-decoder.c`
  - `drivers/media/rc/ir-sony-decoder.c`
  - 等

驱动里设备名定义为：

- `SPACEMIT_IR_DEV = "spacemit-ir"`

K3 上实际存在 4 个 IR-RX 控制器节点：

- A 域：`ircrx0`、`ircrx1`
- R 域：`rircrx0`、`rircrx1`

### 功能介绍

K3 平台外接红外接收头（通常是已解调输出的 IR 接收器）后，控制器驱动从硬件 FIFO 读取脉宽信息，经 `rc-core` 原始事件接口送入内核解码器，再转换成用户空间可见的按键事件。

当前从驱动源码可确认的实现特征包括：

- 使用 `RC_DRIVER_IR_RAW`
- 支持原始脉宽事件上报
- `allowed_protocols = RC_PROTO_BIT_ALL_IR_DECODER`
- FIFO 深度为 `32`
- 驱动内设置了噪声阈值和比较阈值

### 源码结构介绍

```text
drivers/media/rc/
|-- rc-ir-raw.c              # rc-core 原始事件框架
|-- ir-nec-decoder.c         # NEC 协议解码器
|-- ir-rc5-decoder.c         # RC5 协议解码器
|-- ir-rc6-decoder.c         # RC6 协议解码器
|-- ir-sony-decoder.c        # Sony 协议解码器
|-- ir-spacemit-k1.c         # Spacemit IR-RX 控制器驱动
```

## 关键特性

- K3 当前复用 `compatible = "spacemit-k1,irc"`
- 使用 `rc-core` 标准红外输入框架
- 支持原始红外脉宽采集与内核协议解码
- 驱动中 FIFO 深度为 `32`
- 驱动默认超时为 `10000`
- 驱动 `rx_resolution` 由 `IR_FREQ = 100000` 推导，周期分辨率为 `10us`

## 配置介绍

主要包括 **CONFIG 配置** 和 **DTS 配置**。

### CONFIG 配置

K3 SDK 对应的配置项不是 K1 文档里的 `IR_SPACEMIT`，而是：

```text
config IR_SPACEMIT_K1
    tristate "SPACEMIT IR remote Recriver control"
    depends on SOC_SPACEMIT
```

对应 Makefile：

```make
obj-$(CONFIG_IR_SPACEMIT_K1) += ir-spacemit-k1.o
```

因此使用 K3 SDK 时，至少应确认：

- `CONFIG_RC_CORE=y`
- `CONFIG_RC_DEVICES=y`
- `CONFIG_IR_SPACEMIT_K1=y` 或 `m`
- 以及需要的协议解码器，例如：
  - `CONFIG_IR_NEC_DECODER`
  - `CONFIG_IR_RC5_DECODER`
  - `CONFIG_IR_RC6_DECODER`
  - `CONFIG_IR_SONY_DECODER`

### DTS 配置

#### 1. SoC dtsi 控制器节点

K3 主域 `k3.dtsi` 中定义了两个 IR-RX 控制器：

```dts
ircrx0: irc-rx0@d4017e00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017e00 0x0 0x100>;
    interrupts = <69 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR0>;
    resets = <&syscon_apbc RESET_APBC_IR0>;
    clock-frequency = <102400000>;
    status = "disabled";
};

ircrx1: irc-rx1@d4017f00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017f00 0x0 0x100>;
    interrupts = <20 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR1>;
    resets = <&syscon_apbc RESET_APBC_IR1>;
    clock-frequency = <102400000>;
    status = "disabled";
};
```

R 域 `k3-rdomain.dtsi` 中还定义了两个 IR-RX 控制器：

```dts
rircrx0: rirc-rx0@c0887000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc0887000 0x0 0x100>;
    interrupts = <274 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC0>;
    clock-frequency = <102400000>;
    status = "disabled";
};

rircrx1: rirc-rx1@c088e000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc088e000 0x0 0x100>;
    interrupts = <275 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC1>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC1>;
    clock-frequency = <102400000>;
    status = "disabled";
};
```

这说明 K3 至少在设备树定义上给了 4 个 IR 接收控制器入口。

#### 2. 板级 dts 使能示例

当前在 `k3_evb.dts` 中能直接看到：

```dts
&ircrx0 {
    status = "okay";
};

&ircrx1 {
    status = "okay";
};

&rircrx0 {
    status = "okay";
};

&rircrx1 {
    status = "okay";
};
```

所以 K3 EVB 上四个 IR-RX 控制器节点都被打开了。

#### 3. `linux,rc-map-name`

驱动中会读取：

```c
ir->map_name = of_get_property(dn, "linux,rc-map-name", NULL);
ir->rc->map_name = ir->map_name ?: RC_MAP_EMPTY;
```

如果板级 DTS 需要绑定默认按键映射表，可以在节点中补：

```dts
linux,rc-map-name = "rc-empty";
```

或者改成你实际使用的 rc keymap。

如果不配，驱动会回退到：

- `RC_MAP_EMPTY`

## 接口描述

### 驱动关键接口

驱动主要通过 `rc-core` 提供原始红外事件：

```c
ir_raw_event_store_with_filter(struct rc_dev *dev, struct ir_raw_event *ev)
ir_raw_event_handle(struct rc_dev *dev)
```

其中中断处理函数：

- `spacemit_ir_irq()`

会做这些事：

1. 读取中断状态
2. 清中断
3. 从 FIFO 中取脉宽数据
4. 组织 `struct ir_raw_event`
5. 调用 `ir_raw_event_store_with_filter()` 入队
6. 调用 `ir_raw_event_handle()` 触发后续解码

### 驱动初始化相关实现

源码里几个关键函数：

- `spacemit_ir_probe()`
- `spacemit_ir_config()`
- `spacemit_ir_hw_init()`
- `spacemit_ir_set_idle()`
- `spacemit_ir_suspend()`
- `spacemit_ir_resume()`

能确认的实现细节包括：

- 驱动会获取 clock / reset 资源
- 可选读取 `clock-frequency`
- `clkdiv = DIV_ROUND_UP(ir->freq, IR_FREQ) - 1`
- 设置 noise threshold：`SPACEMIT_IR_NOISE = 1`
- 设置 FIFO compare threshold：`IRFIFO_DEF_CMP = 1`
- 使能所有 IR 中断：`IR_IRQ_ENALL`
- 使能控制器：`SPACEMIT_IR_ENABLE`

## 调试介绍

### 1. 先看 dmesg 和节点注册

先确认：

- IR 控制器节点是否为 `okay`
- `CONFIG_IR_SPACEMIT_K1` 是否打开
- 启动日志里是否出现 `spacemit-ir`

建议先看：

```bash
dmesg | grep -i ir
dmesg | grep -i rc
dmesg | grep -i spacemit
```

### 2. 查看输入设备和 rc 设备

常见检查方式：

```bash
ls /sys/class/rc/
cat /sys/class/rc/rc0/protocols
```

如果系统正确注册了 rc 设备，通常还能看到关联输入设备。

### 3. 用 `ir-keytable` 查看协议和事件

用户态常用工具是：

```bash
ir-keytable
ir-keytable -t
```

例如：

```bash
ir-keytable -s rc0
ir-keytable -t
```

按遥控器按键时，若硬件、时钟、引脚和协议都正常，应能看到脉冲 / 解码结果。

## 测试介绍

可以按下面顺序做最小验证：

1. 板级 DTS 把目标 IR 节点设为 `okay`
2. 编译打开 `CONFIG_RC_CORE`、`CONFIG_RC_DEVICES`、`CONFIG_IR_SPACEMIT_K1`
3. 启动后确认 `/sys/class/rc/` 下出现 rc 设备
4. 外接红外接收头并使用遥控器发码
5. 用 `ir-keytable -t` 观察是否有原始或解码后的事件

如果 `ir-keytable -t` 完全无输出，优先检查：

- 节点是不是 `okay`
- 选的是 `ircrx0/1` 还是 `rircrx0/1`
- 时钟 / reset 是否正常
- 硬件接的是不是已经解调后的 IR-RX 信号
- 引脚复用是否正确
- `linux,rc-map-name` / 协议解码器是否匹配

## FAQ

### 1. K3 用的是不是 K3 专用 IR 驱动？

不是。

按当前 SDK 实现，K3 仍然复用：

- `drivers/media/rc/ir-spacemit-k1.c`
- `compatible = "spacemit-k1,irc"`

### 2. K3 上有几个 IR-RX 控制器？

当前 DTS 里能确认 4 个：

- `ircrx0`
- `ircrx1`
- `rircrx0`
- `rircrx1`

### 3. 为什么板级 DTS 只写 `status = "okay";` 也能工作？

因为寄存器、时钟、复位、中断这些基础资源已经在 `k3.dtsi` / `k3-rdomain.dtsi` 里定义好了。板级 DTS 通常只需要选择启用哪个控制器，再补充必要的 pinctrl / keymap 即可。
