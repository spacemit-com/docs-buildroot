# Display

介绍 SpacemiT 平台 Display 模块的功能和使用方法。

## 模块介绍

SpacemiT 平台的 Display 模块基于 **DRM 框架（Direct Rendering Manager）** 实现。DRM 是 Linux 系统主流的显示子系统架构，适配现代显示硬件特性，支持显存管理、显示时序配置、图层混合等多项功能。

### 功能介绍

DRM 框架包括 **用户空间** 与 **内核空间** 两个部分。
![display-drm](static/display-drm.png)

#### 用户空间：Libdrm

DRM 框架在用户空间提供了 libdrm 库，用户或应用程序可通过调用该库的函数，访问和管理显示资源。

#### 内核空间：DRM driver

DRM driver 提供了一组 IOCTL 接口，主要包括以下两类：

1. **Graphics Execution Manager (GEM)**
   GEM 主要是对 FrameBuffer 的管理，包括：
    - 显存申请释放 (Framebuffer managing)
    - 显存共享机制 (Memory sharing objects)
    - 显存同步机制 (Memory synchronization)

2. **Kernel Mode-Setting (KMS)**
   KMS 主要负责管理 **显示模式的设置** 和 **图像输出** 。核心组成包括：

    - **Framebuffer**
      - 一块内存区域，驱动和应用层可访问，单个图层的显示内容。

    - **CRTC（显示控制器）**
      - 负责把要显示的图像，转化为底层硬件层面上的具体时序要求
      - 还负责帧切换、电源控制、色彩调整等等。

    - **Planes（图层）**
      - 每个图像拥有一个 Planes，Planes 的属性控制着图像的显示区域、图像翻转、色彩混合方式等， CRTC 的显示图像实际上是 由 Framebuffer 与 Plane 的组合渲染，得到多个图像的混合显示或单独显示。
      - 图层有以下三种类型：
        1) **主图层（primary plane）：** 用于显示背景或者图像内容
        2) **叠加图层（overlay plane）：** 常用于视频、字幕等附加显示
        3) **光标图层(cursor)：** 专用于鼠标指针显示

    - **Encoder（编码器）**
      - 负责电源管理、视频输出格式封装
      - 把时序转换为显示器所需要的信号，将画面显示到不同的显示设备，例如将视频输出到 HDMI 接口、MIPI DSI 接口等。

    - **Connector（连接器）**
      - 负责硬件显示设备的接入、屏参获取等，例如 HDMI, MIPI DSI 等。

在内核空间中，还包括 **Panel（显示面板）** 模块，负责将接收到的图像信号转换为最终在屏幕上显示的图像内容。

### 源码结构介绍

SpacemiT 平台 DRM 驱动源码结构如下：

```
linux-6.6/drivers/gpu/drm$ tree spacemit
spacemit
|-- dphy
|   `-- spacemit_dphy_drv.c                     // mipi dsi dphy驱动
|-- dpu                                         // dpu 驱动
|   |-- dpu_debug.c
|   |-- dpu_debug.h
|   |-- dpu_saturn.c
|   |-- dpu_saturn.h
|   |-- dpu_trace.h
|   |-- saturn_fbcmem.c
|   |-- saturn_fbcmem.h
|   `-- saturn_regs
|       |-- cmdlist.h
|       |-- cmps_x.h
|       |-- dma_top.h
|       |-- dpu_crg.h
|       |-- dpu_ctl.h
|       |-- dpu_intp.h
|       |-- dpu_top.h
|       |-- mmu.h
|       |-- outctrl_proc_x.h
|       |-- outctrl_top_x.h
|       |-- prepipe_layer_proc_x.h
|       |-- rdma_path_x.h
|       |-- reg_map.h
|       |-- scaler_x.h
|       `-- wb_top.h
|-- dsi                                         // mipi dsi 驱动
|   |-- spacemit_dptc_drv.c
|   |-- spacemit_dptc_drv.h
|   |-- spacemit_dsi_drv.c
|   `-- spacemit_dsi_hw.h
|-- Kconfig
|-- lt8911exb.c                                 // lt8911exb mipi dsi转eDP panel驱动
|-- lt9711.c                                    // lt9711 mipi dsi转DP panel驱动
|-- Makefile
|-- spacemit_bootloader.c
|-- spacemit_bootloader.h
|-- spacemit_cmdlist.c
|-- spacemit_cmdlist.h
|-- spacemit_dmmu.c
|-- spacemit_dmmu.h
|-- spacemit_dphy.c
|-- spacemit_dphy.h
|-- spacemit_dpu.c
|-- spacemit_dpu.h
|-- spacemit_dpu_reg.h
|-- spacemit_drm.c                             // DRM core 驱动
|-- spacemit_drm.h
|-- spacemit_dsi.c
|-- spacemit_dsi.h
|-- spacemit_gem.c                             // GEM 驱动
|-- spacemit_gem.h
|-- spacemit_hdmi.c                            // HDMI 驱动
|-- spacemit_hdmi.h
|-- spacemit_lib.c
|-- spacemit_lib.h
|-- spacemit_mipi_panel.c                      // panel 驱动
|-- spacemit_mipi_panel.h
|-- spacemit_planes.c
|-- spacemit_wb.c                              // write back 驱动
|-- spacemit_wb.h
`-- sysfs
    |-- sysfs_class.c                         
    |-- sysfs_display.h
    |-- sysfs_dphy.c
    |-- sysfs_dpu.c
    |-- sysfs_dsi.c
    `-- sysfs_mipi_panel.c
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 MIPI DSI | 支持 MIPI DPHY v1.1, 支持 DPHY 4 lane，最高速率 1.2Gbps/lane |
| 支持 HDMI | 支持 HDMI 1.4a |

### 性能参数

| 屏幕接口 | 性能规格 |
| :-----| :----|
| MIPI DSI| 1920x1200@60FPS |
| HDMI | 1920x1080@60FPS |

**MIPI DSI 屏幕帧率测试方法:**

- 查看 Connectors：

```
# modetest -M spacemit -D /dev/dri/card1 -c
Connectors:
id      encoder status          name            size (mm)       modes   encoders
130     129     connected       DSI-1           142x228         1       129
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 1200x1920 60.05 1200 1250 1260 1300 1920 1940 1944 1960 153000 flags: phsync, pvsync; type: preferred, driver
  props:
        1 EDID:
                flags: immutable blob
                blobs:

                value:
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 0
        5 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        6 non-desktop:
                flags: immutable range
                values: 0 1
                value: 0
        4 TILE:
                flags: immutable blob
                blobs:

                value:
```

- 查看 Encoders：

```
# modetest -M spacemit -D /dev/dri/card1 -e
Encoders:
id      crtc    type    possible crtcs  possible clones
129     127     DSI     0x00000001      0x00000001
```

- 测试 MIPI DSI 屏幕帧率：

```
# modetest -M spacemit -D /dev/dri/card1 -s 130@127:1200x1920 -v
setting mode 1200x1920-60.05Hz on connectors 130, crtc 127
freq: 60.55Hz
freq: 60.28Hz
freq: 60.28Hz
```

**HDMI 屏幕帧率测试方法:**

- 查看 Connectors：

```
# modetest -M spacemit -D /dev/dri/card2 -c
Connectors:
id      encoder status          name            size (mm)       modes   encoders
130     129     connected       HDMI-A-1        300x260         12      129
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 1920x1080 60.00 1920 2008 2052 2200 1080 1082 1087 1125 148500 flags: phsync, pvsync; type: preferred, driver
  #1 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #2 1920x1080 59.94 1920 2008 2052 2200 1080 1084 1089 1125 148352 flags: phsync, pvsync; type: driver
  #3 1600x900 60.00 1600 1624 1704 1800 900 901 904 1000 108000 flags: phsync, pvsync; type: driver
  #4 1280x1024 60.02 1280 1328 1440 1688 1024 1025 1028 1066 108000 flags: phsync, pvsync; type: driver
  #5 1152x864 59.97 1152 1216 1336 1520 864 865 868 895 81579 flags: nhsync, pvsync; type:
  #6 1280x720 60.00 1280 1390 1430 1650 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #7 1280x720 59.94 1280 1390 1430 1650 720 725 730 750 74176 flags: phsync, pvsync; type: driver
  #8 1024x768 60.00 1024 1048 1184 1344 768 771 777 806 65000 flags: nhsync, nvsync; type: driver
  #9 800x600 60.32 800 840 968 1056 600 601 605 628 40000 flags: phsync, pvsync; type: driver
  #10 640x480 60.00 640 656 752 800 480 490 492 525 25200 flags: nhsync, nvsync; type: driver
  #11 640x480 59.94 640 656 752 800 480 490 492 525 25175 flags: nhsync, nvsync; type: driver
  props:
        1 EDID:
                flags: immutable blob
                blobs:

                value:
                        00ffffffffffff005c73562501000000
                        321d0103801e1a783eee91a3544c9926
                        0f505421080071408180a9c0d1c00101
                        010101010101023a801871382d40582c
                        2500dd0c1100001e000000fc0048444d
                        490a2020202020202020000000ff000a
                        202020202020202020202020000000fd
                        003b3f1e5414000a20202020202001c5
                        02032ef1429004e200d5e305c0002309
                        7f078301000067030c0010001878e606
                        0501626200681a00000101304be6023a
                        801871382d40582c4500dd0c1100001e
                        00000000000000000000000000000000
                        00000000000000000000000000000000
                        00000000000000000000000000000000
                        000000000000000000000000000000f5
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 0
        5 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        6 non-desktop:
                flags: immutable range
                values: 0 1
                value: 0
        4 TILE:
                flags: immutable blob
                blobs:

                value:
```

- 查看 Encoders：

```
# modetest -M spacemit -D /dev/dri/card2 -e
Encoders:
id      crtc    type    possible crtcs  possible clones
129     127     TMDS    0x00000001      0x00000001
```

- 测试 HDMI 屏幕帧率：

```
# modetest -M spacemit -D /dev/dri/card2 -s 130@127:1920x1080 -v
setting mode 1920x1080-60.00Hz on connectors 130, crtc 127
freq: 60.13Hz
freq: 60.00Hz
freq: 60.00Hz
```

## 配置介绍

主要包括 **Display 驱动使能配置** 和 **DTS 配置**，K1 芯片支持 1个 MIPI DSI 硬件接口和 1个 HDMI 硬件接口。

### CONFIG 配置

`CONFIG_DRM_SPACEMIT`：SpacemiT 平台 DRM 驱动配置选项，默认情况，此选项为 `Y`，并作为启用 MIPI DSI 驱动或 HDMI 驱动的前置条件。用户可根据需求选择单独配置启用 MIPI DSI、HDMI 或两者同时启用的显示输出方案。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   < >   MIPI Panel Support For Spacemit
   < >   HDMI Support For Spacemit
```

#### MIPI DSI CONFIG 配置

`CONFIG_SPACEMIT_MIPI_PANEL`：SpacemiT 平台 MIPI DSI 驱动配置选项，具体方案根据需要进行配置。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   MIPI Panel Support For Spacemit
```

#### HDMI CONFIG 配置

`CONFIG_SPACEMIT_HDMI`：SpacemiT 平台 HDMI 驱动配置选项，具体方案根据需要进行配置。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   HDMI Support For Spacemit
```

### DTS 配置

#### MIPI DSI

##### GPIO

MIPI DSI panel GPIO 相关配置，包括 **panel 复位 GPIO 配置** 和 **panel 电源控制 GPIO 配置**。

以 k1-x_deb1 方案为例：
gpio81 配置为 panel 复位 pin，gpio82 和 gpio83 配置为 panel 电源控制 pin。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dsi2 {
        status = "okay";

        panel2: panel2@0 {
                status = "okay";
                compatible = "spacemit,mipi-panel2";
                reg = <0>;

                gpios-reset = <81>;     // 配置panel 复位 gpio
                gpios-dc = <82 83>;     // 配置panel 电源控制 gpio
        };
};
```

#### 电源配置

MIPI DSI 电源配置，包括 MIPI DSI 1.2V 电源控制配置。

以 k1-x_deb1 方案为例：
配置 pmic ldo_5 为 MIPI DSI 1.2V。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
        status = "okay";

        dsi_1v2-supply = <&ldo_5>;      // 引用PMIC DLDO
        vin-supply-names = "dsi_1v2";   // 配置MIPI DSI 1.2v电源
};
```

#### Clock 配置

MIPI DSI 相关 clock配置，包括 MIPI DSI DPU相关clock配置，reset配置，及MIPI DSI DPHY相关 clock 配置。其中 `pixel clock` 和 `bit clock` 通过 timing 参数计算获取，具体计算方法请参见 **Display Timing 配置** 章节。
- `display-timings` 中的 `clock-frequency` 为 pixel clock 值
- MIPI DSI DPU 中的 `spacemit-dpu-bitclk` 和 MIPI DSI DPHY 中的 `phy-bit-clock` 为bit clock 值。
- MIPI DSI DPU ESCCLK 及 MIPI DSI DPHY ESCCLK 配置 `51200000` 或 `76800000`（分辨率 1920x1080 以上推荐使用 `76800000`）
- 其它 clock 参数使用系统默认值，不需在 DTS 文件中配置。

配置平台 MIPI DSI DPU 相关 clock 和 reset。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x-lcd.dtsi
&soc {
	display-subsystem-dsi {
		compatible = "spacemit,saturn-le";
		reg = <0 0xC0340000 0 0x2A000>;
		ports = <&dpu_online2_dsi>;
		interconnects = <&dram_range1>;
		interconnect-names = "dma-mem";
	};

	dpu_online2_dsi: port@c0340000 {
		compatible = "spacemit,dpu-online2";
		interrupt-parent = <&intc>;
		interrupts = <90>, <89>;
		interrupt-names = "ONLINE_IRQ", "OFFLINE_IRQ";
		clocks = <&ccu CLK_DPU_PXCLK>,          // mipi dsi dpu pxclk 配置
			 <&ccu CLK_DPU_MCLK>,           // mipi dsi dpu mclk 配置
			 <&ccu CLK_DPU_HCLK>,           // mipi dsi dpu hclk 配置
			 <&ccu CLK_DPU_ESC>,            // mipi dsi dpu escclk 配置
			 <&ccu CLK_DPU_BIT>;            // mipi dsi dpu bitclk 配置
		clock-names = "pxclk", "mclk", "hclk", "escclk", "bitclk";
		resets = <&reset RESET_MIPI>,           // mipi dsi dpu dsi reset 配置
			 <&reset RESET_LCD_MCLK>,       // mipi dsi dpu mclk reset 配置
			 <&reset RESET_LCD>,            // mipi dsi dpu lcd reset 配置
			 <&reset RESET_DSI_ESC>;        // mipi dsi dpu esc reset 配置
		reset-names= "dsi_reset", "mclk_reset", "lcd_reset","esc_reset";
		power-domains = <&power K1X_PMU_LCD_PWR_DOMAIN>;
		pipeline-id = <ONLINE2>;
		ip = "spacemit-saturn";
		spacemit-dpu-min-mclk = <40960000>;
		type = <DSI>;
		clk,pm-runtime,no-sleep;
		status = "disabled";

		dpu_online2_dsi_out: endpoint@0 {
			remote-endpoint = <&dsi2_in>;
		};

		dpu_offline0_dsi_out: endpoint@1 {
			remote-endpoint = <&wb0_in>;
		};
	};
}
```

配置方案 MIPI DSI DPU bitclk 及 escclk。
以 k1-x_deb1 方案为例：

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
	status = "okay";

	spacemit-dpu-bitclk = <1000000000>;     // mipi dsi dpu bitclk 配置
	spacemit-dpu-escclk = <76800000>;       // mipi dsi dpu escclk 配置
};
```

配置 Panel 型号 MIPI DSI DPHY bitclk 及 escclk。
以 MIPI DSI panel 型号 lcd_gx09inx101_mipi 为例：

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\lcd\lcd_gx09inx101_mipi.dtsi
/ { lcds: lcds {
	lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {

		phy-bit-clock = <1000000000>;   // mipi dsi dphy bitclk 配置
		phy-esc-clock = <76800000>;     // mipi dsi dphy escclk 配置
	};

};};
```

#### Display timing 配置

根据 MIPI DSI panel 提供规格书的 timing 信息填写 DPU timing 配置，及 MIPI DSI timing 配置。其中 pixel clock 和 bit clock 通过 timing 参数计算获取。

![display-timing](static/display-timing.png)

##### Display timing 参数说明

- **HFP:**
hfront porch(horizon front porch): 水平前肩是指水平同步信号之前的空白时间，用于显示设备进行准备。

- **HBP:**
hback porch(horizon back porch): 水平后肩是指水平同步信号之后的空白时间，用于显示设备进行复位和恢复。

- **HSYNC:**
hsync pulse: 水平同步信号用于同步显示设备的行扫描，水平同步脉冲宽度表示水平同步信号的持续时间。

- **VFP:**
vfront porch(vertical front porch): 垂直前肩是指垂直同步信号之前的空白时间，用于显示设备进行准备。

- **VBP:**
vback porch(vertical back porch): 垂直后肩是指垂直同步信号之后的空白时间，用于显示设备进行复位和恢复。

- **VSYNC:**
vsync pulse: 垂直同步信号用于同步显示设备的刷新率，垂直同步脉冲宽度表示垂直同步信号的持续时间。

- **HACTIVE:**
hactive(Horizon display period): 水平行中有效显示的行像素。

- **VAVTIVE:**
vactive(vertical display period): 垂直帧中有效显示的行数。

#### Display timing 计算方法

以下为各相关参数的定义及计算公式：

- **FPS（Frames Per Second）：**
帧率，每秒显示的帧数。

- **Bpp（Bits Per Pixel）：**
位深，每个像素使用的 bit 位数。

- **Htotal（水平总像素数）：**
每一行包含的总像素数。
   ```
   Htotal = hactive + HFP + HSYNC pulse + HBP
   ```

- **vtotal（垂直总像素数）：**
每一帧包含的总像素行数。
   ```
   vtotal = vactive + VFP + VSYNC pulse + VBP
   ```

- **Pixel clock:**
每秒传输或处理像素数据的频率
像素时钟计算方法：
   ```
   pixel clock = htotal \* vtotal \* fps  = (hactive + hfp + hbp + hsync) \* (vactive + vfp + vbp + vsync) \* fps
   ```

- **Bit clock:**
MIPI DSI 数据传输过程中，每 Lane 的数据传输时钟
Bit clock 计算方法：
   ```
   bit clock =  ((htotal \* vtotal \* fps  \* bpp) / lane bumber) \* 1.1 =   (((hactive + hfp + hbp + hsync) \* (vactive + vfp + vbp + vsync) \* fps  \* bpp) / lane bumber) \* 1.1
  ```

- **DSI clock:**
MIPI DSI 的 clock lane 实际时钟信号, 采用双边沿采样，一个时钟可以传两个 bit 数据。
   ```
   dsi clock = bit clock / 2
   ```

**注意：**
在 SpacemiT 平台中计算 MIPI DSI 的 Bit Clock 时，需额外乘以 **1.1 的系数**。

以下以 MIPI DSI 面板型号 `lcd_gx09inx101_mipi` 为例，说明如何计算并配置 pixel clock 与 bit clock。

**Pixel Clock 计算**
```
pixel clock = (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps 
            = （1200 + 50 + 40 + 10）* (1920 + 20 + 16 + 4) * 60 
            = 152880000 Hz
```

**Bit Clock 计算**
```
bit clock = (((hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps  * bpp) / lane bumber) * 1.1 
          = (（（1200 + 50 + 40 + 10）* (1920 + 20 + 16 + 4) * 60 * 24）/ 4) * 1.1 
          = 1009008000 Hz
```

通过 Display timing计算：
pixel clock 值为 152880000 Hz，系统可配置为 153000000 Hz
bit clock 值为 1009008000 Hz，系统可配置为 1000000000 Hz

DTS 配置建议：
- `clock-frequency = 153000000`
- `spacemit-dpu-bitclk = 1000000000`
- `phy-bit-clock = 1000000000`

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\lcd\lcd_gx09inx101_mipi.dtsi
/ { lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {

                // mipi dsi dphy中配置timing
                height = <1920>;              // mipi dsi dphy中配置屏幕高
                width = <1200>;               // mipi dsi dphy中配置屏幕宽
                hfp = <50>;                   // mipi dsi dphy中配置水平前肩（Horizontal Front Porch）
                hbp = <40>;                   // mipi dsi dphy中配置水平后肩（Horizontal Back Porch）
                hsync = <10>;                 // mipi dsi dphy中配置水平同步信号（Horizontal Sync）
                vfp = <20>;                   // mipi dsi dphy中配置垂直前肩（Vertical Front Porch）
                vbp = <16>;                   // mipi dsi dphy中配置垂直后肩（Vertical Back Porch）
                vsync = <4>;                  // mipi dsi dphy中配置垂直同步信号（Vertical Sync）
                fps = <60>;                   // mipi dsi dphy中配置帧率

                // mipi dsi dpu中配置timing
                display-timings {
                        timing0 {
                                clock-frequency = <153000000>;    // 像素时钟
                                hactive = <1200>;                 // 有效显示区域水平像素
                                hfront-porch = <50>;              // hfp
                                hback-porch = <40>;               // hbp
                                hsync-len = <10>;                 // hsync
                                vactive = <1920>;                 // 有效显示区域垂直像素
                                vfront-porch = <20>;              // vfp
                                vback-porch = <16>;               // vbp
                                vsync-len = <4>;                  // vsync
                                vsync-active = <1>;               // vsync信号高电平触发
                                hsync-active = <1>;               // hsync信号高电平触发
                        };
                };
        };
};};
```

已完成功能调试的 MIPI DSI panel，相关 dtsi 文件放置 lcd 目录。

```
linux-6.6/arch/riscv/boot/dts/spacemit/lcd$ tree
.
|-- lcd_gc9503v_mipi.dtsi
|-- lcd_gx09inx101_mipi.dtsi
|-- lcd_icnl9911c_mipi.dtsi
|-- lcd_icnl9951r_mipi.dtsi
|-- lcd_jd9365dah3_mipi.dtsi
|-- lcd_jd9365da_mipi_1280x800.dtsi
|-- lcd_lt8911_edp_1920x1080.dtsi
|-- lcd_lt8911_edp_1920x1200.dtsi
|-- lcd_lt9711_dp_1920x1080.dtsi
`-- lcd_orisetech_ota7290b_mipi.dtsi
```

##### Panel 配置

使能 MIPI DSI Panel，需使能 
- MIPI DSI DPU
- MIPI DSI Host
- LCD 面板节点（lcds）
- 指定的 Panel 型号
- PWM 背光控制

以 k1-x_deb1 方案为例, 需进行如下配置：
- 使能 `dpu_online2_dsi`
- 使能 `dsi2`
- 使能 `lcds`
- 配置 panel 为型号 `lcd_gx09inx101_mipi`
- 配置 PWM 背光

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
	status = "okay";                                       // 使能MIPI DSI dpu
};

&dsi2 {
	status = "okay";                                        // 使能MIPI DSI host
	panel2: panel2@0 {
		status = "okay";                                // 使能panel
		compatible = "spacemit,mipi-panel2";
		reg = <0>;

		gpios-reset = <81>;                             // 配置 panel 复位 gpio
		gpios-dc = <82 83>;                             // 配置 panel 电源控制 gpio
		id = <2>;                                       // 配置 panel id
		delay-after-reset = <10>;                       // 配置 plane 复位延时时间（单位：ms）
		force-attached = "lcd_gx09inx101_mipi";         // 配置 plane 型号
	};

};

&lcds {
	status = "okay";                                          // 使能lcds
};

&pwm14 {
	status = "okay";                                        // 使能pwm
};

&pwm_bl {                                                       // 使能背光
 	status = "okay";
};

```

##### DTS 配置示例

**MIPI DSI Panel配置示例：**

以 MIPI DSI panel 型号 `lcd_gx09inx101_mipi` 为例：
配置 MIPI DSI panel。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\lcd\lcd_gx09inx101_mipi.dtsi
/ { lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {
                dsi-work-mode = <1>;          // panel中配置 mipi dsi工作模式：1 DSI_MODE_VIDEO_BURST；
                dsi-lane-number = <4>;        // panel中配置mipi dsi lane数量
                dsi-color-format = "rgb888";  // panel中配置mipi dsi 数据格式
                width-mm = <142>;             // panel中配置屏幕active area
                height-mm = <228>;            // panel中配置屏幕active area
                use-dcs-write;                // panel中配置是否使用dcs命令模式

                // mipi dsi dphy中配置timing
                height = <1920>;              // mipi dsi dphy中配置屏幕高
                width = <1200>;               // mipi dsi dphy中配置屏幕宽
                hfp = <50>;                   // mipi dsi dphy中配置水平前肩（Horizontal Front Porch）
                hbp = <40>;                   // mipi dsi dphy中配置水平后肩（Horizontal Back Porch）
                hsync = <10>;                 // mipi dsi dphy中配置水平同步信号（Horizontal Sync）
                vfp = <20>;                   // mipi dsi dphy中配置垂直前肩（Vertical Front Porch）
                vbp = <16>;                   // mipi dsi dphy中配置垂直后肩（Vertical Back Porch）
                vsync = <4>;                  // mipi dsi dphy中配置垂直同步信号（Vertical Sync）
                fps = <60>;                   // mipi dsi dphy中配置帧率
                work-mode = <0>;              // mipi dsi dphy中配置mipi dsi工作模式：0 SPACEMIT_DSI_MODE_VIDEO；
                rgb-mode = <3>;               // mipi dsi dphy中配置mipi dsi 数据格式：3 DSI_INPUT_DATA_RGB_MODE_888;
                lane-number = <4>;            // mipi dsi dphy中配置mipi dsi lane数量
                phy-bit-clock = <1000000000>; // mipi dsi dphy中配置mipi dsi dphy bit clock
                phy-esc-clock = <76800000>;   // mipi dsi dphy中配置mipi dsi dphy esc clock
                split-enable = <0>;           // mipi dsi dphy中配置mipi dsi使能split
                eotp-enable = <0>;            // mipi dsi dphy中配置mipi dsi使能eotp
                burst-mode = <2>;             // mipi dsi dphy中配置mipi dsi burst mode: 2 DSI_BURST_MODE_BURST;
                esd-check-enable = <0>;       // panel中配置使能esd check

                // mipi dsi初始化命令序列
                /* DSI_CMD, DSI_MODE, timeout, len, cmd */
                initial-command = [
                        39 01 00 02 B0 01
                        39 01 00 02 C3 4F
                        39 01 00 02 C4 40
                        39 01 00 02 C5 40
                        39 01 00 02 C6 40
                        39 01 00 02 C7 40
                        39 01 00 02 C8 4D
                        39 01 00 02 C9 52
                        39 01 00 02 CA 51
                        39 01 00 02 CD 5D
                        39 01 00 02 CE 5B
                        39 01 00 02 CF 4B
                        39 01 00 02 D0 49
                        39 01 00 02 D1 47
                        39 01 00 02 D2 45
                        39 01 00 02 D3 41
                        39 01 00 02 D7 50
                        39 01 00 02 D8 40
                        39 01 00 02 D9 40
                        39 01 00 02 DA 40
                        39 01 00 02 DB 40
                        39 01 00 02 DC 4E
                        39 01 00 02 DD 52
                        39 01 00 02 DE 51
                        39 01 00 02 E1 5E
                        39 01 00 02 E2 5C
                        39 01 00 02 E3 4C
                        39 01 00 02 E4 4A
                        39 01 00 02 E5 48
                        39 01 00 02 E6 46
                        39 01 00 02 E7 42
                        39 01 00 02 B0 03
                        39 01 00 02 BE 03
                        39 01 00 02 CC 44
                        39 01 00 02 C8 07
                        39 01 00 02 C9 05
                        39 01 00 02 CA 42
                        39 01 00 02 CD 3E
                        39 01 00 02 CF 60
                        39 01 00 02 D2 04
                        39 01 00 02 D3 04
                        39 01 00 02 D4 01
                        39 01 00 02 D5 00
                        39 01 00 02 D6 03
                        39 01 00 02 D7 04
                        39 01 00 02 D9 01
                        39 01 00 02 DB 01
                        39 01 00 02 E4 F0
                        39 01 00 02 E5 0A
                        39 01 00 02 B0 00
                        39 01 00 02 BD 50
                        39 01 00 02 C2 08
                        39 01 00 02 C4 10
                        39 01 00 02 CC 00
                        // 39 01 00 02 B2 41 // BIST pattern
                        39 01 00 02 B0 02
                        39 01 00 02 C0 00
                        39 01 00 02 C1 0A
                        39 01 00 02 C2 20
                        39 01 00 02 C3 24
                        39 01 00 02 C4 23
                        39 01 00 02 C5 29
                        39 01 00 02 C6 23
                        39 01 00 02 C7 1C
                        39 01 00 02 C8 19
                        39 01 00 02 C9 17
                        39 01 00 02 CA 17
                        39 01 00 02 CB 18
                        39 01 00 02 CC 1A
                        39 01 00 02 CD 1E
                        39 01 00 02 CE 20
                        39 01 00 02 CF 23
                        39 01 00 02 D0 07
                        39 01 00 02 D1 00
                        39 01 00 02 D2 00
                        39 01 00 02 D3 0A
                        39 01 00 02 D4 13
                        39 01 00 02 D5 1C
                        39 01 00 02 D6 1A
                        39 01 00 02 D7 13
                        39 01 00 02 D8 17
                        39 01 00 02 D9 1C
                        39 01 00 02 DA 19
                        39 01 00 02 DB 17
                        39 01 00 02 DC 17
                        39 01 00 02 DD 18
                        39 01 00 02 DE 1A
                        39 01 00 02 DF 1E
                        39 01 00 02 E0 20
                        39 01 00 02 E1 23
                        39 01 00 02 E2 07
                        39 01 F0 01 11
                        39 01 28 01 29
                ];
                
                // mipi dsi休眠命令序列
                sleep-in-command = [
                        39 01 78 01 28
                        39 01 78 01 10
                ];
                
                // mipi dsi唤醒命令序列
                sleep-out-command = [
                        39 01 96 01 11
                        39 01 32 01 29
                ];
                
                // mipi dsi read id命令序列
                read-id-command = [
                        37 01 00 01 05
                        14 01 00 05 fb fc fd fe ff
                ];

                // mipi dsi dpu中配置timing
                display-timings {
                        timing0 {
                                clock-frequency = <153000000>;    // 像素时钟
                                hactive = <1200>;                 // 有效显示区域水平像素
                                hfront-porch = <50>;              // hfp
                                hback-porch = <40>;               // hbp
                                hsync-len = <10>;                 // hsync
                                vactive = <1920>;                 // 有效显示区域垂直像素
                                vfront-porch = <20>;              // vfp
                                vback-porch = <16>;               // vbp
                                vsync-len = <4>;                  // vsync
                                vsync-active = <1>;               // vsync信号高电平触发
                                hsync-active = <1>;               // hsync信号高电平触发
                        };
                };
        };
};};
```

**方案配置示例：**

以 k1-x_deb1 方案为例：
选择 MIPI DSI panel 型号 `lcd_gx09inx101_mipi`，配置方案 MIPI DSI panel。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
	status = "okay";                                // 使能mipi dsi dpu
	memory-region = <&dpu_resv>;                    // 配置mipi dsi dpu预留内存
	spacemit-dpu-bitclk = <1000000000>;             // mipi dsi dpu bitclk 配置
	spacemit-dpu-escclk = <76800000>;               // mipi dsi dpu escclk 配置
	dsi_1v2-supply = <&ldo_5>;                      // 引用PMIC DLDO
	vin-supply-names = "dsi_1v2";                   // 配置MIPI DSI 1.2v电源

};

&dsi2 {
	status = "okay";                                // 使能mipi dsi host

	panel2: panel2@0 {
		status = "okay";                        // 使能panel
		compatible = "spacemit,mipi-panel2";
		reg = <0>;

		gpios-reset = <81>;                     // panel 复位 gpio
		gpios-dc = <82 83>;                     // panel 电源控制 gpio
		id = <2>;                               // 配置 panel id
		delay-after-reset = <10>;               // 配置 plane 复位延时时间（单位：ms）
		force-attached = "lcd_gx09inx101_mipi"; // 配置 plane 型号
	};

};

&lcds {
	status = "okay";                                 // 使能lcds
};

&pwm14 {                                                // 配置pwm
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm14_1>;
	status = "okay";

};

&pwm_bl {                                               // 配置背光
 	pwms = <&pwm14 2000>;
	brightness-levels = <
		0   40  40  40  40  40  40  40  40  40  40  40  40  40  40  40
		40  40  40  40  40  40  40  40  40  40  40  40  40  40  40  40
		40  40  40  40  40  40  40  40  40  41  42  43  44  45  46  47
		48  49  50  51  52  53  54  55  56  57  58  59  60  61  62  63
		64  65  66  67  68  69  70  71  72  73  74  75  76  77  78  79
		80  81  82  83  84  85  86  87  88  89  90  91  92  93  94  95
		96  97  98  99  100 101 102 103 104 105 106 107 108 109 110 111
		112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127
		128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
		144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159
		160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175
		176 177 178 179 180 181 182 183 184 185 186 187 188 189 190 191
		192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207
		208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223
		224 225 226 227 228 229 230 231 232 233 234 235 236 237 238 239
		240 241 242 243 244 245 246 247 248 249 250 251 252 253 254 255
	>;
	default-brightness-level = <100>;

	status = "okay";
};

```

#### HDMI

##### pinctrl 配置

支持两组 HDMI pinctrl：`pinctrl_hdmi_0` 和 `pinctrl_hdmi_1`，仅可选择其中任意一组。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_pinctrl.dtsi
pinctrl_hdmi_0: hdmi_0_grp {
        pinctrl-single,pins = <
                K1X_PADCONF(GPIO_86, MUX_MODE1, (EDGE_NONE | PULL_UP   | PAD_1V8_DS2)) /* hdmi_tx_hscl */
                K1X_PADCONF(GPIO_87, MUX_MODE1, (EDGE_NONE | PULL_UP   | PAD_1V8_DS2)) /* hdmi_tx_hsda */
                K1X_PADCONF(GPIO_88, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* hdmi_tx_hcec */
                K1X_PADCONF(GPIO_89, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* hdmi_tx_pdp */
        >;
};

pinctrl_hdmi_1: hdmi_1_grp {
        pinctrl-single,pins = <
                K1X_PADCONF(GPIO_59, MUX_MODE1, (EDGE_NONE | PULL_UP   | PAD_1V8_DS2)) /* hdmi_tx_hscl */
                K1X_PADCONF(GPIO_60, MUX_MODE1, (EDGE_NONE | PULL_UP   | PAD_1V8_DS2)) /* hdmi_tx_hsda */
                K1X_PADCONF(GPIO_61, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* hdmi_tx_hcec */
                K1X_PADCONF(GPIO_62, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* hdmi_tx_pdp */
        >;
};
```

##### Clock 配置

HDMI 模块涉及的 clock 配置主要包括：
- HDMI DPU（数据处理单元）相关 clock 和 reset
- HDMI 控制器本身相关的 clock 和 reset

相关的 clock 参数使用系统默认值，**无需在 DTS 文件中显式配置**。

以下为 HDMI DPU 与 HDMI 控制器在平台上的配置相关 clock 和 reset 示例（以 ·k1-x-hdmi.dtsi· 为例）：

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x-hdmi.dtsi
&soc {
        display-subsystem-hdmi {
                compatible = "spacemit,saturn-hdmi";
                reg = <0 0xc0440000 0 0x2A000>;
                ports = <&dpu_online2_hdmi>;
                interconnects = <&dram_range1>;
                interconnect-names = "dma-mem";
        };

        dpu_online2_hdmi: port@c0440000 {
                compatible = "spacemit,dpu-online2";
                interrupt-parent = <&intc>;
                interrupts = <139>, <138>;
                interrupt-names = "ONLINE_IRQ", "OFFLINE_IRQ";
                interconnects = <&dram_range1>;
                interconnect-names = "dma-mem";
                clocks = <&ccu CLK_HDMI>;                               // hdmi dpu hmclk 配置
                clock-names = "hmclk";
                resets = <&reset RESET_HDMI>;                           // hdmi dpu reset 配置
                reset-names= "hdmi_reset";
                power-domains = <&power K1X_PMU_HDMI_PWR_DOMAIN>;
                pipeline-id = <ONLINE2>;
                ip = "spacemit-saturn";
                type = <HDMI>;
                clk,pm-runtime,no-sleep;
                status = "disabled";

                dpu_online2_hdmi_out: endpoint@0 {
                        remote-endpoint = <&hdmi_in>;
                };
        };

        hdmi: hdmi@C0400500 {
                compatible = "spacemit,hdmi", "simple-bus";
                #address-cells = <2>;
                #size-cells = <2>;
                ranges;
                reg = <0 0xC0400500 0 0x200>;
                interrupt-parent = <&intc>;
                interrupts = <136>;
                clocks = <&ccu CLK_HDMI>;                               // hdmi hmclk 配置
                clock-names = "hmclk";
                resets = <&reset RESET_HDMI>;                           // hdmi reset 配置
                reset-names= "hdmi_reset";
                power-domains = <&power K1X_PMU_HDMI_PWR_DOMAIN>;
                clk,pm-runtime,no-sleep;
                status = "disabled";

                port {
                        #address-cells = <1>;
                        #size-cells = <0>;
                        hdmi_in: endpoint@0 {
                                reg = <0>;
                                remote-endpoint = <&dpu_online2_hdmi_out>;
                        };
                };
        };
};
```

##### DTS 配置示例

以下以 `k1-x_deb1` 方案为例，配置 HDMI。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;                    // 配置hdmi dpu预留内存
	status = "okay";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "okay";
};
```

## 接口介绍

### API 介绍

DRM 驱动 API 介绍请参考 Linux 内核文档 [drm-kms](https://docs.kernel.org/gpu/drm-kms.html)

## Debug 介绍

### debugfs

- **MIPI DSI debugfs 节点**

```
# cd /sys/kernel/debug/dri/1
# ls
DSI-1             crtc-0            gem_names         state
bridge_chains     dump              internal_clients
clients           framebuffer       name
```

- **HDMI debugfs节点**

```
# cd /sys/kernel/debug/dri/2
# ls
HDMI-A-1          crtc-0            gem_names         state
bridge_chains     dump              internal_clients
clients           framebuffer       name
```

- **查看 framebuffer 信息**

```
# cd /sys/kernel/debug/dri/1
# cat framebuffer
framebuffer[132]:
        allocated by = weston
        refcount=2
        format=XR24 little-endian (0x34325258)
        modifier=0x0
        size=1200x1920
        layers:
                size[0]=1200x1920
                pitch[0]=4800
                offset[0]=0
                obj[0]:
                        name=0
                        refcount=3
                        start=0010119c
                        size=9216000
                        imported=no
framebuffer[135]:
        allocated by = weston
        refcount=1
        format=XR24 little-endian (0x34325258)
        modifier=0x0
        size=1200x1920
        layers:
                size[0]=1200x1920
                pitch[0]=4800
                offset[0]=0
                obj[0]:
                        name=0
                        refcount=3
                        start=001008d2
                        size=9216000
                        imported=no
framebuffer[134]:
        allocated by = weston
        refcount=1
        format=AR24 little-endian (0x34325241)
        modifier=0x0
        size=64x64
        layers:
                size[0]=64x64
                pitch[0]=256
                offset[0]=0
                obj[0]:
                        name=0
                        refcount=3
                        start=001008ce
                        size=16384
                        imported=no
framebuffer[133]:
        allocated by = weston
        refcount=1
        format=AR24 little-endian (0x34325241)
        modifier=0x0
        size=64x64
        layers:
                size[0]=64x64
                pitch[0]=256
                offset[0]=0
                obj[0]:
                        name=0
                        refcount=3
                        start=001008ca
                        size=16384
                        imported=no
framebuffer[131]:
        allocated by = [fbcon]
        refcount=1
        format=XR24 little-endian (0x34325258)
        modifier=0x0
        size=1200x1920
        layers:
                size[0]=1200x1920
                pitch[0]=4800
                offset[0]=0
                obj[0]:
                        name=0
                        refcount=2
                        start=00100000
                        size=9216000
                        imported=no
```

- **dump 当前显示的 buffer**

```
# cd /sys/class/drm/card1-DSI-1
# cat dump
[  436.711209] [drm] framebuffer[135]
[  436.752130] [drm] dump framebuffer: /tmp/plane31_fb135_XR24_planes0_1200x1920.rgb
[  436.759796] [drm] framebuffer[134]
[  436.763663] [drm] dump framebuffer: /tmp/plane43_fb134_AR24_planes0_64x64.rgb
```

- **查看连接状态**

```
# cd /sys/class/drm/card1-DSI-1
# cat status
connected
```

- **查看支持的显示模式**

```
# cd /sys/class/drm/card1-DSI-1
# cat modes
1200x1920
```

## 测试介绍

libdrm 是一个用户空间库，提供了与 DRM 驱动进行交互的 API。通过 libdrm 开发者可以直接与 DRM 设备进行通信，执行各种操作，如创建和管理 framebuffer、设置显示模式、处理图层等。**modetest** 是一个使用 libdrm 库的测试工具，通常用于测试和验证 DRM 驱动的功能。

通过 modetest 工具运行 DRM 驱动测试用例如下。

```
# modetest -M spacemit
Encoders:
id      crtc    type    possible crtcs  possible clones
129     127     DSI     0x00000001      0x00000001

Connectors:
id      encoder status          name            size (mm)       modes   encoders
130     129     connected       DSI-1           142x228         1       129
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 1200x1920 60.05 1200 1250 1260 1300 1920 1940 1944 1960 153000 flags: phsync, pvsync; type: preferred, driver
  props:
        1 EDID:
                flags: immutable blob
                blobs:

                value:
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 0
        5 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        6 non-desktop:
                flags: immutable range
                values: 0 1
                value: 0
        4 TILE:
                flags: immutable blob
                blobs:

                value:

CRTCs:
id      fb      pos     size
127     132     (0,0)   (1200x1920)
  #0 1200x1920 60.05 1200 1250 1260 1300 1920 1940 1944 1960 153000 flags: phsync, pvsync; type: preferred, driver
  props:
        24 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
31      127     132     0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        33 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        34 zpos:
                flags: immutable range
                values: 0 0
                value: 0
        35 alpha:
                flags: range
                values: 0 65535
                value: 65535
        36 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        37 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 0
        38 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        41 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        42 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
43      127     134     0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 2
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        45 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        46 zpos:
                flags: immutable range
                values: 1 1
                value: 1
        47 alpha:
                flags: range
                values: 0 65535
                value: 65535
        48 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        49 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 1
        50 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        53 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        54 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
55      0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        57 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        58 zpos:
                flags: immutable range
                values: 2 2
                value: 2
        59 alpha:
                flags: range
                values: 0 65535
                value: 65535
        60 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        61 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        62 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        65 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        66 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
67      0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        69 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        70 zpos:
                flags: immutable range
                values: 3 3
                value: 3
        71 alpha:
                flags: range
                values: 0 65535
                value: 65535
        72 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        73 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        74 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        77 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        78 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
79      0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        81 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        82 zpos:
                flags: immutable range
                values: 4 4
                value: 4
        83 alpha:
                flags: range
                values: 0 65535
                value: 65535
        84 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        85 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        86 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        89 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        90 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
91      0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        93 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        94 zpos:
                flags: immutable range
                values: 5 5
                value: 5
        95 alpha:
                flags: range
                values: 0 65535
                value: 65535
        96 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        97 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        98 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        101 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        102 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
103     0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        105 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        106 zpos:
                flags: immutable range
                values: 6 6
                value: 6
        107 alpha:
                flags: range
                values: 0 65535
                value: 65535
        108 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        109 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        110 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        113 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        114 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0
115     0       0       0,0             0,0     0               0x00000001
  formats: AB30 AR24 AB24 RA24 BA24 XR24 XB24 RX24 BX24 RG16 BG16 YU08 NV12 P010
  props:
        8 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        30 IN_FORMATS:
                flags: immutable blob
                blobs:

                value:
                        01000000000000000e00000018000000
                        01000000500000004142333041523234
                        41423234524132344241323458523234
                        58423234525832344258323452473136
                        42473136595530384e56313250303130
                        ff3f0000000000000000000000000000
                        0000000000000000
                in_formats blob decoded:
                         AB30:  LINEAR
                         AR24:  LINEAR
                         AB24:  LINEAR
                         RA24:  LINEAR
                         BA24:  LINEAR
                         XR24:  LINEAR
                         XB24:  LINEAR
                         RX24:  LINEAR
                         BX24:  LINEAR
                         RG16:  LINEAR
                         BG16:  LINEAR
                         YU08:  LINEAR
                         NV12:  LINEAR
                         P010:  LINEAR
        117 rotation:
                flags: bitmask
                values: rotate-0=0x1 rotate-90=0x2 rotate-180=0x4 rotate-270=0x8 reflect-x=0x10 reflect-y=0x20
                value: 1
        118 zpos:
                flags: immutable range
                values: 7 7
                value: 7
        119 alpha:
                flags: range
                values: 0 65535
                value: 65535
        120 pixel blend mode:
                flags: enum
                enums: None=2 Pre-multiplied=0 Coverage=1
                value: 0
        121 RDMA_ID:
                flags: range
                values: 0 18446744073709551615
                value: 4294967295
        122 SOLID_COLOR:
                flags: range
                values: 0 18446744073709551615
                value: 0
        125 COLOR_ENCODING:
                flags: enum
                enums: ITU-R BT.601 YCbCr=0 ITU-R BT.709 YCbCr=1 ITU-R BT.2020 YCbCr=2
                value: 0
        126 COLOR_RANGE:
                flags: enum
                enums: YCbCr limited range=0 YCbCr full range=1
                value: 0

Frame buffers:
id      size    pitch

# modetest  -M  spacemit -s 130@127:1200x1920
setting mode 1200x1920-60.05Hz on connectors 130, crtc 127
```

## FAQ
