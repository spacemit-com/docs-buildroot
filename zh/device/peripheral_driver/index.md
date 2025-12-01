---
sidebar_position: 6
slug: /development_guide/peripheral_driver
---

# 外设驱动

本文档介绍编写目的、适用范围以及相关使用人员，并详细说明 SpacemiT K1 平台的外设驱动支持情况。


## 编写目的

本手册旨在帮助开发者了解并快速上手 SpacemiT K1 CPU 的外设驱动，包括各接口的板级配置、内核配置（ CONFIG）、驱动测试方法、调试接口说明以及常见问题处理，方便开发者进行二次驱动开发和系统集成。

## 使用范围

适用于基于 SpacemiT K1 CPU 的平台及其软件开发套件。

## 相关人员

- 驱动开发工程师  
- 系统集成工程师

## 功能介绍

外设驱动（或设备驱动）是控制硬件设备与操作系统之间的接口。 Linux 系统中，外设驱动是一个模块化的组件，它充当了硬件和操作系统之间的桥梁，负责对硬件进行初始化、配置、管理、数据传输和错误处理等工作。

SpacemiT K1 包含了各种丰富的 IO 能力，集成多套 PCIe， USB， GMAC、 SPI 等接口，提供了全面的外设连接选型，本文档覆盖了 K1 涉及到的高速扩展接口驱动、音视频接口驱动、工业扩展接口驱动、存储接口驱动等使用说明文档：

- **高速扩展接口驱动**： PCIe、 USB、 GMAC 等   
- **音视频接口驱动**： DSI、 HDMI、 CSI 等   
- **工业控制接口驱动**： UART、 CAN-FD、 I2C、 SPI、 PWM 等  
- **存储接口驱动**： SDHC、 SPI-Flash 等

## 快速索引
高速扩展接口驱动
- [GMAC](09-GMAC.md) 
- [EtherCAT](22-EtherCAT.md)  
- [USB](10-USB/index.md)  
- [PCIe](11-PCIe.md)   

音视频接口驱动
- [Display](12-Display.md)  
- [V2D](13-V2D.md)  
- [Audio](17-Audio.md)

低速扩展接口驱动
- [PWM](03-PWM.md)  
- [IR-RX](04-IR-RX.md)
- [UART](05-UART.md)
- [I2C](06-I2C.md)  
- [QSPI](07-QSPI.md) 
- [SPI](SPI.md) 
- [CAN](15-CAN.md)
- [GPADC](gpadc.md)

存储接口驱动
- [SDHC](08-SDHC.md)

系统基础驱动
- [PINCTRL](01-PINCTRL.md) 
- [GPIO](02-GPIO.md)    
- [Clock](16-Clock.md)  
- [DMA](21-DMA.md)
- [RTC](rtc.md)

功耗子系统驱动
- [Thermal](thermal.md)  
- [CPUFREQ](15-Cpufreq.md)  
- [Standby](../standby.md)
- [PMIC](14-PMIC.md)

第三方外设驱动
- [WIFI](WIFI.md)
- [BT](BT.md)

其他驱动
- [CRYPTO](18-CRYPTO.md)