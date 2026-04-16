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
      - 图层有以下两种类型：
	1) **主图层（primary plane）：** 用于显示背景或者图像内容
	2) **叠加图层（overlay plane）：** 常用于视频、字幕等附加显示

    - **Encoder（编码器）**
      - 负责电源管理、视频输出格式封装
      - 把时序转换为显示器所需要的信号，将画面显示到不同的显示设备，例如将视频输出到 HDMI 接口、MIPI DSI 接口等。

    - **Connector（连接器）**
      - 负责硬件显示设备的接入、屏参获取等，例如 HDMI, MIPI DSI 等。

在内核空间中，还包括 **Panel（显示面板）** 模块，负责将接收到的图像信号转换为最终在屏幕上显示的图像内容。

### 源码结构介绍

SpacemiT 平台 DRM 驱动源码结构如下：

```
spacemit
├── backlight
│   ├── built-in.a
│   ├── Kconfig
│   ├── Makefile
│   ├── modules.order
│   ├── spacemit-backlight.c
│   ├── spacemit-backlight.h
├── built-in.a
├── common
│   ├── spacemit_drm_notifier.c
│   ├── spacemit_drm_notifier.h
├── dphy
│   ├── spacemit_dphy_drv.c // mipi dsi dphy驱动
├── dpu
│   ├── dpu_debug.c
│   ├── dpu_debug.h
│   ├── dpu_saturn.c
│   ├── dpu_saturn.h
│   ├── dpu_saturn_hee.c
│   ├── dpu_trace.h
│   ├── post_process_hee.c
│   ├── post_process_hee.h
│   ├── saturn_fbcmem.c
│   ├── saturn_fbcmem.h
│   └── saturn_regs
│       ├── acad.h
│       ├── cmdlist_top.h
│       ├── composer_x.h
│       ├── dpu_ctl_top.h
│       ├── dpu_int.h
│       ├── dpu_scene_ctl.h
│       ├── dpu_top.h
│       ├── dsc_enc_top.h
│       ├── ee.h
│       ├── ltm.h
│       ├── lut_3d.h
│       ├── mmu_tbu_x.h
│       ├── mmu_top.h
│       ├── ops_hee.h
│       ├── postpipe.h
│       ├── prepipe_x.h
│       ├── rc.h
│       ├── rdma_path.h
│       ├── rdma_top.h
│       ├── reg_map.h
│       ├── reg_map_hee.h
│       ├── scale_x.h
│       ├── tmg.h
│       ├── usr_gma.h
│       └── wb.h
├── dsi
│   ├── spacemit_dptc_drv.c
│   ├── spacemit_dptc_drv.h
│   ├── spacemit_dsi_drv.c
│   └── spacemit_dsi_hw.h
├── Kconfig
├── Makefile
├── raspberrypi_touchscreen.c
├── selftest
│   ├── Kconfig
│   ├── Makefile
│   └── test-spacemit_fbcmem.c
├── spacemit_bootloader.c
├── spacemit_bootloader.h
├── spacemit_cmdlist.c
├── spacemit_cmdlist.h
├── spacemit_crtc.c
├── spacemit_crtc.h
├── spacemit_dmmu.c
├── spacemit_dmmu.h
├── spacemit_dphy.c
├── spacemit_dphy.h
├── spacemit_dpu_reg.h
├── spacemit_drm.c
├── spacemit_drm.h
├── spacemit_dsi.c
├── spacemit_dsi.h
├── spacemit_gem.c
├── spacemit_gem.h
├── spacemit_inno_dp.c
├── spacemit_inno_dp.h
├── spacemit_lib.c
├── spacemit_lib.h
├── spacemit_mipi_panel.c
├── spacemit_mipi_panel.h
├── spacemit_planes.c
├── spacemit_wb.c
├── spacemit_wb.h
└── sysfs
    ├── sysfs_class.c
    ├── sysfs_display.h
    ├── sysfs_dphy.c
    ├── sysfs_dpu.c
    ├── sysfs_dsi.c
    └── sysfs_mipi_panel.c
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 MIPI DSI | 支持 MIPI DPHY v1.2， 最高支持 DPHY 8 lane，最高速率 4.5Gbps/lane |
| 支持 DP | 支持 DP 1.2，支持1/2/4-lane，5.4Gbps/lane |
| 支持 eDP | 支持 eDP 1.4，支持1/2/4-lane，5.4Gbps/lane |
| 支持回写 | 支持在线合成、回写，离线回写。在线离线功能无法同时工作|

### 性能参数

| 屏幕接口 | 性能规格 |
| :-----| :----|
| MIPI DSI| 2560x1400@90FPS |
| DP | 3840x2160@60FPS |
| DP | 2560x1400@90FPS |
| eDP | 3840x2160@60FPS |
| eDP | 2560x1400@90FPS |

**屏幕帧率测试方法:**

- 查看 Connectors：

```
# modetest -M spacemit -D /dev/dri/card0 -c
# 参数说明：
# -M spacemit: 指定 DRM 驱动模块名称为 spacemit
# -D /dev/dri/card0: 指定 DRM 设备节点路径
# -c: 列出所有可用的 connectors（连接器）信息
Connectors:
id      encoder status          name            size (mm)       modes   encoders
257     256     connected       DSI-1           95x53           1       256
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 800x480 57.74 800 801 803 849 480 487 489 510 25000 flags: ; type: preferred, driver
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
        258 panel_name:
                flags: immutable blob
                blobs:

                value:

```

- 查看 Encoders：

```
# modetest -M spacemit -D /dev/dri/card0 -e
# 参数说明：
# -M spacemit: 指定 DRM 驱动模块名称为 spacemit
# -D /dev/dri/card0: 指定 DRM 设备节点路径
# -e: 列出所有可用的 encoders（编码器）信息
Encoders:
id      crtc    type    possible crtcs  possible clones
256     241     DSI     0x00000001      0x00000003
260     0       Virtual 0x00000001      0x00000003

```

- 测试屏幕帧率：

```
# modetest -M spacemit -D /dev/dri/card0 -s 257@241:800x480 -v
# 参数说明：
# -M spacemit: 指定 DRM 驱动模块名称为 spacemit
# -D /dev/dri/card0: 指定 DRM 设备节点路径
# -s <connector_id>@<crtc_id>:<mode>: 设置显示模式
#    connector_id: 来自 -c 输出的 connector id，例如 257（DSI-1）
#    crtc_id: 来自 -e 输出的 crtc id，例如 241
#    mode: 分辨率格式为 <宽>x<高>，例如 800x480
# -v: 开启垂直同步并输出帧率统计信息
setting mode 800x480-57.74Hz on connectors 257, crtc 241
failed to set gamma: Function not implemented
freq: 59.34Hz
freq: 59.12Hz
freq: 59.12Hz

```

## 配置介绍

主要包括 **Display 驱动使能配置** 和 **DTS 配置**，K3 芯片共有两个显示通路，其中通路0支持 MIPI DSI 或者 DP/EDP 硬件接口，通路1支持 DP/EDP 硬件接口。

### CONFIG 配置

`CONFIG_DRM_SPACEMIT`：SpacemiT 平台 DRM 驱动配置选项，默认情况，此选项为 `Y`，并作为启用 MIPI DSI 驱动或 DP/EDP 驱动的前置条件。用户可根据需求选择单独配置启用 MIPI  DP/EDP 或两者同时启用的显示输出方案。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   < >   mipi panel support for SPACEMIT SOCs platform
   < >   Spacemit specific extensions for inno dp/edp
```

#### MIPI DSI CONFIG 配置

`CONFIG_SPACEMIT_MIPI_PANEL`：SpacemiT 平台 MIPI DSI 驱动配置选项，具体方案根据需要进行配置。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   mipi panel support for SPACEMIT SOCs platform
```

#### DP/eDP CONFIG 配置

`CONFIG_SPACEMIT_HDMI`：SpacemiT 平台 DP/eDP 驱动配置选项，具体方案根据需要进行配置。

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   Spacemit specific extensions for inno dp/edp
```

### DTS 配置

#### MIPI DSI

##### GPIO

MIPI DSI panel GPIO 相关配置，包括 **panel 复位 GPIO 配置**、**panel 背光控制 GPIO 配置** 和 **panel 电源控制 GPIO 配置**。

**新版本配置方式（推荐）：**

以 k3_evb 方案为例，使用 `reset-gpios`、`bl-gpios`、`dc0-gpios`、`dc1-gpios` 属性：

```c
// linux-6.18\arch\riscv\boot\dts\spacemit\k3_evb.dts
&dsi0 {
	status = "okay";

	panel0: panel0@0 {
		status = "ok";
		compatible = "spacemit,mipi-panel";
		reg = <0>;
		reset-gpios = <&gpio 1 31 GPIO_ACTIVE_HIGH>;  // 配置 panel 复位 gpio (gpio63)
		bl-gpios = <&gpio 1 29 GPIO_ACTIVE_HIGH>;     // 配置 panel 背光控制 gpio (gpio61)
		dc0-gpios = <&gpio 1 24 GPIO_ACTIVE_HIGH>;    // 配置 panel 电源控制 gpio0 (gpio56)
		dc1-gpios = <&gpio 1 25 GPIO_ACTIVE_HIGH>;    // 配置 panel 电源控制 gpio1 (gpio57)
		id = <2>;
		force-attached = "lcd_icnl9911c_mipi";
	};
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
// linux-6.18/arch/riscv/boot/dts/spacemit/lcd_dsi_panel.dtsi
// SPDX-License-Identifier: (GPL-2.0 OR MIT)
/* Copyright (c) 2025 Spacemit, Inc */

&soc {

	display-subsystem-dsi {
		compatible = "spacemit,saturn-hee";
		reg = <0 0xc0340000 0 0x54000>;
		hw_ver = <2>;
		ports = <&dpu_crtc0>, <&dpu_crtc1>; //dpu_crtc0用于在线合成、回写，dpu_crtc1用于离线回写
	};

	dpu_crtc0: dpu_crtc0 {
		compatible = "spacemit,dpu-saturn";
		reg = <0x0 0xd421a800 0x0 0x200>,
			<0x0 0xd421aa00 0x0 0x200>;
		interrupt-parent = <&saplic>;
		interrupts = <90 IRQ_TYPE_LEVEL_HIGH>;
		interrupt-names = "ONLINE_IRQ";
		clocks = <&syscon_apmu CLK_APMU_LCD_PXCLK>,
			<&syscon_apmu CLK_APMU_LCD_MCLK>,
			<&syscon_apmu CLK_APMU_LCD_HCLK>,
			<&syscon_apmu CLK_APMU_DSI_ESC>,
			<&syscon_apmu CLK_APMU_DPU_ACLK>,
			<&syscon_apmu CLK_APMU_LCD_DSC>;
		clock-names = "pxclk", "mclk", "hclk", "escclk", "aclk", "dscclk";
		resets = <&syscon_apmu RESET_APMU_LCD_MCLK>,
			<&syscon_apmu RESET_APMU_LCD>,
			<&syscon_apmu RESET_APMU_DSI_ESC>,
			<&syscon_apmu RESET_APMU_DPU_ACLK>,
			<&syscon_apmu RESET_APMU_LCD_DSCCLK>;
		reset-names= "mclk_reset", "lcd_reset", "esc_reset", "aclk_reset", "dsc_reset";
		power-domains = <&power K3_PMU_LCD0_PWR_DOMAIN>;
		pipeline-id = <1>;
		is_edp = <0>;
		dpu-id = <0>;
		ip = "spacemit-saturn";
		spacemit-dpu-min-mclk = <40960000>;
		spacemit-dpu-dsipll;
		status = "disabled";

		ports {
			#address-cells = <1>;
			#size-cells = <0>;

			port@0 {
				reg = <0>;
				#address-cells = <1>;
				#size-cells = <0>;

				dpu_crtc0_out0: endpoint@0 {
					reg = <0>;
					remote-endpoint = <&dsi0_in>;
				};

				dpu_crtc0_out1: endpoint@1 {
					reg = <1>;
					remote-endpoint = <&wb0_in0>;
				};
			};
		};
	};

	dpu_crtc1: dpu_crtc1 {
		compatible = "spacemit,dpu-saturn";
		reg = <0x0 0xd421a800 0x0 0x200>,
			<0x0 0xd421aa00 0x0 0x200>;
		interrupt-parent = <&saplic>;
		interrupts = <89 IRQ_TYPE_LEVEL_HIGH>;
		interrupt-names = "OFFLINE_IRQ";
		clocks = <&syscon_apmu CLK_APMU_LCD_PXCLK>,
			<&syscon_apmu CLK_APMU_LCD_MCLK>,
			<&syscon_apmu CLK_APMU_LCD_HCLK>,
			<&syscon_apmu CLK_APMU_DSI_ESC>,
			<&syscon_apmu CLK_APMU_DPU_ACLK>,
			<&syscon_apmu CLK_APMU_LCD_DSC>;
		clock-names = "pxclk", "mclk", "hclk", "escclk", "aclk", "dscclk";
		resets = <&syscon_apmu RESET_APMU_LCD_MCLK>,
			<&syscon_apmu RESET_APMU_LCD>,
			<&syscon_apmu RESET_APMU_DSI_ESC>,
			<&syscon_apmu RESET_APMU_DPU_ACLK>,
			<&syscon_apmu RESET_APMU_LCD_DSCCLK>;
		reset-names= "mclk_reset", "lcd_reset", "esc_reset", "aclk_reset", "dsc_reset";
		power-domains = <&power K3_PMU_LCD0_PWR_DOMAIN>;
		pipeline-id = <2>;
		is_edp = <1>;
		ip = "spacemit-saturn";
		spacemit-dpu-min-mclk = <40960000>;
		status = "disabled";
		ports {
			#address-cells = <1>;
			#size-cells = <0>;

			port@0 {
				reg = <0>;
				dpu_crtc1_out0: endpoint@0 {
					remote-endpoint = <&wb0_in1>;
				};
			};
		};
	};

	dsi0: dsi0@d421a800 {
		compatible = "spacemit,dsi-host";
		#address-cells = <1>;
		#size-cells = <0>;
		reg = <0 0xd421a800 0 0x200>;
		interrupt-parent = <&saplic>;
		interrupts = <79 IRQ_TYPE_LEVEL_HIGH>;
		ip = "synopsys-dhost";
		dev-id = <2>;
		status = "disabled";

		ports {
			#address-cells = <1>;
			#size-cells = <0>;

			port@0 {
				reg = <0>;
				dsi0_out: endpoint {
					remote-endpoint = <&dphy0_in>;
				};
			};

			port@1 {
				reg = <1>;
				dsi0_in: endpoint {
					remote-endpoint = <&dpu_crtc0_out0>;
				};
			};
		};
	};

	wb0 {
		compatible = "spacemit,wb0";
		dev-id = <0>;
		status = "ok";
		ports {
			#address-cells = <1>;
			#size-cells = <0>;
			port@0 {
				reg = <0>;
				wb0_in0: endpoint {
					remote-endpoint = <&dpu_crtc0_out1>;
				};
			};
			port@1 {
				reg = <1>;
				wb0_in1: endpoint {
					remote-endpoint = <&dpu_crtc1_out0>;
				};
			};
		};
	};

	dphy0: dphy0@d421a800 {
		compatible = "spacemit,dsi-phy";
		#address-cells = <1>;
		#size-cells = <0>;
		reg = <0 0xd421a800 0 0x200>;
		ip = "spacemit-dphy";
		dev-id = <2>;
		status = "ok";

		port@0 {
			reg = <0>;
			dphy0_in: endpoint {
				remote-endpoint = <&dsi0_out>;
			};
		};
	};
};
```

配置方案 MIPI DSI DPU bitclk 及 escclk。
以 k3-x_evb 方案为例：

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_evb.dts
&dpu_crtc0 {
	memory-region = <&dpu_resv0>;
	spacemit-dpu-bitclk = <624000000>;
	status = "okay";
};
```

配置 Panel 型号 MIPI DSI DPHY bitclk 及 escclk。
以 MIPI DSI panel 型号 lcd_icnl9911c_mipi 为例：

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_icnl9911c_mipi.dtsi
/ { lcds: lcds {
	lcd_icnl9911c_mipi: lcd_icnl9911c_mipi {

		phy-freq = <624000>;   // mipi dsi dphy bitclk 配置
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
MIPI DSI 数据传输过程中，每 Lane 的数据传输时钟。SpacemiT 平台驱动中使用 `bitclk_min` 和 `bitclk_max` 两个参数约束 PLL 锁相范围，DTS 中配置的 `phy-bit-clock` 需落在此范围内。

Bit clock 计算方法：
   ```
   bitclk_min = (((hactive × bpp(3) + hbp × bpp(3) + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1)×8

   bitclk_max  = pixel clock × 4 × 8 / lane_number

   配置选择 bitclk_min 和 bitclk_max之间的数值。
  ```

- **DSI clock:**
MIPI DSI 的 clock lane 实际时钟信号, 采用双边沿采样，一个时钟可以传两个 bit 数据。
   ```
   dsi clock = bit clock / 2
   ```

**注意：**
在 SpacemiT 平台中计算 MIPI DSI 的 Bit Clock 时，需根据以上公式计算获得 `bitclk_min` 和 `bitclk_max` 的范围，DTS 中配置的 `phy-bit-clock` 应落在该范围内。

以下以 MIPI DSI 面板型号 `lcd_icnl9911c_mipi` 为例，说明如何计算并配置 pixel clock 与 bit clock。

**Pixel Clock 计算**
```
pixel clock = (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps 
	    = (720 + 48 + 48 + 4) * (1600 + 150 + 32 + 4) * 60 
	    = 820 × 1786 × 60
	    = 87,847,200 Hz ≈ 88 MHz
```

**Bit Clock 计算**
```
bitclk_min = (((hactive × bpp + hbp × bpp + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1) × 8
           = (((720 × 3 + 48 × 3 + 2 × 4) × 87.9012) / ((4+48+720-0.65 × 87.9012) × 4) + 1) × 8
           = (((2160 + 144 + 8) × 87.9012) / ((772 - 57.136) × 4) + 1) × 8
           = ((2312 × 87.9012) / (714.864 × 4) + 1) × 8
           = (203226.3744 / 2859.456 + 1) × 8
           = (71.06 + 1) × 8
           = 72.06 × 8
           ≈ 576.48 MHz

bitclk_max = pixel clock × 4 × 8 / lane_number
           = 87,847,200 × 4 × 8 / 4
           = 702,777,600 Hz ≈ 703 MHz

选取配置值 : 600 MHz

其中 `bitclk_min` 和 `bitclk_max` 范围由上述公式直接计算得出。
```

通过 Display timing 计算：
pixel clock 值为 87,847,200 Hz，配置为 88,000,000 Hz
bit clock 值选取 bitclk_min 到 bitclk_max 之间，配置为 624,000,000 Hz（实际配置值）

DTS 配置建议：
- `clock-frequency = 88000000`
- `spacemit-dpu-bitclk = 624000000`

---

已完成功能调试的 MIPI DSI panel，相关 dtsi 文件放置 lcd 目录。

```
linux-6.6/arch/riscv/boot/dts/spacemit/lcd$ tree
.
|-- lcd_gc9503v_mipi.dtsi
|-- lcd_icnl9911c_mipi.dtsi
|-- lcd_nt36523_mipi.dtsi
|-- lcd_tc358762xbg_dpi_800x480.dtsi
```

##### Panel 配置

使能 MIPI DSI Panel，需使能 
- MIPI DSI DPU
- MIPI DSI Host
- LCD 面板节点（lcds）
- 指定的 Panel 型号
- GPIO 背光控制

以 k3_evb 方案为例, 需进行如下配置：
- 使能 `dpu_crtc0`
- 使能 `dsi0`
- 使能 `lcds`
- 配置 panel 为型号 `lcd_icnl9911c_mipi`
- 配置 GPIO 背光，也可配置为PWM。

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_evb.dts
&dpu_crtc0 {
	spacemit-dpu-bitclk = <600000000>;
	memory-region = <&dpu_resv0>;
	status = "okay";
};

&dpu_crtc1 {
	memory-region = <&dpu_resv1>;
	status = "disabled";
}; //用于离线回写，默认不需要该功能

&dsi0 {
	status = "okay";

	panel0: panel0@0 {
		status = "ok";
		compatible = "spacemit,mipi-panel";
		reg = <0>;
		reset-gpios = <&gpio 1 31 GPIO_ACTIVE_HIGH>;
		dc0-gpios = <&gpio 1 24 GPIO_ACTIVE_HIGH>;
		dc1-gpios = <&gpio 1 25 GPIO_ACTIVE_HIGH>;
		id = <2>;
		force-attached = "lcd_icnl9911c_mipi";
	};
};

&lcds {
	status = "okay";
};
```

##### DTS 配置示例

**MIPI DSI Panel配置示例：**

以 MIPI DSI panel 型号 `lcd_icnl9911c_mipi` 为例：
配置 MIPI DSI panel。

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_icnl9911c_mipi.dtsi
// SPDX-License-Identifier: GPL-2.0

/ { lcds: lcds {
	lcd_icnl9911c_mipi: lcd_icnl9911c_mipi {
		dsi-work-mode = <1>; /* video burst mode*/
		dsi-lane-number = <4>;
		dsi-color-format = "rgb888";
		width-mm = <72>;
		height-mm = <126>;
		use-dcs-write;

		/*mipi info*/
		height = <1600>;
		width = <720>;
		hfp = <48>;
		hbp = <48>;
		hsync = <4>;
		vfp = <150>;
		vbp = <32>;
		vsync = <4>;
		fps = <60>;
		work-mode = <0>;
		rgb-mode = <3>;
		lane-number = <4>;
		phy-freq = <624000>;
		phy-escape-clock = <52000>;
		phy-bit-clock = <624000000>;
		split-enable = <0>;
		eotp-enable = <0>;
		burst-mode = <2>;
		esd-check-enable = <0>;
		vpn-tx-dly-cnt = <76>;
		vpn-dly-cnt = <540>;

		/* DSI_CMD, DSI_MODE, timeout, len, cmd */
		initial-command = [
			39 01 00 03 F0 5A 59
			39 01 00 03 F1 A5 A6
			39 01 00 21 B0 83 82 86 87 06 07 04 05 33 33 33 33 20 00 00 77 00 00 3F 05 04 03 02 01 02 03 04 00 00 00 00 00
			39 01 00 1e B1 13 91 8E 81 20 00 00 77 00 00 04 08 54 00 00 00 44 40 02 01 40 02 01 40 02 01 40 02 01
			39 01 00 12 B2 54 C4 82 05 40 02 01 40 02 01 05 05 54 0C 0C 0D 0B
			39 01 00 20 B3 12 00 00 00 00 26 26 91 91 91 91 3C 26 00 18 01 02 08 20 30 08 09 44 20 40 20 40 08 09 22 33
			39 01 00 1D B4 03 00 00 06 1E 1F 0C 0E 10 12 14 16 04 03 03 03 03 03 03 03 03 03 FF FF FC 00 00 00
			39 01 00 1D B5 03 00 00 07 1E 1F 0D 0F 11 13 15 17 05 03 03 03 03 03 03 03 03 03 FF FF FC 00 00 00
			39 01 00 19 B8 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
			39 01 00 03 BA 6B 6B
			39 01 00 0E BB 01 05 09 11 0D 19 1D 55 25 69 00 21 25
			39 01 00 0F BC 00 00 00 00 02 20 FF 00 03 33 01 73 33 02
			39 01 00 0B BD E9 02 4F CF 72 A4 08 44 AE 15
			39 01 00 0D BE 7D 7D 5A 46 0C 77 43 07 0E 0E 00 00
			39 01 00 09 BF 07 25 07 25 7F 00 11 04
			39 01 00 0D C0 10 FF FF FF FF FF 00 FF 00 00 00 00
			39 01 00 14 C1 C0 20 20 96 04 32 32 04 2A 40 36 00 07 CF FF FF C0 00 C0
			39 01 00 02 C2 00
			39 01 00 10 C2 CC 01 10 00 01 30 02 21 43 00 01 30 02 21 43
			39 01 00 0D C3 06 00 FF 00 FF 00 00 81 01 00 00 00
			39 01 00 0D C4 84 03 2B 41 00 3C 00 03 03 3E 00 00
			39 01 00 0C C5 03 1C C0 C0 40 10 42 44 0F 0A 14
			39 01 00 0B C6 87 A0 2A 29 29 00 64 37 08 04
			39 01 00 17 C7 F7 D3 BA A5 80 63 36 8B 56 2A FF CE 23 F4 D3 A4 86 5A 1A 7F E4 00
			39 01 00 17 C8 F7 D3 BA A5 80 63 36 8B 56 2A FF CE 23 F4 D3 A4 86 5A 1A 7F E4 00
			39 01 00 09 D0 80 0D FF 0F 61 0B 08 0C
			39 01 00 0E D2 42 0C 30 01 80 26 04 00 00 C3 00 00 00
			39 01 00 03 F1 5A 59
			39 01 00 03 F0 A5 A6
			39 01 00 02 35 00
			39 01 96 01 11
			39 01 32 01 29
		];
		sleep-in-command = [
			39 01 00 02 26 08
			39 01 78 01 28
			39 01 78 01 10
		];
		sleep-out-command = [
			39 01 96 01 11
			39 01 32 01 29
		];
		read-id-command = [
			37 01 00 01 01
			14 01 00 01 04
		];

		display-timings {
			timing0 {
				clock-frequency = <87870000>;
				hactive = <720>;
				hfront-porch = <48>;
				hback-porch = <48>;
				hsync-len = <4>;
				vactive = <1600>;
				vfront-porch = <150>;
				vback-porch = <32>;
				vsync-len = <4>;
				vsync-active = <1>;
				hsync-active = <1>;
			};
		};
	};
};};

```

**方案配置示例：**

以 k3_evb 方案为例：
选择 MIPI DSI panel 型号 `lcd_icnl9911c_mipi`，配置方案 MIPI DSI panel。

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_evb.dts
&dpu_crtc0 {
	memory-region = <&dpu_resv0>;
	status = "okay";
};

&dpu_crtc1 {
	memory-region = <&dpu_resv1>;
	status = "okay";
};

&dsi0 {
	status = "okay";

	panel0: panel0@0 {
		status = "ok";
		compatible = "spacemit,mipi-panel";
		reg = <0>;
		reset-gpios = <&gpio 1 31 GPIO_ACTIVE_HIGH>;
		bl-gpios = <&gpio 1 29 GPIO_ACTIVE_HIGH>;
		dc0-gpios = <&gpio 1 24 GPIO_ACTIVE_HIGH>;
		dc1-gpios = <&gpio 1 25 GPIO_ACTIVE_HIGH>;
		id = <2>;
		force-attached = "lcd_icnl9911c_mipi";
	};
};

&lcds {
	status = "okay";
};

&pwm19 {
	pinctrl-names = "default";
	pinctrl-0 = <&pwm19_1_cfg>;
	status = "okay";
};

&pwm_bl {                                       // 配置背光
	pwms = <&pwm19 2000>;
	brightness-levels = <
		0   20  20  20  21  21  21  22  22  22  23  23  23  24  24  24
		25  25  25  26  26  26  27  27  27  28  28  29  29  30  30  31
		32  33  34  35  36  37  38  39  40  41  42  43  44  45  46  47
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
	default-brightness-level = <50>;
	status = "okay";
};
```

#### DP

##### pinctrl 配置

支持 DP0、EDP0、EDP1、DP1 四种显示接口，每种接口支持多组 pinctrl 配置可选：
- **EDP0**：`edp0_0_cfg`、`edp0_1_cfg`
- **EDP1**：`edp1_0_cfg`
- **DP0**：`dp0_0_cfg`、`dp0_1_cfg`、`dp0_2_cfg`、`dp0_3_cfg`
- **DP1**：`dp1_0_cfg`、`dp1_1_cfg`、`dp1_2_cfg`、`dp1_3_cfg`

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
	/* edp only for edp */
	edp0_0_cfg: edp0-0-cfg {
		edp0-0-pins {
			pinmux = <K3_PADCONF(30, 6)>;	/* edp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	edp0_1_cfg: edp0-1-cfg {
		edp0-0-pins {
			pinmux = <K3_PADCONF(57, 4)>;	/* edp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	edp1_0_cfg: edp1-0-cfg {
		edp1-0-pins {
			pinmux = <K3_PADCONF(31, 6)>;	/* edp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	/* dp only for edp&dp */
	dp0_0_cfg: dp0-0-cfg {
		dp0-0-pins {
			pinmux = <K3_PADCONF(9, 6)>;	/* dp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp0_1_cfg: dp0-1-cfg {
		dp0-0-pins {
			pinmux = <K3_PADCONF(23, 6)>;	/* dp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp0_2_cfg: dp0-2-cfg {
		dp0-0-pins {
			pinmux = <K3_PADCONF(97, 5)>;	/* dp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
			power-source= <3300>;
		};
	};

	dp0_3_cfg: dp0-3-cfg {
		dp0-0-pins {
			pinmux = <K3_PADCONF(124, 4)>;	/* dp0 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp1_0_cfg: dp1-0-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(10, 6)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp1_1_cfg: dp1-1-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(24, 6)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp1_2_cfg: dp1-2-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(69, 4)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp1_3_cfg: dp1-3-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(72, 4)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};

	dp1_4_cfg: dp1-4-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(98, 5)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
			power-source = <3300>;
		};
	};

	dp1_5_cfg: dp1-5-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(125, 4)>;	/* dp1 hpd */

			bias-disable;			/* normal bias disable */
			drive-strength = <25>;		/* DS8 */
		};
	};
```

##### Clock 配置

DP 模块涉及的 clock 配置主要包括：
- DP DPU（数据处理单元）相关 clock 和 reset
- DP 控制器本身相关的 clock 和 reset

DP DPU 和 DP 控制器的 clock 和 reset 需要在 DTS 文件中显式配置。

以下为 DP DPU 与 DP 控制器在平台上的配置相关 clock 和 reset 示例（以 k3-dp0.dtsi 为例）：

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3-dp0.dtsi
&soc {
	display-subsystem-dp0 {
		compatible = "spacemit,saturn-hee";
		reg = <0 0xc0340000 0 0x54000>;
		hw_ver = <2>;
		ports = <&dpu0_crtc0>;
	};

	dpu0_crtc0: dpu0_crtc0 {
		compatible = "spacemit,dpu-saturn";
		interrupt-parent = <&saplic>;
		interrupts = <90 IRQ_TYPE_LEVEL_HIGH>,
			     <89 IRQ_TYPE_LEVEL_HIGH>;
		interrupt-names = "ONLINE_IRQ",
				  "OFFLINE_IRQ";
		clocks = <&syscon_apmu CLK_APMU_LCD_PXCLK>,
			<&syscon_apmu CLK_APMU_LCD_MCLK>,
			<&syscon_apmu CLK_APMU_LCD_HCLK>,
			<&syscon_apmu CLK_APMU_DSI_ESC>,
			<&syscon_apmu CLK_APMU_DPU_ACLK>,
			<&syscon_apmu CLK_APMU_LCD_DSC>;
		clock-names = "pxclk", "mclk", "hclk", "escclk", "aclk", "dscclk";
		resets = <&syscon_apmu RESET_APMU_LCD_MCLK>,
			<&syscon_apmu RESET_APMU_LCD>,
			<&syscon_apmu RESET_APMU_DSI_ESC>,
			<&syscon_apmu RESET_APMU_DPU_ACLK>,
			<&syscon_apmu RESET_APMU_LCD_DSCCLK>;
		reset-names= "mclk_reset", "lcd_reset", "esc_reset", "aclk_reset", "dsc_reset";
		power-domains = <&power K3_PMU_LCD0_PWR_DOMAIN>;
		pipeline-id = <1>;
		is_edp = <1>;
		dpu-id = <0>;
		ip = "spacemit-saturn";
		spacemit-dpu-min-mclk = <40960000>;
		status = "disabled";

		ports {
			#address-cells = <1>;
			#size-cells = <0>;

			port@0 {
				reg = <0>;
				dpu0_crtc0_out0: endpoint@0 {
					remote-endpoint = <&dp0_in>;
				};
			};
		};
	};

	dp0: dp0@cac84000 {
		compatible = "spacemit,inno-dp0";
		reg = <0x0 0xcac84000 0x0 0x4000>;
		spacemit,apmu = <&syscon_apmu>;
		interrupt-parent = <&saplic>;
		interrupts = <132 IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&syscon_apmu CLK_APMU_EDP0_PXCLK>;
		clock-names = "pxclk";
		resets = <&syscon_apmu RESET_APMU_EDP0>;
		reset-names= "reset";
		dp-id = <0>;
		#sound-dai-cells = <0>;
		status = "disabled";

		port {
			#address-cells = <1>;
			#size-cells = <0>;

			dp0_in: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&dpu0_crtc0_out0>;
			};
		};
	};
};
```

##### DTS 配置示例

以下以 `k3_deb1` 方案为例，配置 DP 和 EDP。

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_deb1.dts
&dpu0_crtc0 {
	memory-region = <&dpu_resv0>;
	status = "okay";
};

&edp0 {
	pinctrl-names = "default";
	pinctrl-0 = <&dp0_1_cfg>;
	backlight = <&backlight>;
	gpios-power = <101>;
	gpios-enable = <118>;
	status = "okay";
};

&dpu1_crtc0 {
	memory-region = <&dpu_resv1>;
	status = "okay";
};

&dp1 {
	pinctrl-names = "default";
	pinctrl-0 = <&dp1_1_cfg>;
	status = "okay";
};
```

## 接口介绍

### API 介绍

DRM 驱动 API 介绍请参考 Linux 内核文档 [drm-kms](https://docs.kernel.org/gpu/drm-kms.html)

## Debug 介绍

### debugfs

- **debugfs 节点**
以 mipi dsi 为例：
```
# cd /sys/kernel/debug/dri/0
# ls
DSI-1             crtc-0            gem_names         state
bridge_chains     dump              internal_clients
clients           framebuffer       name
```

- **查看 framebuffer 信息**

```
# cd /sys/kernel/debug/dri/0
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
# cd /sys/class/drm/card0-DSI-1
# cat dump
[  436.711209] [drm] framebuffer[135]
[  436.752130] [drm] dump framebuffer: /tmp/plane31_fb135_XR24_planes0_1200x1920.rgb
[  436.759796] [drm] framebuffer[134]
[  436.763663] [drm] dump framebuffer: /tmp/plane43_fb134_AR24_planes0_64x64.rgb
```

- **查看连接状态**

```
# cd /sys/class/drm/card0-DSI-1
# cat status
connected
```

- **查看支持的显示模式**

```
# cd /sys/class/drm/card0-DSI-1
# cat modes
1200x1920
```

## 测试介绍

libdrm 是一个用户空间库，提供了与 DRM 驱动进行交互的 API。通过 libdrm 开发者可以直接与 DRM 设备进行通信，执行各种操作，如创建和管理 framebuffer、设置显示模式、处理图层等。**modetest** 是一个使用 libdrm 库的测试工具，通常用于测试和验证 DRM 驱动的功能。

通过 modetest 工具运行 DRM 驱动测试用例如下。

```
# modetest -M spacemit
# 参数说明：
# -M spacemit: 指定 DRM 驱动模块名称为 spacemit
# 不指定其他参数时，默认输出所有 DRM 信息（Encoders、Connectors、CRTCs、Planes、Frame buffers）


# modetest  -M  spacemit -s 257@241:1200x1920
# 参数说明：
# -M spacemit: 指定 DRM 驱动模块名称为 spacemit
# -s <connector_id>@<crtc_id>:<mode>: 设置显示模式
#    connector_id: 257（DSI-1 连接器 ID）
#    crtc_id: 241（CRTC ID）
#    mode: 1200x1920（分辨率）
setting mode 1200x1920-60.05Hz on connectors 257, crtc 241
```

## FAQ
