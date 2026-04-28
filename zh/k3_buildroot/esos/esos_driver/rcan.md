# RCAN

介绍 K3 rcpu（ESOS/RTT）平台 RCAN 模块的定位、软件层次、设备树写法以及应用侧使用方式。

## 模块介绍

RCAN（R-domain CAN）是 K3 小核心侧的 CAN 总线控制器，基于 FlexCAN 硬件 IP，通过 RT-Thread CAN 框架向上提供统一的 CAN 设备接口。在 K3 小核心 ESOS 中，RCAN 模块主要承担以下职责：

- 提供 CAN 2.0A/B 和 CAN FD 协议支持
- 支持标准帧（11-bit ID）和扩展帧（29-bit ID）
- 支持灵活的接收过滤器配置（ID + Mask）
- 通过中断方式收发 CAN 报文
- 支持环回模式（Loopback）用于自测
- 通过 RT-Thread CAN 框架注册为 CAN 设备，供应用层使用
- 与 pinctrl 子系统配合，将 CAN 功能复用到目标 PAD

### 功能介绍

从应用到硬件的大致调用关系如下：

```text
应用/组件驱动
├─ rt_device_find("flexcan0")
├─ rt_device_open()
├─ rt_device_read()
├─ rt_device_write()
└─ rt_device_control()

                │
                ▼
RT-Thread CAN 框架
├─ components/drivers/can/can.c
└─ components/drivers/include/drivers/can.h

                │
                ▼
SoC CAN 驱动层
├─ bsp/spacemit/drivers/can/drv_can.c       # 设备注册与 RT-Thread 接口适配
├─ bsp/spacemit/drivers/can/fsl_flexcan.c   # FlexCAN 控制器底层操作
├─ bsp/spacemit/drivers/can/fsl_flexcan.h   # 寄存器定义与数据结构
└─ bsp/spacemit/drivers/can/flexcan-test.c  # 测试命令实现

                │
                ▼
FlexCAN 控制器硬件
└─ flexcan0 ~ flexcan4
```

### 源码结构介绍

驱动源码位于 `bsp/spacemit/drivers/can` 目录：

```text
bsp/spacemit/drivers/can/
├── drv_can.c         # 设备树解析、设备注册与 RT-Thread CAN 框架适配
├── drv_can.h         # CAN 寄存器位定义与宏
├── fsl_flexcan.c     # FlexCAN 控制器底层操作（初始化、收发、过滤器配置）
├── fsl_flexcan.h     # FlexCAN 寄存器定义与数据结构
├── flexcan-test.c    # MSH 测试命令实现（loopback/send/recv/filter/stress）
└── SConscript        # 编译脚本
```

## 关键特性

| 特性 | 特性说明 |
| :---- | :---- |
| 控制器数量 | 支持 flexcan0 ~ flexcan4，共 5 路 |
| 协议支持 | CAN 2.0A/B、CAN FD |
| 帧类型 | 标准帧（11-bit ID）、扩展帧（29-bit ID） |
| 数据长度 | CAN 2.0：0~8 字节；CAN FD：0~64 字节 |
| 波特率 | 仲裁段：最高 1 Mbps；数据段（FD）：最高 5 Mbps |
| 接收邮箱 | 12 个独立 RX Mailbox，支持 ID + Mask 过滤 |
| 发送邮箱 | 1 个 TX Mailbox（MB13） |
| 工作模式 | 正常模式、环回模式（Loopback）、监听模式（Listen-Only） |
| 时钟源 | 80 MHz（来自 PLL6_80） |

### 支持的波特率配置

| 仲裁段波特率 | 数据段波特率（FD） | 典型应用场景 |
| :---- | :---- | :---- |
| 125 kbps | - | 低速 CAN 网络 |
| 250 kbps | - | 工业控制 |
| 500 kbps | 1 Mbps | 汽车电子（默认配置） |
| 1 Mbps | 2 Mbps | 高速 CAN FD |

## 配置介绍

主要包括 **Kconfig 配置** 和 **DTS 配置**。

### Kconfig 配置

**1. 启用 RT-Thread CAN 框架：** `RT_USING_CAN`（由 `BSP_USING_CAN` 自动选中）

**2. 启用 CAN FD 支持：** `RT_CAN_USING_CANFD`（由 `BSP_USING_CAN` 自动选中）

**3. 启用 CAN 驱动：** `BSP_USING_CAN`（默认：y）

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    [*] Enable CAN
```

### DTS 配置

#### pinctrl

在 `k3-pinctrl.dtsi` 中，RCAN 相关引脚已给出多组默认配置，例如 `rcan0` 提供两组可选引脚：

```dts
rcan0_0_cfg: rcan0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_57, (MUX_MODE3 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 rx */
        K3_PADCONF(GPIO_58, (MUX_MODE3 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 tx */
    >;
};

rcan0_1_cfg: rcan0-1-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_90, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 rx */
        K3_PADCONF(GPIO_91, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 tx */
    >;
};
```

根据板级原理图选择对应的引脚组，参考 [pinctrl](pinctrl.md)。

#### DTSI 配置示例

`k3.dtsi` 中定义了各 CAN 控制器节点，通常无需修改：

```dts
flexcan0: can0@c0710000 {
    compatible = "spacemit,flexcan0";
    reg = <0xc0710000 0x10000>;
    interrupt-parent = <&intc>;
    interrupts = <0 36 0>;
    clocks = <&ccu CLK_RCPU_CAN0>, <&ccu CLK_RST_RCPU_CAN0>;
    clock-frequency = <80000000>;
    status = "disabled";
};
```

#### 板级 DTS 配置示例

在板级 DTS 中启用对应 CAN 并选择引脚组：

```dts
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan0_0_cfg>;
    status = "okay";
};

&flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan1_0_cfg>;
    status = "okay";
};

&flexcan2 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan2_0_cfg>;
    status = "okay";
};

&flexcan3 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan3_0_cfg>;
    status = "okay";
};

&flexcan4 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan4_0_cfg>;
    status = "okay";
};
```

## 接口介绍

RCAN 通过 RT-Thread 标准 CAN 设备接口操作，设备名为 `flexcan0` ~ `flexcan4`。

**查找设备**

```c
rt_device_t rt_device_find(const char *name);
```

- `name`：设备名，如 `"flexcan0"`
- 返回值：设备句柄，失败返回 `RT_NULL`

**配置 CAN 参数**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`：`RT_DEVICE_CTRL_CONFIG`
- `arg`：`struct can_configure *`，配置波特率、工作模式、CAN FD 使能等

**打开设备**

```c
rt_err_t rt_device_open(rt_device_t dev, rt_uint16_t oflag);
```

- `oflag` 常用标志：
  - `RT_DEVICE_FLAG_INT_TX`：中断发送模式
  - `RT_DEVICE_FLAG_INT_RX`：中断接收模式

**发送 CAN 报文**

```c
rt_size_t rt_device_write(rt_device_t dev, rt_off_t pos, const void *buffer, rt_size_t size);
```

- `pos`：固定传 `0`
- `buffer`：`struct rt_can_msg *`
- `size`：`sizeof(struct rt_can_msg)`
- 返回值：成功返回 `sizeof(struct rt_can_msg)`

**接收 CAN 报文**

```c
rt_size_t rt_device_read(rt_device_t dev, rt_off_t pos, void *buffer, rt_size_t size);
```

- `pos`：固定传 `0`
- `buffer`：`struct rt_can_msg *`
- `size`：`sizeof(struct rt_can_msg)`
- 返回值：成功返回 `sizeof(struct rt_can_msg)`

**设置接收过滤器**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`：`RT_CAN_CMD_SET_FILTER`
- `arg`：`struct rt_can_filter_config *`，配置过滤器 ID 和 Mask

**关闭设备**

```c
rt_err_t rt_device_close(rt_device_t dev);
```

## 示例使用

### 发送标准帧示例

```c
rt_device_t dev = rt_device_find("flexcan0");

struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.mode = RT_CAN_MODE_NORMAL;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

rt_device_open(dev, RT_DEVICE_FLAG_INT_TX | RT_DEVICE_FLAG_INT_RX);

struct rt_can_msg msg = {0};
msg.ide = RT_CAN_STDID;
msg.rtr = RT_CAN_DTR;
msg.id = 0x123;
msg.len = 8;
for (int i = 0; i < 8; i++)
    msg.data[i] = i;

rt_device_write(dev, 0, &msg, sizeof(msg));
rt_device_close(dev);
```

### 接收报文示例

```c
rt_device_t dev = rt_device_find("flexcan0");

struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.mode = RT_CAN_MODE_NORMAL;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

rt_device_open(dev, RT_DEVICE_FLAG_INT_RX);

struct rt_can_msg msg;
if (rt_device_read(dev, 0, &msg, sizeof(msg)) == sizeof(msg))
{
    rt_kprintf("RX: ID=0x%X len=%d\n", msg.id, msg.len);
}

rt_device_close(dev);
```

### CAN FD 发送示例

```c
struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.baud_rate_fd = CAN1MBaud;
cfg.enable_canfd = 1;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

struct rt_can_msg msg = {0};
msg.ide = RT_CAN_STDID;
msg.rtr = RT_CAN_DTR;
msg.id = 0x200;
msg.len = 12;
msg.fd_frame = 1;
msg.brs = 1;  /* 数据段使用 1 Mbps */
for (int i = 0; i < 12; i++)
    msg.data[i] = i;

rt_device_write(dev, 0, &msg, sizeof(msg));
```

### 设置接收过滤器示例

```c
struct rt_can_filter_item filter;
filter.ide = RT_CAN_STDID;
filter.rtr = RT_CAN_DTR;
filter.id = 0x200;
filter.mask = 0x7FF;  /* 精确匹配 0x200 */
filter.mode = 0;
filter.hdr_bank = 0;

struct rt_can_filter_config cfg;
cfg.count = 1;
cfg.actived = 1;
cfg.items = &filter;

rt_device_control(dev, RT_CAN_CMD_SET_FILTER, &cfg);
```

## 应用开发

完整测试代码参考：`bsp/spacemit/drivers/can/flexcan-test.c`

## Debug 介绍

### MSH 命令行

可在 MSH 中使用 `can_test` 命令进行调试。

**查看帮助：**

```sh
msh > can_test
```

**环回自测（需 CAN FD 支持）：**

```sh
msh > can_test loopback flexcan0
```

该测试会发送 3 帧（经典 CAN 8 字节、FD 12 字节无 BRS、FD 12 字节有 BRS），并验证全部接收成功。

**发送单帧：**

```sh
msh > can_test send flexcan0 0x123 de ad be ef
```

发送 ID=0x123 的标准帧，数据为 `DE AD BE EF`。

**接收报文：**

```sh
msh > can_test recv flexcan0 4
```

接收 4 帧报文并打印。

**设置过滤器并接收：**

```sh
msh > can_test filter flexcan0 0x200 0x7FF
```

设置过滤器精确匹配 ID=0x200，然后接收 1 帧。

**压力测试（需 TX-RX 短接）：**

```sh
msh > can_test stress flexcan0 200
```

环回模式下发送 200 帧，验证收发计数一致。

### 查看设备注册情况

```sh
msh > list_device
device           type         ref count
-------- -------------------- ----------
flexcan0 CAN Device           0
flexcan1 CAN Device           0
```

## FAQ

### CAN 设备找不到

1. 确认 `BSP_USING_CAN` 已启用
2. 确认板级 DTS 中对应 CAN 节点 `status = "okay"`
3. 查看内核日志确认驱动初始化：`dmesg | grep can`

### 发送失败或接收不到数据

1. 确认 CAN 总线上有终端电阻（120Ω）
2. 确认收发两端波特率一致
3. 检查 pinctrl 引脚组是否与板级原理图一致
4. 使用环回模式（`RT_CAN_MODE_LOOPBACK`）排除硬件问题

### CAN FD 报文发送失败

1. 确认 `cfg.enable_canfd = 1` 已设置
2. 确认 `msg.fd_frame = 1` 已设置
3. 确认对端设备支持 CAN FD 协议
