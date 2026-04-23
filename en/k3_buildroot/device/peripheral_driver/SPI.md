# SPI

This document describes SPI functionality and usage.

## Module Overview

**SPI (Serial Peripheral Interface)** is a serial communication interface between the SoC (System on Chip) and peripherals, and supports only x1 mode. SPI has two modes: **Master** and **Slave**. Typically, one master device controls one or more slave devices for communication. The master device selects a slave device, completes data exchange, provides the clock, and initiates read and write operations. K3 SPI currently supports **Master mode only**.

### Functional Overview

![](static/linux_spi.png)  

The Linux SPI driver framework consists of three layers: **SPI Core**, **SPI Controller Driver**, and **SPI Device Driver**.

- The main functions of **SPI Core** are:
 	- Registering the SPI bus and the `spi_master` class
 	- Adding and removing SPI controllers
 	- Adding and removing SPI devices
 	- Registering and unregistering SPI device drivers

- **SPI Controller Driver:**
 	- The SPI master controller driver, responsible for operating the SPI master controller

- **SPI Device Driver:**
 	- Implements communication with specific SPI peripherals

### Source Code Structure

The controller driver code is located in the `drivers/spi` directory:

```
|-- drivers/spi/spi-spacemit-k1.c              # K3 SPI driver
```

## Key Features

### Features

| Feature | Description |
| :-----| :----|
| Communication | Supports SSP/SPI/MicroWire/PSP protocols |
| Communication Frequency | Maximum supported frequency is 52 Mbps; minimum supported frequency is 800 Kbps |
| Bus Width | x1 |
| Supported Peripherals | Supports SPI-NOR and SPI-NAND flash memory |

### Performance Parameters

- **Communication Frequency**  
The supported communication frequencies are 51.2M / 25.6M / 12.8M / 6.4M / 3.2M / 1.6M / 800k.

- **Bus Width**  
SPI bus width supports x1.

**Testing Method**  
Use an oscilloscope or logic analyzer to measure the SCK (Serial Clock) frequency.

## Configuration

This mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

`CONFIG_SPI`: Provides support for the SPI bus protocol. By default, this option is set to `Y`.

```
Device Drivers
		SPI support (SPI [=y])
```

`CONFIG_SPI_K1`: Provides support for the K1/K3 SPI controller driver. By default, this option is also set to `Y`.

```
Device Drivers
		SPI support (SPI [=y])
				Spacemit K1 SPI Controller (SPI_K1 [=y])

```

### DTS Configuration

#### pinctrl

Refer to the reference design schematic to identify the pin group used by SPI. For pin configuration details, refer to [PINCTRL](01-PINCTRL.md). For example:

Assume SPI0 directly uses the `ssp0_2_cfg` group defined in `k3_pinctrl.dtsi`.

#### SPI Device Configuration

When configuring an SPI device, confirm the **device type** and **communication frequency** parameters.

- **Device Type**
Confirm the type of device connected to the SPI bus, such as SPI-NOR or SPI-NAND.

- **Communication Frequency**
Specify the maximum communication rate between the SPI controller and the SPI device.

- **Communication Width**
SPI communication width supports x1 mode.

SPI device DTS configuration example:
Using SPI NOR as an example, set the maximum communication frequency to 26 MHz, with both transmit and receive operating in x1 mode.

```c
&pinctrl {
 ssp0-2-cfg {
  // 1V8
  ssp0-0-pins {
   power-source = <1800>;
  };
  ssp0-1-pins {
   power-source = <1800>;
  };
 };
};

&spi0 {
 pinctrl-names = "default";
 pinctrl-0 = <&ssp0_2_cfg>;
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

## Interface

### API

**Device Driver Registration and Unregistration**

```
int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);  
void spi_unregister_driver(struct spi_driver *sdrv);
```

**Data Transfer APIs**

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

## Debug

### sysfs

View SPI bus device and driver information in the system.
`/sys/bus/spi`

```
|-- devices                 // Devices on the SPI bus
|-- drivers                 // Device drivers registered on the SPI bus
|-- drivers_autoprobe
|-- drivers_probe
`-- uevent
```

### debugfs

Used to view SPI device information in the system.
`/sys/kernel/debug/spi-nor/spi0.0`

## Testing

### SPI NAND/NOR Read/Write Speed Test

1. Enable `CONFIG_MTD_TESTS`

  ```
  Device Drivers
   Memory Technology Device (MTD) support (MTD [=y])
    MTD tests support (DANGEROUS) (MTD_TESTS [=m])   
  ```

2. Run the test command

  ```
  insmod mtd_speedtest.ko dev=0   # 0 indicates the MTD device number of SPI NAND/NOR
  ```

## FAQ
