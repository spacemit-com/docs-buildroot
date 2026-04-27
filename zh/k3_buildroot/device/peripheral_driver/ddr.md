# DDR 驱动参数配置方法 (K3)

| 版本 | 修改日期   | 说明     |
|------|------------|----------|
| v1.0 | 2026-04-17 | 初版发布 |

## 1. 概述

本文档基于 **SpacemiT K3** 平台，介绍如何修改 DDR 驱动参数，以实现 K3 SDK 与特定 DDR 颗粒的适配。主要涵盖以下内容：

- DDR 料号配置与颗粒匹配机制
- 速率配置
- IO 电气参数（ODT / Drive Strength / 2D Training）调整
- 调试方法与日志说明

### 支持的 DDR 类型

K3 SDK 支持 **LPDDR4X** 和 **LPDDR5** 两种 DDR 类型，通过 DDR 料号（Part Number）自动匹配对应的初始化参数。

内置支持的颗粒列表（`drivers/ddr/spacemit/k3/ddr_init.c`）：

**LPDDR5**

| 写入 DDR 料号   | Ranks | x8 | 容量     | 速率（MT/s） | 兼容颗粒型号                                                          |
|-----------------|-------|----|----------|-------------|-----------------------------------------------------------------------|
| `MT62F1G32D2DS` | 1     | 否 | 4096 MB  | 6400        | MT62F1G32D2DS-023 WT:C、CXDD5DBAM-MC-M                               |
| `MT62F2G32D4DS` | 2     | 否 | 8192 MB  | 6400        | MT62F2G32D4DS-023 WT:C、RS2G32LO5D4FDB-31BT、CXDD6DCBM-MC-M         |
| `MT62F4G32D8DV` | 2     | 是 | 16384 MB | 6400        | MT62F4G32D8DV-023 WT:C、RS4G32LO5D8FDB-31BT                          |

**LPDDR4X**

| 写入 DDR 料号   | Ranks | x8 | 容量     | 速率（MT/s） | 兼容颗粒型号               |
|-----------------|-------|----|----------|-------------|----------------------------|
| `MT53E1G32D2FW` | 1     | 否 | 4096 MB  | 4266        | MT53E1G32D2FW-046 WT:C     |
| `MT53E2G32D4DE` | 2     | 否 | 8192 MB  | 4266        | MT53E2G32D4DE-046 WT:C     |
| `MT53E4G32D8CY` | 2     | 是 | 16384 MB | 4266        | MT53E4G32D8CY-046 WT:C     |

"写入 DDR 料号"为实际写入 EEPROM 或 DTS `part-number` 字段的字符串。若使用规格相同但料号不同的颗粒（如兼容颗粒型号列中的颗粒），直接写入对应的料号即可复用已有参数，无需新增代码条目。

### 配置机制

K3 DDR 驱动通过 **DDR 料号** 匹配颗粒参数，优先级如下：

```
EEPROM (TLV 0x45)  →  DTS part-number  →  代码默认值（ddr_parts_info[0]）
     最高优先级                                    最低优先级
```

1. 优先读取 EEPROM 中存储的 `ddr_partnumber`（TLV code `0x45`）
2. 若 EEPROM 无有效配置，则从 DTS 节点的 `part-number` 属性获取
3. 若两者都未设置，使用 `ddr_parts_info[]` 表的第一项作为默认值

### 启动模式

DDR 驱动支持两种启动模式：

- **Training 模式**：首次启动或无有效训练结果时，执行完整的 PHY Training 流程，耗时较长（数秒）。训练完成后将结果保存到内存缓冲区（`DDR_TRAINING_INFO_BUFF = 0xC08D0000`），在镜像烧写过程中同步更新至代码中。
- **Quick Boot 模式**：检测到有效的训练结果缓存时，直接恢复 PHY 参数，跳过 Training，显著缩短启动时间。

Quick Boot 有效性校验需满足：magic（`0x54524444`）、DDR 类型、CS 数量、速率、CRC32 全部匹配。

## 2. 基本配置

### 2.1 DDR 料号配置

K3 通过 DDR 料号自动匹配颗粒的类型、容量、速率等全部参数，无需分别配置各项参数。根据实际场景选择以下方式之一：

#### 场景 1：通过 EEPROM 配置（无需重烧镜像）

适用于已能正常启动的设备，修改后立即生效，无需重新编译。

**方式一：TitanFlasher 工具写入**

- 按住设备烧录按键上电，进入烧录模式，通过 USB 连接 PC
- 使用 TitanFlasher 工具集的写号功能，写入 DDR 料号（`ddr_partnumber`）

![TitanFlasher 写入 DDR 料号](static/ddr_part_number.png)

**方式二：U-Boot 命令行写入**

前提：DDR 使用默认配置能够完成初始化，设备能够启动到 U-Boot。

```shell
=> tlv_eeprom read
=> tlv_eeprom set 0x45 MT62F2G32D4DS
=> tlv_eeprom write
```

#### 场景 2：固定 DDR 料号（板上无 EEPROM）

修改 `arch/riscv/dts/k3_spl.dts` 中 DDR 节点的 `part-number` 字段，需重新编译生效：

```dts
ddr@cb000000 {
    u-boot,dm-spl;
    compatible = "spacemit,snps-lp45";
    part-number = "MT62F2G32D4DS";    /* 修改为实际使用的料号 */
    reg = <0x00000000 0xcb000000 0x00000000 0x00400000>,
          <0x00000000 0xcc000000 0x00000000 0x00400000>;
    reg-names = "ddrc0", "ddrc1";
    status = "okay";
};
```

#### 场景 3：新增不在列表中的颗粒

若使用的颗粒不在内置列表中，在 `drivers/ddr/spacemit/k3/ddr_init.c` 的 `ddr_parts_info[]` 中添加新条目，需重新编译生效：

```c
const ddr_part_info ddr_parts_info[] = {
    /* part_number,    crc32,      type,           ranks, x8, size_mb, data_rate */
    { "MT62F2G32D4DS", 0x85D1F688, DDR_TYPE_LPDDR5, 2,    0,  8192,   CONFIG_DDR_DATARATE },
    /* 新增条目示例（LPDDR5，双 rank，8GB） */
    { "NEW_PART_NUM",  0xEEFF6244, DDR_TYPE_LPDDR5, 2,    0,  8192,   6400 },
};
```

> `crc32_value` 是料号字符串的 CRC32 值，用于加速匹配，计算方式：
> ```c
> crc32(0, (const uint8_t*)"NEW_PART_NUM", strlen("NEW_PART_NUM"))
> ```

### 2.2 速率配置

`ddr_parts_info[]` 中每个颗粒条目的 `data_rate_mtps` 字段指定该颗粒的实际运行速率。若需修改，直接修改对应条目的值，或修改 `CONFIG_DDR_DATARATE` 宏（影响所有引用该宏的条目）：

```c
// drivers/ddr/spacemit/k3/k3_ddr.h
// 支持 4266 / 5500 / 6000 / 6400 MT/s，取其他值编译报错
#define CONFIG_DDR_DATARATE  (6400)
```

## 3. 高阶配置

> **警告**：本章节涉及的参数直接影响信号完整性，不当修改可能导致系统不稳定或无法启动。建议在硬件工程师指导下进行，修改后务必执行严格的内存稳定性测试。

K3 DDR 驱动的电气参数通过 `ddr_config_t` 结构体统一配置，LPDDR5 和 LPDDR4X 共用同一套结构，默认参数定义在 `drivers/ddr/spacemit/k3/lpddr5_init.c` 的 `ddr_default_io_para[]` 数组中：

```c
static const ddr_config_t ddr_default_io_para[] = {
    // type,            WDS       RX_ODT    DQ_ODT CA_ODT NT_ODT SOC_ODT PDDS   2D
    { DDR_TYPE_LPDDR5,  PHY_R_30, PHY_R_60, R_60,  R_80,  R_OFF, R_OFF,  R_40,  1 },
    { DDR_TYPE_LPDDR4X, PHY_R_40, PHY_R_40, R_60,  R_40,  R_OFF, R_40,   R_40,  1 },
};
```

启动时 `lpddr_init_prepare()` 根据颗粒类型自动选取对应行，调用 `build_lpddr5_io_para()` 或 `build_lpddr4x_io_para()` 将参数转换为 PHY 寄存器配置后写入硬件。

### 3.1 IO 参数字段说明

**`phy_write_ds` — PHY 端写驱动强度（Drive Strength）**

控制 SoC PHY 侧 DQ/DQS 信号的输出驱动阻抗。阻抗越低驱动越强，信号上升沿越陡，但同时增加反射和串扰风险。

| 枚举值      | 等效阻抗 | 说明                              |
|-------------|----------|-----------------------------------|
| `PHY_R_OFF` | 高阻     | 禁用输出，不用于正常工作          |
| `PHY_R_120` | 120 Ω    | 长走线、高阻抗负载                |
| `PHY_R_60`  | 60 Ω     | 中等走线长度                      |
| `PHY_R_40`  | 40 Ω     | 较短走线，LPDDR4X 默认值          |
| `PHY_R_30`  | 30 Ω     | 短走线，LPDDR5 高速场景默认值     |

> LPDDR5 spec（JESD209-5）推荐 SoC 侧驱动阻抗与走线特征阻抗匹配（通常 40–50 Ω），实际值需结合 PCB 仿真确定。

**`phy_rx_odt` — PHY 端接收终端（RX ODT）**

控制 SoC PHY 侧 DQ/DQS 输入端的片上终端电阻（On-Die Termination）。读操作时开启以吸收反射，写操作时通常关闭。

| 枚举值      | 等效阻抗 | 说明                                      |
|-------------|----------|-------------------------------------------|
| `PHY_R_OFF` | 高阻     | 关闭 RX ODT                              |
| `PHY_R_120` | 120 Ω    | 弱终端，适合长走线低反射场景              |
| `PHY_R_60`  | 60 Ω     | LPDDR5 默认值，与 DRAM 侧 PDDS 配合使用  |
| `PHY_R_40`  | 40 Ω     | LPDDR4X 默认值                            |
| `PHY_R_30`  | 30 Ω     | 强终端，适合极短走线                      |

> 典型组合：`phy_rx_odt=PHY_R_60`（60 Ω）+ `pdds=R_40`（40 Ω），DRAM 驱动阻抗低于 SoC 终端阻抗，保证信号电平。

**`dq_odt` — DRAM 侧 DQ 终端电阻（MR11[2:0]）**

控制 DRAM 颗粒内部 DQ 总线的片上终端电阻。写操作时 DRAM 开启 ODT 以减少反射，读操作时关闭。

| 枚举值  | 等效阻抗 | LPDDR5 MR11 编码 |
|---------|----------|-----------------|
| `R_OFF` | 关闭     | 000             |
| `R_240` | 240 Ω    | 001             |
| `R_120` | 120 Ω    | 010             |
| `R_80`  | 80 Ω     | 011             |
| `R_60`  | 60 Ω     | 100（默认）     |
| `R_48`  | 48 Ω     | 101             |
| `R_40`  | 40 Ω     | 110             |

> 典型组合：`phy_write_ds=PHY_R_30`（30 Ω）+ `dq_odt=R_60`（60 Ω）。

**`ca_odt` — DRAM 侧 CA 终端电阻（MR11[6:4]）**

控制 DRAM 颗粒内部 CA（Command/Address）总线的片上终端电阻。CA 总线为单向（SoC→DRAM），DRAM 侧 ODT 用于吸收 CA 信号反射。可选值与 `dq_odt` 相同。

> LPDDR5 spec 中 CA ODT 典型值为 80 Ω，与 SoC 侧 CA 驱动阻抗（通常 40 Ω）形成匹配。

**`nt_odt` — 非目标终端（NT ODT，MR41[7:5]）⚠️ LPDDR5 新增**

NT ODT 是 LPDDR5 相对于 LPDDR4/LPDDR4X 新增的特性，**LPDDR4X 不支持**，该字段对 LPDDR4X 无效，驱动不会将其写入寄存器。

多 rank 配置下，写操作仅针对目标 rank，非目标 rank 开启 NT ODT 以抑制总线反射和串扰。单 rank 配置下必须设为 `R_OFF`。可选值与 `dq_odt` 相同。

> 单 rank 颗粒（`ranks=1`）设为非 `R_OFF` 值会引起额外功耗，无实际收益。

**`soc_odt` — SoC 侧终端电阻（SOC ODT，MR17[2:0]）**

控制 DRAM 颗粒对 SoC 侧的终端电阻配置，用于改善信号在 SoC 端的反射特性。可选值与 `dq_odt` 相同。

> 该参数与 PCB 走线拓扑强相关，通常由信号完整性仿真确定。LPDDR5 默认关闭（`R_OFF`），LPDDR4X 默认 40 Ω。

**`pdds` — DRAM 侧输出驱动强度（Pull-Down Drive Strength，MR3[2:0]）**

控制 DRAM 颗粒 DQ/DQS 输出驱动阻抗。读操作时 DRAM 驱动数据总线，该值决定 DRAM 侧的驱动能力。

| 枚举值  | 等效阻抗 | LPDDR5 MR3 编码       |
|---------|----------|-----------------------|
| `R_OFF` | 高阻     | 000（不用于正常工作） |
| `R_240` | 240 Ω    | 001                   |
| `R_120` | 120 Ω    | 010                   |
| `R_80`  | 80 Ω     | 011                   |
| `R_60`  | 60 Ω     | 100                   |
| `R_48`  | 48 Ω     | 101                   |
| `R_40`  | 40 Ω     | 110（默认）           |

**`enable_2d_training` — 2D Training 开关**

控制 PHY Training 是否执行 2D（二维）训练。1D Training 仅优化时序中心点，2D Training 在此基础上同时扫描电压裕量，生成更精确的训练结果。

| 值  | 含义                                                         |
|-----|--------------------------------------------------------------|
| `1` | 启用 2D Training（默认），训练时间较长，信号裕量更优         |
| `0` | 禁用 2D Training，仅执行 1D Training，训练时间缩短约 50–90% |

> 量产环境建议保持启用（`1`）。仅在调试阶段需要快速迭代时临时禁用。

### 3.2 修改 IO 参数

修改 `ddr_default_io_para[]` 中对应类型的行即可，LPDDR5 和 LPDDR4X 独立配置：

```c
// drivers/ddr/spacemit/k3/lpddr5_init.c
static const ddr_config_t ddr_default_io_para[] = {
    // 修改 LPDDR5 参数示例：将写驱动从 PHY_R_30 改为 PHY_R_40
    { DDR_TYPE_LPDDR5,  PHY_R_40, PHY_R_60, R_60, R_80, R_OFF, R_OFF, R_40, 1 },
    // 修改 LPDDR4X 参数示例：禁用 2D Training
    { DDR_TYPE_LPDDR4X, PHY_R_40, PHY_R_40, R_60, R_40, R_OFF, R_40,  R_40, 0 },
};
```

注意事项：
- LPDDR4X 的 `nt_odt` 字段无效，驱动不会将其写入寄存器
- `build_lpddr4x_io_para()` 中 `add_ddr_tx_odt_config()` 同时处理 `dq_odt`、`ca_odt`、`soc_odt`、`pdds` 四个字段，修改时需整体考虑

## 4. 调试

### 4.1 启动日志

启动时会打印当前使用的模式、颗粒信息和耗时，可用于确认配置是否生效：

```
DDR Part Number: MT62F2G32D4DS, Size: 8192MB, Data Rate: 6400MT/s
DDR training consume 350ms      # Training 模式（首次启动）
# 或
DDR quick boot consume 15ms     # Quick Boot 模式（后续启动）
```

### 4.2 内存验证

DDR 初始化完成后，驱动会自动执行内存读写验证（`test_pattern()`），验证范围为 `CONFIG_SYS_SDRAM_BASE` 起始的 `0x4000` 字节：

```
memory verify pass    # 验证通过，正常继续启动
memory verify fail!   # 验证失败，系统挂起
```

验证失败通常表明 DDR 初始化参数与实际颗粒不匹配，需检查料号配置或 IO 参数。

### 4.3 详细调试日志

如需更多初始化细节，修改 `drivers/ddr/spacemit/k3/k3_ddr.h` 中的日志等级：

```c
#define LOGLEVEL        0    /* 增大数值可输出更多初始化日志 */
```

SPL 日志等级默认为 1（仅 emergency 级别），如需更多日志，在 `menuconfig` 中调高 `CONFIG_SPL_LOGLEVEL`（注意 `FSBL.bin` 大小限制）。

## 5. 编译与部署

修改完成后，按以下步骤生效：

1. 重新编译 U-Boot，生成 `FSBL.bin` 和 `u-boot.itb`
2. 替换镜像包中的对应文件
3. 烧录镜像至设备
