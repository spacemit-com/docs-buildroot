---
sidebar_position: 1
---

# Buildroot 1.0

## v1.0.2 更新说明

发布日期：2026-05-29

### 主要更新

- 支持 KVM/VTVM 虚拟化及 eBPF 动态追踪
- 支持 RISC-V IMSIC Multi MSI 中断
- 支持 AMD GPU（amdgpu/radeon），适配 PCIe Gen3
- 支持 SPI 控制器驱动（含 suspend/resume）
- 支持 DDR 12GB 及 SPL 阶段 CPU 调频
- 支持 Fastboot 刷写 EC 固件
- 支持RT-Linux 专用 defconfig
- 支持 R8169 网卡驱动
- 修复 mailbox 驱动死锁问题
- 修复 USB TypeC fusb301 状态机逻辑错误
- 修复 Bluetooth btrtl 空指针崩溃问题
- 修复 AI DMA 并发链表访问及内存泄漏
- 修复 display 驱动空指针问题

## v1.0.0 更新说明

发布日期：2026-04-30

### 特性

#### 主要组件

- OpenSBI 1.6
- U-Boot 2022.10
- Linux 6.18
- buildroot 2025.02.6
- img-gpu-powervr 24.2: GPU DDK
- mesa3d 24.04.1
- k3x-vpu-firmware: Video Process Unit firmware
- k3x-vpu-test: Video Process Unit test program
- k3x-cam: CSI Unit test progrom
- mpp: Media Process Platform
- FFmpeg 7.1.1 (with Hardware Accelerated)
- GStreamer 1.27.2 (with Hardware Accelerated)
- v2d-test: 2D Unit test program
- factorytest: factory test app

#### 主要驱动

**系统驱动**

- clk
- pinctrl
- timer
- watchdog
- RTC
- DMA
- msgbox

**接口驱动**

- USB 2.0/3.0
- PCIe 3.0
- UART
- I2C
- SPI
- PWM
- CAN

**存储驱动**

- MMC (sdcard/eMMC/SDIO)
- QSPI
- UFS

**网络驱动**

- GMAC
- WiFi
- BT

**显示驱动**

- DPU
- GPU
- MIPI DSI
- eDP/DP

**多媒体驱动**

- VPU
- V2D
- V4L2
- CMOS Sensor
- I2S

**功耗管理**

- cpufreq
- thermal
- PMIC

### 已知问题

- Suspend to ram功能尚不完善
