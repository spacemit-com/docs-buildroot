# Display

The SpacemiT platform's Display module Functionality and Usage Guide.

## Overview

The Display module on the SpacemiT platform uses the DRM (Direct Rendering Manager) framework, which is the mainstream display framework in Linux systems and is well-suited to the characteristics of modern display hardware.

### Function Description

![display-drm](static/display-drm.png)

#### User Space: Libdrm

Libdrm is the user-space library provided by the DRM framework. Users or applications can access and manage display resources by calling the functions provided by libdrm in user space.

#### Kernel Space: DRM Driver

The DRM driver provides a set of IOCTL interfaces, which can be divided into two categories: Graphics Execution Manager (GEM) and Kernel Mode-Setting (KMS).

##### GEM

GEM (Graphics Execution Manager) is mainly responsible for managing the framebuffer, including:
- Allocation and release of video memory
- Mechanism for shared memory objects
- Mechanism for memory synchronization

##### KMS

KMS (Kernel Mode-Setting) primarily handles display mode configuration and image output.
The KMS model is composed of the following components:
- Framebuffer
- CRTC (Cathode Ray Tube Controller)
- Planes
- Encoder
- Connector

###### Framebuffer

A memory region accessible by both the driver and user-space applications, used to hold the display content of a single layer.

###### CRTC

The display controller responsible for:
- Converting images to timing signals required by the display hardware
- Managing frame updates, power control, and color adjustments, among other tasks

###### Plane

A display layer. Each image is associated with a Plane. Plane attributes control display position, image flipping, color blending, etc.
The actual image shown by the CRTC is a composition of the Framebuffer and Planes, allowing either layered or individual image display.
There are three types of Planes:
- Primary Plane – Displays the main content or background.
- Overlay Plane – Used for layered content, such as video overlays.
- Cursor Plane – Dedicated to displaying the mouse cursor.

###### Encoder

The Encoder handles power management and video signal formatting. It converts display timing signals into formats required by output devices. The processed signals are then transmitted to various display interfaces, such as HDMI, MIPI DSI, etc.

###### Connector

The Connector manages the physical connection to display hardware, including tasks like device detection and screen parameter retrieval. Examples include HDMI, MIPI DSI, and others.

###### Panel

The Panel is the display screen, which converts the incoming video signals into visible images.

### Source Code Structure Introduction

The source code structure of the SpacemiT platform's DRM driver is as follows:

```
linux-6.6/drivers/gpu/drm$ tree spacemit
spacemit
|-- dphy
|   `-- spacemit_dphy_drv.c                     // MIPI DSI DPHY driver
|-- dpu                                         // DPU driver
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
|-- dsi                                         // MIPI DSI driver
|   |-- spacemit_dptc_drv.c
|   |-- spacemit_dptc_drv.h
|   |-- spacemit_dsi_drv.c
|   `-- spacemit_dsi_hw.h
|-- Kconfig
|-- lt8911exb.c                                 // lt8911exb MIPI DSI to eDP panel driver
|-- lt9711.c                                    // lt9711 MIPI DSI to DP panel driver
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
|-- spacemit_drm.c                             // DRM core driver
|-- spacemit_drm.h
|-- spacemit_dsi.c
|-- spacemit_dsi.h
|-- spacemit_gem.c                             // GEM driver
|-- spacemit_gem.h
|-- spacemit_hdmi.c                            // HDMI driver
|-- spacemit_hdmi.h
|-- spacemit_lib.c
|-- spacemit_lib.h
|-- spacemit_mipi_panel.c                      // Panel driver
|-- spacemit_mipi_panel.h
|-- spacemit_planes.c
|-- spacemit_wb.c                              // Write back driver
|-- spacemit_wb.h
`-- sysfs
    |-- sysfs_class.c                          
    |-- sysfs_display.h
    |-- sysfs_dphy.c
    |-- sysfs_dpu.c
    |-- sysfs_dsi.c
    `-- sysfs_mipi_panel.c
```

## Key Features

| Feature | Description |
| :-----| :----|
| Supports MIPI DSI | Supports MIPI DPHY V1.1, supports DPHY 4 lanes, maximum rate 1.2Gbps per lane |
| Supports HDMI | Supports HDMI 1.4a |

### Performance Specifications

| Screen Interface | Performance Specification |
| :-----| :----|
| MIPI DSI| 1920x1200@60FPS |
| HDMI | 1920x1080@60FPS |

**MIPI DSI Screen Frame Rate Testing Method:**

To view Connectors:

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

To view Encoders:

```
# modetest -M spacemit -D /dev/dri/card1 -e
Encoders:
id      crtc    type    possible crtcs  possible clones
129     127     DSI     0x00000001      0x00000001
```

Test the frame rate of the MIPI DSI screen:

```
# modetest -M spacemit -D /dev/dri/card1 -s 130@127:1200x1920 -v
setting mode 1200x1920-60.05Hz on connectors 130, crtc 127
freq: 60.55Hz
freq: 60.28Hz
freq: 60.28Hz
```

**HDMI Screen Frame Rate Testing Method:**

To view Connectors:

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

To view Encoders:

```
# modetest -M spacemit -D /dev/dri/card2 -e
Encoders:
id      crtc    type    possible crtcs  possible clones
129     127     TMDS    0x00000001      0x00000001
```

Test the frame rate of the HDMI screen:

```
# modetest -M spacemit -D /dev/dri/card2 -s 130@127:1920x1080 -v
setting mode 1920x1080-60.00Hz on connectors 130, crtc 127
freq: 60.13Hz
freq: 60.00Hz
freq: 60.00Hz
```

## Configuration

This mainly includes the display driver enablement configuration and DTS configuration. The K1 chip supports one MIPI DSI hardware interface and one HDMI hardware interface.

## CONFIG Configuration

CONFIG_DRM_SPACEMIT: This is the configuration option for the SpacemiT platform's DRM driver. By default, this option is set to `Y`. The configuration of the MIPI DSI driver or HDMI driver depends on this option, and you can configure either the MIPI DSI driver or the HDMI driver individually, or both simultaneously.

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   < >   MIPI Panel Support For Spacemit
   < >   HDMI Support For Spacemit
```

#### MIPI DSI CONFIG Configuration

CONFIG_SPACEMIT_MIPI_PANEL: This is the configuration option for the MIPI DSI driver on the SpacemiT platform. Specific configurations should be made according to the needs of the particular scheme.

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   MIPI Panel Support For Spacemit
```

#### HDMI CONFIG Configuration

CONFIG_SPACEMIT_HDMI: This is the configuration option for the HDMI driver on the SpacemiT platform. Specific configurations should be made according to the needs of the particular scheme.

```
 Device Drivers  --->
  Graphics support  ---> 
   <*> DRM Support for Spacemit
   <*>   HDMI Support For Spacemit
```

### DTS Configuration

#### MIPI DSI

##### GPIO

MIPI DSI panel GPIO configurations, including reset and power control pins.

Taking the k1-x_deb1 scheme as an example:
GPIO81 is configured as the panel reset pin, and GPIO82 and GPIO83 are configured as the panel power control pins.

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dsi2 {
        status = "okay";

        panel2: panel2@0 {
                status = "okay";
                compatible = "spacemit,mipi-panel2";
                reg = <0>;

                gpios-reset = <81>;     // Configure panel reset GPIO
                gpios-dc = <82 83>;     // Configure panel power control GPIO  
        };
};
```

#### Power Supply Configuration

MIPI DSI power configuration, including the configuration for the MIPI DSI 1.2V power supply.

Taking the k1-x_deb1 scheme as an example:
Configure PMIC LDO_5 as the MIPI DSI 1.2V power supply.

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
        status = "okay";

        dsi_1v2-supply = <&ldo_5>;      // Reference to PMIC DLDO
        vin-supply-names = "dsi_1v2";   // Configure MIPI DSI 1.2V power supply
};
```

#### Clock Configuration

MIPI DSI-related clock configurations include MIPI DSI DPU clock configurations, reset configurations, and MIPI DSI DPHY clock configurations. The system calculates the pixel clock and bit clock from timing parameters, and the specific calculation methods can be found in the **Display Timing Configuration** section. The clock-frequency in display-timings represents the pixel clock value, while spacemit-dpu-bitclk in MIPI DSI DPU and phy-bit-clock in MIPI DSI DPHY represent the bit clock value.

The MIPI DSI DPU escclk and MIPI DSI DPHY escclk should be configured to 51200000 or 76800000 (for resolutions above 1920x1080, it is recommended to use 76800000). Other clock parameters should use the system default values without configuration in the DTS file.

Here is an example of configuring the MIPI DSI DPU-related clocks and resets for the platform:

```c
// linux-6.6/arch/riscv/boot/dts/spacemit/k1-x-lcd.dtsi
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
		clocks = <&ccu CLK_DPU_PXCLK>,          // MIPI DSI DPU pxclk configuration
			 <&ccu CLK_DPU_MCLK>,           // MIPI DSI DPU mclk configuration
			 <&ccu CLK_DPU_HCLK>,           // MIPI DSI DPU hclk configuration
			 <&ccu CLK_DPU_ESC>,            // MIPI DSI DPU escclk configuration
			 <&ccu CLK_DPU_BIT>;            // MIPI DSI DPU bitclk configuration
		clock-names = "pxclk", "mclk", "hclk", "escclk", "bitclk";
		resets = <&reset RESET_MIPI>,           // MIPI DSI DPU dsi reset configuration
			 <&reset RESET_LCD_MCLK>,       // MIPI DSI DPU mclk reset configuration
			 <&reset RESET_LCD>,            // MIPI DSI DPU lcd reset configuration
			 <&reset RESET_DSI_ESC>;        // MIPI DSI DPU esc reset configuration
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
};
```

To configure the MIPI DSI DPU bitclk and escclk for the k1-x_deb1 scheme, you can follow the example provided in the DTS configuration:

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
	status = "okay";

	spacemit-dpu-bitclk = <1000000000>;     // mipi dsi dpu bitclk Configuration
	spacemit-dpu-escclk = <76800000>;       // mipi dsi dpu escclk Configuration
};
```

Configure the MIPI DSI DPHY bitclk and escclk for the panel model.
For example, using the MIPI DSI panel model `lcd_gx09inx101_mipi`

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\lcd\lcd_gx09inx101_mipi.dtsi
/ { lcds: lcds {
	lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {

		phy-bit-clock = <1000000000>;   // mipi dsi dphy bitclk Configuration
		phy-esc-clock = <76800000>;     // mipi dsi dphy escclk Configuration
	};

};};
```

#### Display Timing Configuration

Fill in the DPU timing configuration and MIPI DSI timing configuration based on the timing information provided in the MIPI DSI panel's specification. The pixel clock and bit clock are calculated from the timing parameters.

![display-timing](static/display-timing.png)

##### Display Timing Parameter Explanation

**HFP:**
Horizontal Front Porch (hfront porch): The horizontal front porch is the blanking time before the horizontal sync signal, used for the display device to prepare.

**HBP:**
Horizontal Back Porch (hback porch): The horizontal back porch is the blanking time after the horizontal sync signal, used for the display device to reset and recover.

**HSYNC:**
Horizontal Sync Pulse (hsync pulse): The horizontal sync signal is used to synchronize the row scanning of the display device. The horizontal sync pulse width indicates the duration of the horizontal sync signal.

**VFP:**
Vertical Front Porch (vfront porch): The vertical front porch is the blanking time before the vertical sync signal, used for the display device to prepare.

**VBP:**
Vertical Back Porch (vback porch): The vertical back porch is the blanking time after the vertical sync signal, used for the display device to reset and recover.

**VSYNC:**
Vertical Sync Pulse (vsync pulse): The vertical sync signal is used to synchronize the refresh rate of the display device. The vertical sync pulse width indicates the duration of the vertical sync signal.

**HACTIVE:**
Horizontal Active Display Period (hactive): The number of pixels in the horizontal line that are actively displayed.

**VACTIVE:**
Vertical Active Display Period (vactive): The number of rows in the vertical frame that are actively displayed.

#### Display Timing Calculation Method

**FPS:**
Frame rate, the number of frames displayed per second.

**Bpp:**
Bits per pixel, the number of bits used for each pixel.

**Htotal:**
Total horizontal pixels.

Htotal=hactive + HFP + HSYNC pulse + HBP

**Vtotal:**
Total vertical pixels.

vtotal = vactive + VFP + VSYNC pulse + VBP

**Pixel clock:**
The frequency at which pixel data is transmitted or processed per second.

Pixel clock calculation method:

pixel clock = htotal \* vtotal \* fps  = (hactive + hfp + hbp + hsync) \* (vactive + vfp + vbp + vsync) \* fps

**Bit clock:**
The data transfer clock for each lane in the MIPI DSI data transmission process.

Bit clock calculation method:

bit clock =  ((htotal \* vtotal \* fps  \* bpp) / lane bumber) \* 1.1 =   (((hactive + hfp + hbp + hsync) \* (vactive + vfp + vbp + vsync) \* fps  \* bpp) / lane bumber) \* 1.1

**DSI clock:**
The actual clock signal of the MIPI DSI clock lane, which uses dual-edge sampling, allowing one clock to transmit two bits of data.

dsi clock = bit clock / 2

**Note:**
When calculating the MIPI DSI Bit clock on the SpacemiT platform, a factor of 1.1 is required.

Taking the MIPI DSI panel model `lcd_gx09inx101_mipi` as an example:
Configure the MIPI DSI DPU timing and MIPI DSI DPHY timing.

Pixel clock = (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps = (1200 + 50 + 40 + 10) * (1920 + 20 + 16 + 4) * 60 = 152,880,000 Hz

Bit clock = (((hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps * bpp) / lane number) * 1.1 = (((1200 + 50 + 40 + 10) * (1920 + 20 + 16 + 4) * 60 * 24) / 4) * 1.1 = 1,009,008,000 Hz

With display timing calculations, the pixel clock value is 152,880,000 Hz, which can be configured in the system as 153,000,000 Hz, and the bit clock value is 1,009,008,000 Hz, which can be configured in the system as 1,000,000,000 Hz.

In the DTS file, the clock-frequency is configured as 153,000,000, and `spacemit-dpu-bitclk` and `phy-bit-clock` are configured as 1,000,000,000.

```c
// linux-6.6/arch/riscv/boot/dts/spacemit/lcd/lcd_gx09inx101_mipi.dtsi
/ {
    lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {

            // Timing configuration in MIPI DSI DPHY
            height = <1920>;              // Screen height configured in MIPI DSI DPHY
            width = <1200>;               // Screen width configured in MIPI DSI DPHY
            hfp = <50>;                   // Horizontal Front Porch configured in MIPI DSI DPHY
            hbp = <40>;                   // Horizontal Back Porch configured in MIPI DSI DPHY
            hsync = <10>;                 // Horizontal Sync configured in MIPI DSI DPHY
            vfp = <20>;                   // Vertical Front Porch configured in MIPI DSI DPHY
            vbp = <16>;                   // Vertical Back Porch configured in MIPI DSI DPHY
            vsync = <4>;                  // Vertical Sync configured in MIPI DSI DPHY
            fps = <60>;                   // Frame rate configured in MIPI DSI DPHY

            // Timing configuration in MIPI DSI DPU
            display-timings {
                timing0 {
                    clock-frequency = <153000000>;    // Pixel clock
                    hactive = <1200>;                 // Horizontal active display pixels
                    hfront-porch = <50>;              // hfp
                    hback-porch = <40>;               // hbp
                    hsync-len = <10>;                 // hsync
                    vactive = <1920>;                 // Vertical active display pixels
                    vfront-porch = <20>;              // vfp
                    vback-porch = <16>;               // vbp
                    vsync-len = <4>;                  // vsync
                    vsync-active = <1>;               // vsync signal high-level trigger
                    hsync-active = <1>;               // hsync signal high-level trigger
                };
            };
        };
    };
};
```

The dtsi file for the MIPI DSI panel that has completed functional debugging is placed in the lcd directory.

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

##### Panel Configuration

To enable the MIPI DSI Panel, you need to enable the MIPI DSI DPU, MIPI DSI host, lcds, configure the panel, and set up the PWM backlight.

Taking the k1-x_deb1 solution as an example: enable dpu_online2_dsi, dsi2, lcds, set the panel model to `lcd_gx09inx101_mipi` and configure the PWM backlight.

```c
// linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
&dpu_online2_dsi {
	status = "okay";                                       // Enable MIPI DSI DPU
};

&dsi2 {
	status = "okay";                                       // Enable MIPI DSI host
	panel2: panel2@0 {
		status = "okay";                              // Enable panel
		compatible = "spacemit,mipi-panel2";
		reg = <0>;

		gpios-reset = <81>;                             // Configure panel reset GPIO
		gpios-dc = <82 83>;                             // Configure panel power control GPIO
		id = <2>;                                       // Configure panel ID
		delay-after-reset = <10>;                       // Configure panel reset delay time (unit: ms)
		force-attached = "lcd_gx09inx101_mipi";         // Configure panel model
	};

};

&lcds {
	status = "okay";                                          // Enable lcds
};

&pwm14 {
	status = "okay";                                        // Enable PWM
};

&pwm_bl {                                                       // Enable backlight
	status = "okay";
};

```

##### DTS Configuration Example

**MIPI DSI Panel Configuration Example:**

Taking the MIPI DSI panel model `lcd_gx09inx101_mipi` as an example: Configure the MIPI DSI panel.

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\lcd\lcd_gx09inx101_mipi.dtsi
/ { lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {
                dsi-work-mode = <1>;          // Configure the MIPI DSI work mode in the panel: 1 DSI_MODE_VIDEO_BURST
                dsi-lane-number = <4>;        // Configure the number of MIPI DSI lanes in the panel
                dsi-color-format = "rgb888";  // Configure the MIPI DSI data format in the panel
                width-mm = <142>;             // Configure the screen active area width in the panel
                height-mm = <228>;            // Configure the screen active area height in the panel
                use-dcs-write;                // Configure whether to use DCS command mode in the panel

                // Timing configuration in MIPI DSI DPHY
                height = <1920>;              // Configure the screen height in MIPI DSI DPHY
                width = <1200>;               // Configure the screen width in MIPI DSI DPHY
                hfp = <50>;                   // Configure the horizontal front porch in MIPI DSI DPHY (Horizontal Front Porch)
                hbp = <40>;                   // Configure the horizontal back porch in MIPI DSI DPHY (Horizontal Back Porch)
                hsync = <10>;                 // Configure the horizontal sync signal in MIPI DSI DPHY (Horizontal Sync)
                vfp = <20>;                   // Configure the vertical front porch in MIPI DSI DPHY (Vertical Front Porch)
                vbp = <16>;                   // Configure the vertical back porch in MIPI DSI DPHY (Vertical Back Porch)
                vsync = <4>;                  // Configure the vertical sync signal in MIPI DSI DPHY (Vertical Sync)
                fps = <60>;                   // Configure the frame rate in MIPI DSI DPHY
                work-mode = <0>;              // Configure the MIPI DSI work mode in MIPI DSI DPHY: 0 SPACEMIT_DSI_MODE_VIDEO
                rgb-mode = <3>;               // Configure the MIPI DSI data format in MIPI DSI DPHY: 3 DSI_INPUT_DATA_RGB_MODE_888
                lane-number = <4>;            // Configure the number of MIPI DSI lanes in MIPI DSI DPHY
                phy-bit-clock = <1000000000>; // Configure the MIPI DSI DPHY bit clock in MIPI DSI DPHY
                phy-esc-clock = <76800000>;   // Configure the MIPI DSI DPHY esc clock in MIPI DSI DPHY
                split-enable = <0>;           // Configure the enablement of split in MIPI DSI DPHY
                eotp-enable = <0>;            // Configure the enablement of EOTP in MIPI DSI DPHY
                burst-mode = <2>;             // Configure the MIPI DSI burst mode in MIPI DSI DPHY: 2 DSI_BURST_MODE_BURST
                esd-check-enable = <0>;       // Configure the enablement of ESD check in the panel

                // MIPI DSI initialization command sequence
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
                
                // MIPI DSI sleep-in command sequence
                sleep-in-command = [
                        39 01 78 01 28
                        39 01 78 01 10
                ];
                
                // MIPI DSI sleep-out command sequence
                sleep-out-command = [
                        39 01 96 01 11
                        39 01 32 01 29
                ];
                
                // MIPI DSI read ID command sequence
                read-id-command = [
                        37 01 00 01 05
                        14 01 00 05 fb fc fd fe ff
                ];

                // Timing configuration in MIPI DSI DPU
                display-timings {
                        timing0 {
                                clock-frequency = <153000000>;    // Pixel clock
                                hactive = <1200>;                 // Horizontal active pixels
                                hfront-porch = <50>;              // Horizontal front porch
                                hback-porch = <40>;               // Horizontal back porch
                                hsync-len = <10>;                 // Horizontal sync length
                                vactive = <1920>;                 // Vertical active pixels
                                vfront-porch = <20>;              // Vertical front porch
                                vback-porch = <16>;               // Vertical back porch
                                vsync-len = <4>;                  // Vertical sync length
                                vsync-active = <1>;               // Vsync signal is active high
                                hsync-active = <1>;               // Hsync signal is active high
                        };
                };
        };
};};
```

**Configuration Example:**

Using the k1-x_deb1 platform as an example: Select the MIPI DSI panel model lcd_gx09inx101_mipi, and configure the solution to use the MIPI DSI panel.

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_dsi {
        status = "okay";                                // Enable MIPI DSI DPU
        memory-region = <&dpu_resv>;                    // Configure the reserved memory for MIPI DSI DPU
        spacemit-dpu-bitclk = <1000000000>;             // Configure the MIPI DSI DPU bitclk
        spacemit-dpu-escclk = <76800000>;               // Configure the MIPI DSI DPU escclk
        dsi_1v2-supply = <&ldo_5>;                      // Reference the PMIC DLDO
        vin-supply-names = "dsi_1v2";                   // Configure the MIPI DSI 1.2v power supply
};


&dsi2 {
	status = "okay";                                // Enable the MIPI DSI host

	panel2: panel2@0 {
		status = "okay";                        // Enable the pane
		compatible = "spacemit,mipi-panel2";
		reg = <0>;

		gpios-reset = <81>;                     // GPIO for panel reset
		gpios-dc = <82 83>;                     // GPIO for panel power control
		id = <2>;                               // Configure panel ID
		delay-after-reset = <10>;               // Configure panel reset delay time
		force-attached = "lcd_gx09inx101_mipi"; // Configure panel mode
	};

};

&lcds {
	status = "okay";                                 // Enable LCDs
};

&pwm14 {                                                // Configure PWM
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm14_1>;
	status = "okay";

};

&pwm_bl {                                               // Configure backlight
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

#### pinctrl

Supports two sets of HDMI pinctrl: `pinctrl_hdmi_0` and `pinctrl_hdmi_1`, only one of them can be selected.

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

#### Clock Configuration

The clock configuration for HDMI includes the clock settings related to HDMI DPU, reset configurations, and other HDMI-related clock settings. The relevant clock parameters use the system default values without  configuration in the DTS file.

Configure the platform's HDMI DPU-related clocks and resets, as well as the HDMI-related clocks and resets.

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
                clocks = <&ccu CLK_HDMI>;                               // hdmi dpu hmclk configuration
                clock-names = "hmclk";
                resets = <&reset RESET_HDMI>;                           // hdmi dpu reset configuration
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
                clocks = <&ccu CLK_HDMI>;                               // hdmi hmclk configuration
                clock-names = "hmclk";
                resets = <&reset RESET_HDMI>;                           // hdmi reset configuration
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

#### DTS Configuration Example

Taking the k1-x_deb1 solution as an example, HDMI is configured in the solution.

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;                    // Configuring HDMI DPU Reserved Memory
	status = "okay";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "okay";
};
```

## Interface Introduction

### API Introduction

For an introduction to the DRM driver API, please refer to the Linux kernel documentation [drm-kms](https://docs.kernel.org/gpu/drm-kms.html)

## Debug Introduction

### debugfs

**MIPI DSI debugfs node**

```
# cd /sys/kernel/debug/dri/1
# ls
DSI-1             crtc-0            gem_names         state
bridge_chains     dump              internal_clients
clients           framebuffer       name
```

**HDMI debugfs node**

```
# cd /sys/kernel/debug/dri/2
# ls
HDMI-A-1          crtc-0            gem_names         state
bridge_chains     dump              internal_clients
clients           framebuffer       name
```

**View framebuffer information**

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

**Dump the currently displayed buffer**

```
# cd /sys/class/drm/card1-DSI-1
# cat dump
[  436.711209] [drm] framebuffer[135]
[  436.752130] [drm] dump framebuffer: /tmp/plane31_fb135_XR24_planes0_1200x1920.rgb
[  436.759796] [drm] framebuffer[134]
[  436.763663] [drm] dump framebuffer: /tmp/plane43_fb134_AR24_planes0_64x64.rgb
```

**View the connection status**

```
# cd /sys/class/drm/card1-DSI-1
# cat status
connected
```

**View the supported display modes**

```
# cd /sys/class/drm/card1-DSI-1
# cat modes
1200x1920
```

## Test Introduction

libdrm is a userspace library that provides an API for interacting with DRM drivers. Through libdrm, developers can directly communicate with DRM devices and perform various operations, such as creating and managing framebuffers, setting display modes, and handling layers. modetest is a test tool that uses the libdrm library and is commonly used to test and verify the functionality of DRM drivers.

Run DRM driver test cases using the **modetest** tool.

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
