# QSPI

介绍 K3 平台 QSPI 控制器驱动的能力、面向 Flash consumer 的设备树配置方法、典型板级用法以及调试方式。

## 模块介绍

QSPI（Quad SPI）是在 SPI 基础上扩展出来的高速串行接口，常用于连接 SPI NOR Flash。相比普通 SPI 控制器，QSPI 更适合处理：

- 启动 Flash；
- 大容量固件存储；
- 高频连续读；
- 通过 memory-mapped / AHB 映射方式进行高效访问的场景。

从客户角度看，QSPI 文档最重要的不是控制器寄存器定义，而是：

- K3 上 QSPI 怎么启用；
- 该控制器主要适合接哪类设备；
- `jedec,spi-nor` 这类 consumer 节点怎么写；
- 分区表怎么配；
- DTS 里那些 `spacemit,qspi-*` 属性是什么意思；
- 为什么同样是 Flash，K3 板级更倾向于挂在 QSPI 而不是普通 SPI 控制器下面。

K3 SDK 的实际板级 DTS 已经给出了很明确的答案：

- **普通 SPI 控制器** 更适合通用寄存器型 SPI 外设；
- **QSPI 控制器** 更适合 SPI NOR Flash 等高吞吐存储型设备。

### 功能介绍

Linux 中，K3 QSPI 方案涉及三层：

1. **SPI Core / SPI Memory Framework**：为 Flash 类设备提供统一访问模型；
2. **QSPI 控制器驱动**：实现 `spi-mem` 控制器能力；
3. **Flash consumer 驱动**：通常是 SPI NOR 驱动，如 `jedec,spi-nor`。

K3 QSPI 驱动采用的是 **spi-mem 框架**，也就是说它不是简单地把 Flash 当成一个“普通 SPI 字节流设备”去处理，而是围绕 Flash 的典型操作来优化，例如：

- opcode / address / dummy / data 组合操作；
- LUT（Look-Up Table）序列配置；
- AHB 映射读；
- DMA 辅助收发；
- 针对读写/擦除后 AHB buffer 的失效处理。

这也是为什么 K3 上的 Flash consumer 文档，必须结合 **驱动代码 + DTS + consumer 设备树用法** 来写，而不能只从 generic SPI 角度描述。

### 源码结构介绍

K3 QSPI 相关代码主要位于：

```text
linux-6.18/
|-- drivers/spi/
|   |-- spi-mem.c                        # SPI memory framework
|   |-- spi-nor/                         # SPI NOR 相关支持（由 MTD 子系统接管）
|   |-- Kconfig
|   `-- spi-k1-qspi.c                   # K3 QSPI 控制器驱动
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                         # K3 QSPI 控制器节点
    `-- k3*.dts                         # 板级 QSPI Flash 使用示例
```

K3 的控制器节点 `compatible` 为：

```dts
compatible = "spacemit,k3-qspi";
```

K1 的板级 DTS 中也存在 QSPI 节点和使用方式，说明两代平台都把 QSPI 作为 Flash 接口的重要方案；但 K3 文档仍需以 `linux-6.18` 当前驱动和 DTS 实际写法为准。

## 关键特性

### 控制器能力

结合 `drivers/spi/spi-k1-qspi.c` 可知，K3 QSPI 驱动具备如下能力：

| 特性 | 说明 |
| :----- | :---- |
| 框架模型 | 基于 `spi-mem` 框架实现 |
| 适用设备 | 重点面向 SPI NOR Flash 类设备 |
| 访问模式 | 支持命令模式（IP command）和 AHB memory-mapped read |
| LUT 序列 | 根据 `spi_mem_op` 动态生成 LUT 序列 |
| 多线模式 | 支持 single / dual / quad 线宽操作（取决于 Flash op） |
| DMA | 支持 TX DMA / RX DMA，可通过 DTS 打开或关闭 |
| 时钟控制 | 驱动会设置 QSPI 功能时钟 |
| 缓冲失效 | 写/擦除后会触发 AHB buffer 失效处理 |
| 调试接口 | 提供 sysfs 信息节点查看 DMA / buffer / 错误状态 |

### 从客户使用角度要重点理解的能力

#### 1. 它不是普通 SPI 控制器的替代写法，而是更适合 Flash

普通 SPI 控制器也能理论上挂 Flash，但在 K3 实际 SDK 中，板级 DTS 明确优先把 SPI NOR 放在 `&qspi` 下面。原因是 QSPI 驱动：

- 更适合 Flash 的命令访问模型；
- 支持 memory mapped 读；
- 支持更高效的连续读；
- 已经在 K3 板级 DTS 中形成标准用法。

#### 2. DTS 中的 consumer 几乎总是 Flash

K3 现有板级 DTS 中，QSPI 下面的典型 consumer 为：

```dts
flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <26500000>;
        m25p,fast-read;
        broken-flash-reset;
        status = "okay";
};
```

因此客户如果接的是启动 Flash 或 NOR Flash，优先参考 QSPI 文档，而不是普通 SPI 文档。

#### 3. DTS 可控制 DMA 开关

K3 板级 DTS 中常见：

```dts
spacemit,qspi-tx-dma = <0>;
spacemit,qspi-rx-dma = <0>;
```

这说明 QSPI 驱动支持通过设备树控制 TX / RX DMA 是否启用。对于 bring-up 阶段，如果怀疑 DMA 路径有问题，可以先关闭 DMA，优先验证基础读写是否正常。

## 配置介绍

### CONFIG 配置

K3 QSPI 常用配置包括：

- `CONFIG_SPI`
- `CONFIG_SPI_MEM`
- `CONFIG_SPI_K1_QSPI`
- `CONFIG_MTD`
- `CONFIG_MTD_SPI_NOR`

菜单路径大致如下：

```text
Device Drivers
    SPI support
        SPI support (SPI [=y])
        SPI Master Controller Drivers
            Spacemit K1 QSPI Controller (SPI_K1_QSPI [=y])

Device Drivers
    Memory Technology Device (MTD) support
        SPI-NOR device support (MTD_SPI_NOR [=y])
```

> 说明：Kconfig 名称虽然仍带 `K1`，但 K3 SDK 中同样用于 `spacemit,k3-qspi` 控制器。

### DTS 配置

#### 1. 控制器节点

K3 的 QSPI 控制器节点位于 `arch/riscv/boot/dts/spacemit/k3.dtsi`：

```dts
qspi: spi@d420c000 {
        compatible = "spacemit,k3-qspi";
        #address-cells = <1>;
        #size-cells = <0>;
        reg = <0x0 0xd420c000 0x0 0x1000>,
              <0x0 0xb8000000 0x0 0xc00000>;
        reg-names = "QuadSPI", "QuadSPI-memory";
        clocks = <&syscon_apmu CLK_APMU_QSPI>,
                 <&syscon_apmu CLK_APMU_QSPI_BUS>;
        clock-names = "qspi", "qspi_en";
        resets = <&syscon_apmu RESET_APMU_QSPI>,
                 <&syscon_apmu RESET_APMU_QSPI_BUS>;
        reset-names = "qspi_reset", "qspi_bus_reset";
        interrupts = <117 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&saplic>;
        status = "disabled";
};
```

各属性含义如下：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | 固定为 `spacemit,k3-qspi` |
| `reg` | 第一段是 QSPI 控制器寄存器，第二段是 QSPI memory-mapped window |
| `reg-names` | 控制器寄存器区与 memory window 的名称 |
| `clocks` / `clock-names` | QSPI 功能时钟和总线时钟 |
| `resets` / `reset-names` | 控制器和 bus 复位 |
| `interrupts` | QSPI 中断号 |
| `status` | `okay` 表示启用控制器 |

这里最值得客户注意的是：**QSPI 比普通 SPI 多了一个 memory-mapped 的地址窗口**，这正是它适合 Flash 访问的重要原因。

#### 2. pinctrl 配置

QSPI 使用前需要先确认板级引脚复用，一般包括：

- CLK
- CS
- DAT0
- DAT1
- DAT2
- DAT3

典型启用方式如下：

```dts
&qspi {
        pinctrl-names = "default";
        pinctrl-0 = <&qspi_cfg>;
        status = "okay";
};
```

不同板级 DTS 中 pin group 名称可能不同，但 K3 现有板级大多使用 `&qspi_cfg`。

#### 3. QSPI 控制器私有属性

结合 K3 驱动和板级 DTS，可看到以下常用属性：

| 属性 | 含义 |
| ---- | ---- |
| `spacemit,qspi-tx-dma` | 控制 TX DMA 是否启用，板级常设为 `<0>` 关闭 |
| `spacemit,qspi-rx-dma` | 控制 RX DMA 是否启用，板级常设为 `<0>` 关闭 |

板级示例：

```dts
&qspi {
        spacemit,qspi-tx-dma = <0>;
        spacemit,qspi-rx-dma = <0>;
};
```

从客户调试角度建议：

- **首板 bring-up**：建议先关闭 DMA，优先验证基础识别、读 ID、分区枚举；
- **性能调优阶段**：再按需打开 DMA 做吞吐测试。

#### 4. Flash consumer 节点配置

这是 QSPI 文档最核心的部分。K3 现有 DTS 中，QSPI 下面一般挂的是 `jedec,spi-nor`。

典型 consumer 写法如下：

```dts
flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <26500000>;
        m25p,fast-read;
        broken-flash-reset;
        status = "okay";
};
```

各字段说明：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | 使用通用 SPI NOR 驱动 |
| `reg` | 片选号，通常为 0 |
| `spi-max-frequency` | Flash 工作频率，建议参考芯片手册并从较低频率起调 |
| `m25p,fast-read` | 启用 fast read 能力 |
| `broken-flash-reset` | 某些 Flash 没有可靠 reset 时使用 |
| `status` | 设为 `okay` 表示启用该 Flash |

#### 5. Flash 分区配置

如果该 Flash 用于启动或固件存储，通常还需要添加固定分区：

```dts
partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        bootinfo@0 {
                reg = <0x00000000 0x00020000>;
        };

        fsbl@20000 {
                reg = <0x00020000 0x00080000>;
        };

        env@a0000 {
                reg = <0x000a0000 0x00010000>;
        };
};
```

客户在配置分区时，应特别注意：

- 分区地址和大小必须与实际 Flash 容量匹配；
- 启动链各阶段（FSBL、OpenSBI、U-Boot、环境区）的地址必须和固件烧写方案一致；
- 修改分区表前要先确认量产工具、升级流程、启动 ROM 约束。

### DTS 示例

#### 示例 1：最小可用 QSPI + SPI NOR 配置

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

这个例子适合客户在新板 bring-up 早期做最基础验证。

#### 示例 2：带固定分区的启动 Flash 配置

该形式参考 `k3_deb1.dts`：

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

                partitions {
                        compatible = "fixed-partitions";
                        #address-cells = <1>;
                        #size-cells = <1>;

                        bootinfo@0 {
                                reg = <0x00000000 0x00020000>;
                        };
                        fsbl@20000 {
                                reg = <0x00020000 0x00080000>;
                        };
                        env@a0000 {
                                reg = <0x000a0000 0x00010000>;
                        };
                        esos@b0000 {
                                reg = <0x000b0000 0x00100000>;
                        };
                        opensbi@1b0000 {
                                reg = <0x001b0000 0x00060000>;
                        };
                        uboot@210000 {
                                reg = <0x00210000 0x005f0000>;
                        };
                };
        };
};
```

## K1 与 K3 的差异理解

为了理解 K3 文档应该怎么写，也需要参考 K1 的做法。

### 1. 两代平台都把 QSPI 作为 Flash 重要接口

从 K1 DTS 可以看到，QSPI 在多块板卡上都被启用并挂接 Flash；K3 也延续了这个方向。因此在面向客户写文档时，QSPI 的重点天然应该放在 **Flash consumer**，而不是泛泛介绍 SPI 总线。

### 2. K3 设备树更强调控制器资源完整描述

K3 `qspi` 节点中明确给出了：

- 两段 `reg`：控制器 + memory window；
- `clock-names = "qspi", "qspi_en"`；
- `reset-names = "qspi_reset", "qspi_bus_reset"`；
- 板级 DTS 通过 `spacemit,qspi-tx-dma` / `spacemit,qspi-rx-dma` 控制 DMA。

这些都是 K3 文档里必须明确写给客户的内容。

### 3. K3 驱动是典型 spi-mem/Flash 优化写法

驱动里可以直接看到：

- 通过 `spi_mem_op` 生成 LUT；
- 支持 AHB 读；
- 写/擦除后做 AHB buffer invalid；
- 暴露 sysfs 调试节点查看 DMA 和 buffer 情况；
- 适合 SPI NOR 这类存储设备。

所以 K3 的 QSPI 文档写法应围绕：

- consumer 节点如何写；
- 分区如何规划；
- bring-up 阶段如何先保守配置；
- 性能和稳定性如何逐步调优。

## 接口介绍

### 内核态接口

K3 QSPI 控制器驱动对上层主要提供的是 `spi-mem` 能力，因此 consumer 侧通常不直接调用控制器私有 API，而是通过：

- SPI NOR 框架；
- MTD 子系统；
- `spi-mem` 标准操作接口；

完成识别、读写和擦除。

如果最终挂接的是 `jedec,spi-nor`，那客户更多接触到的是：

- `/proc/mtd`
- `/dev/mtdX`
- `/dev/mtdblockX`

### 用户态接口

当 Flash 作为 MTD 设备暴露后，用户态可通过：

- `cat /proc/mtd`
- `mtd_debug`
- `flash_erase`
- `nandwrite`（不适用于 NOR）
- `dd`（谨慎使用）

等方式进行验证和维护。

## 调试方法

### 1. 查看 QSPI 和 SPI NOR 是否识别成功

```bash
dmesg | grep -Ei "qspi|spi-nor|mtd"
cat /proc/mtd
```

### 2. 查看 sysfs 调试节点

K3 QSPI 驱动会创建调试属性，可用于查看当前 DMA / buffer / 错误情况。可以在对应设备目录下查看，例如：

```bash
find /sys -path '*qspi_dev*' 2>/dev/null
```

典型可看到类似信息：

- `qspi_info`：RX/TX DMA 开关、buffer 大小、AHB read 状态；
- `qspi_err_resp`：TX underrun、RX overflow、AHB overflow 等错误计数。

### 3. 验证分区是否正确创建

```bash
cat /proc/mtd
```

如果 DTS 中配置了固定分区，系统启动后应能看到对应分区名，例如：

- `bootinfo`
- `fsbl`
- `env`
- `opensbi`
- `uboot`

### 4. 读 ID / 读写验证

可结合 `mtd_debug` 做基础验证，例如：

```bash
mtd_debug info /dev/mtd0
mtd_debug read /dev/mtd0 0x0 0x100 /tmp/qspi_read.bin
```

### 5. bring-up 调试建议

如果首板上 QSPI Flash 无法识别，建议按以下顺序排查：

1. 确认 pinctrl 与原理图一致；
2. 确认 Flash 型号、电压和接线方式（尤其 DAT2/DAT3）；
3. 先关闭 TX/RX DMA；
4. 先降低 `spi-max-frequency`；
5. 确认 `compatible = "jedec,spi-nor"` 是否合适；
6. 观察 `dmesg` 是否有读 ID 失败、超时或中断错误；
7. 再检查分区表是否越界。

## 测试建议

### 面向客户的最小验证路径

推荐按下面顺序验证：

1. **控制器起得来**：`qspi` probe 成功；
2. **Flash 识别成功**：能看到 `spi-nor` 相关日志；
3. **MTD 创建设备成功**：`/proc/mtd` 中有目标设备；
4. **分区正确**：分区名和地址范围符合预期；
5. **基础读正常**：可读 JEDEC ID / 读 flash 内容；
6. **擦写正常**：擦除后重读验证；
7. **重启一致性正常**：重启后数据与分区状态一致。

## FAQ

### 1. 为什么建议 Flash 优先挂 QSPI，而不是普通 SPI？

因为 K3 QSPI 驱动基于 `spi-mem`，更适合 Flash 的命令模型，并支持 memory-mapped read、LUT 序列和更高效的数据访问，板级 DTS 也已经把这作为标准用法。

### 2. 为什么 DTS 里要先把 `spacemit,qspi-tx-dma` / `spacemit,qspi-rx-dma` 设为 0？

这是 bring-up 阶段常见做法。先用最简单路径确认 Flash 基础访问正确，再逐步启用 DMA 做性能优化，能更快定位问题。

### 3. 为什么系统里能看到 QSPI 控制器，但看不到 `/proc/mtd` 分区？

通常说明控制器已经 probe，但 Flash consumer 没有正确识别。请检查：

- `flash@0` 子节点是否存在；
- `compatible` 是否正确；
- 频率是否过高；
- Flash 供电、复位、引脚连接是否正常；
- 分区表是否写在正确节点下。
