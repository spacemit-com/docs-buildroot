# CAN

介绍 CAN 的配置和调试方式

## 模块介绍  

CAN（Controller Area Network，控制器局域网络）是一种用于控制器和设备之间进行通信的串行通信协议。广泛应用于汽车工业、工业自动化、医疗设备、航空航天、机器人等多个领域。

### 功能介绍  

CAN 控制器支持 CAN 2.0 和 CAN FD 协议，可实现多种类型的帧传输，包括：

- 标准数据帧
- 标准远程帧
- 扩展数据帧

CAN 驱动通过网络设备接口注册为网络设备。在用户层可以通过指定网络工具或接口完成 CAN 驱动调用实现报文收发。

### 源码结构介绍

CAN 控制器驱动代码位于 `drivers/net/can` 目录下：  

```
drivers/net/can  
|--dev.c                     # 内核 CAN 框架代码，包含计算波特率参数，注册 CAN 设备等
|--flexcan/                  # K3 CAN 驱动
 |--flexcan-core.c
 |--flexcan.h
```  

## 关键特性  

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 CAN FD | 支持 CAN FD 协议，兼容 CAN 2.0 |
| 支持最大 64B 数据 | CAN FD 协议支持 8，16，32，64B 数据传输 |
| 多控制器 | K3 平台提供 10 个 CAN 控制器：5 个主域 CAN（CAN0 ~ CAN4）+ 5 个 R-domain CAN（RCAN0 ~ RCAN4） |
| 工作模式 | 通过 compatible 属性配置：`spacemit,k1-flexcan` 支持 CAN FD 模式，`spacemit,k1-flexcan-can2.0` 仅支持 CAN 2.0 模式 |

### 性能参数

- CAN FD 模式：支持最高 8M 数据域波特率
- CAN 2.0 模式：支持最高 1M 波特率

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

`CONFIG_CAN_DEV`：此为内核平台 CAN 框架提供支持，支持 K3 CAN 驱动情况下，应为 `Y`

```shell
Symbol: CAN_DEV [=y]
Device Drivers
    -> Network device support (NETDEVICES [=y]) 
  -> CAN Device Drivers (CAN_DEV [=y])
```

在支持平台层 CAN 框架后，配置 `CONFIG_CAN_FLEXCAN` 为 `Y`，支持 K3 CAN 驱动

```shell
Symbol: CAN_FLEXCAN [=y]
    -> CAN device drivers with Netlink support (CAN_NETLINK [=y])
  -> Support for Freescale FLEXCAN based chips (CAN_FLEXCAN [=y])
```

### DTS 配置

K3 平台 **CAN 控制器不含收发器**，仅提供 TX / RX 信号，需外接收发器。

#### pinctrl 配置示例

可查看 Linux 仓库的 `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi`，参考已配置好的 CAN 节点配置，如下：

```dts
can0_0_cfg: can0-0-cfg {
    can0-0-pins {
        pinmux = <K3_PADCONF(11, 3)>,     /* can0 tx */
                 <K3_PADCONF(12, 3)>;     /* can0 rx */
        bias-pull-up;
        drive-strength = <25>;
        power-source = <3300>;
    };
};

can1_0_cfg: can1-0-cfg {
    can1-0-pins {
        pinmux = <K3_PADCONF(48, 3)>,     /* can1 rx */
                 <K3_PADCONF(49, 3)>;     /* can1 tx */
        bias-pull-up;
        drive-strength = <25>;
        power-source = <3300>;
    };
};
```

#### dtsi 配置示例

dtsi 中配置 CAN 控制器基地址和时钟复位资源，正常情况**无需改动**

K3 平台提供 10 个 CAN 控制器，分为主域和 R-domain 两部分。

**compatible 属性说明：**

- `compatible = "spacemit,k1-flexcan"`：支持 CAN FD 模式（兼容 CAN 2.0）
- `compatible = "spacemit,k1-flexcan-can2.0"`：仅支持 CAN 2.0 模式

**主域 CAN 控制器（k3.dtsi）**

```dts
flexcan0: fdcan@d4028000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xd4028000 0x0 0x4000>;
    interrupts = <161 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN0>,<&syscon_apbc CLK_APBC_CAN0_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN0>;
    status = "disabled";
};

flexcan1: fdcan@d402c000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xd402c000 0x0 0x4000>;
    interrupts = <163 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN1>,<&syscon_apbc CLK_APBC_CAN1_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN1>;
    status = "disabled";
};

flexcan2: fdcan@d4034000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xd4034000 0x0 0x4000>;
    interrupts = <165 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN2>,<&syscon_apbc CLK_APBC_CAN2_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN2>;
    status = "disabled";
};

flexcan3: fdcan@d4038000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xd4038000 0x0 0x4000>;
    interrupts = <167 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN3>,<&syscon_apbc CLK_APBC_CAN3_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN3>;
    status = "disabled";
};

flexcan4: fdcan@d403c000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xd403c000 0x0 0x4000>;
    interrupts = <169 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN4>,<&syscon_apbc CLK_APBC_CAN4_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN4>;
    status = "disabled";
};
```

**R-domain CAN 控制器（k3-rdomain.dtsi）**

R-domain 提供额外 5 个 CAN 控制器，用于实时域应用：

```dts
r_flexcan0: fdcan@c0710000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xc0710000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0_BUS>;
    clock-names = "per","ipg";
    interrupts = <241 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN0>;
    status = "disabled";
};

r_flexcan1: fdcan@c0720000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xc0720000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN1>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN1_BUS>;
    clock-names = "per","ipg";
    interrupts = <243 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN1>;
    status = "disabled";
};

r_flexcan2: fdcan@c0730000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xc0730000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN2>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN2_BUS>;
    clock-names = "per","ipg";
    interrupts = <245 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN2>;
    status = "disabled";
};

r_flexcan3: fdcan@c0740000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xc0740000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN3>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN3_BUS>;
    clock-names = "per","ipg";
    interrupts = <247 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN3>;
    status = "disabled";
};

r_flexcan4: fdcan@c0750000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD 模式 */
    reg = <0x0 0xc0750000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN4>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN4_BUS>;
    clock-names = "per","ipg";
    interrupts = <249 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN4>;
    status = "disabled";
};
```

**如需使用 CAN 2.0 模式，修改 compatible 属性：**

```dts
/* 示例：将 flexcan0 配置为 CAN 2.0 模式 */
&flexcan0 {
    compatible = "spacemit,k1-flexcan-can2.0";   /* 仅支持 CAN 2.0 */
    /* 其他配置保持不变 */
};
```

#### DTS 配置示例

DTS 完整配置，如下所示。可选择配置时钟频率为 20M，40M，80M 以支持不同波特率。

**主域 CAN 配置示例：**

```dts
/* can0 */
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};

/* can1 */
&flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&can1_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

**R-domain CAN 配置示例：**

```dts
/* rcan0 */
&r_flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};

/* rcan1 */
&r_flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan1_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

## 接口描述

### API 介绍

CAN 驱动主要实现了**发送接收报文的接口**

- 常用（设备打开）：

    ```c
    static int flexcan_open(struct net_device *dev)  
    ```

- 开启 CAN 设备时调用（报文发送）

    ```c
    static netdev_tx_t flexcan_start_xmit(struct sk_buff *skb, struct net_device *dev) 
    ```

驱动初始化阶段会从设备树读取波特率信息并保存至私有数据结构体中。

## 高级配置

### CAN FD 模式下的 Mailbox 配置

**重要说明：** FlexCAN 在 CAN FD 模式下默认只使用一个 mailbox 接收报文，在高负载场景下可能出现低概率丢包。

#### 解决方案

**方案 1：使用固定帧 ID 的 Mailbox 过滤（推荐）**

如果应用场景中有固定的 CAN 帧 ID，可以通过 DTS 配置多个 mailbox 并设置 ID 过滤，提高接收可靠性。

在 DTS 中添加以下属性：

```dts
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    
    /* 配置固定帧 ID 的 mailbox */
    flexcan-mailbox-id = <0x123 0x456 0x789>;           /* 需要接收的 CAN ID */
    flexcan-mailbox-id-bits = /bits/ 8 <11 11 29>;      /* 对应 ID 的位数：11 位标准帧或 29 位扩展帧 */
    
    status = "okay";
};
```

**配置说明：**

- `flexcan-mailbox-id`：指定需要接收的 CAN 帧 ID 列表（最多支持 MAX_RX_MAILBOX 个）
- `flexcan-mailbox-id-bits`：对应每个 ID 的位数
  - `11`：标准帧（11 位 ID）
  - `29`：扩展帧（29 位 ID）

**示例：**

```dts
/* 示例 1：接收 3 个标准帧 ID */
flexcan-mailbox-id = <0x100 0x200 0x300>;
flexcan-mailbox-id-bits = /bits/ 8 <11 11 11>;

/* 示例 2：混合标准帧和扩展帧 */
flexcan-mailbox-id = <0x123 0x18FF1234>;
flexcan-mailbox-id-bits = /bits/ 8 <11 29>;
```

**方案 2：使用 CAN 2.0 模式**

如果确定应用场景中只有 CAN 2.0 帧（无 CAN FD 帧），建议使用 CAN 2.0 模式配置，可以使用 RX FIFO 提高接收性能：

```dts
&flexcan0 {
    compatible = "spacemit,k1-flexcan-can2.0";    /* 使用 CAN 2.0 模式 */
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

CAN 2.0 模式下驱动会自动启用 RX FIFO，提供更好的接收缓冲能力。

## Debug 介绍

1. 查看 CAN 设备是否加载成功

   ```shell
   ifconfig -a
   ```

2. K3 配置 CAN 的仲裁域和数据域波特率  

   ```shell
   ip link set can0 type can bitrate 125000 dbitrate 250000 berr-reporting on fd on  
   ```

3. 打开 CAN 设备（同时另一端准备接收）  

   ```shell
   ip link set can0 up  
   ```

4. K3 端发送报文  

   cansend 格式：`cansend can-dev id##data`

   ```shell
   cansend can0 123##3.11223344556677881122334455667788aabbccdd  
   ```

5. K3 端接收报文（另一端发送）  

   ```shell
   candump can0
   ```

## 测试介绍

基于 K3 平台可以外接 CAN 收发器进行测试，通讯的另一端一般选择 USB CAN 分析仪连接电脑模拟 CAN 设备。由于通信的另一端设备和用法不确定，这里主要介绍 K3 的测试用法。

以下将以 K3 开发板为例，基于 Buildroot 系统做 demo 演示，DTS 配置请参考 DTS 配置示例章节。

### 测试步骤

1. K3 开发板连接 CAN 收发器

   将 CAN TX/RX 信号连接到外部 CAN 收发器（如 TJA1050、SN65HVD230 等）

2. PC 端安装 CAN 软件，以及接入 PC CAN（可以接入两个 CAN 外设相互收发）

   推荐使用 PEAK 的 PC CAN 工具（[PEAK 官网链接](https://www.peak-system.com)）

3. 查看 CAN 设备是否加载成功

   ```shell
   # ifconfig -a
   can0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
             NOARP  MTU:16  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:10 
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
             Interrupt:161
   ```

4. 配置 CAN 波特率并启动设备

   ```shell
   # ip link set can0 type can bitrate 500000 dbitrate 2000000 berr-reporting on fd on
   # ip link set can0 up
   ```

5. K3 端发送 CAN FD 报文

   ```shell
   # cansend can0 123##3.11223344556677881122334455667788aabbccdd
   ```

   PC 端 CAN 工具应能接收到该报文

6. K3 端接收报文

   ```shell
   # candump can0
   ```

   PC 端发送报文，K3 端应能接收并显示

7. 停止 CAN 设备

   ```shell
   # ip link set can0 down
   ```

## 控制器资源汇总

K3 平台 CAN 控制器资源分布：

| 控制器 | 基地址 | 中断号 | 时钟源 | 所属域 |
| :-----| :----| :----| :----| :----|
| flexcan0 | 0xd4028000 | 161 | CLK_APBC_CAN0 | 主域 |
| flexcan1 | 0xd402c000 | 163 | CLK_APBC_CAN1 | 主域 |
| flexcan2 | 0xd4034000 | 165 | CLK_APBC_CAN2 | 主域 |
| flexcan3 | 0xd4038000 | 167 | CLK_APBC_CAN3 | 主域 |
| flexcan4 | 0xd403c000 | 169 | CLK_APBC_CAN4 | 主域 |
| r_flexcan0 | 0xc0710000 | 241 | CLK_RCPU_SYSCTRL_RCAN0 | R-domain |
| r_flexcan1 | 0xc0720000 | 243 | CLK_RCPU_SYSCTRL_RCAN1 | R-domain |
| r_flexcan2 | 0xc0730000 | 245 | CLK_RCPU_SYSCTRL_RCAN2 | R-domain |
| r_flexcan3 | 0xc0740000 | 247 | CLK_RCPU_SYSCTRL_RCAN3 | R-domain |
| r_flexcan4 | 0xc0750000 | 249 | CLK_RCPU_SYSCTRL_RCAN4 | R-domain |

## FAQ

### CAN 设备无法启动

检查以下几点：

1. 确认内核配置已启用 `CONFIG_CAN` 和 `CONFIG_CAN_FLEXCAN`
2. 确认 DTS 中 CAN 节点 status 为 "okay"
3. 确认 pinctrl 配置正确
4. 检查外部 CAN 收发器供电和连接

### CAN 通信异常

1. 检查波特率配置是否与对端一致
2. 检查 CAN_H 和 CAN_L 连接是否正确
3. 检查总线终端电阻（120Ω）是否正确连接
4. 使用 `ip -details -statistics link show can0` 查看错误统计

### CAN FD 和 CAN 2.0 如何选择

- **CAN FD 模式**（`compatible = "spacemit,k1-flexcan"`）：
  - 支持更高的数据传输速率（最高 8M）
  - 支持更大的数据负载（最大 64 字节）
  - 向下兼容 CAN 2.0
  - 推荐用于新设计
  - **注意**：默认只使用 1 个 mailbox，高负载下可能丢包，建议配置固定帧 ID 过滤

- **CAN 2.0 模式**（`compatible = "spacemit,k1-flexcan-can2.0"`）：
  - 仅支持传统 CAN 2.0 协议
  - 最大波特率 1M
  - 数据负载最大 8 字节
  - 使用 RX FIFO，接收性能更好
  - 用于需要与旧设备兼容的场景或纯 CAN 2.0 应用

### 高负载场景下出现丢包怎么办

CAN FD 模式下默认只使用 1 个 mailbox 接收，高负载时可能丢包。解决方法：

1. **配置固定帧 ID 过滤**（推荐）：
   - 在 DTS 中添加 `flexcan-mailbox-id` 和 `flexcan-mailbox-id-bits` 属性
   - 为每个需要接收的固定 ID 分配独立的 mailbox
   - 参考"高级配置"章节的详细说明

2. **切换到 CAN 2.0 模式**：
   - 如果应用中只有 CAN 2.0 帧，修改 compatible 为 `spacemit,k1-flexcan-can2.0`
   - CAN 2.0 模式使用 RX FIFO，缓冲能力更强

3. **优化应用层处理**：
   - 提高应用程序的接收处理速度
   - 使用更高优先级的线程处理 CAN 接收

### 如何查看 CAN 错误信息

```shell
# 查看详细统计信息
ip -details -statistics link show can0

# 查看内核日志
dmesg | grep -i can
```
