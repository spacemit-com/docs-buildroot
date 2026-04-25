# RUART

介绍 K3 rcpu（ESOS/RTT）平台 RUART 模块的定位、软件层次、设备树写法以及应用侧使用方式。

## 模块介绍

RUART（R-domain UART）是 K3 小核心侧的串行通信控制器，基于 PXA UART 硬件 IP，通过 RT-Thread Serial 框架向上提供统一的字符设备接口。在 K3 小核心 ESOS 中，RUART 模块主要承担以下职责：

- 提供串行数据收发能力
- 支持波特率、数据位、停止位、奇偶校验等参数配置
- 通过中断方式接收数据
- 通过 RT-Thread Serial 框架注册为字符设备，供应用层使用
- 与 pinctrl 子系统配合，将 UART 功能复用到目标 PAD

### 功能介绍

从应用到硬件的大致调用关系如下：

```text
应用/组件驱动
├─ rt_device_find("uart0")
├─ rt_device_open()
├─ rt_device_read()
├─ rt_device_write()
└─ rt_device_control()

                │
                ▼
RT-Thread Serial 框架
├─ components/drivers/serial/serial.c
└─ components/drivers/include/drivers/serial.h

                │
                ▼
SoC UART 驱动层
├─ bsp/spacemit/drivers/uart/board_uart.c   # 设备注册与初始化
├─ bsp/spacemit/drivers/uart/pxa_uart.c     # PXA UART 控制器操作
└─ bsp/spacemit/drivers/uart/pxa_uart.h     # 寄存器定义与数据结构

                │
                ▼
UART 控制器硬件
└─ uart0 ~ uart4
```

### 源码结构介绍

驱动源码位于 `bsp/spacemit/drivers/uart` 目录：

```text
bsp/spacemit/drivers/uart/
├── board_uart.c      # 设备树解析、设备注册与初始化
├── pxa_uart.c        # PXA UART 控制器底层操作（波特率、数据位等配置）
├── pxa_uart.h        # 寄存器位定义与私有数据结构
├── drv_uart.h        # UART 枚举类型与底层接口声明
├── uart_service.c    # RPMSG UART 服务（用于异构通信场景）
└── uart-test.c       # 测试命令实现
```

## 关键特性

| 特性 | 特性说明 |
| :---- | :---- |
| 控制器数量 | 支持 uart0 ~ uart4，共 5 路 |
| 数据位 | 支持 5 / 6 / 7 / 8 位 |
| 停止位 | 支持 1 位、2 位 |
| 奇偶校验 | 支持无校验、奇校验、偶校验 |
| 波特率 | 支持 9600 ~ 3686400 bps |
| 接收方式 | 中断接收，RX FIFO 缓冲大小 2048 字节 |
| 默认波特率 | 115200 bps |

### 支持的波特率

| 波特率 | 时钟源 |
| :---- | :---- |
| 9600 ~ 460800 | 14.48 MHz 低速时钟 |
| 921600 / 1843200 / 3686400 | 波特率 × 16 |

## 配置介绍

主要包括 **Kconfig 配置** 和 **DTS 配置**。

### Kconfig 配置

**1. 启用 RT-Thread Serial 框架：** `RT_USING_SERIAL`（由 `BSP_USING_UART` 自动选中）

**2. 启用 UART 驱动：** `BSP_USING_UART`（默认：y）

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    [*] Enable UART
```

**3. 启用 UART 测试命令（可选）：** `BSP_UART_TEST`（默认：n）

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    -> Enable UART (BSP_USING_UART [=y])
      [ ] Enable uart test driver
```

### DTS 配置

#### pinctrl

在 `k3-pinctrl.dtsi` 中，RUART 相关引脚已给出多组默认配置，例如 `ruart0` 提供三组可选引脚：

```dts
ruart0_0_cfg: ruart0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_tx */
        K3_PADCONF(GPIO_135, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_rx */
    >;
};

ruart0_1_cfg: ruart0-1-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_147, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_tx */
        K3_PADCONF(GPIO_148, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_rx */
    >;
};
```

根据板级原理图选择对应的引脚组，参考 [pinctrl](pinctrl.md)。

#### DTSI 配置示例

`k3.dtsi` 中定义了各 UART 控制器节点，通常无需修改：

```dts
uart1: serial1@c0881200 {
    compatible = "spacemit,pxa-uart1";
    reg = <0xc0881200 0x100>;
    interrupt-parent = <&intc>;
    interrupts = <0 19 0>;
    clocks = <&ccu CLK_RCPU_UART2>, <&ccu CLK_RST_RCPU_UART2>;
    status = "disabled";
};
```

#### 板级 DTS 配置示例

在板级 DTS 中启用对应 UART 并选择引脚组：

```dts
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ruart0_0_cfg>;
    status = "okay";
};
```

## 接口介绍

RUART 通过 RT-Thread 标准设备接口操作，设备名为 `uart0` ~ `uart4`。

**查找设备**

```c
rt_device_t rt_device_find(const char *name);
```

- `name`：设备名，如 `"uart0"`
- 返回值：设备句柄，失败返回 `RT_NULL`

**打开设备**

```c
rt_err_t rt_device_open(rt_device_t dev, rt_uint16_t oflag);
```

- `oflag` 常用标志：
  - `RT_DEVICE_FLAG_RDWR`：读写模式
  - `RT_DEVICE_FLAG_INT_RX`：中断接收模式

**发送数据**

```c
rt_size_t rt_device_write(rt_device_t dev, rt_off_t pos, const void *buffer, rt_size_t size);
```

- `pos`：固定传 `0`
- 返回值：实际发送字节数

**接收数据**

```c
rt_size_t rt_device_read(rt_device_t dev, rt_off_t pos, void *buffer, rt_size_t size);
```

- `pos`：固定传 `0`
- 返回值：实际接收字节数

**配置串口参数**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`：`RT_DEVICE_CTRL_CONFIG`
- `arg`：`struct serial_configure *`，配置波特率、数据位等

**关闭设备**

```c
rt_err_t rt_device_close(rt_device_t dev);
```

## 示例使用

### 发送数据示例

```c
rt_device_t dev = rt_device_find("uart0");
rt_device_open(dev, RT_DEVICE_FLAG_RDWR);

char *msg = "Hello RUART!\r\n";
rt_device_write(dev, 0, msg, rt_strlen(msg));

rt_device_close(dev);
```

### 接收数据示例

```c
rt_device_t dev = rt_device_find("uart0");
rt_device_open(dev, RT_DEVICE_FLAG_INT_RX);

char ch;
while (rt_device_read(dev, 0, &ch, 1) == 1) {
    rt_kprintf("recv: 0x%02x\n", ch);
}

rt_device_close(dev);
```

### 修改波特率示例

```c
rt_device_t dev = rt_device_find("uart0");

struct serial_configure config = RT_SERIAL_CONFIG_DEFAULT;
config.baud_rate = 921600;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &config);
```

## 应用开发

完整测试代码参考：`bsp/spacemit/drivers/uart/uart-test.c`

## Debug 介绍

### MSH 命令行

启用 `BSP_UART_TEST` 后，可在 MSH 中使用 `uart` 命令进行调试。

**查看帮助：**

```sh
msh > uart
```

**初始化目标端口：**

```sh
msh > uart init uart0
```

**发送测试数据：**

```sh
msh > uart send
```

**接收数据（默认超时 5000ms）：**

```sh
msh > uart recv 3000
```

**切换波特率：**

```sh
msh > uart baud 921600
```

**运行完整测试（需 TX-RX 短接）：**

```sh
msh > uart test
```

该测试会依次遍历 9600 / 19200 / 38400 / 57600 / 115200 / 230400 / 460800 / 921600 / 1843200 / 3686400 bps，逐字节收发验证，最终输出 PASS / FAIL。

### 查看设备注册情况

```sh
msh > list_device
device           type         ref count
-------- -------------------- ----------
uart0    Character Device     0
uart1    Character Device     0
```

## FAQ

### UART 设备找不到

1. 确认 `BSP_USING_UART` 已启用
2. 确认板级 DTS 中对应 UART 节点 `status = "okay"`
3. 查看内核日志确认驱动初始化：`dmesg | grep uart`

### 接收乱码

1. 确认收发两端波特率一致
2. 确认数据位、停止位、奇偶校验配置匹配
3. 检查 pinctrl 引脚组是否与板级原理图一致

### 波特率切换失败

调用 `rt_device_control` 修改波特率前，设备必须处于关闭状态（未被其他线程打开），否则返回 `-RT_EBUSY`。
