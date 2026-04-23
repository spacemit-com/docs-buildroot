# QSPI

This document describes QSPI functionality and usage.

## Module Overview

QSPI (Quad SPI) is a serial interface bus between the SoC and peripheral devices that supports x4 mode. SPI supports both master and slave modes. Typically, one master device is connected to one or more slave devices. The master selects the target slave for communication, provides the clock, and initiates all read and write operations. K3 QSPI currently supports master mode only.

### Functional Overview

![](static/linux_spi.png)  

The Linux SPI driver framework is divided into three parts: **SPI core**, **SPI controller driver**, and **SPI device driver**.
**SPI core** mainly provides the following functions:

- Registers SPI buses and `spi_master` classes
- Adds and removes SPI controllers
- Adds and removes SPI devices
- Registers and unregisters SPI device drivers

**SPI controller driver**:

- The SPI master controller driver, which operates the SPI master controller

**SPI device driver**:

- The SPI device driver

### Source Code Structure

The controller driver code is located in the `drivers/spi` directory:

```
|-- spi-fsl-qspi.c              #k3 qspi driver
```

## Key Features

### Feature

| Feature | Description |
| :-----| :----|
| Communication Protocol | Supports SSP/SPI/MicroWire/PSP protocols |
| Communication Frequency | Supports a maximum frequency of 102MHz and a minimum frequency of 30MHz |
| Communication Multiplier | x1/x2/x4 |
| Supported Peripherals  | Supports `spi-nor` and `spi-nand` |

### Performance Parameters

#### Communication Frequency

The current QSPI controller supports up to 102MHz. The list of supported communication frequencies is as follows:

- 30.72MHz
- 35.10MHz
- 40.96MHz
- 53.57MHz
- 68.26MHz
- 75MHz
- 81.92MHz
- 93.75MHz
- 98.3MHz
- 102.4MHz

#### Communication Multiplier

QSPI supports communication multipliers of x1/x2/x4.

Testing method:
Use an oscilloscope or logic analyzer to measure the SCK frequency and confirm the communication multiplier.

## Configuration

Configuration mainly includes driver enablement and DTS settings.

### CONFIG Configuration

- `CONFIG_SPI`: Provides support for the SPI bus protocol. The default value is `Y`.
```
Device Drivers
        SPI support (SPI [=y])
```

- `CONFIG_SPI_MEM`: Provides support for simplified operations on SPI memory devices. The default value is `Y`.
```
Device Drivers
        SPI support (SPI [=y])
                SPI memory extension (SPI_MEM [=y])
```

- `CONFIG_SPI_FSL_QUADSPI`: Provides support for the K3 QSPI controller driver. The default value is `Y`.
```
Device Drivers
        SPI support (SPI [=y])
                Freescale QSPI controller (SPI_FSL_QUADSPI [=y])

```

### DTS Configuration

#### pinctrl

Check the board schematic to determine which pin group is used by QSPI. You can refer to the **Pin Configuration Definitions** section in [PINCTRL](01-PINCTRL.md) to identify the QSPI pin group.

Assume that QSPI can directly use the `pinctrl_qspi` group defined in `k3_pinctrl.dtsi`.

#### SPI Device Configuration

You need to confirm the SPI device type, as well as the communication frequency and multiplier used between QSPI and the SPI device.

##### Device Type

Confirm whether the SPI device connected to QSPI is `spi-nor` or `spi-nand`.

##### Communication Frequency

Confirm the maximum communication rate supported by both the QSPI controller and the SPI device.
The list of communication frequencies supported by the current QSPI controller is given in **Performance Parameters** > **Communication Frequency** in this document.


##### SPI Device DTS

Taking SPI NOR as an example, use a maximum communication frequency of 30MHz, with both transmit and receive operating in x4 mode.

```c
// Specify the voltage required by the pins here. 1.8V is used as an example. For more details, refer to the pinctrl chapter.
&pinctrl {
    qspi-cfg {
        qspi-0-pins {
            power-source = <1800>;
        };

        qspi-1-pins {
            power-source = <1800>;
        };
    };
};

&qspi {
    pinctrl-names = "default";
    pinctrl-0 = <&qspi_cfg>;
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <30000000>;
        m25p,fast-read;
        spi-tx-bus-width = <4>;
        spi-rx-bus-width = <4>;
        broken-flash-reset;
        status = "okay";
    };
};
```

#### DTS Example

##### SPI-NOR

Based on the above information, QSPI is connected to SPI-NOR flash with a maximum communication frequency of 30MHz and x4 communication.

Example DTS configuration:

```c
&pinctrl {
    qspi-cfg {
        qspi-0-pins {
            power-source = <1800>;
        };

        qspi-1-pins {
            power-source = <1800>;
        };
    };
};

&qspi {
    pinctrl-names = "default";
    pinctrl-0 = <&qspi_cfg>;
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <30000000>;
        m25p,fast-read;
        spi-tx-bus-width = <4>;
        spi-rx-bus-width = <4>;
        broken-flash-reset;
        status = "okay";
    };
};
```

##### SPI-NAND

QSPI is connected to SPI-NAND flash with a maximum communication frequency of 30MHz and x4 communication.

The board-level DTS configuration can refer to the SPI-NOR example; only the flash device node needs to be modified.

```c
&pinctrl {
    qspi-cfg {
        qspi-0-pins {
            power-source = <1800>;
        };

        qspi-1-pins {
            power-source = <1800>;
        };
    };
};

&qspi {
    pinctrl-names = "default";
    pinctrl-0 = <&qspi_cfg>;
    status = "okay";

    flash@0 {
        compatible = "spi-nand";
        reg = <0>;
        spi-max-frequency = <30000000>;
        m25p,fast-read;
        spi-tx-bus-width = <4>;
        spi-rx-bus-width = <4>;
        broken-flash-reset;
        status = "okay";
    };
};
```

## Interface

### API

Device driver registration and unregistration:

```
int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);  
void spi_unregister_driver(struct spi_driver *sdrv);
```

Data transfer APIs:

- Initialize `spi_message`

```
void spi_message_init(struct spi_message *m);
```

- Add `spi_transfer` to the transfer list of `spi_message`

```
void spi_message_add_tail(struct spi_transfer *t, struct spi_message *m);
```

- Write data

```
int spi_write(struct spi_device *spi, const void *buf, size_t len);
```

- Read data

```
int spi_read(struct spi_device *spi, void *buf, size_t len);
```

- Synchronously transfer `spi_message`

```
int spi_sync(struct spi_device *spi, struct spi_message *message);
```

## Debugging

### sysfs

View SPI bus device and driver information in the system:
`/sys/bus/spi`

```
|-- devices                 // Devices on the SPI bus
|-- drivers                 // Drivers registered on the SPI bus
|-- drivers_autoprobe
|-- drivers_probe
`-- uevent
```

### debugfs

Used to view SPI device information in the system:
`/sys/kernel/debug/spi-nor/spi4.0`

```
|-- capabilities
`-- params

# cat capabilities
Supported read modes by the flash
 1S-1S-1S
  opcode        0x03
  mode cycles   0
  dummy cycles  0
 1S-1S-1S (fast read)
  opcode        0x0b
  mode cycles   0
  dummy cycles  8
 1S-1S-2S
  opcode        0x3b
  mode cycles   0
  dummy cycles  8
 1S-2S-2S
  opcode        0xbb
  mode cycles   2
  dummy cycles  2
 1S-1S-4S
  opcode        0x6b
  mode cycles   0
  dummy cycles  8
 1S-4S-4S
  opcode        0xeb
  mode cycles   2
  dummy cycles  4
 4S-4S-4S
  opcode        0xeb
  mode cycles   2
  dummy cycles  2

Supported page program modes by the flash
 1S-1S-1S
  opcode        0x02

# cat params
name            gd25lq64c
id              c8 60 17 c8 60 17
size            8.00 MiB
write size      1
page size       256
address nbytes  3
flags           HAS_SR_TB | BROKEN_RESET | HAS_LOCK | HAS_16BIT_SR | NO_READ_CR | SOFT_RESET

opcodes
 read           0xeb
  dummy cycles  6
 erase          0x20
 program        0x02
 8D extension   none

protocols
 read           1S-4S-4S
 write          1S-1S-1S
 register       1S-1S-1S

erase commands
 20 (4.00 KiB) [1]
 52 (32.0 KiB) [2]
 d8 (64.0 KiB) [3]
 c7 (8.00 MiB)

sector map
 region (in hex)   | erase mask | overlaid
 ------------------+------------+----------
 00000000-007fffff |     [ 1  ] | no
```

## Testing

### SPI-NAND/NOR Read/Write Speed Test

Enable `CONFIG_MTD_TESTS`

```
Device Drivers
         Memory Technology Device (MTD) support (MTD [=y])
                MTD tests support (DANGEROUS) (MTD_TESTS [=m])   
```

Test command:

```
insmod mtd_speedtest.ko dev=0   # 0 indicates the MTD device number of SPI-NAND/NOR
```

## FAQ
