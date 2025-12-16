---
sidebar_position: 1
---

# Buildroot 1.0更新说明 [已停服]

## Buildroot 1.0 停服公告  
Buildroot V1.0 基于linux-6.1内核构建，现已决定对Buildroot V1.0停止维护（End of Life）。
| 版本 | 停止支持日期 | 延长生命周期（Extended Life-cycle Support, ELS）停止日期|
| :-----| :----| :----|
| Buildroot 1.x.x |	2024/12/31 |	NO|

建议用户更新Buildroot V2.0版本使用

## v1.0.15更新说明

发布日期：2024-9-7

### 主要更新

- 更新aic8800固件路径

## v1.0.14更新说明

发布日期：2024-8-31

### 主要更新

- 支持u-boot pwm控制LCD背光
- 支持u-boot 电量计框架
- 支持MIPI DSI屏jd9365dah3
- 支持Type-C芯片husb239
- 修复u-boot efi free memory问题
- 修复u-boot LCD低概率花屏的问题
- 修复SPINOR分区创建失败的问题
- 更新rcpu固件
- 优化内核加载模块速度

## v1.0.13更新说明

发布日期：2024-8-16

### 主要更新

- 支持`inxi`获取CPU温度
- 修复u-boot SSD兼容性
- 修复u-boot env bootarg参数超过16个会出错的问题
- 修复CAN 8M波特率传输卡死的问题
- 【MUSE Book】修复频繁开关盖状态异常的问题
- 【MUSE Book】支持ADB

## v1.0.12更新说明

发布日期：2024-8-2

### 主要更新

- 支持aic8800 WiFi模组

## v1.0.11更新说明

发布日期：2024-8-1

### 主要更新

- 支持EEPROM烧录part number
- 支持rtl8852be休眠开关电
- 更新pin脚上、下拉状态
- 修复u-boot下pwm时钟配置错误
- 修复GMAC Phy在休眠时无法进入低功耗的问题
- 修复CPU跑`1.8GHz@1.16v` WiFi、GMAC功能异常的问题
- 修复Samsung 970pro 1T SSD被识别为1 lane的问题
- 【MUSE Book】修复充电充满后电量显示仅99%的问题
- 【MUSE Book】修复休眠后唤醒速度到2s内

## v1.0.9更新说明

发布日期：2024-7-20

### 主要更新

- 修复MIPI DSI屏JD9365DA无显示的问题
- 【MUSE Book】修复启动或刷机低概率屏幕没显示的问题
- 【MUSE Book】修复启动过程中logo低概率闪烁的问题
- 【MUSE Book】修复充电充满后电量显示仅99%的问题
- 【MUSE Book】修复休眠后低概率立即唤醒，及休眠功耗异常（高）的问题

## v1.0.8更新说明

发布日期：2024-7-16

### 主要更新

- 更新rtl8852bu驱动和固件，修复蓝牙部分问题
- 修复个别HDMI显示器没有声音的问题
- 修改个别场景休眠唤醒后声音异常问题
- 【MUSE Card】支持MUSE Card

## v1.0.7更新说明

发布日期：2024-7-11

### 主要更新

- 支持更多CPU调频策略
- 支持WiFi/USB唤醒
- 支持查阅芯片名称和版本号等
- 修复pwm频率配置错误的问题
- 修复16GB DDR部分模块无法访问的问题
- 修复低概率耳机检测出错的问题
- 修复休眠唤醒后低概率无声音的问题
- 修复rtl8852be极低概率kernel panic的问题
- 【BPI-F3】修复touchpac无法使用的问题
- 【MUSE Book】修复u-boot无法在线升级的问题
- 【MUSE Book】修复启动logo低概率闪烁的问题

## v1.0.5更新说明

发布日期：2024-6-28

### 主要更新

- u-boot支持CPU `1.6GHz@1.05v`
- 支持LPDDR4类型的DDR颗粒
- 支持ota7290b MIPI屏
- 支持es7210/es8516 audio codec
- 修复音频播放杂音问题
- 修复HDMI获取EDID缺陷，支持完整256 byte EDID
- 修复Crypto非16 byte对齐处理错误问题
- 修复rtl8852be休眠唤醒异常相关问题
- 修复u-boot SPINOR低概率擦写失败的问题
- 修复u-boot Sandisk 128GB eMMC HS400模式读写出错问题
- 修复u-boot NVMe SSD某些条件下读写出问题
- 【MUSE Box】支持DP插拔动态检测
- 【MUSE Book】修复启动和唤醒概率性花屏的问题
- 【MUSE Book】修复休眠概率性无法唤醒屏幕的问题
- 【MUSE Book】修复摄像头高分辨率卡顿问题
- 【MUSE Book】优化eDP屏初始化速度
- 【MUSE Book】优化eDP屏背光亮度曲线

## v1.0.3更新说明

发布日期：2024-6-19

### 主要更新

- linux-6.1支持codec芯片es8316
- u-boot支持SPI驱动
- 修复长时间休眠无法唤醒问题
- 修复低概率创建声卡失败问题
- 修复低概率i2c通信timeout问题
- 修复pcie 2 lane无法自适应支持1 lane问题
- 修复SDIO WiFi低温概率性CRC错误问题
- 【BPI-F3】支持风扇降温功能
- 【MUSE Pi】支持MIPI-CSI Sensor
- 【MUSE Box】支持声卡耳机、Mic检测功能
- 【MUSE Box】支持DP显示功能（注：不支持热插拔）
- 【MUSE Book】支持u-boot阶段显示logo
- 【MUSE Book】更新产测工具
- 【MUSE Book】修复获取不到显示器EDID问题

## v1.0更新说明

发布日期：2024-5-30

### 特性

#### 主要组件

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.1
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

### 已知问题

- Suspend to ram功能尚不完善
