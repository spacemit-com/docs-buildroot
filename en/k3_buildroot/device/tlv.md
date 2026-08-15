---
sidebar_position: 8
---

# K3 Production Programming Guide

## 1. Background

Before K3-based products are flashed and booted during mass production, specific configuration data must be preprogrammed into the device EEPROM.
At power-on, the system reads the EEPROM to obtain the following device parameters:

1. Board type
2. DDR part number
3. Serial number
4. Ethernet address
5. Number of MAC addresses
6. Product model
7. DCDC model

The **board type**, **DDR part number**, and **DCDC model** are required.

## 2. TLV Data Structure

The K3 EEPROM stores configuration data in the standard ONIE TLV (Type-Length-Value) format. This extensible format allows new fields to be added without affecting backward compatibility.

### 2.1 Overall Structure

EEPROM data is organized as follows:

```text
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

The header has a fixed size of 11 bytes:

| Field | Length | Description |
| - | - | - |
| ID String | 8 bytes | Fixed as `"TlvInfo"` (0x54 0x6C 0x76 0x49 0x6E 0x66 0x6F 0x00) |
| Version | 1 byte | TLV format version, currently `0x01` |
| Total Length | 2 bytes | Total length of the TLV data area (big-endian), including all TLV entries and CRC |

### 2.3 TLV Entry Format

Each TLV entry consists of three parts:

```text
+------+--------+-----------------+
| Type | Length |     Value       |
+------+--------+-----------------+
1 byte  1 byte   Length bytes
```

- **Type**: A 1-byte field that identifies the field type (TLV code).
- **Length**: A 1-byte field that specifies the length of the Value field, from 0 to 255 bytes.
- **Value**: A variable-length field containing the data.

### 2.4 TLV Code Definitions

The TLV codes used by K3 include ONIE standard fields and SpaceMIT-defined fields.

#### 2.4.1 ONIE Standard Fields

| TLV Code | Field Name | Description | Data Type |
| - | - | - | - |
| `0x21` | product_name | Product name (board type) | string |
| `0x22` | part# | Product part number | string |
| `0x23` | serial# | Serial number | string |
| `0x24` | ethaddr | Base MAC address | mac address (xx:xx:xx:xx:xx:xx) |
| `0x25` | manufacture_date | Date of manufacture | string[19] (MM/DD/YYYY hh:mm:ss) |
| `0x26` | device_version | Device version | uint8 |
| `0x27` | LabelRevision | Label revision | string |
| `0x28` | PlatformName | Platform name | string |
| `0x29` | ONIEVersion | ONIE version | string |
| `0x2A` | ethsize | Number of MAC addresses | uint16 |
| `0x2B` | manufacturer | Manufacturer name | string |
| `0x2C` | CountryCode | Country code of manufacture | string |
| `0x2D` | VendorName | Vendor name | string |
| `0x2E` | DiagVersion | Diagnostic software version | string |
| `0x2F` | ServiceTag | Service tag | string |
| `0xFD` | VendorExtension | Vendor-specific extension | byte array |
| `0xFE` | CRC-32 | Checksum (must be last) | uint32 |

#### 2.4.2 SpaceMIT Custom Fields

| TLV Code | Field Name | Description | Data Type |
| - | - | - | - |
| `0x40` | sdk_version | SDK version | string |
| `0x45` | ddr_partnumber | DDR part number | string |
| `0x60` | wifi_addr | WiFi MAC address | mac address (xx:xx:xx:xx:xx:xx) |
| `0x61` | bt_addr | Bluetooth MAC address | mac address (xx:xx:xx:xx:xx:xx) |
| `0x80` | pmic_type | DCDC model | string |
| `0x83` | SecondBootDev | Secondary boot device | string |

### 2.5 Data Example

The following is a simplified TLV data example in hexadecimal format:

```text
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

### 2.6 Read and Write Operations

The U-Boot command line supports the following commands:

- **Read**: `tlv_eeprom` reads all data from EEPROM, parses the TLV structure, and displays the contents.
- **Modify**:
   1. `tlv_eeprom read` reads EEPROM content into the memory buffer.

   2. `tlv_eeprom set <code> <value>` modifies an existing TLV entry or adds a new entry to the buffer.

   3. `tlv_eeprom write` recalculates Total Length and CRC-32, then writes the buffer back to EEPROM.

## 3. Production Programming

Production programming is supported through the following methods:

- **fastboot**: Writes data using fastboot commands issued from a PC. This method is suitable for high-volume production; the Titan Flasher tool can also be used.
- **U-Boot shell**: Writes data directly with the `tlv_eeprom` command after the device enters the U-Boot command line. This method is suitable for debugging.

### 3.1 fastboot

Complete the following preparation steps before programming through fastboot.

#### 3.1.1 Device State

Before programming, the device must be placed in **flashing mode**. This mode can be entered by either of the following methods:

1. Hold the force-flash button during power-on.
2. Erase the bootloader from the device boot media.

#### 3.1.2 Tools

Select one of the following tools:

1. **fastboot**

   Download the latest fastboot package from the [official website](https://developer.android.com/tools/releases/platform-tools?hl=zh-cn), then add the installation path to the `PATH` environment variable.

2. **titanflasher**

   Download and install the tool from the [SpaceMIT official channel](https://spacemit.com/community/resources-download/Tools/Flashing%20tool).

#### 3.1.3 Flashing Service

The flashing service runs on the target board and communicates with the PC through the fastboot protocol. It receives programming commands and data, then updates the EEPROM.
The program is also available in the titanflasher installation directory.

MD5 checksum:

```text
c31cc9e89bb188a22801d584b1a29f43  SPA3609_v1.2.0.bin
```

Before issuing fastboot programming commands from the PC, stage and run the flashing service on the device:

```bash
fastboot stage SPA3609.bin
fastboot continue
```

### 3.2 Writing Data

After completing the preparation steps, execute the write commands in the following sections in sequence. Replace the **sample values** in each command with the applicable values.

#### 3.2.1 Board Type

```bash
fastboot oem config:write product_name@eeprom:k3_com260
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x21`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x21 k3_com260
=> tlv_eeprom write
```

#### 3.2.2 DDR Part Number

K3 uses two DDR devices; the part number of one device is programmed. Part numbers from different manufacturers can share the same driver configuration. Use the **programmed part number** corresponding to the installed DDR part number in the following table:

| DDR Part Number | Programmed Part Number | Capacity | Type | Data Rate |
| - | - | - | - | - |
| MT53E1G32D2FW-046 WT:C | MT53E1G32D2FW | 4 GB | LPDDR4x | 4266 MT/s |
| MT53E2G32D4DE-046 WT:C | MT53E2G32D4DE | 8 GB | LPDDR4x | 4266 MT/s |
| MT53E4G32D8CY-046 WT:C | MT53E4G32D8CY | 16 GB | LPDDR4x | 4266 MT/s |
| MT62F1G32D2DS-023 WT:C | MT62F1G32D2DS | 4 GB | LPDDR5 | 6400 MT/s |
| MT62F2G32D4DS-023 WT:C; RS2G32LO5D4FDB-31BT | MT62F2G32D4DS | 8 GB | LPDDR5 | 6400 MT/s |
| RS3G32LG5D8FDB-31BT | RS3G32LG5D8FDB | 12 GB | LPDDR5 | 6400 MT/s |
| MT62F4G32D8DV-023 WT:C; RS4G32LO5D8FDB-31BT | MT62F4G32D8DV | 16 GB | LPDDR5 | 6400 MT/s |

```bash
fastboot oem config:write ddr_partnumber@eeprom:MT62F4G32D8DV
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x45`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x45 MT62F4G32D8DV
=> tlv_eeprom write
```

#### 3.2.3 DCDC Model

Different board types use different DCDCs to supply the core power rail. Identify the DCDC part name on the board, then use the corresponding programmed model in the following table. If no value is programmed, the default is `mpq8655`.

| DCDC Part Number | Programmed Model | Applicable Board Type |
| - | - | - |
| mpq8655 | mpq8655 | com260 / deb1 / evb / dc_board / com260_kit_v20 / gemini_c0 / gemini_c1 |
| tda38740 | tda38740 | evb2-1 |
| is6615a | is6615a | evb2-2 |
| au4562 | au4562 | pico-itx |

```bash
fastboot oem config:write pmic_type@eeprom:mpq8655
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x80`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x80 mpq8655
=> tlv_eeprom write
```

#### 3.2.4 Serial Number

```bash
fastboot oem config:write serial#@eeprom:12345678ABCDEFG
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x23`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x23 12345678ABCDEFG
=> tlv_eeprom write
```

#### 3.2.5 MAC Address

The MAC address must conform to the IEEE 802 format, for example, `FE:FE:FE:01:02:03`.

```bash
fastboot oem config:write ethaddr@eeprom:FE:FE:FE:01:02:03
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x24`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x24 FE:FE:FE:01:02:03
=> tlv_eeprom write
```

#### 3.2.6 MAC Address Count

```bash
fastboot oem config:write ethsize@eeprom:2
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x2A`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x2A 2
=> tlv_eeprom write
```

#### 3.2.7 Product Model

```bash
fastboot oem config:write part#@eeprom:123456789ABCDEFG
fastboot oem config:flush
```

**U-Boot shell (TLV code: `0x22`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x22 123456789ABCDEFG
=> tlv_eeprom write
```

#### 3.2.8 Secondary Boot Device

This setting applies only to SPI NOR flash-based solutions. If no value is programmed, the boot media are tried in the default order.

```bash
fastboot oem config:write SecondBootDev@eeprom:SSD
```

**U-Boot shell (TLV code: `0x83`):**

```text
=> tlv_eeprom read
=> tlv_eeprom set 0x83 SSD
=> tlv_eeprom write
```

The supported boot media and corresponding boot orders are as follows:

| SecondBootDev | Boot Order |
| - | - |
| (not programmed) | UFS -> NVME -> emmc -> USB |
| NVME | NVME -> UFS -> emmc -> USB |
| SSD | NVME -> UFS -> emmc -> USB |
| MMC | emmc -> UFS -> NVME -> USB |
| USB | USB -> UFS -> NVME -> emmc |
| PXE | Boot from a PXE server (server IP specified in the environment) |
| PXE@\<IP\> | Boot from a PXE server (server IP specified as a parameter) |
