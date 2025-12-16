---
sidebar_position: 2
---

# Buildroot 2.0更新说明 [已停服]

Buildroot 2.0 于 2025年7月31日 停止维护，推荐使用 2.2 或后续版本，如有疑问，请联系我们。

## v2.0.4更新说明

发布日期：2024-12-11

### 主要更新

- 修复 openssl speed -evp chacha20 非法指令的问题
- 修复个别场景 rwsem/spinlock 等出错问题
- u-boot 优化 PCIe 枚举时间
- 优化内核启动速度
- 优化 PCIe memory range 定义，提升 NVMe SSD 读写性能
- 更新 GPU 驱动至 v24.2
- 支持 GRUB

## v2.0.2更新说明

发布日期：2024-11-11

### 主要更新

- u-boot 支持非 AI 扩展核作为启动核
- 支持 Hynix LPDDR4x DRAM 颗粒
- 支持 DDR 带宽统计工具
- 支持 es8323 audio codec 芯片
- 支持 rt-linux patch
- 支持 AMD 显卡
- 修复关闭 CPU 1 ~ 7 后，rtl8852bs 中断过多导致系统卡死的问题
- 修复 gpio74 引脚的 pwm 无法正常工作的问题
- 修复 Node.js 运行概率性出现非法指令的问题
- 关闭 LOCKDEP 选项，避免其影响系统运行性能

## v2.0.1更新说明

发布日期：2024-10-28

### 主要更新

- 修复开启内核 LOCKDEP 选项后的警告信息
- 【MUSE Book】修复休眠唤醒异常的问题

## v2.0更新说明

发布日期：2024-10-22

### 主要更新

- 修复 u-boot keyboard 兼容性问题
- 修复 SPINOR 引导 eMMC 无 bootlogo 问题
- 增加 u-boot 支持 NFS 启动
- 修复内核若干个 Oops 问题
- 临时修复内核个别场景 Hung 问题

## v2.0rc7更新说明

发布日期：2024-9-30

### 主要更新

- 修复卡量产失败的问题
- 修复rtl8852bs驱动内存越界访问题
- 修复rf驱动gpio设置报错问题
- 修复bootlogo闪烁问题
- 更新sdio delay参数，提升sdio稳定性
- 增加 P1 芯片的 ADC 功能

## v2.0rc6更新说明

发布日期：2024-9-10

### 主要更新

- 【MUSE Book】优化屏幕唤醒时间

## v2.0rc5更新说明

发布日期：2024-9-2

### 主要更新

- 更新rcpu固件

## v2.0rc4更新说明

发布日期：2024-8-29

### 主要更新

- 支持u-boot pwm控制LCD背光
- 支持u-boot 电量计框架
- 支持MIPI DSI屏jd9365dah3
- 支持Type-C芯片husb239
- 修复u-boot efi free memory问题
- 修复u-boot LCD低概率花屏的问题
- 修复关机过程中i2c异常的问题
- 优化内核加载模块速度

### 已知问题

- WiFi功能不稳定

## v2.0rc3更新说明

发布日期：2024-8-10

### 特性

#### 主要组件

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.6.36
- buildroot 2023.02.9
- onnxruntime 1.15.1
- img-gpu-powervr 23.2: GPU DDK
- mesa3d 22.3.5
- QT 5.15 (with GPU enabled)
- k1x-vpu-firmware: Video Process Unit firmware
- k1x-vpu-test: Video Process Unit test program
- k1x-jpu: JPEG Process Unit API
- k1x-cam: CMOS Sensor and ISP API
- mpp: Media Process Platform
- FFmpeg 4.4.4 (with Hardware Accelerated)
- GStreamer 1.22.8 (with Hardware Accelerated)
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
- spinlock
- TCM

**接口驱动**

- USB 2.0/3.0
- PCIe 2.1
- UART
- I2C
- SPI
- PWM
- CAN

**存储驱动**

- MMC (sdcard/eMMC/SDIO)
- SPI NOR
- NVMe SSD

**网络驱动**

- GMAC
- WiFi
- BT

**显示驱动**

- DPU
- GPU
- MIPI DSI
- HDMI 1.4

**多媒体驱动**

- VPU
- JPU
- V2D
- V4L2
- CMOS Sensor
- ISP
- I2S
- HDMI Audio

**功耗管理**

- cpufreq
- thermal
- PMIC

**加密驱动**

- AES
