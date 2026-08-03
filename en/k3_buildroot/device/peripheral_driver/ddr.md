# DDR Driver Configuration Guide

| Version | Date | Description |
|------|------------|----------|
| v1.0 | 2026-04-17 | Initial release |

## 1. Overview

This document explains how to modify DDR driver parameters on the **SpacemiT K3** platform. The goal is to adapt the K3 SDK to a specific DDR device. It covers the following topics:

- DDR part number configuration and device matching
- Data rate configuration
- IO electrical parameter tuning, including ODT, drive strength, and 2D training
- Debugging methods and log interpretation

### Supported DDR Types

The K3 SDK supports two DDR types: **LPDDR4X** and **LPDDR5**. It automatically matches initialization parameters by DDR part number.

The built-in supported device list is defined in `drivers/ddr/spacemit/k3/ddr_init.c`.

**LPDDR5**

| Programmed DDR part number | Ranks | x8 | Capacity | Data rate (MT/s) | Compatible device models |
|-----------------|-------|----|----------|-------------|-----------------------------------------------------------------------|
| `MT62F1G32D2DS` | 1     | No | 4096 MB  | 6400        | MT62F1G32D2DS-023 WT:C, CXDD5DBAM-MC-M                               |
| `MT62F2G32D4DS` | 2     | No | 8192 MB  | 6400        | MT62F2G32D4DS-023 WT:C, RS2G32LO5D4FDB-31BT, CXDD6DCBM-MC-M         |
| `MT62F4G32D8DV` | 2     | Yes | 16384 MB | 6400        | MT62F4G32D8DV-023 WT:C, RS4G32LO5D8FDB-31BT                          |

**LPDDR4X**

| Programmed DDR part number | Ranks | x8 | Capacity | Data rate (MT/s) | Compatible device models |
|-----------------|-------|----|----------|-------------|----------------------------|
| `MT53E1G32D2FW` | 1     | No | 4096 MB  | 4266        | MT53E1G32D2FW-046 WT:C     |
| `MT53E2G32D4DE` | 2     | No | 8192 MB  | 4266        | MT53E2G32D4DE-046 WT:C     |
| `MT53E4G32D8CY` | 2     | Yes | 16384 MB | 4266        | MT53E4G32D8CY-046 WT:C     |

The **programmed DDR part number** is the string written to EEPROM or to the DTS `part-number` field.

For a compatible device with the same specifications but a different part number:

- use the corresponding mapped part number
- refer to the **Compatible device models** column for valid mappings
- reuse the existing parameter set without adding a new code entry

### Configuration Mechanism

The K3 DDR driver matches device parameters by **DDR part number**. The priority order is as follows:

```
EEPROM (TLV 0x45)  →  DTS part-number  →  code default (`ddr_parts_info[0]`)
    highest priority                                 lowest priority
```

1. First read `ddr_partnumber` from EEPROM using TLV code `0x45`
2. If EEPROM does not contain a valid configuration, read the `part-number` property from the DTS node
3. If neither is set, use the first entry in `ddr_parts_info[]` as the default

### Boot Modes

The DDR driver supports two boot modes:

- **Training mode**: On first boot, or when no valid training result is available, the driver runs the full PHY training flow. This process usually takes several seconds. After training completes, the result is saved to the memory buffer at `DDR_TRAINING_INFO_BUFF = 0xC08D0000`. It is then synchronized back into the code during image programming.
- **Quick Boot mode**: When a valid training result cache is detected, the driver restores the PHY parameters directly. It then skips training, which significantly reduces boot time.

Quick Boot is valid only when all of the following match: the magic value (`0x54524444`), DDR type, chip-select count, data rate, and CRC32.

## 2. Basic Configuration

### 2.1 DDR Part Number Configuration

K3 automatically matches DDR type, capacity, data rate, and related parameters from the DDR part number. Separate configuration of these items is not required.

Select one of the following methods based on the actual use case.

#### Scenario 1: Configure Through EEPROM, Without Reflashing the Image

This method is suitable for devices that can already boot normally. Changes take effect immediately and do not require recompilation.

**Method 1: Write with the TitanFlasher Tool**

- Power on the device while holding the flashing button to enter flashing mode
- then connect the device to the PC via USB
- Use the part-number programming function in the TitanFlasher toolset to write the DDR part number (`ddr_partnumber`).

![Writing a DDR part number with TitanFlasher](static/ddr_part_number.png)

**Method 2: Programming via U-Boot Command Line**

Prerequisite: DDR must initialize successfully with the default configuration, and the device must be able to boot to U-Boot.

```shell
=> tlv_eeprom read
=> tlv_eeprom set 0x45 MT62F2G32D4DS
=> tlv_eeprom writeb 
```

#### Scenario 2: Use a Fixed DDR Part Number (No EEPROM on Board)

Modify the `part-number` field in the DDR node of `arch/riscv/dts/k3_spl.dts`. Recompilation is required for the change to take effect.

```dts
ddr@cb000000 {
    u-boot,dm-spl;
    compatible = "spacemit,snps-lp45";
    part-number = "MT62F2G32D4DS";    /* Change to the actual part number in use */
    reg = <0x00000000 0xcb000000 0x00000000 0x00400000>,
          <0x00000000 0xcc000000 0x00000000 0x00400000>;
    reg-names = "ddrc0", "ddrc1";
    status = "okay";
};
```

#### Scenario 3: Add a Device That Is Not in the Built-In List

If the target device is not included in the built-in list, add a new entry to `ddr_parts_info[]` in `drivers/ddr/spacemit/k3/ddr_init.c`. Recompile for the change to take effect.

```c
const ddr_part_info ddr_parts_info[] = {
    /* part_number,    crc32,      type,           ranks, x8, size_mb, data_rate */
    { "MT62F2G32D4DS", 0x85D1F688, DDR_TYPE_LPDDR5, 2,    0,  8192,   CONFIG_DDR_DATARATE },
    /* Example new entry: LPDDR5, dual-rank, 8 GB */
    { "NEW_PART_NUM",  0xEEFF6244, DDR_TYPE_LPDDR5, 2,    0,  8192,   6400 },
};
```

> `crc32_value` is the CRC32 value of the part-number string. It is used to speed up matching. Calculate it as follows:
> ```c
> crc32(0, (const uint8_t*)"NEW_PART_NUM", strlen("NEW_PART_NUM"))
> ```

### 2.2 Data Rate Configuration

The `data_rate_mtps` field in each `ddr_parts_info[]` entry defines the operating data rate for that device.

To change the rate:

- update the value in the target entry directly, or
- modify the `CONFIG_DDR_DATARATE` macro

Changing the macro affects all entries that reference it.

```c
// drivers/ddr/spacemit/k3/k3_ddr.h
// Supported values: 4266 / 5500 / 6000 / 6400 MT/s. Any other value causes a build error.
#define CONFIG_DDR_DATARATE  (6400)
```

## 3. Advanced Configuration

> **Warning:**
> The parameters described in this section directly affect signal integrity.
> Incorrect settings may cause system instability or boot failure.
> It is strongly recommended to make changes under the guidance of a hardware engineer and to perform thorough memory stress testing afterward.

The K3 DDR driver configures electrical parameters through the `ddr_config_t` structure.

- LPDDR5 and LPDDR4X use the same structure definition.
- Default values are defined in the `ddr_default_io_para[]` array in `drivers/ddr/spacemit/k3/ddr_init.c`.

```c
static const ddr_config_t ddr_default_io_para[] = {
    // type,            WDS       RX_ODT    DQ_ODT CA_ODT NT_ODT SOC_ODT PDDS   2D
    { DDR_TYPE_LPDDR5,  PHY_R_30, PHY_R_60, R_60,  R_80,  R_OFF, R_OFF,  R_40,  1 },
    { DDR_TYPE_LPDDR4X, PHY_R_40, PHY_R_40, R_60,  R_40,  R_OFF, R_40,   R_40,  1 },
};
```

At boot time, `lpddr_init_prepare()` automatically selects the matching row based on the DDR type.

It then calls `build_lpddr5_io_para()` or `build_lpddr4x_io_para()` to:

- convert the parameters into PHY register settings
- write the settings to hardware

### 3.1 IO Parameter Descriptions

**`phy_write_ds` — PHY-side write drive strength**

This field controls the output drive impedance of the DQ/DQS signals on the SoC PHY side.

- Lower impedance provides stronger drive strength.
- Lower impedance also produces steeper signal edges.
- At the same time, it increases the risk of reflections and crosstalk.

| Enum value | Effective impedance | Description |
|-------------|----------|-----------------------------------|
| `PHY_R_OFF` | High-Z | Output disabled, not used in normal operation |
| `PHY_R_120` | 120 Ω | Long traces and high-impedance loads |
| `PHY_R_60`  | 60 Ω | Medium trace length |
| `PHY_R_40`  | 40 Ω | Shorter traces, default for LPDDR4X |
| `PHY_R_30`  | 30 Ω | Short traces, default for high-speed LPDDR5 |

> The LPDDR5 specification (JESD209-5) recommends matching the SoC-side drive impedance to the trace characteristic impedance, typically 40–50 Ω. The final value should be determined with PCB simulation.

**`phy_rx_odt` — PHY-side receive termination (RX ODT)**

This field controls the on-die termination resistance on the DQ/DQS input side of the SoC PHY.

- It is enabled during reads to absorb reflections.
- It is typically disabled during writes.

| Enum value | Effective impedance | Description |
|-------------|----------|-------------------------------------------|
| `PHY_R_OFF` | High-Z | RX ODT disabled |
| `PHY_R_120` | 120 Ω | Weak termination, suitable for long traces with low reflection |
| `PHY_R_60`  | 60 Ω | Default for LPDDR5, used with DRAM-side PDDS |
| `PHY_R_40`  | 40 Ω | Default for LPDDR4X |
| `PHY_R_30`  | 30 Ω | Strong termination, suitable for very short traces |

> A typical combination is `phy_rx_odt=PHY_R_60` (60 Ω) + `pdds=R_40` (40 Ω). In this case, the DRAM drive impedance is lower than the SoC termination impedance, which helps maintain signal levels.

**`dq_odt` — DRAM-side DQ termination resistance (`MR11[2:0]`)**

This field controls the on-die termination resistance of the internal DQ bus in the DRAM device.

- During writes, DRAM enables ODT to reduce reflections.
- During reads, ODT is disabled.

| Enum value | Effective impedance | LPDDR5 MR11 encoding |
|---------|----------|-----------------|
| `R_OFF` | Disabled | 000 |
| `R_240` | 240 Ω    | 001             |
| `R_120` | 120 Ω    | 010             |
| `R_80`  | 80 Ω     | 011             |
| `R_60`  | 60 Ω     | 100 (default) |
| `R_48`  | 48 Ω     | 101             |
| `R_40`  | 40 Ω     | 110             |

> A typical combination is `phy_write_ds=PHY_R_30` (30 Ω) + `dq_odt=R_60` (60 Ω).

**`ca_odt` — DRAM-side CA termination resistance (`MR11[6:4]`)**

This field controls the on-die termination resistance of the internal CA (Command/Address) bus in the DRAM device.

- The CA bus is unidirectional, from SoC to DRAM.
- DRAM-side ODT is therefore used to absorb CA signal reflections.
- The supported values are the same as for `dq_odt`.

> In the LPDDR5 specification, a typical CA ODT value is 80 Ω. This is commonly matched with a SoC-side CA drive impedance of about 40 Ω.

**`nt_odt` — non-target termination (NT ODT, `MR41[7:5]`) ⚠️ LPDDR5 only**

NT ODT is a feature introduced in LPDDR5 and is not supported by **LPDDR4X**. This field is ignored for LPDDR4X, and the driver does not write it to registers.

In multi-rank configurations, write operations target only one rank.

- NT ODT is enabled on the non-target rank to suppress bus reflections and crosstalk.
- In single-rank configurations, this field must be set to `R_OFF`.
- The supported values are the same as for `dq_odt`.

> Setting this field to any value other than `R_OFF` for a single-rank device (`ranks=1`) increases power consumption without providing any practical benefit.

**`soc_odt` — SoC-side termination resistance (SOC ODT, `MR17[2:0]`)**

This field controls the DRAM termination setting associated with the SoC side.

- It is used to improve reflection characteristics at the SoC end.
- The supported values are the same as for `dq_odt`.

> This parameter depends heavily on PCB routing topology and is usually determined through signal integrity simulation. The default is disabled (`R_OFF`) for LPDDR5 and 40 Ω for LPDDR4X.

**`pdds` — DRAM-side output drive strength (Pull-Down Drive Strength, `MR3[2:0]`)**

This field controls the DQ/DQS output drive impedance of the DRAM device.

- During reads, DRAM drives the data bus.
- This value therefore determines DRAM-side drive strength.

| Enum value | Effective impedance | LPDDR5 MR3 encoding |
|---------|----------|-----------------------|
| `R_OFF` | High-Z | 000 (not used in normal operation) |
| `R_240` | 240 Ω    | 001                   |
| `R_120` | 120 Ω    | 010                   |
| `R_80`  | 80 Ω     | 011                   |
| `R_60`  | 60 Ω     | 100                   |
| `R_48`  | 48 Ω     | 101                   |
| `R_40`  | 40 Ω     | 110 (default) |

**`enable_2d_training` — 2D training enable**

This field controls whether PHY training runs 2D training.

- 1D training optimizes only the timing center point.
- 2D training also scans voltage margin.
- As a result, 2D training produces a more accurate training result.

| Value | Meaning |
|-----|--------------------------------------------------------------|
| `1` | Enable 2D training (default). Training takes longer, but signal margin is better. |
| `0` | Disable 2D training and run only 1D training. Training time is reduced by about 50–90%. |

> For production systems, keep this option enabled (`1`). Disable it only temporarily during debug when faster iteration is needed.

### 3.2 Modifying IO Parameters

To change IO parameters, modify the row for the corresponding DDR type in `ddr_default_io_para[]`.

- LPDDR5 and LPDDR4X are configured independently.

```c
// drivers/ddr/spacemit/k3/ddr_init.c
static const ddr_config_t ddr_default_io_para[] = {
    // LPDDR5 example: change write drive strength from PHY_R_30 to PHY_R_40
    { DDR_TYPE_LPDDR5,  PHY_R_40, PHY_R_60, R_60, R_80, R_OFF, R_OFF, R_40, 1 },
    // LPDDR4X example: disable 2D training
    { DDR_TYPE_LPDDR4X, PHY_R_40, PHY_R_40, R_60, R_40, R_OFF, R_40,  R_40, 0 },
};
```

Notes:

- The `nt_odt` field is invalid for LPDDR4X, and the driver does not write it to registers.
- In `build_lpddr4x_io_para()`, `add_ddr_tx_odt_config()` handles `dq_odt`, `ca_odt`, `soc_odt`, and `pdds` together. Review these four fields as a set when making changes.

## 4. Debugging

### 4.1 Boot Log

At boot time, the driver prints the current mode, device information, and elapsed time.

Use this output to confirm that the configuration has taken effect.

```
DDR Part Number: MT62F2G32D4DS, Size: 8192MB, Data Rate: 6400MT/s
DDR training consume 350ms      # Training mode, first boot
# or
DDR quick boot consume 15ms     # Quick Boot mode, subsequent boots
```

### 4.2 Memory Verification

After DDR initialization completes, the driver automatically runs a memory read/write verification test through `test_pattern()`. The test range is `0x4000` bytes starting at `CONFIG_SYS_SDRAM_BASE`.

```
memory verify pass    # Verification passed, boot continues normally
memory verify fail!   # Verification failed, system halts
```

A verification failure usually indicates that the DDR initialization parameters do not match the actual device.

Check the following items:

- the part-number configuration
- the IO parameters

### 4.3 Detailed Debug Logs

To print more initialization details, adjust the log level in `drivers/ddr/spacemit/k3/k3_ddr.h`:

```c
#define LOGLEVEL        0    /* Increase this value to print more initialization logs */
```

The default SPL log level is 1. At this level, only emergency logs are printed.

To enable more output:

- increase `CONFIG_SPL_LOGLEVEL` in `menuconfig`
- keep the `FSBL.bin` size limit in mind

## 5. Build and Deployment

After making changes, apply them with the following steps:

1. Rebuild U-Boot to generate `FSBL.bin` and `u-boot.itb`
2. Replace the corresponding files in the image package
3. Flash the image to the device