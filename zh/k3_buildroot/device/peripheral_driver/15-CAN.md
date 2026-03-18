# CAN

介绍 K3 平台 CAN / CAN FD 控制器驱动的能力、设备树配置方法、用户空间使用方式以及调试方法。

## 模块介绍

CAN（Controller Area Network）是一种广泛应用于工业控制、汽车电子、机器人和边缘设备的串行总线。相比 UART、SPI 这类点对点接口，CAN 的典型特点是：

- 多节点共享同一总线；
- 以报文为中心，而不是字节流；
- 具有仲裁、校验、错误处理和自动重发机制；
- 更适合控制类网络和现场总线场景。

从用户角度出发，CAN 文档通常最需要回答的是：

- K3 上有哪些 CAN 控制器；
- 控制器对应哪个 Linux 驱动；
- 设备树该怎么写；
- 收发器电源、终端电阻、引脚复用怎么配；
- Linux 下如何起网口、收发报文、做回环测试；
- 如果波特率不通、总线报错，应该怎么排查。

K3 平台的 CAN 控制器使用的是 **FlexCAN 驱动体系**，K3 DTS 中控制器节点的 `compatible` 为：

```dts
compatible = "spacemit,k1-flexcan";
```

因此，K3 文档应同时结合：

- K1 文档写法；
- K3 DTS / DTSI 中控制器节点和板级用法；
- K3 Linux 驱动 `drivers/net/can/flexcan/flexcan-core.c`；
- DT binding `Documentation/devicetree/bindings/net/can/fsl,flexcan.yaml`；

来说明当前 SDK 的真实使用方式。

### 功能介绍

Linux CAN 软件栈主要分为三层：

1. **CAN 网络核心（PF_CAN / SocketCAN）**  
   为用户态提供标准 socket 接口；
2. **CAN 控制器驱动**  
   负责驱动 SoC 内部 CAN / CAN FD 控制器；
3. **CAN 收发器与板级硬件**  
   负责把控制器的数字信号转换为真正的 CAN 总线差分信号。

K3 的 CAN 控制器在 Linux 中表现为网络设备，例如：

- `can0`
- `can1`
- `can2`
- `can3`
- `can4`

如果启用了 R-domain CAN，则还可能出现对应的 `r_flexcanX` 控制器实例。

### 源码结构介绍

K3 CAN 相关代码主要位于：

```text
linux-6.18/
|-- drivers/net/can/
|   |-- dev/                            # CAN 核心框架
|   `-- flexcan/flexcan-core.c         # FlexCAN 控制器驱动
|-- Documentation/devicetree/bindings/net/can/
|   |-- fsl,flexcan.yaml               # FlexCAN 控制器 binding
|   `-- can-transceiver.yaml           # CAN transceiver 子节点约束
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                        # A-domain CAN 控制器节点
    |-- k3-rdomain.dtsi                # R-domain CAN 控制器节点
    `-- k3*.dts                        # 板级使能和 pinctrl 示例
```

结合驱动代码可以看到，K3 当前匹配的 compatible 包括：

- `spacemit,k1-flexcan`
- `spacemit,k1-flexcan-can2.0`

说明 K3 继续沿用 SpacemiT 这套 FlexCAN 兼容描述，文档中关于 DTS 属性解释必须以 binding 和 K3 实际代码为准。

## 关键特性

### 控制器能力

从 K3 的 DTS 和 FlexCAN 驱动可以提炼出以下能力：

| 特性 | 说明 |
| :----- | :---- |
| 协议类型 | 支持 CAN，驱动体系也支持 CAN FD 能力（取决于具体 compatible / 控制器能力） |
| Linux 接口 | 使用 SocketCAN，表现为标准网络设备 |
| 控制器数量 | K3 A-domain 提供 `flexcan0 ~ flexcan4`，R-domain 提供 `r_flexcan0 ~ r_flexcan4` |
| 时钟源选择 | 支持 `fsl,clk-source` 属性 |
| 收发器支持 | 支持 `xceiver-supply` 和 `can-transceiver` 描述外部收发器 |
| 板级终端控制 | binding 支持 `termination-gpios` |
| 回环测试 | 可通过 SocketCAN 的 loopback / listen-only / berr-reporting 进行调试 |

### 从用户角度最值得关注的点

#### 1. CAN 控制器本身不能直接上总线，必须配收发器

这是用户最容易忽略的地方。SoC 内部 CAN 控制器只输出控制器侧数字信号，真正接到 CANH/CANL 总线上，还需要外部 CAN transceiver 芯片。因此 DTS / 硬件设计里通常要关注：

- 收发器供电是否打开；
- 收发器 standby / enable 引脚是否控制正确；
- 总线两端是否有 120Ω 终端电阻；
- 引脚复用是否和板子连线一致。

#### 2. Linux 中它是网络设备，不是字符设备

CAN 的用户态访问方式和 UART/I2C 不一样，不是去打开 `/dev/xxx`，而是通过：

- `ip link set can0 up type can bitrate ...`
- `candump can0`
- `cansend can0 ...`

来使用。这一点在文档里必须向用户讲清楚。

#### 3. 能不能通信，通常不是“驱动有没有加载”这么简单

CAN 联调要同时满足：

- 控制器 probe 正常；
- pinctrl 正确；
- 收发器供电正常；
- 对端节点存在；
- 双方波特率一致；
- 终端电阻合理；
- 总线未短路、未悬空。

所以文档里不能只讲驱动，也必须讲板级和用户空间 bring-up 方法。

### K3 板级现状和使用建议

从 K3 当前 SDK 的 `k3.dtsi`、`k3-rdomain.dtsi` 以及板级 DTS 来看，可以总结出几个很实用的事实：

1. **A-domain 一共有 5 路 CAN**：`flexcan0 ~ flexcan4`；
2. **R-domain 也有 5 路 CAN**：`r_flexcan0 ~ r_flexcan4`；
3. 当前板级里最常见的启用方式，是直接在板级 DTS 里给某一路控制器补：
   - `clock-frequency = <80000000>;`
   - `pinctrl-names = "default";`
   - `pinctrl-0 = <&canX_Y_cfg>;`
   - `status = "okay";`
4. `k3_com260_kit_v02.dts` 是一个很好的参考板，里面一次启用了多路：
   - `&flexcan0`
   - `&flexcan1`
   - `&flexcan2`
   - `&flexcan3`
   - `&flexcan4`
   - `&r_flexcan2`
5. `k3_com260.dts` 里则只启用了 `&flexcan2`，这也说明不同板子通常只会打开实际引出的那几路，而不是把所有控制器都开起来。

所以对用户来说，**最稳妥的做法不是“看到 dtsi 里有 10 路 CAN 就全开”**，而是：

- 先看原理图/接口定义；
- 再看板级 pinctrl 有没有对应 `canX_Y_cfg` / `rcanX_Y_cfg`；
- 再确认对应收发器、电源、终端电阻和外部接口是否真的焊出。

## 配置介绍

### CONFIG 配置

K3 CAN 常用配置包括：

- `CONFIG_CAN`
- `CONFIG_CAN_RAW`
- `CONFIG_CAN_BCM`
- `CONFIG_CAN_GW`
- `CONFIG_CAN_DEV`
- `CONFIG_CAN_FLEXCAN`

常见菜单路径如下：

```text
Networking support
    CAN bus subsystem support
        CAN bus subsystem support (CAN [=y])
        Raw CAN Protocol (CAN_RAW [=y])
        Broadcast Manager CAN Protocol (CAN_BCM [=y])
        CAN Device Drivers
            Platform CAN drivers with Netlink support
                Support for Freescale FLEXCAN based chips
```

> 说明：K3 虽然使用的是 SpacemiT compatible，但底层驱动仍然是 FlexCAN 驱动框架，因此配置项仍然是 `CONFIG_CAN_FLEXCAN`。

### DTS 配置

#### 1. 控制器基础节点

K3 A-domain 的控制器节点位于 `k3.dtsi`，例如：

```dts
flexcan0: fdcan@d4028000 {
        compatible = "spacemit,k1-flexcan";
        reg = <0x0 0xd4028000 0x0 0x4000>;
        interrupts = <161 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&saplic>;
        fsl,clk-source = <0>;
        clocks = <&syscon_apbc CLK_APBC_CAN0>,
                 <&syscon_apbc CLK_APBC_CAN0_BUS>;
        clock-names = "per", "ipg";
        resets = <&syscon_apbc RESET_APBC_CAN0>;
        status = "disabled";
};
```

K3 R-domain 中也有对应节点，例如：

```dts
r_flexcan0: fdcan@c0710000 {
        compatible = "spacemit,k1-flexcan";
        reg = <0x0 0xc0710000 0x0 0x4000>;
        interrupts = <241 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&saplic>;
        fsl,clk-source = <0>;
        clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0>,
                 <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0_BUS>;
        clock-names = "per", "ipg";
        resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN0>;
        status = "disabled";
};
```

各属性含义如下：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | 控制器匹配字符串，K3 当前使用 `spacemit,k1-flexcan` |
| `reg` | 控制器寄存器空间 |
| `interrupts` | 中断号 |
| `fsl,clk-source` | 时钟源选择，当前 DTS 中常设为 `<0>` |
| `clocks` / `clock-names` | CAN 外设时钟与 IPG 时钟 |
| `resets` | 控制器复位 |
| `status` | `okay` 表示启用该控制器 |

#### 2. pinctrl 配置

CAN 使用前，必须先确认 TX / RX 对应引脚复用是否正确。典型方式如下：

```dts
&flexcan0 {
        pinctrl-names = "default";
        pinctrl-0 = <&can0_0_cfg>;
        status = "okay";
};
```

具体 pin group 名称请以实际板级 DTS / pinctrl dtsi 为准。对用户来说，CAN bring-up 时最常见问题之一就是：

- DTS 里启用了 `flexcan0`；
- 但实际上 pinctrl 配到了别的 CAN 实例，或用了错误的复用组；
- 最终表现为 `can0` 能创建，但总线永远不通。

#### 3. 收发器与电源描述

根据 `fsl,flexcan.yaml`，FlexCAN 节点支持：

- `xceiver-supply`
- `can-transceiver`
- `termination-gpios`

这些属性的意义非常重要。

##### `xceiver-supply`

用于描述外部 CAN 收发器供电。如果板上收发器由可控 regulator 供电，建议显式配置。

示例：

```dts
&flexcan0 {
        xceiver-supply = <&vcc_3v3_can>;
};
```

##### `can-transceiver`

用于描述外部收发器的最大速率等约束，符合 `can-transceiver.yaml` 约定。

示例：

```dts
&flexcan0 {
        can-transceiver {
                max-bitrate = <5000000>;
        };
};
```

##### `termination-gpios`

如果板上终端电阻通过 GPIO 控制，可增加该属性，用于软件使能或关闭终端匹配。

示例：

```dts
&flexcan0 {
        termination-gpios = <&gpio 1 12 GPIO_ACTIVE_LOW>;
};
```

#### 4. 板级启用示例

一个更接近实际项目的 CAN 节点可以写成：

```dts
&flexcan0 {
        pinctrl-names = "default";
        pinctrl-0 = <&can0_0_cfg>;
        xceiver-supply = <&vcc_3v3_can>;
        status = "okay";

        can-transceiver {
                max-bitrate = <5000000>;
        };
};
```

### 用户空间使用

#### 1. 查看设备是否创建成功

系统启动后可查看：

```bash
ip link show | grep can
```

如果控制器 probe 成功，通常可看到 `can0`、`can1` 等设备。

#### 2. 设置波特率并启动接口

最常用的 bring-up 方式：

```bash
ip link set can0 down
ip link set can0 type can bitrate 500000
ip link set can0 up
```

如果需要 CAN FD：

```bash
ip link set can0 down
ip link set can0 type can bitrate 500000 dbitrate 2000000 fd on
ip link set can0 up
```

> 前提是控制器、收发器和对端都支持 CAN FD。

#### 3. 回环测试

如果想先不接外部总线，只验证控制器软件通路，可启用 loopback：

```bash
ip link set can0 down
ip link set can0 type can bitrate 500000 loopback on
ip link set can0 up
```

发送测试报文：

```bash
cansend can0 123#11223344
```

抓取报文：

```bash
candump can0
```

#### 4. 正常总线测试

在双节点或多节点 CAN 网络中：

发送端：

```bash
cansend can0 123#1122334455667788
```

接收端：

```bash
candump can1
```

如果是单机双口回环，也可以用两路控制器互连验证。

## K1 与 K3 的差异理解

### 1. 文档结构可以参考 K1，但 K3 要更强调用户使用方式

K1 的 CAN 文档更多是基础介绍，而 K3 文档需要进一步强调：

- 用户空间通过 SocketCAN 使用；
- 收发器和终端电阻必须同时考虑；
- DTS 不只是写控制器节点，还要考虑 transceiver 供电和板级限制。

### 2. K3 DTS 同时包含 A-domain 与 R-domain CAN 控制器

这意味着在 K3 上做方案设计时，需要先明确：

- 哪一路 CAN 属于 A-domain；
- 哪一路属于 R-domain；
- 是否涉及 RCPU 场景；
- 板子实际把哪一路引到了外部接口。

用户如果仅凭 `can0/can1` 名称理解，很容易和原理图实例号混淆，所以文档里必须强调“先对照 DTS 和原理图确认控制器实例”。

### 3. K3 继续沿用 FlexCAN 驱动体系

从驱动 `flexcan-core.c` 可见，K3 compatible 已经接入 FlexCAN 驱动，因此 Linux 用户空间使用方式与标准 SocketCAN 保持一致，这对应用开发是好事：

- 可以直接使用现成工具链；
- 可以沿用 Linux CAN 网络调试经验；
- 无需学习私有字符设备接口。

## 接口介绍

### 内核态接口

上层如果是内核态开发，一般通过 Linux 标准 CAN / netdev 机制工作，而不是直接调用控制器私有 API。应用驱动一般不需要和 FlexCAN 私有寄存器交互。

### 用户态接口

K3 CAN 用户态推荐使用 SocketCAN 工具链：

- `ip`
- `candump`
- `cansend`
- `cangen`
- `canfdtest`

这些工具通常来自 `iproute2` 与 `can-utils`。

## 调试方法

### 1. 查看驱动是否加载、设备是否创建

```bash
dmesg | grep -i can
ip -details link show can0
```

### 2. 查看错误状态

```bash
ip -details -statistics link show can0
```

重点关注：

- state（ERROR-ACTIVE / ERROR-PASSIVE / BUS-OFF）
- bitrate
- sample-point
- restart-ms
- error counters

### 3. 常用联调命令

启动接口：

```bash
ip link set can0 type can bitrate 500000 berr-reporting on
ip link set can0 up
```

发送：

```bash
cansend can0 123#DEADBEEF
```

接收：

```bash
candump can0
```

压力测试：

```bash
cangen can0 -g 0 -I 123 -L 8
```

### 4. 排查总线不通的建议顺序

推荐用户按以下顺序排查：

1. **控制器是否创建成功**：`ip link show` 能看到 `can0`；
2. **引脚复用是否正确**：TX/RX 是否接到了正确的实例；
3. **收发器是否上电**：`xceiver-supply`、使能脚是否正常；
4. **波特率是否一致**：两端必须一致；
5. **终端电阻是否正确**：总线两端通常各 120Ω；
6. **总线物理连接是否正常**：CANH/CANL 是否接反、短路、悬空；
7. **错误状态是否已经 BUS-OFF**：若是，需要先恢复后再测。

## FAQ

### 1. 为什么 `can0` 创建成功了，但收不到报文？

通常说明驱动已经加载，但物理链路未通。优先检查：

- pinctrl 是否正确；
- 收发器是否上电；
- 波特率是否一致；
- 是否真的接入了有报文的 CAN 总线；
- 终端电阻是否合适。

### 2. 为什么一发送就报错，甚至进入 BUS-OFF？

常见原因：

- 总线上没有其他节点应答；
- 波特率不匹配；
- 收发器异常；
- 总线接线错误；
- 终端匹配不正确。

### 3. 为什么只做软件联调时建议先开 loopback？

因为 loopback 可以先验证：

- 驱动是否正常；
- netdev 配置是否正确；
- 用户空间工具链是否能正常收发；

这样可以把“软件问题”和“总线物理问题”先分开。
