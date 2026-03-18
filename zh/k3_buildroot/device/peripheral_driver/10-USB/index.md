---
sidebar_position: 10
slug: /development_guide/peripheral_driver/10-USB
---

# USB

USB 全称 Universal Serial Bus（通用串行总线），是一种广泛使用的高速串行总线标准。K3 平台当前 USB 文档按功能拆成三部分，分别覆盖通用配置、Gadget 设备侧开发和信号质量测试。

USB 开发指南：

- [USB 通用开发指南](1-USB-General-Developer-Guide.md)
- [USB Gadget 开发指南](2-USB-Gadget-Developer-Guide.md)
- [USB 信号质量测试指南](3-USB-SQ-Test-Guide.md)

阅读建议：

- 做板级端口规划、Type-C / role-switch / PHY 方案确认时，先看 **USB 通用开发指南**；
- 做设备侧模式、configfs、UDC bring-up 时，看 **USB Gadget 开发指南**；
- 做 USB2 test mode、debugfs、Host/Gadget 测试闭环时，看 **USB 信号质量测试指南**。
