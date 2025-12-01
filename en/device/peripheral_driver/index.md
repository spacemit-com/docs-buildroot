---
sidebar_position: 6
slug: /development_guide/peripheral_driver
---

# Peripheral Drivers

This document introduces the purpose of writing, the scope of application, and the relevant personnel involved. It also provides a detailed explanation of the peripheral driver support for the SpacemiT K1 platform.

## Purpose

This manual is designed to assist developers in understanding and quickly getting started with the peripheral drivers for the SpacemiT K1 CPU.
It covers the board-level configuration of various interfaces, kernel configuration (CONFIG), driver testing methods, debugging interface descriptions, and common issue troubleshooting. This information is intended to facilitate secondary driver development and system integration for developers.

## Scope of Usage

This document is applicable to platforms based on the SpacemiT K1 CPU and their software development kits.

## Relevant Personnel

- Driver Development Engineers
- System Integration Engineers

## Functional Description

Peripheral drivers (or device drivers) act as interfaces between hardware devices and the operating system. In Linux systems, peripheral drivers are modular components that act as a bridge between hardware and the operating system. They are responsible for initializing, configuring, managing hardware, transferring data, and handling errors.

The SpacemiT K1 features a wide range of IO capabilities, integrating multiple sets of interfaces such as PCIe, USB, GMAC, SPI, and more, providing comprehensive peripheral connection options. This document covers the usage instructions for the high-speed expansion interface drivers, audio/video interface drivers, industrial expansion interface drivers, and storage interface drivers involved in K1:

- **High-Speed Expansion Interface Drivers**: PCIe, USB, GMAC, etc.
- **Audio/Video Interface Drivers**: DSI, HDMI, CSI, etc.
- **Industrial Control Interface Drivers**: UART, CAN-FD, I2C, SPI, PWM, etc.
- **Storage Interface Drivers**: SDHC, SPI-Flash, etc.

## Quick Index

High-Speed Expansion Interface Drivers
- [GMAC](09-GMAC.md)
- [EtherCAT](22-EtherCAT.md)
- [USB](10-USB/index.md)  
- [PCIe](11-PCIe.md)

Audio/Video Interface Drivers
- [Display](12-Display.md)  
- [V2D](13-V2D.md)  
- [Audio](17-Audio.md)

Low-Speed Expansion Interface Drivers
- [PWM](03-PWM.md)  
- [IR-RX](04-IR-RX.md)
- [UART](05-UART.md)
- [I2C](06-I2C.md)  
- [QSPI](07-QSPI.md)
- [SPI](SPI.md)
- [CAN](15-CAN.md)
- [GPADC](gpadc.md)

Storage Interface Drivers
- [SDHC](08-SDHC.md)

System Basic Drivers
- [PINCTRL](01-PINCTRL.md)
- [GPIO](02-GPIO.md)
- [Clock](16-Clock.md)  
- [DMA](21-DMA.md)
- [RTC](rtc.md)

Power Subsystem Drivers
- [Thermal](thermal.md)  
- [CPUFREQ](15-Cpufreq.md)  
- [Standby](../standby.md)
- [PMIC](14-PMIC.md)

Third-Party Peripheral Drivers
- [WIFI](WIFI.md)
- [BT](BT.md)

Other Drivers
- [CRYPTO](18-CRYPTO.md)