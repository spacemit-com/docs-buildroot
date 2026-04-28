# DMA

介绍 DMA 的功能和使用方法。

## 模块介绍

K3 DMA 支持内存到内存、内存到外设、外设到内存三种传输方向。在 K3 平台该模块由 RCPU 访问，基于 RT-Thread DMA 框架实现 DMA 传输功能，可为 SPI 等外设提供 DMA 数据搬运能力，也可用于高效的内存拷贝。

### 功能介绍

![DMA 驱动框架](dma-arch.drawio.png)

- **应用层 / 外设驱动层：** 面向用户或外设驱动提供 DMA 传输服务。
- **RT-Thread DMA 框架层：** 提供统一的通道请求、配置、传输接口，屏蔽底层硬件差异。
- **设备驱动层：** 实现 DMA 控制器的寄存器操作、通道管理和中断处理。
- **物理层：** DMA 控制器硬件。

### 源码结构介绍
驱动源码位于 esos/bsp/spacemit/drivers/dma 目录，主要文件如下：

```bash
esos/bsp/spacemit/drivers/dma
.
|-- dma_k1x.c            # DMA 控制器核心驱动
|-- dma_test_k1x.c       # DMA 内存到内存自测工具
|-- dma_k1_spi_test.c    # SPI + DMA 联合测试工具
`-- SConscript
```

## 关键特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 16 通道 | 最多支持 16 个独立 DMA 通道并行工作 |
| 支持 MEM_TO_MEM | 内存到内存传输 |
| 支持 MEM_TO_DEV | 内存到外设传输 |
| 支持 DEV_TO_MEM | 外设到内存传输 |
| 支持多种总线宽度 | 1 / 2 / 4 字节（8 / 16 / 32 bit） |
| 支持多种 burst 大小 | 8 / 16 / 32 / 64 字节 |
| 支持中断回调 | 传输完成后通过中断触发用户回调 |
| 支持地址窗口重映射 | K3 平台通过 remap 寄存器映射 DMA 地址窗口，当前实现固定映射到 4G 段（地址高 32 位必须为 1） |
| 最大单次传输长度 | 16MB（24 bit 计数器，0xFFFFFF 字节），且受 256MB 窗口边界约束 |

## 配置介绍

1. `BSP_USING_DMA`：启用 DMA 驱动

```bash
config BSP_USING_DMA
    bool "Enable DMA"
    default n
```

2. `BSP_DMA_TEST`：启用 DMA 内存到内存自测工具

```bash
if BSP_USING_DMA
    config BSP_DMA_TEST
        bool "Enable dma test driver"
        default n
endif
```

3. `BSP_SPI_DMA_TEST`：启用 SPI + DMA 联合测试工具

```bash
if BSP_USING_SPI && BSP_USING_DMA
    config BSP_SPI_DMA_TEST
        bool "Enable spi dma test driver"
        default n
endif
```

> **注：** `dma_k1_spi_test` 需要同时具备 SPI 与 DMA 运行环境。仅打开该选项但未启用 SPI 控制器时，命令会因找不到 SPI 设备而无法完成测试。

### DTS 属性说明

| 属性 | 说明 |
| :-----| :----|
| `compatible` | `"spacemit,k1x-rdma"` |
| `#dma-channels` | DMA 通道数量（默认 16） |
| `max-burst-size` | 最大 burst 大小（8 / 16 / 32 / 64，默认 64） |

### 其他模块使用 DMA 的配置

外设驱动（如 SPI）通过 DTS 中的 `dmas` 和 `dma-names` 属性绑定 DMA 通道，并通过握手号指定与外设的硬件握手信号。

**握手号说明**

握手号由 `DMA_RSSR(n)` 寄存器配置，用于选择 DMA 通道与哪个外设的 TX/RX 请求信号相连。以 SPI0 为例，握手号分配如下：

| 外设 | TX 握手号 | RX 握手号 |
| :-----| :----|:----|
| SPI0 | 4 | 5 |

**DTS 配置示例（SPI0 使用 DMA）**

`#dma-cells = <2>`，每个 `dmas` 条目格式为 `<&rdma 握手号 1>`：

```dts
&rspi0 {
    dmas = <&rdma 5 1
            &rdma 4 1>;
    dma-names = "rx", "tx";
};
```

## 示例使用

在小核串口可以通过测试工具进行基本功能测试。

### DMA 内存到内存测试

命令格式：
```bash
dma_k1x_test [mode] [channel] [width_bytes] [burst_bytes] [count]
```

参数说明：

| 参数 | 说明 | 默认值 |
| :-----| :----|:----|
| mode | 0=顺序测试（16 通道依次跑）；1=并行测试（16 通道同时跑）；2=单通道测试 | 0 |
| channel | mode=2 时指定通道号（0~15），其他模式忽略 | 0 |
| width_bytes | 总线宽度：1 / 2 / 4 字节 | 4 |
| burst_bytes | burst 大小：8 / 16 / 32 / 64 字节 | 32 |
| count | 测试重复次数 | 1 |

示例：

1. 16 通道顺序测试
```bash
dma_k1x_test 0
```

2. 16 通道并行测试
```bash
dma_k1x_test 1
```

3. 单通道测试（通道 3，2 字节宽度，16 字节 burst，重复 5 次）
```bash
dma_k1x_test 2 3 2 16 5
```

### SPI + DMA 联合测试

```bash
dma_k1_spi_test
```
该命令会依次执行：读取 SPI Flash JEDEC ID → 读取 256 字节 → 写入测试数据 → 回读验证。

## 应用开发

DMA 驱动对接了 RT-Thread DMA 框架，外设驱动或应用可通过标准接口使用 DMA 传输。

- 请求 DMA 通道

```c
struct rt_dma_chan *chan = rt_dma_chan_request(dev, NULL);
```

- 配置 DMA 传输参数

```c
struct rt_dma_slave_config config;

config.direction      = RT_DMA_MEM_TO_MEM;  /* 或 RT_DMA_MEM_TO_DEV / RT_DMA_DEV_TO_MEM */
config.src_addr       = src_phys_addr;
config.dst_addr       = dst_phys_addr;
config.src_addr_width = RT_DMA_SLAVE_BUSWIDTH_4_BYTES;
config.dst_addr_width = RT_DMA_SLAVE_BUSWIDTH_4_BYTES;
config.src_maxburst   = 32;
config.dst_maxburst   = 32;

rt_dma_chan_config(chan, &config);
```

- 准备内存到内存传输

```c
struct rt_dma_slave_transfer transfer;

transfer.src_addr   = src_phys_addr;
transfer.dst_addr   = dst_phys_addr;
transfer.buffer_len = len;

rt_dma_prep_memcpy(chan, &transfer);
```

- 设置回调并启动传输

```c
chan->callback = my_dma_callback;
rt_dma_chan_start(chan);
```

- 停止传输并释放通道

```c
rt_dma_chan_stop(chan);
rt_dma_chan_release(chan);
```

## FAQ

### DMA 传输超时
当测试出现超时，串口会打印如下 log：
```
[dma_k1x_test:430] seq: timeout on ch0: -1
```
可能原因：

**1. DMA 控制器未初始化**

检查 `BSP_USING_DMA` 是否启用，DTS 中 DMA 节点 status 是否为 `okay`。

**2. 中断未正常触发**

DMA 传输完成依赖中断通知，检查中断号配置是否正确，中断处理函数是否正常注册。

**3. 地址窗口越界**

K3 平台 DMA 引擎仅能寻址 256MB 窗口（0x4000_0000~0x4FFF_FFFF），通过 remap 寄存器映射到实际物理地址。如果源地址或目标地址超出窗口范围，驱动会打印：
```
DMA addr outside 256MB window: off_src=0x... off_dst=0x... len=0x...
DMA xfer crosses window: off_src=0x... off_dst=0x... len=0x...
```

**4. 地址高位不满足 remap 约束**

当前实现要求 DMA 源/目标地址高 32 位为 1（4G 段）；否则会打印：
```
DMA remap only supports upper==1 (4G segment): src=0x... dst=0x...
```

### DMA 数据校验失败
当测试出现如下 log：
```
[dma_k1x_test:448] seq: ch0 data mismatch.
```
可能原因：

**1. Cache 一致性问题**

DMA 传输前需要对源缓冲区执行 cache flush，传输完成后需要对目标缓冲区执行 cache invalidate。示例：
```c
rt_hw_cpu_dcache_ops(RT_HW_CACHE_FLUSH, src, len);       /* 传输前 */
rt_hw_cpu_dcache_ops(RT_HW_CACHE_FLUSH, dst, len);       /* 传输前 */
/* ... DMA 传输 ... */
rt_hw_cpu_dcache_ops(RT_HW_CACHE_INVALIDATE, dst, len);  /* 传输后 */
```

**2. burst 配置不匹配**

burst 大小必须为 8 / 16 / 32 / 64 之一，其他值会导致配置失败。

### 通道分配失败
当出现 `no free dma channel`，说明 16 个通道已全部被占用，需要先释放不再使用的通道。
