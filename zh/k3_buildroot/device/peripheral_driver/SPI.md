# SPI

介绍 K3 平台 SPI 控制器驱动的能力、设备树配置方法、典型 consumer 接入方式，以及调试与验证方法。

## 模块介绍

SPI（Serial Peripheral Interface）是一种同步串行总线，常用于连接 SPI NOR Flash、SPI NAND、ADC、DAC、TPM、安全芯片、显示控制器、传感器、EtherCAT 从站芯片等外设。SPI 通常由一个主控制器（Master）配合一个或多个从设备（Slave）组成，主控制器负责产生时钟并发起数据传输。

从客户使用角度看，SPI 文档最关心的通常不是“SPI 控制器内部寄存器长什么样”，而是以下问题：

- K3 上有哪些 SPI 控制器可用；
- 某个 SPI 外设应该挂在哪个控制器下面；
- 设备树节点怎么写；
- `compatible`、`reg`、`spi-max-frequency`、`mode` 这些参数怎么选；
- 控制器支持哪些能力，什么时候会走 DMA；
- 怎么验证板级连接和驱动是否工作正常。

K3 当前提供两类 SPI 控制器相关接口：

- **普通 SPI 控制器**：用于连接常规 SPI 外设；
- **QSPI 控制器**：更适合连接 SPI NOR 等高吞吐 Flash，单独由 QSPI 驱动支持。

本文档主要描述 **普通 SPI 控制器**；QSPI 在独立文档中介绍。

### 功能介绍

Linux SPI 软件栈主要分为三层：

1. **SPI Core**：负责统一注册控制器、匹配 SPI 设备与 SPI 设备驱动、调度数据传输；
2. **SPI 控制器驱动**：负责把 SoC SPI 控制器抽象为 `spi_controller`；
3. **SPI 设备驱动（consumer driver）**：具体外设驱动，如 SPI NOR、TPM、ADC、显示控制器等。

K3 SPI 控制器驱动实现的是第 2 层，它为上层 consumer 提供标准 Linux SPI 接口，因此客户在接入设备时，通常重点放在：

- 控制器节点是否启用；
- pinctrl 是否正确；
- SPI 外设子节点是否挂在正确的控制器下面；
- 外设的 `compatible`、片选、最大频率和模式是否与硬件一致。

### 源码结构介绍

K3 SPI 相关代码和描述主要位于以下位置：

```text
linux-6.18/
|-- drivers/spi/
|   |-- spi.c                           # SPI core 框架
|   |-- Kconfig                         # SPI 相关配置项
|   `-- spi-spacemit-k1.c              # K3 普通 SPI 控制器驱动
|-- Documentation/devicetree/bindings/spi/
|   `-- spi-controller.yaml            # SPI 控制器通用 binding 约束
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                         # K3 A-domain SPI 控制器节点
    |-- k3-rdomain.dtsi                 # K3 R-domain SPI 控制器节点
    `-- k3*.dts                         # 板级 consumer 使用示例
```

对比 K1 SDK 可见：

- K1 SDK 中驱动文件为 `drivers/spi/spi-k1x.c`；
- K3 SDK 中驱动文件为 `drivers/spi/spi-spacemit-k1.c`；
- K1 DTS 中控制器 `compatible = "spacemit,k1x-spi"`；
- K3 DTS 中控制器 `compatible = "spacemit,k3-spi"`。

说明 K3 SPI 文档不能简单照搬 K1，而应结合 K3 当前驱动和 DTS 实际用法来写。

## 关键特性

### 控制器能力

结合 `drivers/spi/spi-spacemit-k1.c` 可知，K3 SPI 控制器具备如下能力：

| 特性 | 说明 |
| :----- | :---- |
| 工作模式 | 当前驱动仅支持 **主机模式（Master）** |
| SPI 模式 | 支持 `CPOL` / `CPHA`，可组合为 SPI mode 0/1/2/3 |
| 回环测试 | 支持 `SPI_LOOP`，便于板级联调 |
| 位宽 | 支持 `bits_per_word = 4 ~ 32` |
| 频率范围 | 驱动限定范围约为 **6.25 KHz ~ 51.2 MHz** |
| DMA/PIO | 同时支持 **PIO** 和 **DMA** 两种传输路径 |
| DMA 对齐约束 | DMA 对齐要求 64B，超小包或超大包时可能退回 PIO |
| 片选数 | 当前驱动将 `num_chipselect` 固定为 **1** |
| 典型 consumer | SPI NOR、各类寄存器型 SPI 外设、自定义板级设备 |

### K3 驱动中的几个关键实现点

从驱动代码看，客户在使用时尤其需要关注以下几点：

1. **最大频率来源于控制器节点的 `spi-max-frequency`**  
   驱动在 probe 时会读取控制器节点上的 `spi-max-frequency`，若缺省或非法，则回退到默认值 `25600000`。

2. **每个 transfer 也可以带自己的 `speed_hz`**  
   SPI core 会在传输时根据设备和 transfer 的配置调整速度，但前提是不能超过控制器支持范围。

3. **DMA 不是所有传输都一定启用**  
   仅当：
   - 控制器成功申请到 `tx/rx` DMA 通道；
   - 传输长度足够大；
   - 长度不超过驱动上限；
   - DMA mapping 成功；
   时才会走 DMA。否则自动回退到 PIO。

4. **控制器当前只暴露 1 个片选**  
   驱动中 `host->num_chipselect = 1`，因此当前 DTS 使用时通常只挂一个片选设备，`reg` 一般写 `0`。

## 配置介绍

主要包括驱动使能配置、控制器节点配置和 consumer 节点配置。

### CONFIG 配置

K3 SPI 相关常用配置如下：

- `CONFIG_SPI`
- `CONFIG_SPI_MASTER`
- `CONFIG_SPI_K1`
- 与具体 consumer 对应的配置，例如：
  - `CONFIG_MTD_SPI_NOR`
  - `CONFIG_SPI_SPIDEV`（如 SDK 开启且允许）

菜单路径如下：

```text
Device Drivers
    SPI support
        SPI support (SPI [=y])
        SPI Master Controller Drivers
            Spacemit K1 SPI Controller (SPI_K1 [=y])
```

> 说明：Kconfig 名称虽然仍叫 `SPI_K1`，但在 K3 SDK 中该驱动同样用于 `spacemit,k3-spi` 控制器。

### DTS 配置

#### 1. 控制器节点

K3 的普通 SPI 控制器节点位于：

- `arch/riscv/boot/dts/spacemit/k3.dtsi`
- `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi`

A-domain 控制器包括：

- `spi0`
- `spi1`
- `spi2`
- `spi3`

R-domain 控制器包括：

- `rspi0`
- `rspi1`
- `rspi2`

典型控制器节点如下：

```dts
spi0: spi@d4040000 {
        compatible = "spacemit,k3-spi";
        reg = <0x0 0xd4040000 0x0 0x30>;
        dmas = <&pdma DMA_SSP0_TX
                &pdma DMA_SSP0_RX>;
        dma-names = "tx", "rx";
        interrupt-parent = <&saplic>;
        interrupts = <283 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&syscon_apbc CLK_APBC_SPI0>,
                 <&syscon_apbc CLK_APBC_SPI0_BUS>;
        clock-names = "func", "bus";
        resets = <&syscon_apbc RESET_APBC_SPI0>;
        spi-max-frequency = <51200000>;
        #address-cells = <1>;
        #size-cells = <0>;
        status = "disabled";
};
```

各属性含义如下：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | 固定为 `spacemit,k3-spi`，匹配 K3 SPI 控制器驱动 |
| `reg` | 控制器寄存器基址和长度 |
| `dmas` / `dma-names` | DMA 发送和接收通道，名称固定为 `tx`、`rx` |
| `interrupts` | SPI 控制器中断号 |
| `clocks` / `clock-names` | 功能时钟与总线时钟，名称为 `func`、`bus` |
| `resets` | SPI 控制器复位 |
| `spi-max-frequency` | 控制器支持的最大时钟频率上限 |
| `#address-cells` / `#size-cells` | 子设备地址格式，SPI 设备通常使用 `reg = <chip_select>` |
| `status` | 设为 `okay` 表示启用控制器 |

#### 2. pinctrl 配置

SPI 引脚通常至少包括：

- SCLK
- MOSI / TXD
- MISO / RXD
- CS / SS

客户接板时，最容易出问题的环节就是 **引脚复用和片选脚**，因此建议严格按以下步骤确认：

1. 根据原理图确认控制器实例，例如 SPI0 / SPI1 / SPI3；
2. 根据板级 pinctrl dtsi 确认该 SPI 的 pin group 名称；
3. 确认片选脚是否由控制器原生 CS 输出，还是经 GPIO 扩展；
4. 再启用控制器节点。

典型写法如下：

```dts
&spi0 {
        pinctrl-names = "default";
        pinctrl-0 = <&spi0_0_cfg>;
        status = "okay";
};
```

不同板型上 pinctrl 组名称可能不同，实际请以对应 `k3*.dts` / pinctrl dtsi 为准。

#### 3. consumer（SPI 从设备）节点配置

这部分是客户最关心的内容。控制器节点启用后，需要在其下面添加具体 SPI 设备子节点。

一个 SPI consumer 节点通常至少包含：

- `compatible`：匹配设备驱动；
- `reg`：片选号，当前 K3 普通 SPI 控制器一般用 `0`；
- `spi-max-frequency`：该设备可承受的最大 SPI 时钟；
- 视设备需要补充 mode、GPIO、中断、电源、reset 等属性。

##### 3.1 `compatible`

它决定该设备由哪个 Linux driver 接管。必须以 **设备真实型号的数据手册** 和 **内核现有驱动支持情况** 为准。

例如：

- SPI NOR：`jedec,spi-nor`
- 某些通用测试设备：`spidev`（仅限 SDK 允许时）
- 其他传感器/控制器：请查其对应 binding 文档

##### 3.2 `reg`

对 SPI 子设备来说，`reg` 表示 **chip select 编号**。在当前 K3 驱动实现里，由于控制器只暴露 1 个片选，因此一般写：

```dts
reg = <0>;
```

##### 3.3 `spi-max-frequency`

这是 consumer 侧最关键的参数之一。它并不是“越大越好”，而应综合考虑：

- 外设手册允许的最大时钟；
- 板级走线质量；
- 电平、电源完整性；
- 当前工作模式（mode 0/1/2/3）；
- 是否有高速连续读需求。

建议初期联调从 **较低频率** 开始，例如 1MHz / 6.4MHz / 12.8MHz，确认稳定后再逐步升高。

##### 3.4 mode 配置

若外设要求非默认 mode 0，需要增加：

- `spi-cpol;`
- `spi-cpha;`

两者组合关系如下：

| 模式 | 属性 |
| ---- | ---- |
| mode 0 | 默认，无需额外属性 |
| mode 1 | `spi-cpha;` |
| mode 2 | `spi-cpol;` |
| mode 3 | `spi-cpol;` + `spi-cpha;` |

##### 3.5 其他常见设备属性

根据 consumer 不同，还可能需要：

- `interrupts` / `interrupt-parent`
- `reset-gpios`
- `wp-gpios`
- `hold-gpios`
- `vcc-supply`
- 分区表、校准信息、厂商私有属性等

### DTS 示例

#### 示例 1：普通 SPI 控制器挂接一个自定义寄存器型设备

```dts
&spi0 {
        pinctrl-names = "default";
        pinctrl-0 = <&spi0_0_cfg>;
        status = "okay";

        my_device@0 {
                compatible = "vendor,my-spi-device";
                reg = <0>;
                spi-max-frequency = <10000000>;
                status = "okay";
        };
};
```

这个示例的含义是：

- 启用 `spi0` 控制器；
- 在片选 0 下挂一个 SPI 从设备；
- 该设备最高工作在 10MHz；
- 具体设备驱动通过 `vendor,my-spi-device` 匹配。

#### 示例 2：设备要求 SPI mode 3

```dts
&spi1 {
        pinctrl-names = "default";
        pinctrl-0 = <&spi1_0_cfg>;
        status = "okay";

        sensor@0 {
                compatible = "vendor,my-sensor";
                reg = <0>;
                spi-max-frequency = <8000000>;
                spi-cpol;
                spi-cpha;
                status = "okay";
        };
};
```

#### 示例 3：K3 板级上挂接 SPI NOR 的真实用法应优先考虑 QSPI

在 K3 SDK 现有板级 DTS 中，SPI NOR 的真实量产用法主要挂在 **QSPI 控制器** 下，而不是普通 SPI 控制器。例如：

```dts
&qspi {
        pinctrl-names = "default";
        pinctrl-0 = <&qspi_cfg>;
        spacemit,qspi-tx-dma = <0>;
        spacemit,qspi-rx-dma = <0>;
        status = "okay";

        flash@0 {
                compatible = "jedec,spi-nor";
                reg = <0>;
                spi-max-frequency = <26500000>;
                m25p,fast-read;
                broken-flash-reset;
                status = "okay";
        };
};
```

这说明从客户选型角度：

- 如果是 **普通寄存器类 SPI 外设**，优先使用普通 SPI 控制器；
- 如果是 **启动 Flash / 大容量 SPI NOR**，优先评估 QSPI 控制器是否更合适。

## K1 与 K3 SPI 驱动差异说明

理解 K1/K3 差异，有助于理解为什么 K3 文档不能原样照搬 K1。

### 1. compatible 已变化

- K1：`spacemit,k1x-spi`
- K3：`spacemit,k3-spi`

这说明虽然两代控制器架构相近，但在 DTS 侧已经明确区分平台。

### 2. 驱动文件不同

- K1：`drivers/spi/spi-k1x.c`
- K3：`drivers/spi/spi-spacemit-k1.c`

K3 驱动实现里可直接看到：

- 通过 `of_match_table` 匹配 `spacemit,k3-spi`；
- `mode_bits = SPI_CPOL | SPI_CPHA | SPI_LOOP`；
- `bits_per_word_mask = SPI_BPW_RANGE_MASK(4, 32)`；
- `num_chipselect = 1`；
- 按条件自动选择 DMA 或 PIO。

因此 K3 文档应重点突出 **consumer 怎么写、频率怎么选、DMA 何时生效、mode 如何配置**，而不是沿用 K1 时代不一定还成立的旧描述。

### 3. K3 DTS 资源描述更完整

K3 控制器节点里普遍加入了：

- `resets`
- 双时钟 `func/bus`
- `dmas` / `dma-names`
- A-domain 和 R-domain 双域分布

这些都是 K3 板级文档中需要向客户讲清楚的内容。

## 接口介绍

### 内核态 API

SPI consumer 驱动通常使用 Linux 标准 SPI API：

- `spi_write()`
- `spi_read()`
- `spi_sync()`
- `spi_sync_transfer()`
- `spi_message_init()`
- `spi_message_add_tail()`

如果客户是开发自己的 SPI 设备驱动，建议优先使用 `spi_sync_transfer()` 或标准 `spi_message` 机制，不要直接依赖控制器私有实现。

### 用户态接口

若系统启用了 spidev，并且对应设备节点使用 `compatible = "spidev"` 或 SDK 允许的兼容方式绑定到 spidev，则用户空间可通过：

- `/dev/spidevB.C`

进行访问，其中：

- `B` 是 SPI bus 号；
- `C` 是 chip select 号。

用户态通常通过 `ioctl()` 配置：

- mode
- bits per word
- max speed

然后发起全双工传输。

### 用户态 demo 示例

```c
#include <stdio.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/spi/spidev.h>

int main(void)
{
    int fd;
    uint8_t mode = SPI_MODE_0;
    uint8_t bits = 8;
    uint32_t speed = 1000000;
    uint8_t tx[2] = {0x9f, 0x00};
    uint8_t rx[2] = {0};
    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = sizeof(tx),
        .speed_hz = speed,
        .bits_per_word = bits,
    };

    fd = open("/dev/spidev0.0", O_RDWR);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    ioctl(fd, SPI_IOC_WR_MODE, &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

    if (ioctl(fd, SPI_IOC_MESSAGE(1), &tr) < 0) {
        perror("SPI transfer");
        close(fd);
        return -1;
    }

    printf("rx: %02x %02x\n", rx[0], rx[1]);
    close(fd);
    return 0;
}
```

## 调试方法

### 1. 查看控制器是否 probe 成功

```bash
dmesg | grep -i spi
```

### 2. 查看 SPI 总线和设备

```bash
ls /sys/bus/spi/devices/
ls /sys/bus/spi/drivers/
```

### 3. 查看是否生成 spidev 节点

```bash
ls /dev/spidev*
```

### 4. 关注 consumer 是否绑定成功

例如 SPI NOR：

```bash
dmesg | grep -Ei "spi|mtd|spi-nor"
cat /proc/mtd
```

### 5. 回环联调建议

如果板上允许把 MOSI 与 MISO 短接，可配合驱动支持的 loopback 模式进行最小闭环验证。若客户在自定义设备 bring-up 早期怀疑硬件连线、时钟或 mode 配置有问题，建议：

1. 先降频到 1MHz 左右；
2. 先确认 mode 0；
3. 再切换到设备要求的 mode；
4. 使用逻辑分析仪观察 SCLK / CS / MOSI / MISO；
5. 再逐步提高频率。

## 测试建议

### 面向 consumer 的验证思路

客户验证一个 SPI 设备是否接通，建议优先按以下顺序进行：

1. **看控制器是否起来**：SPI 控制器 probe 成功；
2. **看设备是否枚举**：子节点是否被创建为 `spiB.C`；
3. **看设备驱动是否绑定**：是否进入对应 consumer 驱动；
4. **看读写是否正确**：寄存器读 ID、读状态寄存器是否正确；
5. **看高频是否稳定**：低频可用后再加速；
6. **看 DMA 是否生效**：大包传输时观察性能与日志。

### 对 SPI NOR 类设备的验证

如果最终设备是 Flash，建议额外做：

- 识别 JEDEC ID；
- 擦写测试；
- 分区读写测试；
- 重启后数据一致性检查。

## FAQ

### 1. 为什么设备树写了 SPI 节点，但设备没有工作？

优先检查：

- pinctrl 是否正确；
- 片选脚是否真的接到了该设备；
- `compatible` 是否与驱动匹配；
- `spi-max-frequency` 是否过高；
- mode 是否配置错误；
- 外设电源、reset、中断 GPIO 是否已就绪。

### 2. 为什么频率设置成 26MHz，但示波器看到不是精确 26MHz？

驱动底层受时钟分频约束，实际输出频率通常是若干离散值逼近目标值，不一定完全等于请求值。这是正常现象，关键是 **不超过设备时序约束且通信稳定**。

### 3. 为什么有时走 DMA，有时走 PIO？

这是驱动设计使然。DMA 只有在 DMA 通道存在、长度合适、对齐满足且映射成功时才启用；否则自动回落到 PIO。
