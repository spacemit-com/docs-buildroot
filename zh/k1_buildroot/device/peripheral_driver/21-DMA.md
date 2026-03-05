# DMA

介绍 DMA 控制器的配置方法及外设通过 DMA-Slave 使用 DMA 的方式。

## 模块介绍  

**DMA（Direct Memory Access，直接内存访问）** 是一种无需 CPU 直接控制的高速数据传输方式。它通过硬件建立起源地址与目标地址之间的数据通路，从而提高系统效率。

本模块为 DMA Controller（即 DMA Master），主要负责调度 DMA Channel 资源，实现数据搬运任务。

### 功能介绍  

![](static/dma.JPEG)

通过 DMA 框架和 K1 平台的 DMA 控制器驱动，实现以下传输方式：

- 内存到内存
- 内存到外设
- 外设到内存

并支持如下三种传输形式：

- 内存搬运
- 散列表传输
- 环形 Buffer 传输

### 源码结构介绍

DMA 控制器相关源码位于 `drivers/dma` 目录下：  

```  
drivers/dma  
|--dmaengine.c dmaengine.h      #内核 DMA 框架代码
|--dmatest.c         #内核 DMA 测试代码
|--mmp_pdma_k1x.c          #K1 DMA 控制器驱动  
```  

## 关键特性  

### 特性

- 支持三种方向传输: 内存到内存，内存到外设，外设到内存
- 16 路 channel 可用
- 支持 Burst 长度：8 / 16 / 32 / 64
- 单次传输支持最大 8191 Bytes

### 性能参数

DMA 进行 内存到内存 方向的数据搬运的极限速度为 **220MB/s**

**测试方法：** 可使用 `dmatest.c` 进行测试，具体工具使用方法见 本文的 Debug 章节

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG 配置

`CONFIG_DMADEVICES`：为内核平台 DMA 框架提供支持，开启内核 DMA 框架支持，应为 `Y`

```
Symbol: DMADEVICES [=y]
Device Drivers
      -> DMA Engine support (DMADEVICES [=y])
```

在支持平台层 DMA 框架后，配置 `CONFIG_MMP_PDMA_SPACEMIT_K1X` 为 `Y`，启用 K1 DMA 控制器驱动

```
Symbol: MMP_PDMA_SPACEMIT_K1X [=y]
      ->Spacemit mmp_pdma support (MMP_PDMA_SPACEMIT_K1X [=y])
```

### DTS 配置

DMA 的使用方法是 **选择一个通道** 并 **指定起始地址和目标地址**，如内存到内存，内存到外设等。

本节介绍：
- 在 DTS 中使能 DMA 控制器
- 为具体设备（如 UART）配置 DMA 支持

#### DMA 控制器配置示例

可参考 Linux 源码中的 `arch/riscv/boot/dts/spacemit/k1-x.dtsi` 文件，里面已包含完整的 DMA 控制器节点配置。示例如下：

```dts
    pdma0: pdma@d4000000 {
  compatible = "spacemit,pdma-1.0";
  reg = <0x0 0xd4000000 0x0 0x4000>;
  interrupts = <72>;
  interrupt-parent = <&intc>;
  clocks = <&ccu CLK_DMA>;
  resets = <&reset RESET_DMA>;
  #dma-cells= <2>;
  #dma-channels = <16>;
  max-burst-size = <64>;
  reserved-channels = <15 45>;
  power-domains = <&power K1X_PMU_BUS_PWR_DOMAIN>;
  clk,pm-runtime,no-sleep;
  cpuidle,pm-runtime,sleep;
  interconnects = <&dram_range4>;
  interconnect-names = "dma-mem";
  status = "ok";
 };
```

此节点配置了 DMA 的时钟复位资源，通道数，最大 burst 数量，并将通道 15 预留给握手号为 45 号的设备

#### DMA-Slave 使用 DMA 的配置示例

这里以 UART0 为例，在 UART0 的节点下添加下述属性，其中宏 `DMA_UART0_RX` 和 `DMA_UART0_TX` 在 `include/dt-bindings/dma/k1x-dmac.h` 文件里定义

```dts
 dmas = <&pdma0 DMA_UART0_RX 1
   &pdma0 DMA_UART0_TX 1>;
 dma-names = "rx", "tx";
```

## 接口介绍

### API 介绍

Linux 内核实现了申请 DMA 通道，配置 DMA 传输，准备资源，启动传输等接口。

常用：

- 申请通道
```
struct dma_chan *dma_request_chan(struct device *dev, const char *name)
```

- 配置通道参数，如传输宽度，数据量，起始/目标地址
```
static inline int dmaengine_slave_config(struct dma_chan *chan,
                                          struct dma_slave_config *config)
```

- 下述三个接口实现了 DMA 开始传输前的准备资源
```
dmaengine_prep_dma_memcpy
dmaengine_prep_slave_sg
dmaengine_prep_dma_cyclic
```

- 将描述符加入到传输任务的链表里
```
static inline dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc)
```

- 开始传输
```
static inline void dma_async_issue_pending(struct dma_chan *chan)
```

- 释放通道
```
static inline void dma_release_channel(struct dma_chan *chan)
```

- 停止传输，如音频播放时暂停
```
static inline int dmaengine_terminate_all(struct dma_chan *chan)
```

## 测试介绍

由于在内存到设备或设备到内存的数据通路工作中需要外设的驱动配合，如 UART0 在发送数据时要将数据通过 DMA 从内存搬运到 UART TX FIFO 里，这里 DMA 需要 UART 来控制启动。所以一般在测试时我们采用内存到内存的数据通路，可直接使用内核自带的 `dmatest.c` 程序。

```
echo dma0chan8 > /sys/module/dmatest/parameters/channel  #选择一个没有使用的通道
echo 1 > /sys/module/dmatest/parameters/iterations  #设置传输次数，这里以1为例
echo 4096 > /sys/module/dmatest/parameters/transfer_size #设置传输数据量大小(不大于16k)
echo 1 > /sys/module/dmatest/parameters/run   #开始传输
```

> **注：** 使用前请确保内核启用了 ·CONFIG_DMATEST·，编译生成新固件后启动系统执行测试。

## FAQ
