# DMA

This document describes DMA controller configuration and how peripherals use DMA through DMA-Slave.

## Module Overview

**DMA (Direct Memory Access)** is a high-speed data transfer method that does not require direct CPU control. It establishes a hardware data path between the source and destination addresses, thereby improving system efficiency.

This module covers the DMA controller (that is, the DMA master), which is mainly responsible for scheduling DMA channel resources and carrying out data-transfer tasks.

### Functional Overview

![](static/dma.JPEG)

By combining the DMA framework with the K3 platform DMA controller driver, the following transfer directions are supported:

- Memory-to-memory
- Memory-to-peripheral
- Peripheral-to-memory

It also supports the following three transfer forms:

- Memory copy
- Scatter-gather transfer
- Ring buffer transfer

### Source Code Structure

DMA controller-related source code is located in the `drivers/dma` directory:

```
drivers/dma
|--dmaengine.c dmaengine.h      # Kernel DMA framework code
|--dmatest.c                    # Kernel DMA test code
|--mmp_pdma.c                   # K3 PDMA controller driver
```

The PDMA driver file used on K3 is `mmp_pdma.c`, with `compatible = "spacemit,k1-pdma"`.

## Key Features

### Features

- Supports three transfer directions: memory-to-memory, memory-to-peripheral, and peripheral-to-memory
- Two PDMA controllers: `pdma` (16 channels, enabled by default) and `pdma1` (16 channels, disabled by default, used for the secure domain)
- Supported burst lengths: 8 / 16 / 32 / 64
- Maximum transfer size per descriptor: 8191 bytes
- Supports 64-bit addressing (LPAE, Long Physical Address Extension), allowing access to memory above 4 GB
- Supports data widths of 1 / 2 / 4 bytes

## Configuration

It mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

`CONFIG_DMADEVICES`: provides support for the kernel DMA framework. To support the DMA driver, this option should be set to `Y`.

```
Symbol: DMADEVICES [=y]
Device Drivers
      -> DMA Engine support (DMADEVICES [=y])
```

After enabling the platform DMA framework, set `CONFIG_MMP_PDMA` to `Y` to enable the K3 PDMA controller driver.

```
Symbol: MMP_PDMA [=y]
Device Drivers
      -> DMA Engine support (DMADEVICES [=y])
            -> MMP PDMA support (MMP_PDMA [=y])
```

The above options and `CONFIG_DMATEST` are all enabled in the K3 SDK defconfig.

### DTS Configuration

Two things need to be completed in DTS: first, enable the PDMA controller node; second, declare `dmas` / `dma-names` on peripheral nodes that use DMA. This allows the driver to request the corresponding channels at runtime. Examples of both are given below.

#### DMA Controller Configuration Example

The Linux source file `arch/riscv/boot/dts/spacemit/k3.dtsi` already contains the DMA controller node. The following is an example written according to the DT binding recommendations:

```c
pdma: pdma@d4000000 {
    compatible = "spacemit,k1-pdma";
    reg = <0x0 0xd4000000 0x0 0x4000>;
    interrupt-parent = <&saplic>;
    interrupts = <72 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&syscon_apmu CLK_APMU_DMA>;
    resets = <&syscon_apmu RESET_APMU_DMA>;
    #dma-cells= <1>;
    dma-channels = <16>;
    status = "okay";
};
```

This node declares the DMA register address, clock, reset resources, and the number of channels (16).

K3 also has a secure-domain PDMA controller, `pdma1` (address `0xf0600000`), which is disabled by default in the current DTS:

```c
pdma1: pdma@f0600000 {
    compatible = "spacemit,k1-pdma";
    reg = <0x0 0xf0600000 0x0 0x4000>;
    interrupt-parent = <&saplic>;
    interrupts = <150 IRQ_TYPE_LEVEL_HIGH>;
    #dma-cells= <1>;
    dma-channels = <16>;
    status = "disabled";
};
```

> **Note:** On K3, `#dma-cells = <1>`, meaning that only the DMA request number needs to be specified. Therefore, when a device node references DMA resources, it should use the format **one phandle + one request number**, such as `dmas = <&pdma 64>` or `dmas = <&pdma DMA_SSP0_TX>`.

#### DMA-Slave Configuration Example

Taking SPI0 as an example, add the following properties under the SPI0 node. The macros `DMA_SSP0_TX` and `DMA_SSP0_RX` are defined in `include/dt-bindings/dma/k3-dmac.h`:

```c
dmas = <&pdma DMA_SSP0_TX
        &pdma DMA_SSP0_RX>;
dma-names = "tx", "rx";
```

#### DMA Channel Mapping

The DMA request numbers for K3 PDMA are defined in `include/dt-bindings/dma/k3-dmac.h`. The main mappings are as follows:

| Peripheral | TX Request Number | RX Request Number |
|------|-----------|-----------|
| UART0 | 3 | 4 |
| UART2 | 5 | 6 |
| UART3 | 7 | 8 |
| UART4 | 9 | 10 |
| UART5 | 25 | 26 |
| UART6 | 27 | 28 |
| UART7 | 29 | 30 |
| UART8 | 31 | 32 |
| UART9 | 33 | 34 |
| UART10 | 53 | 54 |
| I2C0 | 11 | 12 |
| I2C1 | 13 | 14 |
| I2C2 | 15 | 16 |
| I2C4 | 17 | 18 |
| I2C5 | 35 | 36 |
| I2C6 | 37 | 38 |
| I2C8 | 41 | 42 |
| SSP3 | 19 | 20 |
| SSPA0 | 21 | 22 |
| SSPA1 | 23 | 24 |
| SSPA2 | 56 | 57 |
| SSPA3 | 58 | 59 |
| SSPA4 | 60 | 61 |
| SSPA5 | 62 | 63 |
| SSP0 | 64 | 65 |
| SSP1 | 66 | 67 |
| QSPI | 85 | 84 |
| CAN0 | — | 43 |
| CAN1 | — | 44 |
| CAN2 | — | 51 |
| CAN3 | — | 52 |

The channel mapping for `pdma1` (secure domain) is defined by the `DMA_SEC_*` macros in the same header file and is used for secure-domain peripherals, such as `SEC_UART1`, `SEC_SSP2`, and `SEC_I2C3`.

## Interface

### API

The Linux kernel provides interfaces for managing DMA operations, including requesting DMA channels, configuring DMA transfers, preparing resources, starting transfers, and more.

Commonly used APIs include:


- Request a DMA channel
```c
struct dma_chan *dma_request_chan(struct device *dev, const char *name)
```

- Configure channel parameters, such as transfer width, data size, and source/destination addresses
```c
static inline int dmaengine_slave_config(struct dma_chan *chan,
                                          struct dma_slave_config *config)
```

- The following three interfaces prepare resources before starting a DMA transfer
```c
dmaengine_prep_dma_memcpy
dmaengine_prep_slave_sg
dmaengine_prep_dma_cyclic
```

- Add the descriptor to the transfer task list
```c
static inline dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc)
```

- Start the transfer
```c
static inline void dma_async_issue_pending(struct dma_chan *chan)
```

- Release the DMA channel
```c
static inline void dma_release_channel(struct dma_chan *chan)
```

- Stop all transfers, for example when pausing audio playback
```c
static inline int dmaengine_terminate_all(struct dma_chan *chan)
```

## Testing

Since memory-to-device or device-to-memory data transfers require coordination with the peripheral driver—for example, when SPI transmits data, DMA must move the data from memory to the SPI TX FIFO, and SPI must trigger the DMA—testing usually focuses on memory-to-memory transfers. In this case, the kernel's built-in `dmatest.c` program can be used directly.


```
echo dma0chan8 > /sys/module/dmatest/parameters/channel  # Select an unused channel
echo 1 > /sys/module/dmatest/parameters/iterations  # Set the number of transfers; 1 is used as an example
echo 4096 > /sys/module/dmatest/parameters/transfer_size # Set the transfer size
echo 1 > /sys/module/dmatest/parameters/run   # Start the transfer
```

> **Note:** Before using this test, make sure the kernel option `CONFIG_DMATEST` is enabled. It is enabled by default in the K3 SDK defconfig.

## FAQ
