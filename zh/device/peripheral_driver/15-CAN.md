# CAN

介绍 CAN 的配置和调试方式

## 模块介绍  

CAN（Controller Area Network，控制器局域网络）是一种用于控制器和设备之间进行通信的串行通信协议。广泛应用于汽车工业，工业自动化、医疗设备、航空航天、机器人等多个领域。

### 功能介绍  

![cat](static/can.png)

CAN 控制器支持 CAN 2.0 和 CAN FD 协议，可实现多种类型的帧传输，包括：

- 标准数据帧
- 标准远程帧
- 扩展数据帧

CAN 驱动通过网络设备接口注册为网络设备。在用户层可以通过指定网络工具或接口完成 CAN 驱动调用实现报文收发。

### 源码结构介绍

CAN 控制器驱动代码位于 `drivers/net/can` 目录下：  

```c
drivers/net/can  
|--dev.c                     #内核can框架代码，包含计算波特率参数，注册can设备等
|--flexcan/                #k1 can驱动
 |--flexcan-core.c
 |--flexcan.h
```  

## 关键特性  

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 CANFD | 支持 CANFD 协议，兼容 CAN2.0 |
| 支持最大 64B 数据 | CANFD 协议支持 8，16，32，64B 数据传输 |

### 性能参数

支持最高 8M 数据域波特率

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

`CONFIG_CAN_DEV`：此为内核平台 CAN 框架提供支持，支持 K1 CAN 驱动情况下，应为 `Y`

```shell
Symbol: CAN_DEV [=y]
Device Drivers
    -> Network device support (NETDEVICES [=y]) 
  -> CAN Device Drivers (CAN_DEV [=y])
```

在支持平台层 CAN 框架后，配置 `CONFIG_CAN_FLEXCAN` 为 `Y`，支持 K1 CAN 驱动

```shell
Symbol: CAN_FLEXCAN [=y]
    -> CAN device drivers with Netlink support (CAN_NETLINK [=y])
  -> Support for Freescale FLEXCAN based chips (CAN_FLEXCAN [=y])
```

### DTS 配置

K1 平台 **CAN 控制器不含收发器**，仅提供 TX / RX 信号，需外接收发器。

#### pinctrl 配置示例

可查看 Linux 仓库的 `arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi`，参考已配置好的 CAN 节点配置，如下：

```dts
    pinctrl_can_0: can_0_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(GPIO_75, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* can_tx0 */
            K1X_PADCONF(GPIO_76, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* can_rx0 */
     >;
    };
```

#### dtsi 配置示例

dtsi 中配置 CAN 控制器基地址和时钟复位资源，正常情况**无需改动**

```dts
    flexcan0: fdcan@d4028000 {
        compatible = "spacemit,k1x-flexcan";
        reg = <0x0 0xd4028000 0x0 0x4000>;
        interrupts = <16>;
        interrupt-parent = <&intc>;
        clocks = <&ccu CLK_CAN0>,<&ccu CLK_CAN0_BUS>;
        clock-names = "per","ipg";
        resets = <&reset RESET_CAN0>;
        fsl,clk-source = <0>;
        status = "disabled";
    };
```

#### DTS 配置示例

DTS 完整配置，如下所示
可选择配置时钟频率为 20M，40M，80M 以支持不同波特率

```dts
/*can0*/
&flexcan0 {
  pinctrl-names = "default";
  pinctrl-0 = <&pinctrl_can_0>;
  clock-frequency = <80000000>;
  status = "okay";
};

/*rcan*/
&r_flexcan {
       pinctrl-names = "default";
       pinctrl-0 = <&pinctrl_r_can_0>;
       clock-frequency = <80000000>;
       status = "okay";
       mboxes = <&mailbox 2>;
       mbox-names = "mcan0";
};

```

## 接口描述

### API 介绍

CAN 驱动主要实现了**发送接收报文的接口**
- 常用 (设备打开)：

    ```c
    static int flexcan_open(struct net_device *dev)  
    ```

- 开启 CAN 设备时调用 (报文发送)

    ```c
    static netdev_tx_t flexcan_start_xmit(struct sk_buff *skb, struct net_device *dev) 
    ```

驱动初始化阶段会从设备树读取波特率信息并保存至私有数据结构体中。

### Demo 示例

## Debug 介绍

1. 查看 CAN 设备是否加载成功

   ```
   ifconfig -a
   ```

2. K1 配置 CAN 的仲裁域和数据域波特率  

   ```
   ip link set can0 type can bitrate 125000 dbitrate 250000 berr-reporting on fd on  
   ```

3. 打开 CAN 设备 (同时另一端准备接收)  

   ```
   ip link set can0 up  
   ```

4. K1 端发送报文  
cansend格式：cansend can-dev id#data

   ```
   cansend can0 123##3.11223344556677881122334455667788aabbccdd  
   ```

5. K1 端接收报文 (另一端发送)  

   ```
   candump can0
   ```

## 测试介绍

基于 K1 平台可以外接 CAN 收发器进行测试，通讯的另一端一般选择 USBCAN 分析仪连接电脑模拟 CAN 设备，由于通信的另一端设备和用法不确定，这里主要介绍 K1 的测试用法。
以下将以 MUSE Pi 开发板为例，基于 Buildroot 系统做 demo 演示，DTS 配置请参考 DTS 配置示例章节。

1. MUSE Pi 连接 CAN 设备
   ![alt text](static/can_image_1.png)
   pin脚方向如上图所示，从上往下的绿色箭头，分别为
   - rcan tx (gpio47, 26 pin 接口的 8pin)
   - rcan rx (gpio48, 26 pin 接口的 10pin)
   - can0 tx (gpio75, 26 pin 接口的 23pin)
   - can0 rx (gpio76,26 pin 接口的 24pin)

2. PC 端安装 CAN 软件，以及接入 PC CAN (可以接入两个 CAN 外设相互收发)。本次使用的是 PEAK的 PC CAN 工具 ([PEAK官网链接](https://www.peak-system.com))
下图所示为 rcan 的接线，can0 的接线类似。

   ![alt text](static/can_image_2.jpg)

3. 查看 CAN 设备是否加载成功

```shell
# ifconfig -a
can0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          NOARP  MTU:16  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Interrupt:77 

can1      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:169.254.185.103  Mask:255.255.0.0
          UP RUNNING NOARP  MTU:72  Metric:1
          RX packets:4226044 errors:1411370 dropped:0 overruns:0 frame:1411370
          TX packets:1428220 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10 
          RX bytes:50946992 (48.5 MiB)  TX bytes:28564400 (27.2 MiB)
          Interrupt:255 


```

4. K1 配置 CAN 的仲裁域和数据域波特率，两个 CAN 设备必须要配置成相同的仲裁、数据波特率才能正常收发数据。

```shell
ip link set can1 up type can bitrate 4000000 sample-point 0.75 dbitrate 8000000 sample-point 0.8 fd on

#接收数据
candump can1
```

5. 打开另外一个 CAN 设备作为数据发送端(可以是 PC CAN，也可以是开发板的另外一个 CAN 设备，这里以另外一个 CAN 设备发送数据，PC CAN 可自行验证)

   ```shell
   cansend can1 456##3.8877665544332211aabbccddeeffaabbaabb
   ```

6. 停止 CAN设备

    ```shell
    ifconfig can1 down
    ```

## FAQ

- 在 MUSE-Pi 开发版调试 rcan，需要关闭以下的 dts 引脚配置

```dts
diff --git a/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts b/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
index 9107d43c3091..a34272ce8318 100644
--- a/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
+++ b/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
@@ -578,12 +578,12 @@ &range GPIO_124 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_125 3 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
        >;
 
-       pinctrl_rcpu: pinctrl_rcpu_grp {
-               pinctrl-single,pins = <
-                       K1X_PADCONF(GPIO_47, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* r_uart0_tx */
-                       K1X_PADCONF(GPIO_48, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* r_uart0_rx */
-               >;
-       };
+       /* pinctrl_rcpu: pinctrl_rcpu_grp { */
+       /*      pinctrl-single,pins = < */
+       /*              K1X_PADCONF(GPIO_47, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /1* r_uart0_tx *1/ */
+       /*              K1X_PADCONF(GPIO_48, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /1* r_uart0_rx *1/ */
+       /*      >; */
+       /* }; */
 
        pinctrl_gmac0: gmac0_grp {
                pinctrl-single,pins =<
@@ -1062,7 +1062,7 @@ &vi {
 
 &rcpu {
        pinctrl-names = "default";
-       pinctrl-0 = <&pinctrl_rcpu>;
+       /* pinctrl-0 = <&pinctrl_rcpu>; */
        mboxes = <&mailbox 0>, <&mailbox 1>;
        mbox-names = "vq0", "vq1";
        memory-region = <&rcpu_mem_0>, <&vdev0vring0>, <&vdev0vring1>, <&vdev0buffer>, <&rsc_table>, <&rcpu_mem_snapshots>;

```
