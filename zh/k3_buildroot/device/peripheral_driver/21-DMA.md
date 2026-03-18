# DMA

介绍 K3 平台 DMA 的组成、驱动实现、DTS 配置方式、外设侧接入方法以及常见调试思路。

## 模块介绍

DMA（Direct Memory Access）用于在外设与内存、内存与内存之间搬运数据，尽量减少 CPU 逐字节参与，提高吞吐并降低中断负担。

对 K3 来说，DMA 不是一个单点能力，而是分成了几类：

- 面向通用外设的 PDMA；
- DTS 中的 `udma` 占位/控制节点；
- 面向 AI/专用加速域的 `spacemit-ai-dmac`；
- 外设作为 DMA consumer，通过 `dmas` / `dma-names` 接入。

用户最常遇到的问题其实是这些：

- K3 通用外设到底挂的是哪路 DMA；
- `dmas = <&pdma ...>` 里的参数表示什么；
- 一个设备为什么要同时写 `dmas` 和 `dma-names`；
- DMA 没生效时，是驱动没开、DTS 没配、还是 consumer 退回了 PIO。

### 功能介绍

![](static/dma.png)

Linux DMAEngine 体系通常包括：

1. **DMA controller driver**  
   提供 channel 分配、描述符提交、启动/停止、中断处理；
2. **DMA client / consumer**  
   各外设驱动通过 DMAEngine API 申请通道并发起搬运；
3. **DTS 绑定关系**  
   用 `dmas` / `dma-names` 把 consumer 和 controller 连起来；
4. **用户调试接口**  
   通过 dmesg、驱动日志、功能表现判断是否真正进入 DMA 路径。

K3 当前 SDK 里，**最核心、最通用的一条 DMA 路线是 PDMA**：

- controller 节点：`pdma@d4000000`
- compatible：`spacemit,k1-pdma`
- 驱动：`drivers/dma/mmp_pdma_spacemit.c`

也就是说，虽然平台是 K3，但通用外设 DMA 这条链路仍然复用了 `k1-pdma` 这套 binding / driver 命名。

## 源码结构介绍

K3 DMA 相关代码主要位于：

```text
linux-6.18/
|-- drivers/dma/
|   |-- dmaengine.c
|   |-- virt-dma.c
|   |-- mmp_pdma_spacemit.c        # SpacemiT PDMA 主驱动
|   |-- udma.c                     # UDMA 相关实现
|   |-- k3dma.c                    # 其他通用 DMA 组件（内核自带）
|   `-- dmatest.c                  # DMAEngine 测试模块
|-- Documentation/devicetree/bindings/dma/
|   |-- spacemit,k1-pdma.yaml      # K3 当前实际用到的 PDMA binding
|   `-- k3dma.txt                  # 内核通用 k3dma binding（非 SpacemiT 这条主线）
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    `-- k3*.dts
```

本轮和 K3 最相关的几个对象是：

- `drivers/dma/mmp_pdma_spacemit.c`
- `Documentation/devicetree/bindings/dma/spacemit,k1-pdma.yaml`
- `arch/riscv/boot/dts/spacemit/k3.dtsi`

### `mmp_pdma_spacemit.c` 做了什么

从源码可见，这个驱动实现的是一套标准 DMAEngine controller，支持：

- `DMA_SLAVE`
- `DMA_MEMCPY`
- `DMA_CYCLIC`
- `DMA_PRIVATE`

驱动里直接可以看到：

```c
dma_cap_set(DMA_SLAVE, pdev->device.cap_mask);
dma_cap_set(DMA_MEMCPY, pdev->device.cap_mask);
dma_cap_set(DMA_CYCLIC, pdev->device.cap_mask);
dma_cap_set(DMA_PRIVATE, pdev->device.cap_mask);
```

还实现了这些典型接口：

- `device_prep_memcpy`
- `device_prep_slave_sg`
- `device_prep_dma_cyclic`
- `device_issue_pending`
- `device_pause`
- `device_terminate_all`
- `device_tx_status`

这说明它不是只给某一个外设专用，而是标准的 **通用 DMA controller**。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| 通用外设 DMA 主线是 PDMA | K3 当前 DTS 中大量外设都挂 `&pdma` |
| 兼容串仍是 `spacemit,k1-pdma` | K3 通用 PDMA 复用 K1 命名体系 |
| 支持 DMAEngine 标准能力 | `DMA_SLAVE` / `DMA_MEMCPY` / `DMA_CYCLIC` |
| 通道数可配 | DTS 通过 `#dma-channels` 指定，当前默认 16 |
| consumer 通过 request number 接入 | `#dma-cells = <1>`，参数表示外设 DMA request number |
| 支持保留通道 | 驱动可解析 `reserved-channels` |
| 支持 `max-burst-size` | DTS 可控制 burst 配置 |
| 存在专用 AI DMAC | `spacemit-ai-dmac` 面向 AI/DCIU 域，不等同于通用 PDMA |

### K3 当前通用 PDMA 节点

`k3.dtsi` 里可见两个 PDMA 节点：

#### 1. 主 PDMA

```dts
pdma: pdma@d4000000 {
	compatible = "spacemit,k1-pdma";
	reg = <0x0 0xd4000000 0x0 0x4000>;
	interrupt-parent = <&saplic>;
	interrupts = <72 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&syscon_apmu CLK_APMU_DMA>;
	resets = <&syscon_apmu RESET_APMU_DMA>;
	#dma-cells = <1>;
	#dma-channels = <16>;
	max-burst-size = <64>;
	status = "okay";
};
```

#### 2. 第二个 PDMA 控制器

```dts
pdma1: pdma@f0600000 {
	compatible = "spacemit,k1-pdma";
	reg = <0x0 0xf0600000 0x0 0x4000>;
	interrupt-parent = <&saplic>;
	interrupts = <150 IRQ_TYPE_LEVEL_HIGH>;
	#dma-cells = <1>;
	#dma-channels = <16>;
	max-burst-size = <64>;
	status = "disabled";
};
```

所以从当前默认 DTS 来看：

- 主用的是 `pdma`
- `pdma1` 目前是关闭的

### `#dma-cells = <1>` 的含义

`spacemit,k1-pdma.yaml` 里已经写明：

```yaml
'#dma-cells':
  const: 1
  description:
    The DMA request number for the peripheral device.
```

也就是说，consumer 侧这样写：

```dts
dmas = <&pdma DMA_SSP0_TX &pdma DMA_SSP0_RX>;
```

其中 `DMA_SSP0_TX` / `DMA_SSP0_RX` 不是 channel index，而是 **外设 DMA request number**。

这点很重要，别把它误认为“直接选第几个物理 channel”。

## 配置介绍

### CONFIG 配置

K3 通用 PDMA 驱动实际文件是：

```text
drivers/dma/mmp_pdma_spacemit.c
```

虽然本轮没有专门展开对应 Kconfig 片段，但从源码功能与 DTS 绑定关系看，K3 通用 DMA 文档当前重点应该放在：

- `mmp_pdma_spacemit.c` 的实际实现
- `spacemit,k1-pdma.yaml` 的 DTS 约束
- 各外设 consumer 的接法

实际编译时，一般还会涉及：

- `CONFIG_DMADEVICES`
- `CONFIG_DMA_ENGINE`
- SpacemiT PDMA 对应选项
- `CONFIG_DMATEST`（如果需要做 DMAEngine 测试）

> 这里不建议在文档里硬填未核实的具体 Kconfig 名称，优先以驱动、binding 和 DTS 为准。

### DTS 配置

#### 1. controller 侧

PDMA 控制器节点核心字段包括：

| 属性 | 作用 |
| :--- | :--- |
| `compatible = "spacemit,k1-pdma"` | 绑定 PDMA 控制器驱动 |
| `reg` | 控制器寄存器地址 |
| `interrupts` | DMA 中断 |
| `clocks` | DMA 控制器时钟 |
| `resets` | DMA 控制器 reset |
| `#dma-cells = <1>` | consumer 参数个数 |
| `#dma-channels = <16>` | 可用 channel 数 |
| `max-burst-size = <64>` | 最大 burst 大小 |

#### 2. consumer 侧通用写法

外设一般这样接 DMA：

```dts
xxx: device@addr {
	dmas = <&pdma REQ_TX>, <&pdma REQ_RX>;
	dma-names = "tx", "rx";
};
```

或者只有单向通道时：

```dts
xxx: device@addr {
	dmas = <&pdma REQ_TX>;
	dma-names = "tx-dma";
};
```

### 典型 K3 consumer 示例

#### 1. SPI

```dts
spi0: spi@d4040000 {
	dmas = <&pdma DMA_SSP0_TX
		&pdma DMA_SSP0_RX>;
	dma-names = "tx", "rx";
};
```

`spi1`、`spi3` 也是同样模式。

而 `spi2` 当前 DTS 里相关 DMA 配置还是注释状态：

```dts
// dmas = <&pdma 19
// 	&pdma 20>;
// dma-names = "rx", "tx";
```

这类细节就很适合写进文档：**不是所有控制器默认都已经开 DMA**。

#### 2. QSPI

K3 QSPI 的用法更特殊：

```dts
qspi: spi@d420c000 {
	spacemit,qspi-tx-dma = <1>;
	spacemit,qspi-rx-dma = <1>;
	dmas = <&pdma 45>;
	dma-names = "tx-dma";
};
```

它不仅有标准 `dmas` / `dma-names`，还额外用了平台私有属性：

- `spacemit,qspi-tx-dma`
- `spacemit,qspi-rx-dma`

而且从当前 DTS 看只给了一路：

- `dmas = <&pdma 45>`
- `dma-names = "tx-dma"`

所以 QSPI 这条链路不能简单按 SPI/I2S 那种双向通道模板去理解，必须按它自己的驱动逻辑看。

#### 3. I2S / Audio

K3 的多个 I2S 节点都挂了 PDMA，例如：

```dts
i2s1: i2s1@d4026800 {
	dmas = <&pdma 24>, <&pdma 23>;
	dma-names = "rx", "tx";
};
```

类似的还有：

- `v2d`
- `i2s2`
- `i2s3`
- `i2s4`
- `i2s5`

这说明 PDMA 在 K3 上不仅服务 SPI/UART 一类低速外设，也广泛参与音频/多媒体数据搬运。

## AI DMAC 说明

除了通用 PDMA，`k3.dtsi` 里还定义了 8 路 AI DMAC：

- `ai_dmac0 ~ ai_dmac7`

它们的节点形态例如：

```dts
ai_dmac0: hdma0@d8804000 {
	compatible = "spacemit-ai-dmac";
	reg = <0x0 0xd8804000 0x0 0x1000>;
	interrupts = <7 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&syscon_dciu CLK_DCIU_HDMA>;
	resets = <&syscon_dciu RESET_DCIU_HDMA>,
		 <&syscon_dciu RESET_DCIU_AXIDMA0>;
	reset-names = "bus", "dma";
	dma-coherent;
	status = "okay";
};
```

从这些节点可以确定：

- 它们位于 DCIU / AI 相关域；
- 有独立中断、clock、双 reset；
- 标了 `dma-coherent`；
- 它们和通用 `pdma` 不是同一种控制器。

所以写业务文档时，**不要把 AI DMAC 和通用外设 PDMA 混为一谈**。

## 驱动实现细节

### 1. PDMA 支持的传输类型

`mmp_pdma_spacemit.c` 当前明确支持：

- slave scatter-gather
- memcpy
- cyclic DMA

其中 cyclic 对音频特别重要。

### 2. 支持 1/2/4 字节总线宽度

驱动里配置了：

```c
DMA_SLAVE_BUSWIDTH_1_BYTE |
DMA_SLAVE_BUSWIDTH_2_BYTES |
DMA_SLAVE_BUSWIDTH_4_BYTES;
```

### 3. 支持 burst 配置

驱动里根据 `max_burst_size` 和 slave config 转成：

- `DCMD_BURST8`
- `DCMD_BURST16`
- `DCMD_BURST32`
- `DCMD_BURST64`

并且 DTS controller 节点允许配置：

```dts
max-burst-size = <64>;
```

### 4. 支持 reserved channel

驱动会解析：

```dts
reserved-channels
```

把某些通道和 request number 预留出来。

这对于一些平台私有固定用途、或者不希望被通用分配逻辑抢走的场景很重要。

### 5. 支持 64-bit DMA 地址

源码里可以看到：

```c
#ifdef CONFIG_SPACEMIT_PDMA_SUPPORT_64BIT
	dma_set_mask(pdev->dev, DMA_BIT_MASK(64));
#endif
```

同时描述符结构也扩展了高 32 位地址字段。

所以如果平台开启了这项支持，PDMA 可以覆盖 64-bit 地址空间场景。

## 使用与调试介绍

### 1. 先判断外设是否真的走了 DMA

这一步最重要。

很多外设即使 DTS 写了 `dmas`，驱动也可能因为某些条件退回 PIO。判断方法通常是：

- 看 dmesg 是否打印申请 DMA channel 成功/失败；
- 看吞吐和 CPU 占用是否符合 DMA 特征；
- 必要时在 consumer 驱动里确认 `dma_request_chan()` 是否成功。

### 2. 优先检查 DTS 三件套

DMA consumer 侧先检查：

- `dmas`
- `dma-names`
- 对应 controller 节点是否 `status = "okay"`

这一步在 K3 上很有效，因为大量问题都不是 controller 驱动本身坏了，而是 consumer 接法不匹配。

### 3. 检查 `dma-names` 是否和驱动预期一致

例如：

- SPI 常见是 `"tx", "rx"`
- QSPI 当前用的是 `"tx-dma"`

名字不对，consumer 驱动就可能拿不到通道。

### 4. QSPI 这类带私有 DMA 属性的设备要单独看

K3 QSPI 不是简单一套通用模板，它额外还要看：

- `spacemit,qspi-tx-dma`
- `spacemit,qspi-rx-dma`

所以如果 QSPI 没走 DMA，不要只盯 `dmas`，还要把私有属性一起核对。

### 5. 音频类优先看 cyclic DMA 是否跑起来

I2S / PCM 场景里，真正关键的不只是“有没有 DMA”，而是：

- 有没有进入 cyclic DMA 路径；
- period 中断是否稳定；
- 是否出现 underrun / overrun。

因为音频类 DMA 和简单 memcpy / SPI 搬运的关注点不一样。

### 6. controller 侧排查项

如果怀疑 PDMA 控制器本身没起来，优先检查：

- `pdma@d4000000` 是否 `status = "okay"`
- `clocks = <&syscon_apmu CLK_APMU_DMA>` 是否有效
- `resets = <&syscon_apmu RESET_APMU_DMA>` 是否释放
- 中断是否注册成功

## 测试介绍

### 1. 基础存在性检查

先看设备树和启动日志，确认：

- `pdma` 已经 probe；
- 目标 consumer 没有报 `failed to request dma` 一类错误。

### 2. 外设功能压测

对用户来说，最实用的 DMA 测试往往不是直接测 controller，而是测外设场景：

- SPI 大包收发
- I2S 长时间播放/录音
- QSPI 大块读写

观察：

- 吞吐
- CPU 占用
- 是否丢包 / underrun / overrun

### 3. DMAEngine 通用测试

如果内核使能了 `dmatest`，也可以用 `drivers/dma/dmatest.c` 做 DMAEngine 层验证。

不过对 K3 文档来说，更建议把它作为“补充测试手段”，而不是唯一依据，因为实际用户更关心的是**外设链路里的 DMA 是否真正可用**。

## Debug 介绍

### 1. consumer 明明配了 `dmas`，为什么还是像 PIO？

常见原因：

- `dma-names` 不匹配驱动预期；
- request number 配错；
- controller 没起来；
- 该外设驱动本身在某些传输模式下就不会走 DMA；
- 通道申请失败后 silently fallback 到 PIO。

### 2. 为什么 `dmas = <&pdma N>` 里的 `N` 不是物理通道号？

因为根据 binding，`#dma-cells = <1>` 传的是 **DMA request number**，不是“第 N 个 channel”。

channel 最终由 controller 驱动按映射和分配逻辑处理。

### 3. 为什么 `spi2` 没用 DMA，但 `spi0/1/3` 用了？

因为当前 DTS 就是这么写的。`spi2` 的 DMA 配置在 `k3.dtsi` 里仍然是注释状态，说明它当前默认并没有按其它 SPI 控制器那样接通 DMA。

### 4. 为什么 K3 文档里要同时提 PDMA 和 AI DMAC？

因为它们确实都存在，但用途不同：

- PDMA：通用外设 DMA 主线
- AI DMAC：AI/DCIU 专用域 DMA

如果不分开写，后面用户看 DTS 很容易混淆。

## FAQ

### 1. K3 当前最常见的 DMA controller 是哪个？

是：

```text
pdma@d4000000
compatible = "spacemit,k1-pdma"
```

### 2. 为什么 K3 上还是 `k1-pdma`？

这是当前 SDK 的实际复用关系。和前面的 I2C 一样，**平台是 K3，但部分通用 IP 仍沿用 K1 的 compatible / binding / driver 命名**。

### 3. DMA 文档里最该记住哪条经验？

**先从 consumer 节点看 `dmas` / `dma-names`，不要一上来就只盯 controller 驱动。**

因为 K3 上 DMA 问题最常见的入口，不是 controller 坏了，而是 consumer 接法和驱动预期没对齐。
