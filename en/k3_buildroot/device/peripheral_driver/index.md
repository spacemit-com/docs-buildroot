---
sidebar_position: 6
---

# Peripheral Drivers

This document describes the purpose of this guide, its intended scope, and the target audience. It also provides an overview of peripheral driver support on the SpacemiT K3 platform.

## Purpose

This guide is intended to help developers understand and quickly get started with peripheral drivers for the SpacemiT K3 CPU. It covers board-level configuration for each interface, kernel configuration (`CONFIG`), driver test methods, debugging interfaces, and common issue handling, making follow-up driver development and system integration easier.

## Scope

This guide applies to platforms based on the SpacemiT K3 CPU and their software development kits.

## Target Audience

- Driver development engineers  
- System integration engineers

## Functional Description

Peripheral drivers, also called device drivers, provide the interface between hardware devices and the operating system. In Linux, peripheral drivers are modular components that act as the bridge between hardware and the operating system. They are responsible for hardware initialization, configuration, management, data transfer, and error handling.

SpacemiT K3 provides a wide range of I/O capabilities and integrates multiple interfaces such as PCIe, USB, GMAC, and SPI. This document covers usage guides for the interface drivers supported on K3, including high-speed expansion interfaces, audio and video interfaces, industrial control interfaces, and storage interfaces:

- **High-speed expansion interface drivers**: PCIe, USB, GMAC, and others
- **Audio and video interface drivers**: DSI, HDMI, CSI, and others
- **Industrial control interface drivers**: UART, CAN-FD, I2C, SPI, PWM, and others  
- **Storage interface drivers**: SDHC, SPI Flash, and others

## Quick Index

High-speed expansion interface drivers

- [GMAC](09-GMAC.md)
- [EtherCAT](22-EtherCAT.md)  
- [USB](10-USB/index.md)  
- [PCIe](11-PCIe.md)

Audio and video interface drivers

- [Display](12-Display.md)  
- [V2D](13-V2D.md)  
- [Audio](17-Audio.md)

Low-speed expansion interface drivers

- [PWM](03-PWM.md)  
- [IR-RX](04-IR-RX.md)
- [UART](05-UART.md)
- [I2C](06-I2C.md)  
- [QSPI](07-QSPI.md)
- [SPI](SPI.md)
- [CAN](15-CAN.md)
- [GPADC](gpadc.md)

Storage interface drivers

- [SDHC](08-SDHC.md)
- [UFS](ufs.md)
- [DDR](ddr.md)

System foundation drivers

- [PINCTRL](01-PINCTRL.md)
- [GPIO](02-GPIO.md)
- [Clock](16-Clock.md) 
- [Reset](Reset.md) 
- [DMA](21-DMA.md)
- [RTC](rtc.md)

Power subsystem drivers

- [Thermal](thermal.md)  
- [CPUFREQ](15-Cpufreq.md)  
- [Standby](../standby.md)
- [PMIC](14-PMIC.md)

Third-party peripheral drivers

- [WIFI](WIFI.md)
- [BT](BT.md)

Other drivers
- [WDT](23-WDT.md)
- [Timer](Timer.md)
