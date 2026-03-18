# DDR

介绍 K3 平台 DDR 初始化主线、当前支持的 DDR 类型、参数来源与调试入口。

## 概述

K3 平台这条 DDR 文档，不能直接照搬 K1 那套“类型 / CS / 速率 / ODT 都通过 EEPROM 或 SPL DTS 灵活覆盖”的写法。

这轮按 K3 SDK 实际代码回查下来，K3 DDR 的主线更明确也更收敛：

- **DDR 初始化主要发生在 U-Boot SPL 阶段**
- 驱动位于：
  - `uboot-2022.10/drivers/ddr/spacemit/k3/`
- 当前已明确支持：
  - `LPDDR4X`
  - `LPDDR5`
- K3 DDR 驱动优先读取：
  - **TLV 中的 DDR Part Number**
- 如果 TLV 没给，则回退到：
  - **SPL DTS 中的 `part-number`**
- 驱动再根据颗粒型号，匹配内部表项，得到：
  - DDR 类型
  - 容量
  - 默认速率

所以 K3 这篇更适合写成：

1. DDR 初始化主线
2. 颗粒型号识别机制
3. SPL DTS 配置方式
4. 当前支持的 LPDDR4X / LPDDR5 颗粒表
5. 频点 / training / 调试思路

## 初始化主线

当前 K3 DDR 驱动的核心代码在：

```text
uboot-2022.10/drivers/ddr/spacemit/k3/
```

其中最关键的文件包括：

- `ddr_init.c`
- `ddr_freq.c`
- `k3_ddr.h`
- `lpddr4x_init.c`
- `lpddr5_init.c`
- `lpddr4x_*training_table.c`
- `lpddr5_*training_table.c`

Makefile 里可以直接看到当前构成：

```make
ifdef CONFIG_SPL_BUILD
obj-y += ddr_init.o ddr_freq.o
obj-$(CONFIG_K3_BOARD_FPGA) += ddr-snps-lp4x-fpga-init.o
obj-$(CONFIG_K3_BOARD_ASIC) += lpddr5_init.o lpddr5_training_table.o lpddr5_pre_training_table.o
obj-$(CONFIG_K3_BOARD_ASIC) += lpddr4x_init.o lpddr4x_training_table.o lpddr4x_pre_training_table.o
else
obj-y += ddr_freq.o
endif
```

这段非常关键，说明：

- **SPL 阶段**会真正把 DDR 初始化代码带进去；
- ASIC 板当前明确支持 LPDDR4X 和 LPDDR5 两条初始化路径；
- Linux 阶段不是重新“初始化 DDR”，而是更多保留频率切换相关逻辑。

## 当前支持的 DDR 类型

从 `k3_ddr.h` 可直接确认：

```c
typedef enum {
	DDR_TYPE_LPDDR4X = 0,
	DDR_TYPE_LPDDR5,
	DDR_TYPE_UNKNOWN
} ddr_part_type;
```

也就是说，这轮 K3 当前实际能确认的类型只有：

- `LPDDR4X`
- `LPDDR5`

**没有看到 K1 文档里那种 LPDDR3 / LPDDR4 / LPDDR4X 三套并列的模型。**

所以 K3 文档不能沿用 K1 那套类型说明。

## 参数来源与识别机制

### 1. 优先从 TLV 读取 DDR Part Number

`ddr_init.c` 里能直接看到：

```c
ret = get_tlvinfo(TLV_CODE_DDR_PARTNUMBER, ddr_part_number, sizeof(ddr_part_number) - 1);
```

如果 TLV 中取到颗粒型号，驱动会优先用这个值。

### 2. TLV 没有时，从 DTS `part-number` 读取

如果 TLV 读取失败，则继续读 SPL DTS：

```c
temp = dev_read_string(dev, "part-number");
```

也就是说，K3 当前更像是：

- **TLV 优先**
- **DTS 兜底**

而不是 K1 那种把 DDR type / CS / datarate / TX ODT 分成多个字段分别可配。

### 3. 通过 `part-number` 匹配内部表项

驱动内部有颗粒信息表：

```c
const ddr_part_info ddr_parts_info[] = {
	{ "MT62F1G32D2DS", 0x0FD38DD9, DDR_TYPE_LPDDR5, 4096, CONFIG_DDR_DATARATE },
	{ "MT62F2G32D4DS", 0x85D1F688, DDR_TYPE_LPDDR5, 8192, CONFIG_DDR_DATARATE },
	{ "MT62F4G32D8DV", 0x3ACEF2E4, DDR_TYPE_LPDDR5, 16384, CONFIG_DDR_DATARATE },
	{ "MT53E1G32D2FW", 0x75251AB8, DDR_TYPE_LPDDR4X, 4096, 4266 },
	{ "MT53E2G32D4DE", 0x3EA87223, DDR_TYPE_LPDDR4X, 8192, 4266 },
	{ "MT53E4G32D8CY", 0xAA9D4848, DDR_TYPE_LPDDR4X, 16384, 4266 },
};
```

表项中已经直接绑定了：

- Part Number
- CRC32
- DDR 类型
- 容量（MB）
- 数据率（MT/s）

这就是当前 K3 DDR 参数组织的核心思路。

### 4. 默认表项回退

如果找不到匹配颗粒，`find_ddr_info()` 会回退到第一项：

- `MT62F1G32D2DS`

这意味着如果 TLV / DTS 的 `part-number` 写错，SPL 可能会按默认颗粒去初始化，风险非常大。

## DTS 配置

### SPL DTS 节点

当前 U-Boot SPL 设备树里能直接看到 DDR 节点，例如：

```dts
ddr@cb000000 {
	u-boot,dm-spl;
	compatible = "spacemit,snps-lp45";
	part-number = "MT62F2G32D4DS";
	reg = <0x00000000 0xcb000000 0x00000000 0x00400000>,
	      <0x00000000 0xcc000000 0x00000000 0x00400000>;
	reg-names = "ddrc0", "ddrc1";
	status = "okay";
};
```

这里最关键的属性是：

| 属性 | 作用 |
| :--- | :--- |
| `compatible = "spacemit,snps-lp45"` | 匹配 K3 DDR 控制器 SPL 驱动 |
| `part-number` | 当 TLV 没有颗粒信息时，作为识别输入 |
| `reg` | 两个 DDRC 寄存器空间 |
| `reg-names = "ddrc0", "ddrc1"` | 对应双通道 DDRC |
| `status = "okay"` | 启用节点 |

### 顶层 U-Boot DTS 中也有 DDR 节点

在 `uboot-2022.10/arch/riscv/dts/k3.dtsi` 中也可看到：

```dts
ddr@3000000 {
	u-boot,dm-spl;
	compatible = "spacemit,snps-lp45";
	reg = <0x00000000 0xc0000000 0x00000000 0x00400000>;
	status = "okay";
};
```

但当前真正带 `part-number`、而且更接近 SPL 初始化入口的，是 `k3_spl.dts` 这类 SPL DTS。

## 驱动实现要点

### 1. probe 流程

K3 DDR SPL 驱动入口在：

- `spacemit_ddr_probe()`

主要步骤是：

1. 读取 DDRC 地址
2. 获取 DDR Part Number（TLV 优先，DTS 兜底）
3. 匹配 `ddr_parts_info[]`
4. 打印颗粒信息
5. 调用 `lpddr_silicon_init()` 对 `ddrc0` / `ddrc1` 分别初始化
6. 做一次基础 memory verify

驱动里能直接看到：

```c
printf("DDR Part Number: %s, Size: %dMB, Data Rate: %dMT/s\n",
	part_info->part_number, part_info->size_mb, part_info->data_rate_mtps);
```

这也是 bring-up 时最值得看的早期日志之一。

### 2. 当前仅支持 LPDDR4X / LPDDR5

驱动里还显式判断：

```c
if ((DDR_TYPE_LPDDR5 != part_info->type) && (DDR_TYPE_LPDDR4X != part_info->type)) {
	pr_err("unsupported ddr type %d\n", part_info->type);
	return 1;
}
```

这点要在文档里写死，避免用户按 K1 经验去配 LPDDR3/LPDDR4。

### 3. 初始化后会做基础内存校验

`ddr_init.c` 里有：

- `test_pattern()`

会在一段地址范围上做：

- 地址写入模式校验
- 反码写入模式校验
- 恢复原值

日志成功时会打印：

```text
memory verify pass
```

失败则会打印：

```text
memory verify fail!
```

这能帮助你在 SPL 阶段尽早发现训练/初始化问题。

## 频率与训练

### 默认数据率

`k3_ddr.h` 中当前默认定义为：

```c
#define CONFIG_DDR_DATARATE	(6400)
```

并且注释明确写了当前支持范围：

- `4266MT/s`
- `5500MT/s`
- `6000MT/s`
- `6400MT/s`

### DFC 频点表

`ddr_freq.c` 中可见当前频点表：

- `600`
- `800`
- `1066`
- `1200`
- `1600`
- `2400`
- `2666`

这张表对应的是频率切换/级别控制逻辑，而不是简单等同于最终宣传口径里的 LPDDR MT/s 数字，但它能说明：

- 当前 K3 DDR 已经考虑了多级频点切换；
- `ddr_freq.c` 不只是空壳。

### 2D training 配置

`k3_ddr.h` 中还能看到：

```c
#define DISABLE_DDR_2D_TRAINING	(0)
```

说明当前默认并**没有关闭** 2D training。

另外 LPDDR4X / LPDDR5 都有各自的：

- `pre_training_table`
- `training_table`

这说明 K3 DDR 初始化不是只靠固定寄存器表硬写，而是包含较完整的训练流程支持。

## 配置建议

### 1. 最常改的是 `part-number`

如果你是在适配新的 DDR 颗粒，当前 K3 最直接、最稳的入口通常是：

- 先确认 TLV 里有没有正确的 `DDR_PARTNUMBER`
- 如果没有，就改 SPL DTS 里的：
  - `part-number = "...";`

### 2. 改颗粒支持时，不要只改 DTS

因为 K3 驱动不是看到任意 `part-number` 都能自动工作，它必须在：

- `ddr_parts_info[]`

里找到对应表项。

也就是说，如果你上的是一个当前表里没有的新颗粒，通常至少要补：

- part number
- type
- size_mb
- data_rate_mtps

并确认对应 LPDDR4X / LPDDR5 初始化与训练参数是否真的适配。

### 3. K3 当前文档不建议照 K1 那种颗粒参数逐项开放修改

这轮按源码看，K3 现阶段更像是：

- 通过颗粒型号选择一套初始化/训练方案

而不是像 K1 文档那样，把：

- `type`
- `cs-num`
- `datarate`
- `tx-odt`

都作为 SPL DTS / EEPROM 中可直接逐项开放给用户改。

所以这篇文档先不写 K1 那种逐寄存器 / 逐字段调参教程，避免误导。

## 调试建议

### 1. 先看 SPL 打印的颗粒识别结果

优先关注：

```text
DDR Part Number: ..., Size: ...MB, Data Rate: ...MT/s
```

如果这里 Part Number 就不对，后面一切都不用看了。

### 2. 再看 memory verify

如果出现：

- `memory verify pass`

说明最基础的初始化至少过了一轮快速校验。

如果是：

- `memory verify fail!`

那就先回头查：

- `part-number`
- TLV 内容
- 训练表是否匹配实际颗粒
- LPDDR4X / LPDDR5 类型是否选对

### 3. 注意双 DDRC 资源

驱动里会分别对：

- `ddrc0`
- `ddrc1`

调用初始化。

因此如果 DTS `reg` 或 `reg-names` 写错，不一定是“完全不起”，也可能表现为初始化过程异常、容量不对或稳定性问题。

## FAQ

### 1. K3 DDR 当前支持哪些类型？

按这轮源码确认，当前明确支持：

- `LPDDR4X`
- `LPDDR5`

### 2. K3 还是像 K1 一样支持通过 EEPROM/TLV 配一堆 DDR 字段吗？

不是同一种模型。

K3 当前能明确确认的是：

- 通过 `TLV_CODE_DDR_PARTNUMBER` 读取颗粒型号
- 或者 DTS `part-number` 兜底
- 再由内部颗粒表决定类型、容量和默认速率

### 3. K3 当前默认数据率是多少？

从 `k3_ddr.h` 看，当前默认：

- `CONFIG_DDR_DATARATE = 6400`

但具体某颗粒实际使用值，还要看 `ddr_parts_info[]` 里的表项。

### 4. 这篇最该记住哪条经验？

**先把 `part-number` 这条线搞对，再谈 DDR 调优。**

K3 当前 DDR bring-up 的第一关键点，不是先改一堆寄存器，而是确保 SPL 识别到的颗粒型号就是你板子上那颗。