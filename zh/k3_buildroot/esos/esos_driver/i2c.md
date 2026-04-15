# I2C

介绍 I2C 的功能和使用方法。

## 模块介绍

K3 I2C 模块符合 I2C 总线规范。在 K3 平台该模块由 RCPU 访问，基于 RT-Thread I2C 驱动框架实现完整的 I2C 主机接口功能，可用于与各类 I2C 从设备（如 EEPROM、传感器等）进行通信。同时驱动还提供了 rpmsg 通道的 I2C 中断转发服务，支持大小核协同场景下的 I2C 访问。

### 功能介绍

![I2C 驱动框架](i2c-arch.drawio.png)

- **应用层：** 面向用户提供 I2C 设备访问服务。
- **RT-Thread I2C 框架层：** 提供统一的 `rt_i2c_transfer` 接口，屏蔽底层硬件差异。
- **设备驱动层：** 实现 I2C 控制器的寄存器操作、数据传输和中断处理。
- **物理层：** I2C 总线硬件（SCL/SDA）。

### 源码结构介绍
驱动源码位于 esos/bsp/spacemit/drivers/i2c 目录，主要文件如下：

```bash
esos/bsp/spacemit/drivers/i2c
.
|-- i2c-k1.c            # I2C 控制器核心驱动
|-- i2c-k1.h            # I2C 控制器寄存器定义及数据结构
|-- i2c-test.c           # I2C 测试工具，用于 debug 和基本功能测试
|-- i2c_service.c        # I2C rpmsg 中断转发服务（大小核协同场景）
`-- SConscript
```

## 关键特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持标准模式 (100 KHz) | 默认速率模式 |
| 支持快速模式 (400 KHz) | 通过 DTS 属性 `spacemit,i2c-fast-mode` 启用 |
| 支持高速模式 (1.5 MHz) | 通过 DTS 属性 `spacemit,i2c-high-mode` 启用，需配置 master code |
| 支持中断传输模式 | 逐字节中断驱动传输，适用于任意长度消息 |
| 支持 FIFO 传输模式 | 当 `intr_enable = false` 且消息总长度不超过 FIFO 深度（8）时，驱动软件判断选用 FIFO 模式，减少中断次数 |
| 支持 SMBus 块读取 | 支持 `RT_I2C_M_RECV_LEN` 标志，从设备动态返回数据长度 |
| 支持总线恢复 | 当总线被从设备锁定时，自动发送最多 9 个时钟脉冲恢复 |
| 支持多实例 | 支持 ri2c0、ri2c1、ri2c2 三个 I2C 控制器实例 |

## 配置介绍

1. `BSP_USING_I2C`：启用 I2C 驱动

```bash
config BSP_USING_I2C
    bool "Enable I2C"
    default n
```

2. `BSP_I2C_TEST`：启用 I2C 测试工具

```bash
if BSP_USING_I2C
    config BSP_I2C_TEST
        bool "Enable I2C test driver"
        default n
endif
```

### DTS 属性说明

| 属性 | 说明 |
| :-----| :----|
| `spacemit,i2c-fast-mode` | 启用快速模式（400 KHz） |
| `spacemit,i2c-high-mode` | 启用高速模式（1.5 MHz） |
| `spacemit,i2c-master-code` | 高速模式 master code（默认 0x0e） |
| `spacemit,i2c-clk-rate` | I2C 工作时钟频率 |
| `spacemit,i2c-lcr` | Load Count Register 值，控制时钟时序 |
| `spacemit,i2c-wcr` | Wait Count Register 值，控制等待时序 |
| `spacemit,sda-glitch-nofix` | 绕过 SDA glitch fix 逻辑。某些硬件场景下 SDA glitch fix 会导致传输 timeout，设置此属性后驱动会在 RST_CYC 寄存器中置位 `RCR_SDA_GLITCH_NOFIX` 位 |

## 示例使用

I2C 驱动对接了 RT-Thread I2C 框架，应用开发可通过标准接口进行 I2C 通信。

- 查找 I2C 总线设备

```c
struct rt_i2c_bus_device *bus = rt_i2c_bus_device_find("ri2c0");
```

- I2C 数据传输（支持组合消息）

```c
struct rt_i2c_msg msgs[2];

/* 写：发送寄存器地址 */
msgs[0].addr  = slave_addr;
msgs[0].flags = RT_I2C_WR;
msgs[0].buf   = &reg_addr;
msgs[0].len   = 1;

/* 读：接收数据 */
msgs[1].addr  = slave_addr;
msgs[1].flags = RT_I2C_RD;
msgs[1].buf   = recv_buf;
msgs[1].len   = recv_len;

rt_i2c_transfer(bus, msgs, 2);
```

- 单字节写入

```c
rt_uint8_t buf[2] = {reg_addr, value};
struct rt_i2c_msg msg = {
    .addr  = slave_addr,
    .flags = RT_I2C_WR,
    .buf   = buf,
    .len   = 2,
};

rt_i2c_transfer(bus, &msg, 1);
```

## Debug 介绍

### 调试工具

在小核串口可以通过 I2C 测试工具进行基本功能测试，基本命令如下：

1. 列出可用的 I2C 总线设备
```bash
i2c_list
```

2. 全量读写验证测试（默认从设备地址 0x50）
```bash
i2c_test <bus>
```
例如：
```bash
i2c_test ri2c1
```
该命令会依次执行：备份原始数据 → 写入 pattern 数据（0x00~0xFF）→ 回读验证 → 恢复原始数据。

> **注：** 该测试针对 EEPROM 类设备（8 位寄存器地址、容量至少 256 字节），每次写入后有 50ms 延时等待 EEPROM 写周期完成，不适用于一般 I2C 从设备。

3. 读取指定长度的数据
```bash
i2c_read <bus> <addr> <len>
```
例如：
```bash
i2c_read ri2c0 0x50 64
```
从寄存器地址 0x00 开始逐字节读取 64 字节数据并以十六进制格式打印。

> **注：** 该命令假定从设备支持 8 位寄存器地址，不适用于无寄存器地址或寄存器地址宽度不同的设备。

### 传输模式选择
驱动内部支持 FIFO 与中断两种传输模式：
- 当 `intr_enable = false` 且消息总长度 ≤ 8 字节时，使用 FIFO 模式（减少中断开销）
- 当消息总长度 > 8 字节、包含 SMBus 块读取（`RT_I2C_M_RECV_LEN`）或 `intr_enable = true` 时，使用中断模式

> **注：** 当前驱动 probe 默认设置 `intr_enable = true`，因此默认总是中断模式；如需启用 FIFO 自动选择，需修改驱动内部 `intr_enable` 标志。

## FAQ

### I2C 传输超时
当 I2C 传输出现超时，串口会打印如下 log：
```
msg completion timeout
```
可能原因：

**1. 从设备未响应**

检查从设备地址是否正确，从设备是否正常上电。可通过 `i2c_read` 命令尝试读取确认设备是否在线。

**2. 总线被锁定**

驱动会自动尝试总线恢复（发送最多 9 个时钟脉冲），如果恢复失败会打印：
```
bus reset clk reaches the max 9-clocks
```
此时需检查硬件连接。

**3. 时钟配置不当**

检查 DTS 中 `spacemit,i2c-clk-rate` 配置是否正确，以及 `spacemit,i2c-lcr`（Load Count Register）和 `spacemit,i2c-wcr`（Wait Count Register）的时序参数是否匹配目标速率。

### I2C 传输错误
当出现传输错误，串口会打印如下 log：
```
i2c error status: 0x00400000
```
根据状态位判断：

- **SR_BED（Bus Error）：** 总线错误，未收到期望的 ACK/NAK，检查从设备是否在线及地址是否正确。
- **SR_ALD（Arbitration Loss）：** 仲裁丢失，多主机环境下可能出现，当前驱动未启用自动重试，需上层重新发起传输。
- **SR_RXOV（RX Overrun）：** 接收 FIFO 溢出，通常不应出现，如出现请检查中断响应是否及时。
