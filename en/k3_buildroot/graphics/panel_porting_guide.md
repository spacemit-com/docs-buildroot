sidebar_position: 5

# SpacemiT Display Debugging Guide

This document describes the MIPI DSI and DP/eDP display driver use cases and debugging methods for U-Boot and the kernel on the SpacemiT K3 platform.

## Introduction

The Display module on SpacemiT platforms is built on the **DRM** framework. DRM (Direct Rendering Manager) is the graphics display subsystem in the Linux kernel. It manages display hardware, framebuffers, plane composition, display modes, and interactions among user space, the GPU, and the display controller, making it well suited to modern display hardware.

In the Linux kernel, DRM, as the display device management subsystem, is mainly responsible for the following:

- Managing display controller devices
- Allocating, mapping, and sharing graphics buffers
- Configuring resolution, refresh rate, and display timings
- Managing hardware layers and layer composition
- Controlling output interfaces such as HDMI, MIPI DSI, DP, and eDP
- Handling vertical sync and page flip
- Supporting display hot-plug and EDID
- Providing kernel interfaces for Wayland, Xorg, Android HWC, OpenHarmony Display HDI, and direct DRM applications

![display-kms](static/diaplay-kms.jpg)

Core objects in the Linux DRM/KMS display pipeline:

**framebuffer:**
drm_framebuffer represents an image buffer that can be used for display, usually created by user space.
The information it describes mainly includes:

- Image width and height
- Pixel format, for example XR24, AR24, or NV12
- Row pitch
- GEM buffer or DMA buffer
- Modifier and memory layout

**plane:**
drm_plane is a hardware layer. It reads the image from the framebuffer and feeds it into the CRTC.
Each plane can be configured with:

- Which framebuffer to use
- Source image crop region
- On-screen display position
- Scaling size
- Pixel format
- Transparency
- Layer order (zpos)
- Rotation or mirroring

**crtc:**
drm_crtc represents an independent display scanout pipeline.
Its main responsibilities are:

- Composing one or more planes
- Generating horizontal and vertical scan timings
- Setting resolution and refresh rate
- Producing the pixel clock
- Controlling vblank
- Sending the final pixel stream to the encoder

**encoder:**
drm_encoder converts the generic pixel stream output by the CRTC into the signal format required by a specific display interface.

**connector:**
drm_connector represents the connection endpoint that a display interface presents externally.
The connector is responsible for storing and detecting:

- Current connection state
- Supported display modes
- EDID
- Physical display dimensions
- Color format
- HDR, audio, and other capabilities
- Connector properties

## U-Boot Development Environment

### Source Code Structure

#### uboot-2022.10

| Category | Path |
| --- | --- |
| Board DTS | `bsp-src/uboot-2022.10/arch/riscv/dts/k3-***.dts` |
| SoC DTSI | `bsp-src/uboot-2022.10/arch/riscv/dts/k3.dtsi` |
| DPU driver | `bsp-src/uboot-2022.10/drivers/video/spacemit/spacemit_dpu.c` |
| DP/eDP driver | `bsp-src/uboot-2022.10/drivers/video/spacemit/spacemit_inno_dp.c` and `drivers/video/spacemit/dp/` |
| DSI driver | `bsp-src/uboot-2022.10/drivers/video/spacemit/spacemit_mipi.c` and `drivers/video/spacemit/dsi/drv/` |
| DSI Panel registration | `bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/video/spacemit_mipi_port.c` |
| Supported DSI Panels | `bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_***.c` |

### U-Boot DP/eDP Configuration

#### CONFIG Configuration

1. Run `make uboot-menuconfig`.

2. Go to **Device Drivers -> Graphics support** and enable the SpacemiT display driver and the DP/eDP port. The default configuration usually already enables these options.

```text
Device Drivers  --->
  Graphics support  --->
     <*> Enable SPACEMIT Video Support
     <*>    INNO DP/eDP Port
```

The corresponding main configuration options are:

```text
CONFIG_VIDEO=y
CONFIG_DISPLAY=y
CONFIG_VIDEO_SPACEMIT=y
DISPLAY_SPACEMIT_INNO_DP=y
```

For the exact option names used by the current SDK, see `bsp-src/uboot-2022.10/drivers/video/spacemit/Kconfig`.

#### DTS Configuration

For the DP solution, the DTS must enable the DPU and DP nodes and configure the DP pinctrl. The following example is based on `k3_com260.dts`:

```c
// bsp-src/uboot-2022.10/arch/riscv/dts/k3_com260.dts
&dpu {
	status = "okay";

	dpu_out: port {
		#address-cells = <1>;
		#size-cells = <0>;

		dpu_out_dp1: endpoint@1 {
			reg = <1>;
			remote-endpoint = <&dp1_in_dpu>;
		};
	};

};

&dp1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_dp1_3>;
	status = "okay";

	ports {
		dp1_in: port {
			#address-cells = <1>;
			#size-cells = <0>;
			dp1_in_dpu: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&dpu_out_dp1>;
			};
		};
	};
};
```

For the eDP solution, the DTS must enable the DPU and eDP nodes and configure the eDP pinctrl, backlight power, and enable GPIO. The following example is based on `k3-pico.dtsi`:

```c
// bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico.dtsi
&dpu {
	status = "okay";

	dpu_out: port {
		#address-cells = <1>;
		#size-cells = <0>;

		dpu_out_edp0: endpoint@0 {
			reg = <1>;
			remote-endpoint = <&edp0_in_dpu>;
		};
	};
};

&edp0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_dp0_1>;
	power-gpios = <&gpio 101 0>;
	enable-gpios = <&gpio 118 0>;
	backlight = <&backlight>;
	status = "okay";

	ports {
		edp0_in: port {
			#address-cells = <1>;
			#size-cells = <0>;
			edp0_in_dpu: endpoint@0 {
				reg = <0>;
				remote-endpoint = <&dpu_out_edp0>;
			};
		};
	};
};
```

#### Common Issues

If no LOGO is displayed on the DP/eDP interface under U-Boot, first confirm that DP/eDP is enabled in the board DTS file and that the monitor is already connected before power-on.

Log when DP/eDP is connected and link training succeeds:

```text
DP: Training successful
```

Log when DP/eDP is connected but link training fails:

```text
DP: Training failed
```

Log when DP/eDP is not connected:

```text
dp cannot get HPD signal
```

### U-Boot MIPI DSI Configuration

#### CONFIG Configuration

1. Run `make uboot-menuconfig`.

2. Go to **Device Drivers -> Graphics support** and enable the SpacemiT display driver and the MIPI port:

```text
Device Drivers  --->
  Graphics support  --->
     <*> Enable SPACEMIT Video Support
     <*>    MIPI Port
```

The corresponding main configuration options are:

```text
CONFIG_VIDEO=y
CONFIG_DISPLAY=y
CONFIG_VIDEO_SPACEMIT=y
CONFIG_DISPLAY_SPACEMIT_MIPI=y
```

#### DTS Configuration

The board DTS must enable DSI power, panel power, the DPU, MIPI DSI, the panel, and the backlight. The following example is based on `k3_evb.dts`:

```c
// bsp-src/uboot-2022.10/arch/riscv/dts/k3_evb.dts
// In the U-Boot development environment, the two DSI power rails AVDD18_DSI (1.8 V) and AVDD12_DSI (1.2 V) are enabled by default
&dpu {
	status = "okay";
};

&mipi_dsi {
	status = "okay";
};

&panel {
	force-attached = "icnl9911c";		// DSI Panel name
	dcp-gpios = <&gpio 56 0>;		// Panel power control GPIO
	dcn-gpios = <&gpio 57 0>;		// Panel power control GPIO
	reset-gpios = <&gpio 63 0>;		// Panel reset GPIO
	status = "okay";
};

&pwm19 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm19_1>;
	status = "okay";
};

&backlight {
	pwms = <&pwm19 0 2000>;
	default-brightness-level = <6>;		// Backlight brightness level
	status = "okay";
};

```

The GPIO level control, PWM channel, and period must be modified according to the actual schematic. A lit backlight only indicates that the backlight circuit works; it does not prove that the DPU, DSI Host, DPHY, or Panel was initialized successfully.

MIPI DSI node properties can be overridden from the board DTS file. If those properties are absent from the board DTS file, the current driver uses the following default values:

| Property | Default | Description |
| --- | ---: | --- |
| `pix-clk` | 88 MHz | Default DPU pixel clock |
| `bit-clk` | 614.4 MHz | Default DSI lane bit clock |

The defaults are not necessarily suitable for every panel. Calculate the target clock frequencies from the panel timings, pixel format, and lane count.

### DSI Panel Driver Configuration

DSI Panel configuration files are located in:

```text
bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/
```

Each DSI Panel file mainly contains the following kinds of data:

| Configuration | Purpose |
| --- | --- |
| `struct spacemit_mode_modeinfo` | Describes the DPU output resolution, display timings, pixel clock, pixel format, and physical dimensions |
| `struct spacemit_mipi_info` | Describes the DSI Host and DPHY working mode, lanes, link rate, and video/command mode parameters |
| `struct spacemit_dsi_cmd_desc` arrays | Describe the DSI Panel initialization, sleep, wake-up, Panel ID read, and power state commands |
| `struct lcd_mipi_panel_info` | Aggregates the Panel name, ID, command arrays, mode, and MIPI parameters, and registers them with the framework |

##### spacemit_mode_modeinfo Configuration

The structure is defined in:

```text
bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/include/spacemit_video_tx.h
```

| Member | Unit | Description | DSI Panel porting requirement |
| --- | --- | --- | --- |
| `name` | string | Display mode name, for example `1200x1920-60` | Recommended to include resolution and refresh rate |
| `refresh` | Hz | Target refresh rate | Fill in per the Panel specification, usually 60 |
| `xres` | pixel | DPU output horizontal active pixels | Corresponds to `hactive` |
| `yres` | line | DPU output vertical active lines | Corresponds to `vactive` |
| `real_xres` | pixel | Panel actual horizontal resolution | Usually equals `xres` when there is no rotation or special mapping |
| `real_yres` | line | Panel actual vertical resolution | Usually equals `yres` when there is no rotation or special mapping |
| `left_margin` | pixel | Horizontal back porch | Corresponds to `hbp` or `hback-porch` |
| `right_margin` | pixel | Horizontal front porch | Corresponds to `hfp` or `hfront-porch` |
| `upper_margin` | line | Vertical back porch | Corresponds to `vbp` or `vback-porch` |
| `lower_margin` | line | Vertical front porch | Corresponds to `vfp` or `vfront-porch` |
| `hsync_len` | pixel | Horizontal sync pulse width | Corresponds to `hsync` or `hsync-len` |
| `vsync_len` | line | Vertical sync pulse width | Corresponds to `vsync` or `vsync-len` |
| `hsync_invert` | boolean | Horizontal sync polarity inversion | Fill in per the Panel timing polarity; do not copy from a reference panel |
| `vsync_invert` | boolean | Vertical sync polarity inversion | Fill in per the Panel timing polarity |
| `invert_pixclock` | boolean | Pixel clock sampling edge inversion | Enable only when the hardware requires the opposite sampling edge |
| `pixclock_freq` | kHz | Target pixel clock | Calculate from the total timings, in kHz; default 88*1000 kHz |
| `pix_fmt_out` | enum | DPU output pixel format | Default format is RGB888 |
| `width` | mm | Panel visible area physical width | From the datasheet, not the horizontal resolution |
| `height` | mm | Panel visible area physical height | From the datasheet, not the vertical resolution |

Pixel clock formula:

```text
htotal = xres + left_margin + right_margin + hsync_len
vtotal = yres + upper_margin + lower_margin + vsync_len
pixel_clock_hz = htotal x vtotal x refresh
pixclock_freq = pixel_clock_hz / 1000
```

##### spacemit_mipi_info Configuration

The structure is defined in:

```text
bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/include/spacemit_dsi_common.h
```

| Member | Unit | Description | DSI Panel porting requirement |
| --- | --- | --- | --- |
| `height` | line | DSI input image height | Corresponds to the DSI Panel vertical active resolution |
| `width` | pixel | DSI input image width | Corresponds to the DSI Panel horizontal active resolution |
| `hfp` | pixel | Horizontal front porch | Should equal `right_margin` |
| `hbp` | pixel | Horizontal back porch | Should equal `left_margin` |
| `hsync` | pixel | Horizontal sync pulse width | Should equal `hsync_len` |
| `vfp` | line | Vertical front porch | Should equal `lower_margin` |
| `vbp` | line | Vertical back porch | Should equal `upper_margin` |
| `vsync` | line | Vertical sync pulse width | Should equal `vsync_len` |
| `fps` | Hz | DSI target frame rate | Should equal `refresh` |
| `work_mode` | enum | DSI working mode | Use `SPACEMIT_DSI_MODE_VIDEO` for video mode, and the corresponding CMD enum for command mode |
| `rgb_mode` | enum | DSI input color format | For the default RGB888 format use `DSI_INPUT_DATA_RGB_MODE_888` |
| `lane_number` | lane | Number of data lanes | Fill in per the schematic and Panel capability; commonly 1, 2, or 4 |
| `phy_bit_clock` | Hz | Bit clock per data lane | Calculated from the pixel clock, bpp, and lane count; corresponds to the `bit-clk` value in the board DTS file; default 614400000 Hz |
| `phy_esc_clock` | Hz | DPHY escape clock | Used for LP command transfer; corresponds to the `esc-clk` value in the board DTS file; default 51200000 Hz |
| `split_enable` | boolean | DSI split mode enable | Usually 0 for a single Panel and single link; set to 1 when using an 8-lane DSI Panel |
| `eotp_enable` | boolean | End Of Transmission Packet enable | Set according to whether the Panel requires eotp |
| `burst_mode` | enum | Video mode transfer type | Burst panels can use `DSI_BURST_MODE_BURST`; choose other modes per the specification |

Clock frequency estimation:

- **H Total**
```text
htotal = hactive + hfp + hbp + hsync
```

- **V Total**
```text
vtotal = vactive + vfp + vbp + vsync
```

- **Pixel Clock**
pixel clock = (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps

- **Minimum Bit Clock**
`bitclk_min` = (((hactive × bpp + hbp × bpp + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1) × 8

- **Maximum Bit Clock**
`bitclk_max` = pixel clock × 4 × 8 / lane_number

The DSI driver outputs RGB888 by default, using `bpp=24`. The `pixel clock` must be a frequency that the K3 platform clock tree can generate or accept within tolerance, and the selected `phy_bit_clock` should be a value between `bitclk_min` and `bitclk_max`. The resolution, porch, sync, and fps values in `spacemit_mode_modeinfo` and `spacemit_mipi_info` must be consistent.

### DSI Panel Debug Example

The following uses `icnl9911c` as the reference. Follow these steps to debug a DSI Panel in the U-Boot development environment.

**1. Add the DSI panel file**

Create the file:

```text
bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_icnl9911c.c
```

Main parameters of `icnl9911c`:

| Parameter | Value |
| --- | ---: |
| Resolution | 720 x 1600 |
| Refresh rate | 60 Hz |
| Horizontal timing | hbp=48, hfp=48, hsync=4 |
| Vertical timing | vbp=32, vfp=150, vsync=4 |
| Output format | RGB888 |
| DSI working mode | Video Burst Mode |
| Lane count | 4 |
| `pixclock_freq` | 88000 kHz |
| `phy_bit_clock` | 614400000 Hz |
| `phy_esc_clock` | 51200000 Hz |
| Physical size | 70 mm x 126 mm |

Calculated from the timings above:

```text
htotal = 720 + 48 + 48 + 4 = 820
vtotal = 1600 + 150 + 32 + 4 = 1786
pixel_clock = 820 x 1786 x 60 = 87871200 Hz
```

`pixel_clock` = htotal x vtotal x fps
= 820 x 1786 x 60
= 87871200 Hz

`bitclk_min` = (((hactive × bpp + hbp × bpp + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1) × 8
= (((720 × 3 + 48 × 3 + 2 × 4) × 87.9012) / ((4+48+720-0.65 × 87.9012) × 4) + 1) × 8
≈ 576.48 MHz

`bitclk_max` = pixel clock × 4 × 8 / lane_number
= 87,847,200 × 4 × 8 / 4
≈ 703 MHz

In this example the `pixel clock` is set to `88000000 Hz`, a value the K3 chip platform supports, and the bit clock is set to `614400000 Hz`, which lies between `bitclk_min` and `bitclk_max`.

Modify `spacemit_mode_modeinfo` according to the new DSI Panel datasheet:

- Update the resolution, hfp/hbp/hsync, vfp/vbp/vsync, and refresh rate.
- Calculate `.pixclock_freq` in kHz.
- Update the sync polarity and output pixel format.
- Set `.width` and `.height` to the Panel physical size in mm.

```c
struct spacemit_mode_modeinfo icnl9911c_spacemit_modelist[] = {
	{
		.name = "720x1600-60",
		.refresh = 60,
		.xres = 720,
		.yres = 1600,
		.real_xres = 720,
		.real_yres = 1600,
		.left_margin = 48,
		.right_margin = 48,
		.hsync_len = 4,
		.upper_margin = 32,
		.lower_margin = 150,
		.vsync_len = 4,
		.hsync_invert = 0,
		.vsync_invert = 0,
		.invert_pixclock = 0,
		.pixclock_freq = 88*1000,
		.pix_fmt_out = OUTFMT_RGB888,
		.width = 72,
		.height = 126,
	},
};
```

Modify `spacemit_mipi_info` accordingly:

- `width/height`, porch, sync, and fps must match the modeinfo.
- Set the video/command mode, burst mode, RGB format, and lane count according to the interface specification.
- Calculate `phy_bit_clock` and choose a clock frequency that the K3 chip platform supports.
- Set `phy_esc_clock`, eotp, and TE parameters according to the K3 chip platform and DSI Panel requirements.

```c
struct spacemit_mipi_info icnl9911c_mipi_info = {
	.height = 1600,
	.width = 720,
	.hfp = 48,
	.hbp = 48,
	.hsync = 4,
	.vfp = 150,
	.vbp = 32,
	.vsync = 4,
	.fps = 60,

	.work_mode = SPACEMIT_DSI_MODE_VIDEO,
	.rgb_mode = DSI_INPUT_DATA_RGB_MODE_888,
	.lane_number = 4,
	.phy_bit_clock = 614400000,
	.phy_esc_clock = 51200000,
	.split_enable = 0,
	.eotp_enable = 0,

	.burst_mode = DSI_BURST_MODE_BURST,
};
```

Add the Panel driver IC initialization commands and information provided by the DSI Panel vendor:

- Put the DSI Panel vendor initialization sequence into `init_cmds[]`.
- Configure `sleep_in_cmds[]` and `sleep_out_cmds[]`.
- To identify the DSI Panel automatically, configure `set_id_cmds[]`, `read_id_cmds[]`, and the Panel ID.
- To detect the power state (ESD), configure `set_power_cmds[]` and `read_power_cmds[]`.
- Verify the command type, LP/HS mode, delay (ms), payload length, and payload data.

```c
static struct spacemit_dsi_cmd_desc icnl9911c_set_id_cmds[] = {
	{SPACEMIT_DSI_SET_MAX_PKT_SIZE, SPACEMIT_DSI_LP_MODE, UNLOCK_DELAY, 1, {0x01}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_read_id_cmds[] = {
	{SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_LP_MODE, UNLOCK_DELAY, 1, {0x04}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_set_power_cmds[] = {
	{SPACEMIT_DSI_SET_MAX_PKT_SIZE, SPACEMIT_DSI_HS_MODE, UNLOCK_DELAY, 1, {0x1}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_read_power_cmds[] = {
	{SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_HS_MODE, UNLOCK_DELAY, 1, {0xA}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_init_cmds[] = {
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   3, {0xF0, 0x5A, 0x59}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   3, {0xF1, 0xA5, 0xA6}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  33, {0xB0, 0x83, 0x82, 0x86, 0x87, 0x06, 0x07, 0x04, 0x05, 0x33, 0x33, 0x33, 0x33, 0x20, 0x00, 0x00, 0x77, 0x00, 0x00, 0x3F, 0x05, 0x04, 0x03, 0x02, 0x01, 0x02, 0x03, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  30, {0xB1, 0x13, 0x91, 0x8E, 0x81, 0x20, 0x00, 0x00, 0x77, 0x00, 0x00, 0x04, 0x08, 0x54, 0x00, 0x00, 0x00, 0x44, 0x40, 0x02, 0x01, 0x40, 0x02, 0x01, 0x40, 0x02, 0x01, 0x40, 0x02, 0x01}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  18, {0xB2, 0x54, 0xC4, 0x82, 0x05, 0x40, 0x02, 0x01, 0x40, 0x02, 0x01, 0x05, 0x05, 0x54, 0x0C, 0x0C, 0x0D, 0x0B}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  32, {0xB3, 0x12, 0x00, 0x00, 0x00, 0x00, 0x26, 0x26, 0x91, 0x91, 0x91, 0x91, 0x3C, 0x26, 0x00, 0x18, 0x01, 0x02, 0x08, 0x20, 0x30, 0x08, 0x09, 0x44, 0x20, 0x40, 0x20, 0x40, 0x08, 0x09, 0x22, 0x33}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  29, {0xB4, 0x03, 0x00, 0x00, 0x06, 0x1E, 0x1F, 0x0C, 0x0E, 0x10, 0x12, 0x14, 0x16, 0x04, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0xFF, 0xFF, 0xFC, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  29, {0xB5, 0x03, 0x00, 0x00, 0x07, 0x1E, 0x1F, 0x0D, 0x0F, 0x11, 0x13, 0x15, 0x17, 0x05, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0x03, 0xFF, 0xFF, 0xFC, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  25, {0xB8, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   3, {0xBA, 0x6B, 0x6B}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  14, {0xBB, 0x01, 0x05, 0x09, 0x11, 0x0D, 0x19, 0x1D, 0x55, 0x25, 0x69, 0x00, 0x21, 0x25}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  15, {0xBC, 0x00, 0x00, 0x00, 0x00, 0x02, 0x20, 0xFF, 0x00, 0x03, 0x33, 0x01, 0x73, 0x33, 0x02}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  11, {0xBD, 0xE9, 0x02, 0x4F, 0xCF, 0x72, 0xA4, 0x08, 0x44, 0xAE, 0x15}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  13, {0xBE, 0x7D, 0x7D, 0x5A, 0x46, 0x0C, 0x77, 0x43, 0x07, 0x0E, 0x0E, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   9, {0xBF, 0x07, 0x25, 0x07, 0x25, 0x7F, 0x00, 0x11, 0x04}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  13, {0xC0, 0x10, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x00, 0xFF, 0x00, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  20, {0xC1, 0xC0, 0x20, 0x20, 0x96, 0x04, 0x32, 0x32, 0x04, 0x2A, 0x40, 0x36, 0x00, 0x07, 0xCF, 0xFF, 0xFF, 0xC0, 0x00, 0xC0}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC2, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  16, {0xC2, 0xCC, 0x01, 0x10, 0x00, 0x01, 0x30, 0x02, 0x21, 0x43, 0x00, 0x01, 0x30, 0x02, 0x21, 0x43}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  13, {0xC3, 0x06, 0x00, 0xFF, 0x00, 0xFF, 0x00, 0x00, 0x81, 0x01, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  13, {0xC4, 0x84, 0x03, 0x2B, 0x41, 0x00, 0x3C, 0x00, 0x03, 0x03, 0x3E, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  12, {0xC5, 0x03, 0x1C, 0xC0, 0xC0, 0x40, 0x10, 0x42, 0x44, 0x0F, 0x0A, 0x14}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  11, {0xC6, 0x87, 0xA0, 0x2A, 0x29, 0x29, 0x00, 0x64, 0x37, 0x08, 0x04}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  23, {0xC7, 0xF7, 0xD3, 0xBA, 0xA5, 0x80, 0x63, 0x36, 0x8B, 0x56, 0x2A, 0xFF, 0xCE, 0x23, 0xF4, 0xD3, 0xA4, 0x86, 0x5A, 0x1A, 0x7F, 0xE4, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  23, {0xC8, 0xF7, 0xD3, 0xBA, 0xA5, 0x80, 0x63, 0x36, 0x8B, 0x56, 0x2A, 0xFF, 0xCE, 0x23, 0xF4, 0xD3, 0xA4, 0x86, 0x5A, 0x1A, 0x7F, 0xE4, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   9, {0xD0, 0x80, 0x0D, 0xFF, 0x0F, 0x61, 0x0B, 0x08, 0x0C}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,  15, {0xD2, 0x42, 0x0C, 0x30, 0x01, 0x80, 0x26, 0x04, 0x00, 0x00, 0xC3, 0x00, 0x00, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   3, {0xF1, 0x5A, 0x59}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   3, {0xF0, 0xA5, 0xA6}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0x35, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 150, 2, {0x11, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE,  50, 2, {0x29, 0x00}},
	{SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE,   5, 2, {0x26, 0x00}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_sleep_out_cmds[] = {
	{SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,150,1,{0x11}},
	{SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,50,1,{0x29}},
};

static struct spacemit_dsi_cmd_desc icnl9911c_sleep_in_cmds[] = {
	{SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,50,1,{0x28}},
	{SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,150,1,{0x10}},
};
```

Initialize the DSI Panel `struct lcd_mipi_panel_info` data structure:

```c
struct lcd_mipi_panel_info lcd_icnl9911c = {
	.lcd_name = "icnl9911c",
	.lcd_id = 0x7202,
	.panel_id0 = 0x99,
	.power_value = 0x9c,
	.panel_type = LCD_MIPI,
	.width_mm = 72,
	.height_mm = 126,
	.dft_pwm_bl = 128,
	.set_id_cmds_num = ARRAY_SIZE(icnl9911c_set_id_cmds),
	.read_id_cmds_num = ARRAY_SIZE(icnl9911c_read_id_cmds),
	.init_cmds_num = ARRAY_SIZE(icnl9911c_init_cmds),
	.set_power_cmds_num = ARRAY_SIZE(icnl9911c_set_power_cmds),
	.read_power_cmds_num = ARRAY_SIZE(icnl9911c_read_power_cmds),
	.sleep_out_cmds_num = ARRAY_SIZE(icnl9911c_sleep_out_cmds),
	.sleep_in_cmds_num = ARRAY_SIZE(icnl9911c_sleep_in_cmds),
	.spacemit_modeinfo = icnl9911c_spacemit_modelist,
	.mipi_info = &icnl9911c_mipi_info,
	.set_id_cmds = icnl9911c_set_id_cmds,
	.read_id_cmds = icnl9911c_read_id_cmds,
	.set_power_cmds = icnl9911c_set_power_cmds,
	.read_power_cmds = icnl9911c_read_power_cmds,
	.init_cmds = icnl9911c_init_cmds,
	.sleep_out_cmds = icnl9911c_sleep_out_cmds,
	.sleep_in_cmds = icnl9911c_sleep_in_cmds,
	.bitclk_sel = 3,
	.bitclk_div = 1,
	.pxclk_sel = 2,
	.pxclk_div = 6,
};

int lcd_icnl9911c_init(void)
{
	int ret;

	ret = lcd_mipi_register_panel(&lcd_icnl9911c);
	return ret;
}
```

**2. Add the DSI Panel file to the Makefile**

```text
// bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/Makefile
obj-y += video/lcd/lcd_icnl9911c.o
```

**3. Declare the DSI Panel initialization function**

```c
// bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/include/spacemit_dsi_common.h
int lcd_icnl9911c_init(void);
```

**4. Register the DSI Panel**

```c
// bsp-src/uboot-2022.10/drivers/video/spacemit/dsi/video/spacemit_mipi_port.c
	else if (strcmp("icnl9911c", priv->panel_name) == 0) {
		tx_device_client.panel_type = LCD_MIPI;
		tx_device.panel_type = tx_device_client.panel_type;
		lcd_icnl9911c_init();
	}
```

**5. Configure the DSI Panel in the board DTS file**

In the board DTS file, configure the DPU, MIPI DSI, Panel, power, reset, and backlight. To use clocks other than the DSI driver defaults, set `pix-clk` and `bit-clk` to override the clock frequencies. The DSI Panel name must be exactly the same in the DSI Panel file and the board DTS file.

```c
// bsp-src/uboot-2022.10/arch/riscv/dts/k3_evb.dts

&dpu {
	status = "okay";
};

&mipi_dsi {
	pix-clk = <88000000>;			// pixel clock
	bit-clk = <614400000>;			// bit clock
	status = "okay";
};

&panel {
	force-attached = "icnl9911c";		// DSI Panel name
	dcp-gpios = <&gpio 56 0>;		// Panel power control GPIO
	dcn-gpios = <&gpio 57 0>;		// Panel power control GPIO
	reset-gpios = <&gpio 63 0>;		// Panel reset GPIO
	status = "okay";
};

&pwm19 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm19_1>;
	status = "okay";
};

&backlight {
	pwms = <&pwm19 0 2000>;
	default-brightness-level = <6>;		// Backlight brightness level
	status = "okay";
};
```

#### Common Issues

If no LOGO is displayed on the DSI Panel interface under U-Boot, first confirm that DSI Panel support is enabled in the board DTS file, that the hardware DSI Panel is connected correctly, and that the power, reset, and DSI initialization timings are configured correctly.

Boot log when DSI Panel communication starts successfully:

```text
[   0.850] Found device 'mipi@d421a800', disp_uc_priv=000000007dea6b60
[   0.854] spacemit_panel_of_to_plat panel icnl9911c
[   1.058] read panel id OK: read value = 0x99, 0x0, 0x0
[   1.060] Panel is icnl9911c
[   1.318] spacemit_display_init: lcd name icnl9911c
[   1.321] fb=7f700000, size=720x1600
```

Boot log when DSI Panel communication fails:

```text
[   0.855] Found device 'mipi@d421a800', disp_uc_priv=000000007dea6b60
[   0.859] spacemit_panel_of_to_plat panel icnl9911c
[   1.083] dsi_write_cmd: DSI send out packet maybe failed.
[   1.086] dsi_read_cmd_array: dsi didn't receive packet, irq status 0x1001
[   1.092] lcd_mipi_readid failed!
[   1.095] lcd_port (0) is not the corrected video_tx!
[   1.100] Can not found the corrected panel!
[   1.104] probe: failed to find video tx
[   1.108] spacemit_display_init: Failed to init panel
```

| Symptom | Possible cause | Check and action |
| --- | --- | --- |
| Backlight does not light up | PWM, GPIO, or backlight node not enabled | Check `pwm`, `backlight`, pinctrl, and the active level |
| Backlight is on but the screen is black | DPU/DSI not enabled, DSI Panel not initialized, clock error, lane wiring or signal problem | Check Panel registration, initialization log, and the actual clock frequency. The requested clock value is not necessarily a frequency the hardware can output; always use the value returned by `clk_get_rate()`. Check the lane hardware connection and inspect the lane signals with an oscilloscope. Before bringing up the display, try enabling the DSI Panel BIST mode to confirm that the hardware path works |
| Reading the Panel ID fails | Reset timing, power supply, LP command, or ID mismatch | Confirm power and reset with an oscilloscope, and check `read_id_cmds`, the Panel ID value, and the delays |
| The default DSI Panel is always loaded | DSI Panel name mismatch or not registered | Check the initialization function name and the DSI Panel name configured in the board DTS file |
| Artifacts or wrong colors | Wrong RGB format, lane count, timing, or initialization commands | Verify that the host output format matches the DSI Panel input format, and check the lane configuration, porch values, and the DSI Panel vendor initialization commands |
| Scrolling picture or abnormal refresh rate | Pixel clock does not match the total timings, or porch parameters are wrong | Recalculate `htotal x vtotal x fps`, check the actual pxclk, and adjust the porch parameters to verify |
| High-resolution black screen or flicker | Bit clock problem, MIPI DSI timing problem, or MIPI DSI signal integrity problem | Calculate the bitclk from bpp/lane, choose a suitable clock frequency, and inspect the lane signal quality with an oscilloscope |

## Kernel Development Environment

### Source Code Structure

#### linux-6.18

| Category | Path |
| --- | --- |
| Board DTS | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-***.dts` |
| SoC DTSI | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3.dtsi` |
| Main DRM driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/spacemit_drm.c` |
| DP DTS node | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-dp*.dtsi` |
| eDP DTS node | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-edp*.dtsi` |
| DP/eDP driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/spacemit_inno_dp.c` |
| DSI DTS node | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd_dsi_panel.dtsi` |
| Supported DSI Panels | `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_***.dtsi` |
| DPU driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/dpu/dpu_saturn.c` |
| DSI Host driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/dsi/spacemit_dsi_drv.c` |
| DSI DPHY driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/dphy/spacemit_dphy_drv.c` |
| DSI Panel driver | `bsp-src/linux-6.18/drivers/gpu/drm/spacemit/spacemit_mipi_panel.c` |

### Kernel DP/eDP Configuration

#### CONFIG Configuration

Run the following command to open the Linux kernel configuration menu:

```bash
make linux-menuconfig
```

Go to **Device Drivers -> Graphics support -> DRM Support for Spacemit** and enable the SpacemiT DRM and DP/eDP support:

```text
Device Drivers  --->
  Graphics support  --->
    DRM Support for Spacemit  --->
      <*> Spacemit specific extensions for inno dp/edp
```

The corresponding main configuration options are:

```text
CONFIG_DRM=y
CONFIG_DRM_SPACEMIT=y
SPACEMIT_INNO_DP=y
```

`CONFIG_DRM_SPACEMIT` selects the common dependencies such as DRM/KMS, CMA GEM, AUX, Panel, and backlight. The exact option names and dependencies are those in the following file for the current SDK:

```text
bsp-src/linux-6.18/drivers/gpu/drm/spacemit/Kconfig
```

#### DTS Configuration

For the DP solution, the DTS must include the common DP node and enable the DPU output path and the DP node. The following example is based on `k3_com260.dts`:

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_com260.dtsi
#include "k3-dp1.dtsi"

&pinctrl {
	dp1_3_cfg: dp1-3-cfg {
		dp1-0-pins {
			pinmux = <K3_PADCONF(72, 4)>;

			bias-disable;
			drive-strength = <25>;
			power-source = <3300>;
		};
	};
}

&dpu1_crtc0 {
	memory-region = <&dpu_resv1>;
	status = "okay";
};

&dp1 {
	pinctrl-names = "default";
	pinctrl-0 = <&dp1_3_cfg>;
	status = "okay";
};
```

If DP audio is required, enable the DP audio and sound card nodes as well:

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3_com260.dtsi

&adma3 {
	status = "okay";
};

&ri2s3 {
	status = "okay";
};

&sound_card_dp1 {
	status = "okay";
	simple-audio-card,name = "snd-dp1";
	simple-audio-card,mclk-fs = <512>;
	simple-audio-card,dai-link@0 {
		format = "left_j";
		frame-master = <&dp1_link_cpu>;
		bitclock-master = <&dp1_link_cpu>;
		dp1_link_codec: codec {
			playback-only;
			sound-dai = <&dp1>;
		};
	};
};
```

For the eDP solution, the DTS must include the common eDP node and enable the DPU output path, the eDP node, and the PWM configuration. The following example is based on `k3-pico-itx.dts`:

```c
// linux-6.18/arch/riscv/boot/dts/spacemit/k3-pico.dtsi
#include "k3-edp0.dtsi"

&pinctrl {
	dp0_1_cfg: dp0-1-cfg {
		dp0-0-pins {
			pinmux = <K3_PADCONF(23, 6)>;

			bias-disable;
			drive-strength = <25>;
			power-source = <3300>;
		};
	};

	pwm18_2_cfg: pwm18-2-cfg {
		pwm18-0-pins {
			pinmux = <K3_PADCONF(106, 6)>;

			bias-pull-up;
			drive-strength = <25>;
			power-source = <3300>;
		};
	};
}

&pwm18 {
	pinctrl-names = "default";
	pinctrl-0 = <&pwm18_2_cfg>;
	status = "okay";
};

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

backlight: backlight {
	compatible = "pwm-backlight";
	pwms = <&pwm18 0 20000 0>;
	brightness-levels = <
		0   1   2   3   4   5   6   7
		8   9  10  11  12  13  14  15
		16  17  18  19  20  21  22  23
		24  25  26  27  28  29  30  31
		32  33  34  35  36  37  38  39
		40  41  42  43  44  45  46  47
		48  49  50  51  52  53  54  55
		56  57  58  59  60  61  62  63
		64  65  66  67  68  69  70  71
		72  73  74  75  76  77  78  79
		80  81  82  83  84  85  86  87
		88  89  90  91  92  93  94  95
		96  97  98  99 100 101 102 103
		104 105 106 107 108 109 110 111
		112 113 114 115 116 117 118 119
		120 121 122 123 124 125 126 127
		128 129 130 131 132 133 134 135
		136 137 138 139 140 141 142 143
		144 145 146 147 148 149 150 151
		152 153 154 155 156 157 158 159
		160 161 162 163 164 165 166 167
		168 169 170 171 172 173 174 175
		176 177 178 179 180 181 182 183
		184 185 186 187 188 189 190 191
		192 193 194 195 196 197 198 199
		200 201 202 203 204 205 206 207
		208 209 210 211 212 213 214 215
		216 217 218 219 220 221 222 223
		224 225 226 227 228 229 230 231
		232 233 234 235 236 237 238 239
		240 241 242 243 244 245 246 247
		248 249 250 251 252 253 254 255>;
	default-brightness-level = <128>;
	status = "okay";
};
```

#### Run Check

```bash
# View DRM, DP/eDP, HPD, and EDID related logs
dmesg

# Check the DP connector status and the modes supported by the monitor
cat /sys/class/drm/card*-DP-*/status
cat /sys/class/drm/card*-DP-*/modes

# Check the eDP connector status and the modes supported by the monitor
cat /sys/class/drm/card*-eDP-*/status
cat /sys/class/drm/card*-eDP-*/modes

# With libdrm-tests installed, the full KMS resources can be inspected
modetest -c

modetest -p

modetest -e
```

#### Common Issues

| Symptom | Possible cause | Check and action |
| --- | --- | --- |
| No DP/eDP connector | Driver CONFIG not enabled, or the DRM component failed to bind | Check `CONFIG_DRM_SPACEMIT`, `SPACEMIT_INNO_DP`, and the probe/bind errors in `dmesg` |
| Connector stays disconnected | Cable, monitor, HPD pinctrl, or hardware connection problem | Check `/sys/class/drm/*DP/eDP*/status`, measure the HPD level, and verify the DP/eDP pinctrl configuration |
| HPD is fine but no mode is reported | AUX failed to read the EDID | Check the EDID logs |
| Modes are reported but the screen is black | DPU output path not enabled, clock or DP/eDP PHY configuration failed, or DP/eDP link training failed | Check the atomic commit and the DP/eDP PHY logs |
| DP shows a picture but no audio | DP audio or sound card node not enabled | Check the `sound_card` configuration in the board DTS file, the ALSA card, and the audio routing |

### Kernel MIPI DSI Configuration

#### CONFIG Configuration

Run `make linux-menuconfig`, go to **Device Drivers -> Graphics support -> DRM Support for Spacemit**, and enable the SpacemiT DRM and MIPI Panel support:

```text
Device Drivers  --->
  Graphics support  --->
    DRM Support for Spacemit  --->
      <*> MIPI Panel Support For Spacemit
```

The corresponding main configuration options are:

```text
CONFIG_DRM=y
CONFIG_DRM_SPACEMIT=y
CONFIG_SPACEMIT_MIPI_PANEL=y
```

#### DTS Configuration

On the Linux side, the DSI Panel parameters live in a separate DTSI file, while the board DTS selects the Panel, GPIOs, power, backlight, and display path. The following example is based on `k3_evb.dts`:

```c
// bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3_evb.dts
// In the kernel development environment, the DSI power rails AVDD18_DSI (1.8 V) and AVDD12_DSI (1.2 V) are enabled by default
#include "lcd_dsi_panel.dtsi"
#include "lcd/lcd_icnl9911c_mipi.dtsi"

&dpu_crtc0 {
	memory-region = <&dpu_resv0>;
	status = "okay";
};

&dsi0 {
	status = "okay";

	panel0: panel0@0 {
		status = "okay";
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

&pwm19 {
	pinctrl-names = "default";
	pinctrl-0 = <&pwm19_1_cfg>;
	status = "okay";
};

&pwm_bl {
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

The DSI power, DSI Panel power, reset, PWM, backlight brightness, and GPIO active levels must be modified according to the schematic. The `force-attached` string must match the DSI Panel name in the Panel DTSI file.

### DSI Panel Configuration Description

DSI Panel files are located in:

```text
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_<panel>_mipi.dtsi
```

#### Main Properties

| Property | Description | Porting requirement |
| --- | --- | --- |
| `dsi-work-mode` | MIPI DSI transfer mode | Choose command/video burst mode according to the Panel specification |
| `dsi-lane-number` | Number of DSI data lanes | Must match the schematic and the DSI Panel capability; commonly 1, 2, or 4 |
| `dsi-color-format` | DSI output format | Defaults to `rgb888` and must match the DSI Panel input format |
| `width-mm`, `height-mm` | DSI Panel physical size | In mm; used for the physical size information in the DRM mode |
| `use-dcs-write` | Whether to use the DSI DCS data format | Configure when the DCS data format is used; otherwise the Generic data format is used |
| `width`, `height` | Active display resolution | Correspond to the horizontal and vertical active pixels |
| `hfp/hbp/hsync` | Horizontal front porch, back porch, and sync width | Must be consistent with `display-timings` |
| `vfp/vbp/vsync` | Vertical front porch, back porch, and sync width | Must be consistent with `display-timings` |
| `fps` | Target refresh rate | Must match the total timings and the pixel clock |
| `lane-number` | Number of DPHY lanes | Should equal `dsi-lane-number` |
| `phy-bit-clock` | Bit clock of the DPHY data lanes | In Hz; must not be lower than the minimum link bandwidth requirement |
| `phy-esc-clock` | DPHY escape clock | In Hz; used for LP command transfer |
| `eotp-enable` | eotp switch | Set according to the DSI Panel requirements |
| `esd-check-enable` | ESD status detection switch | Before enabling, a correct read status command and the expected ESD status value must be configured |
| `initial-command` | DSI Panel initialization command sequence | Sent in the order DSI data type, LP/HS mode, delay, payload length, payload data |
| `sleep-in-command`, `sleep-out-command` | Sleep and wake-up command sequences | Sent in the order DSI data type, LP/HS mode, delay, payload length, payload data |
| `read-id-command` | Panel ID read command sequence | Used to identify the DSI Panel or verify communication |
| `display-timings` | DRM display mode | Configure the pixel clock, active, porch, sync, and polarity |

The clock frequency formulas are:

```text
htotal = hactive + hfront-porch + hback-porch + hsync-len
vtotal = vactive + vfront-porch + vback-porch + vsync-len

pixel_clock = htotal x vtotal x fps

bitclk_min = (((hactive × bpp + hbp × bpp + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1) × 8

bitclk_max = pixel clock × 4 × 8 / lane_number
```

RGB888 uses `bpp=24`. `bit_clk` must lie between `bitclk_min` and `bitclk_max`, and `clock-frequency` should be close to the calculated `pixel_clock`. `phy-bit-clock` must be consistent with `spacemit-dpu-bitclk` in the board DTS, and `phy-esc-clock` must be consistent with `spacemit-dpu-escclk` in the board DTS. Choose clock frequencies that the K3 chip platform clock tree can actually output. The frequency requested by the driver is not necessarily the frequency the hardware finally outputs; during debugging, read back the actual clock rate or measure the MIPI CLOCK waveform.
Because the MIPI D-PHY high-speed data uses DDR (Double Data Rate, sampling on both edges) transfer, one bit is transferred on each rising and falling clock edge, so the `bit_clock` frequency is twice the measured MIPI CLOCK frequency.

#### DSI Command Encoding

DSI Panel command sequences use the format "DSI data type, transfer mode, delay, length, payload":

```text
DATA_TYPE  DSI_MODE  DELAY  LENGTH  PAYLOAD...
```

The fields are described below:

| Field | Length | Description |
| --- | ---: | --- |
| `DATA_TYPE` | 1 byte | MIPI DSI packet data type, for example `0x39` for DCS long write |
| `DSI_MODE` | 1 byte | Command transfer mode, LP mode or HS mode |
| `DELAY` | 1 byte | Wait time after the current command is sent, in ms, parsed as hexadecimal |
| `LENGTH` | 1 byte | Number of payload bytes that follow, parsed as hexadecimal |
| `PAYLOAD` | `LENGTH` bytes | DCS/Generic command and parameters |

`DATA_TYPE` values commonly used for DCS and Generic request packets:

| Command type | Data Type | Binary | Payload content | Packet type |
| --- | ---: | --- | --- | --- |
| DCS Write | `0x05` | `00 0101` | 1 byte: DCS command without parameters | Short Packet |
| DCS Write | `0x15` | `01 0101` | 2 bytes: DCS command + 1 parameter | Short Packet |
| DCS Write | `0x39` | `11 1001` | DCS command + 2 or more parameters | Long Packet |
| DCS Read | `0x06` | `00 0110` | 1 byte: the DCS command to read | Short Packet |
| Generic Write | `0x13` | `01 0011` | 1 parameter | Short Packet |
| Generic Write | `0x23` | `10 0011` | 2 parameters | Short Packet |
| Generic Write | `0x29` | `10 1001` | 3 or more parameters | Long Packet |
| Generic Read | `0x14` | `01 0100` | 1 parameter | Short Packet |

For DCS Write, the first payload byte is always the DCS command and the following bytes are the command parameters. For Generic Write, the whole payload is treated as Generic parameters. For example:

```text
05 01 00 01 11       # DCS Short Write: Sleep Out, no parameters
15 01 00 02 51 FF    # DCS Short Write: Write Display Brightness, 1 parameter 0xFF
39 01 00 03 B0 01 02 # DCS Long Write: DCS command 0xB0 with 2 parameters

13 01 00 01 B0       # Generic Short Write: 1 parameter
23 01 00 02 B0 01    # Generic Short Write: 2 parameters
29 01 00 03 B0 01 02 # Generic Long Write: 3 parameters

06 01 00 01 0A       # DCS Read: read the DCS Power Mode register 0x0A
14 01 00 01 FB       # Generic Read: 1 parameter 0xFB
```

The Read command table describes the read request issued by the host. To limit the maximum number of returned bytes before reading, send `Set Maximum Return Packet Size` (Data Type `0x37`) first. The short/long response Data Types returned by the DSI Panel are parsed by the DSI Host receive path and must not be confused with the read request Data Types above.

For example:

```text
39 01 78 01 28
|  |  |  |  +-- payload: 0x28, DCS Display OFF
|  |  |  +----- payload length: 0x01, that is 1 byte
|  |  +-------- delay: 0x78, that is 120 ms
|  +----------- transfer mode: 0x01, LP mode
+-------------- data type: 0x39, DCS long write
```

Note that both the delay and the length in a DTS byte array are hexadecimal bytes:

```text
0x78 = 7 x 16 + 8 = 120 ms
```

When porting a DSI Panel vendor command sequence, check every entry for the following:

1. Whether `DATA_TYPE` matches the DCS/Generic and short/long write or read type.
2. Whether the command must use LP mode or HS mode.
3. Whether `DELAY` is encoded in hexadecimal and meets the Panel datasheet requirement.
4. Whether `LENGTH` equals the actual payload byte count, so that the driver does not parse into the next command.
5. Whether the payload command and parameters are complete and in the same order as the DSI Panel vendor command table.

Pay particular attention to the wait time between Sleep Out (`0x11`) and Display On (`0x29`).

### DSI Panel Debug Example

The following uses `lcd_icnl9911c_mipi` as the reference. Follow these steps to debug a DSI Panel in the kernel development environment.

**1. Add the DSI Panel file**

```text
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_icnl9911c_mipi.dtsi
```

**2. Configure the pixel clock and bit clock**

Modify the resolution, porch, sync, refresh rate, sync polarity, and physical size according to the DSI Panel datasheet, based on the output RGB format and lane count.
Calculate the pixel clock and the minimum bit clock, then choose clock frequencies the K3 chip platform can output.

- **Pixel Clock**
pixel clock = (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps = (720 + 48 + 48 + 4) * (1600 + 150 + 32 + 4) * 60 = 87871200 HZ

- **Bit Clock**
`bitclk_min` = (((hactive × bpp + hbp × bpp + 2 × lane_number) × pixel clock/1000000) / ((hsync+hbp+hactive-0.65 × pixel clock/1000000) × lane_number) + 1) × 8
= (((720 × 3 + 48 × 3 + 2 × 4) × 87.9012) / ((4+48+720-0.65 × 87.9012) × 4) + 1) × 8
≈ 576.48 MHz

`bitclk_max` = pixel clock × 4 × 8 / lane_number
= 87,847,200 × 4 × 8 / 4
≈ 703 MHz

In this example the pixel clock `88000000 Hz` and the bit clock `614400000 Hz` are chosen, both supported by the K3 chip platform.

- **Pixel Clock**: the system is configured with 88000000 HZ
- **Bit Clock**: the system is configured with 614400000 HZ

```c
// bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_icnl9911c_mipi.dtsi

lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {
	dsi-work-mode = <1>;		// MIPI DSI working mode configured in the panel: 1 = DSI_MODE_VIDEO_BURST
	dsi-lane-number = <4>;		// Number of MIPI DSI lanes configured in the panel
	dsi-color-format = "rgb888";	// DSI output format
	width-mm = <72>;		// DSI Panel physical size in the panel (unit: mm)
	height-mm = <126>;		// DSI Panel physical size in the panel (unit: mm)
	use-dcs-write;			// Whether the DCS command mode is used in the panel

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
	phy-freq = <614400>;
	phy-escape-clock = <52000>;
	split-enable = <0>;
	eotp-enable = <0>;
	burst-mode = <2>;
	esd-check-enable = <0>;

	......

	display-timings {
		timing0 {
			clock-frequency = <88000000>;
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
```

**3. Configure the DSI Panel command sequences**

Convert the DSI initialization code, sleep, wake-up, and read ID commands provided by the DSI Panel vendor into DSI command sequences.

```c
// bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_icnl9911c_mipi.dtsi

/* DATA_TYPE, DSI_MODE, DELAY, LENGTH, PAYLOAD... */
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
	05 01 96 01 11
	05 01 32 01 29
];

sleep-in-command = [
	39 01 00 02 26 08
	05 01 78 01 28
	05 01 32 01 10
];

sleep-out-command = [
	05 01 96 01 11
	05 01 32 01 29
];

read-id-command = [
	37 01 00 01 01
	14 01 00 01 04
];
```

**4. Configure the DSI Panel in the board DTS file**

In the board DTS file, set `force-attached` to the DSI Panel name, and configure the DSI power, DSI Panel power, reset, PWM, backlight brightness, and clock frequencies according to the schematic.

```c
// bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3_evb.dts
// In the kernel development environment, the DSI power rails AVDD18_DSI (1.8 V) and AVDD12_DSI (1.2 V) are enabled by default
#include "lcd_dsi_panel.dtsi"
#include "lcd/lcd_icnl9911c_mipi.dtsi"

&dpu_crtc0 {
	memory-region = <&dpu_resv0>;		// DPU reserved memory
	spacemit-dpu-bitclk = <614400000>;	// bit clock
	status = "okay";
};

&dsi0 {
	status = "okay";

	panel0: panel0@0 {
		status = "okay";
		compatible = "spacemit,mipi-panel";
		reg = <0>;

		reset-gpios = <&gpio 1 31 GPIO_ACTIVE_HIGH>;	// Panel reset GPIO
		dc0-gpios = <&gpio 1 24 GPIO_ACTIVE_HIGH>;	// Panel power control GPIO
		dc1-gpios = <&gpio 1 25 GPIO_ACTIVE_HIGH>;	// Panel power control GPIO
		id = <2>;					// Configure the panel ID
		force-attached = "lcd_icnl9911c_mipi";		// Configure the DSI Panel name
	};
};

&lcds {								// Enable the DSI Panel configuration
	status = "okay";
};

&pwm19 {							// Configure PWM
	pinctrl-names = "default";
	pinctrl-0 = <&pwm19_1_cfg>;
	status = "okay";
};

&pwm_bl {							// Configure the backlight brightness
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

The GPIO level control, PWM channel, and period must be modified according to the actual schematic. A lit backlight only indicates that the backlight circuit works; it does not prove that the DPU, DSI Host, DPHY, or Panel was initialized successfully.

#### Run Check

```bash
# View DRM, DSI Host, DPHY, and DSI Panel related logs
dmesg

# Check the DRM connector status and the modes supported by the display
cat /sys/class/drm/card*-DSI-*/status
cat /sys/class/drm/card*-DSI-*/modes

# With libdrm-tests installed, the full KMS resources can be inspected
modetest -c

modetest -p

modetest -e
```

#### Common Issues

| Symptom | Possible cause | Check and action |
| --- | --- | --- |
| Panel driver does not probe | CONFIG not enabled, node disabled, or wrong compatible / component link | Check the final DTB, `CONFIG_SPACEMIT_MIPI_PANEL`, and the deferred probe logs |
| Backlight does not light up | Wrong PWM, pinctrl, backlight node, or power GPIO configuration | Measure PWM/power, and verify the pinctrl configuration, period, and polarity |
| Backlight is on but the screen is black | Panel initialization failed, DPU/DSI/DPHY not started, clock frequency error, lane wiring or signal problem | Check reset and power sequencing, the DSI initialization commands, and the actual pixel clock and bit clock frequencies. Check the lane hardware connection and inspect the lane signals with an oscilloscope. Before bringing up the display, try enabling the DSI Panel BIST mode to confirm that the hardware path works |
| Artifacts or wrong colors | Wrong RGB format, lane count, lane mapping, or initialization registers | Check the DSI Panel data format, the physical wiring and lane configuration, and the porch values, and confirm that the DSI Panel vendor initialization sequence is correct |
| Scrolling or shaking picture | Pixel clock does not match the total timings, or wrong porch/sync parameters | Confirm that the DSI Panel vendor initialization sequence is correct, and adjust the porch parameters |
| High-resolution black screen or flicker | Insufficient bit clock bandwidth, unstable DPHY timing, or poor lane signal quality | Calculate the bit clk from bpp/lane, choose a suitable clock frequency, and inspect the lane signal quality with an oscilloscope |
| Black screen after suspend/resume | Wrong sleep commands, power/reset restore order, or backlight timing | Check `sleep-in`/`sleep-out-command`, runtime PM, and the logs of the resume stage |

## FAQ

### 1. How to Configure 8-Lane MIPI

The 8-lane panel is described here using `linux-6.18/arch/riscv/boot/dts/spacemit/lcd/lcd_nt36523_mipi.dtsi` as an example.
Split mode is used and must be enabled:

```c
	split-enable = <1>;
```

When calculating the timing, the original resolution is:

```c
	height = <2560>;
	width = <1600>;
	lane-number = <8>;
```

But when substituting into the formulas, use:

```c
	height = <1280>; // = 2560/2
	width = <1600>;
	lane-number = <4>; // = 8/2
```

### 2. Weston Primary Screen Configuration

The Weston startup script is located at:

```c
// buildroot/package/weston/run_weston.sh
```

**Default script**

```c
#export EGL_LOG_LEVEL=debug
export MESA_LOADER_DRIVER_OVERRIDE=pvr
export XDG_RUNTIME_DIR=/root
export QT_QPA_PLATFORM_PLUGIN_PATH=/usr/lib/qt/plugins/platforms
export QT_QPA_PLATFORM=wayland
weston --log=/var/log/weston --tty=1 --idle-time=0
```

When both DP and MIPI DSI are configured as `okay`, Weston uses the **first DRM display device** by default, that is, the first card detected in `/sys/class/drm`.

**View the DRM devices**

```bash
cd /sys/class/drm
ls
```

- `card0`: bound to MIPI DSI by default
- `card2`: bound to DP by default

**Specify DP as the primary screen**

To make Weston display on DP by default, modify the `run_weston.sh` script and specify the DRM device explicitly in the startup parameters:

```c
//buildroot/package/weston/run_weston.sh
#export EGL_LOG_LEVEL=debug
export MESA_LOADER_DRIVER_OVERRIDE=pvr
export XDG_RUNTIME_DIR=/root
export QT_QPA_PLATFORM_PLUGIN_PATH=/usr/lib/qt/plugins/platforms
export QT_QPA_PLATFORM=wayland
weston --log=/var/log/weston --tty=1 --drm-device=card2 --idle-time=0
```

With the `--drm-device=card2` parameter, Weston uses DP directly as the primary display.

### 3. Bianbu Primary Screen Configuration

In the **Bianbu** environment, `mutter` identifies **DSI** as the primary display by default.
To change the primary display to **DP**, the `mutter` source must be modified and recompiled.

```c
//mutter/src/backends/meta-monitor.c
meta_monitor_is_laptop_panel (MetaMonitor *monitor)
{
  const MetaOutputInfo *output_info =
    meta_monitor_get_main_output_info (monitor);

  switch (output_info->connector_type)
    {
    case META_CONNECTOR_TYPE_eDP:
    case META_CONNECTOR_TYPE_LVDS:
    case META_CONNECTOR_TYPE_DP: // Originally defaulted to META_CONNECTOR_TYPE_DSI
      return TRUE;
    default:
      return FALSE;
    }
}
```
