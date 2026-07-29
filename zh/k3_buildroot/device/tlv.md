# K3 产线写号说明

## 1. 背景

使用 K3 的产品在产线量产时，需在线烧写和启动之前，预先将部分配置数据写入设备 EEPROM。
上电启动后，系统通过读取 EEPROM 中的信息获取以下设备参数：

1. 板型
2. DDR 料号
3. 序列号
4. 网卡地址
5. 网卡数量
6. 产品型号
7. DCDC 型号

其中**板型**、**DDR 料号**与 **DCDC 型号**为必填项。

## 2. TLV 数据结构

K3 EEPROM 采用 ONIE TLV（Type-Length-Value）标准格式存储配置数据。TLV 格式具有良好的扩展性，
便于新增字段而不影响向后兼容。

### 2.1 整体结构

EEPROM 中的数据按以下结构组织：

```
+-------------------+
|   TLV Header      |  (11 bytes)
+-------------------+
|   TLV Entry 1     |  (Type + Length + Value)
+-------------------+
|   TLV Entry 2     |
+-------------------+
|       ...         |
+-------------------+
|   TLV Entry N     |
+-------------------+
|   CRC-32          |  (6 bytes: Type 0xFE + Length 0x04 + CRC Value)
+-------------------+
```

### 2.2 TLV Header

Header 固定为 11 字节：

| 字段 | 长度 | 说明 |
|---|---|---|
| ID String | 8 bytes | 固定为 `"TlvInfo"` (0x54 0x6C 0x76 0x49 0x6E 0x66 0x6F 0x00) |
| Version | 1 byte | TLV 格式版本，当前为 `0x01` |
| Total Length | 2 bytes | TLV 数据区总长度（big-endian），包含所有 TLV entries 和 CRC |

### 2.3 TLV Entry 格式

每个 TLV Entry 由三部分组成：

```
+------+--------+-----------------+
| Type | Length |     Value       |
+------+--------+-----------------+
1 byte  1 byte   Length bytes
```

- **Type**：1 字节，标识字段类型（TLV Code）
- **Length**：1 字节，Value 字段的长度（0-255）
- **Value**：可变长度，实际数据内容

### 2.4 TLV Code 定义

K3 使用的 TLV Code 包括 ONIE 标准字段和 SpaceMIT 自定义字段：

#### 2.4.1 ONIE 标准字段

| TLV Code | 字段名 | 说明 | 数据类型 |
|---|---|---|---|
| `0x21` | product_name | 产品名称（板型） | string |
| `0x22` | part# | 产品零件号 | string |
| `0x23` | serial# | 序列号 | string |
| `0x24` | ethaddr | 基础 MAC 地址 | mac address (xx:xx:xx:xx:xx:xx) |
| `0x25` | manufacture_date | 生产日期 | string[19] (MM/DD/YYYY hh:mm:ss) |
| `0x26` | device_version | 设备版本号 | uint8 |
| `0x27` | LabelRevision | 标签版本 | string |
| `0x28` | PlatformName | 平台名称 | string |
| `0x29` | ONIEVersion | ONIE 版本 | string |
| `0x2A` | ethsize | MAC 地址数量 | uint16 |
| `0x2B` | manufacturer | 制造商名称 | string |
| `0x2C` | CountryCode | 制造国家代码 | string |
| `0x2D` | VendorName | 厂商名称 | string |
| `0x2E` | DiagVersion | 诊断程序版本 | string |
| `0x2F` | ServiceTag | 服务标签 | string |
| `0xFD` | VendorExtension | 厂商自定义扩展 | byte array |
| `0xFE` | CRC-32 | 校验和（必须位于最后） | uint32 |

#### 2.4.2 SpaceMIT 自定义字段

| TLV Code | 字段名 | 说明 | 数据类型 |
|---|---|---|---|
| `0x40` | sdk_version | SDK 版本号 | string |
| `0x45` | ddr_partnumber | DDR 料号 | string |
| `0x60` | wifi_addr | WiFi MAC 地址 | mac address (xx:xx:xx:xx:xx:xx) |
| `0x61` | bt_addr | 蓝牙 MAC 地址 | mac address (xx:xx:xx:xx:xx:xx) |
| `0x80` | pmic_type | DCDC 型号 | string |
| `0x83` | SecondBootDev | 第二级启动介质 | string |

### 2.5 数据示例

以下是一个简化的 TLV 数据示例（十六进制表示）：

```
54 6C 76 49 6E 66 6F 00          // Header: "TlvInfo"
01                                // Version: 0x01
00 3A                             // Total Length: 58 bytes

21 09 6B 33 5F 63 6F 6D 32 36 30 // Type 0x21, Len 9, Value "k3_com260"
23 10 31 32 33 34 35 36 37 38 ... // Type 0x23, Len 16, Value "12345678ABCDEFG"
24 06 FE FE FE 01 02 03           // Type 0x24, Len 6, Value MAC address
2A 02 00 02                       // Type 0x2A, Len 2, Value 2 (MAC count)
...
FE 04 12 34 56 78                 // Type 0xFE, Len 4, CRC-32 value
```

### 2.6 读写流程

uboot 命令行支持如下命令：

- **读取**：`tlv_eeprom` 从 EEPROM 读取全部数据，解析 TLV 结构并显示
- **修改**：
  1. `tlv_eeprom read`：将 EEPROM 内容读入内存缓冲区
  2. `tlv_eeprom set <code> <value>`：在缓冲区中修改或新增 TLV Entry
  3. `tlv_eeprom write`：重新计算 Total Length 和 CRC-32，写回 EEPROM

## 3. 写号

写号支持以下两种方式：

- **fastboot**：通过 PC 端命令行调用 fastboot 写入，适合产线批量操作；也可使用 Titan Flasher 工具
- **uboot shell**：设备进入 uboot 命令行后，使用 `tlv_eeprom` 命令直接写入，便于调试

### 3.1 fastboot

如通过 fastboot 写入，需按如下方法完成准备工作。

#### 3.1.1 设备状态

写号前，设备需进入**烧写模式**。可通过以下任一方式触发：

1. 上电时按住设备上的强制烧录键
2. 擦除设备启动介质上的 bootloader

#### 3.1.2 工具

选择以下工具之一：

1. **fastboot**

   从[官网](https://developer.android.com/tools/releases/platform-tools?hl=zh-cn)下载最新 fastboot 工具包，并将其路径添加到环境变量 `PATH`。

2. **titanflasher**

   从[ spacemit 官方渠道](https://spacemit.com/community/resources-download/Tools/Flashing%20tool)下载并安装。

#### 3.1.3 烧写服务

烧写服务程序运行于目标板上，与 PC 端通过 fastboot 协议通信，接收写号命令和数据并更新 EEPROM。
该程序也可从 titanflasher 安装目录中获取。

MD5 校验：

```
c31cc9e89bb188a22801d584b1a29f43  SPA3609_v1.2.0.bin
```

在 PC 端使用 fastboot 写号前，还需将烧写服务程序推送到设备并运行：

```bash
fastboot stage SPA3609.bin
fastboot continue
```

### 3.2 写入

完成准备工作后，依次执行以下各节中的写入命令，将命令中的**示例值**替换为实际值后执行。

#### 3.2.1 板型

```bash
fastboot oem config:write product_name@eeprom:k3_com260
fastboot oem config:flush
```

**uboot shell（TLV code: `0x21`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x21 k3_com260
=> tlv_eeprom write
```

#### 3.2.2 DDR 料号

K3 使用两颗 DDR，写入的是其中一颗的料号。不同厂家的料号可能共用同一份驱动配置，
需从下表查找对应的**写入料号**后填入命令：

| DDR 料号 | 写入料号 | 容量 | 类型 | 速率 |
|---|---|---|---|---|
| MT53E1G32D2FW-046 WT:C | MT53E1G32D2FW | 4 GB | LPDDR4x | 4266 MT/s |
| MT53E2G32D4DE-046 WT:C | MT53E2G32D4DE | 8 GB | LPDDR4x | 4266 MT/s |
| MT53E4G32D8CY-046 WT:C | MT53E4G32D8CY | 16 GB | LPDDR4x | 4266 MT/s |
| MT62F1G32D2DS-023 WT:C | MT62F1G32D2DS | 4 GB | LPDDR5 | 6400 MT/s |
| MT62F2G32D4DS-023 WT:C<br>RS2G32LO5D4FDB-31BT | MT62F2G32D4DS | 8 GB | LPDDR5 | 6400 MT/s |
| RS3G32LG5D8FDB-31BT | RS3G32LG5D8FDB | 12 GB | LPDDR5 | 6400 MT/s |
| MT62F4G32D8DV-023 WT:C<br>RS4G32LO5D8FDB-31BT | MT62F4G32D8DV | 16 GB | LPDDR5 | 6400 MT/s |

```bash
fastboot oem config:write ddr_partnumber@eeprom:MT62F4G32D8DV
fastboot oem config:flush
```

**uboot shell（TLV code: `0x45`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x45 MT62F4G32D8DV
=> tlv_eeprom write
```

#### 3.2.3 DCDC 型号

不同板型使用不同的 DCDC 为 Core 供电。根据板上 DCDC 物料名称从下表查找对应写入型号，未写入时默认为 `mpq8655`。

| DCDC 料号 | 写入型号 | 对应板型 |
|---|---|---|
| mpq8655 | mpq8655 | com260 / deb1 / evb / dc_board / com260_kit_v20 / gemini_c0 / gemini_c1 |
| tda38740 | tda38740 | evb2-1 |
| is6615a | is6615a | evb2-2 |
| au4562 | au4562 | pico-itx |

```bash
fastboot oem config:write pmic_type@eeprom:mpq8655
fastboot oem config:flush
```

**uboot shell（TLV code: `0x80`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x80 mpq8655
=> tlv_eeprom write
```

#### 3.2.4 序列号

```bash
fastboot oem config:write serial#@eeprom:12345678ABCDEFG
fastboot oem config:flush
```

**uboot shell（TLV code: `0x23`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x23 12345678ABCDEFG
=> tlv_eeprom write
```

#### 3.2.5 网卡地址

MAC 地址格式须符合 IEEE 802 标准（如 `FE:FE:FE:01:02:03`）。

```bash
fastboot oem config:write ethaddr@eeprom:FE:FE:FE:01:02:03
fastboot oem config:flush
```

**uboot shell（TLV code: `0x24`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x24 FE:FE:FE:01:02:03
=> tlv_eeprom write
```

#### 3.2.6 网卡数量

```bash
fastboot oem config:write ethsize@eeprom:2
fastboot oem config:flush
```

**uboot shell（TLV code: `0x2A`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x2A 2
=> tlv_eeprom write
```

#### 3.2.7 产品型号

```bash
fastboot oem config:write part#@eeprom:123456789ABCDEFG
fastboot oem config:flush
```

**uboot shell（TLV code: `0x22`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x22 123456789ABCDEFG
=> tlv_eeprom write
```

#### 3.2.8 第二级启动介质

仅适用于 SPI NOR flash based 方案。未写入时按默认顺序依次尝试各启动介质。

```bash
fastboot oem config:write SecondBootDev@eeprom:SSD
```

**uboot shell（TLV code: `0x83`）：**

```
=> tlv_eeprom read
=> tlv_eeprom set 0x83 SSD
=> tlv_eeprom write
```

支持的启动介质及对应启动顺序如下：

| SecondBootDev | 启动顺序 |
|---|---|
| （未写入） | UFS → NVME → emmc → USB |
| NVME | NVME → UFS → emmc → USB |
| SSD | NVME → UFS → emmc → USB |
| MMC | emmc → UFS → NVME → USB |
| USB | USB → UFS → NVME → emmc |
| PXE | 通过 PXE 服务器启动（服务器 IP 在 env 中指定） |
| PXE@\<IP\> | 通过 PXE 服务器启动（服务器 IP 通过参数指定） |
