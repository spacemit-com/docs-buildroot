sidebar_position: 5

# MIPI-CSI

On the camera input side, K3 retains only the **MIPI-CSI capture path** and does not include the ISP and CPP hardware modules available on K1.
This section describes K3 usage, including the `MIPI D-PHY + CCIC + CCIC DMA + Sensor` driver architecture, device tree configuration, and test program.

## 1. Specification

### MIPI CSI (CSI-2 V1.1) 4 lane (x3)

- Supports multiple lane combination modes:
  - 4 Lane + 4 Lane + 4 Lane
  - 4 Lane + 4 Lane + 2 Lane + 2 Lane
- D-PHY V1.1, with a maximum data rate of up to 1.5 Gbps/lane.
- Supports RAW8, RAW10, and RAW12 formats, as well as the traditional 8-bit YUV420 input format.

## 2. DTS Configuration

### 2.1 Driver Code Structure

The MIPI-CSI code for K3 is located at `linux-6.18/drivers/media/platform/spacemit/`.  
`cam_ccic/`: responsible for CSI PHY, CCIC, DMA, video nodes, and the IOMMU.
`cam_sensor/`: responsible for sensor power-up, register initialization, and user-space control interfaces.

Code directory structure:

```text
linux-6.18/drivers/media/platform/spacemit/
├── cam_ccic/
│   ├── Kconfig
│   ├── Makefile
│   ├── csiphy.c                  // D-PHY initialization and link parameter configuration
│   ├── csiphy.h
│   ├── ccic_drv.c                // CCIC core driver: detecting, resource initialization and path management 
│   ├── ccic_drv.h
│   ├── ccic_hwreg.c              // CCIC register read/write and low-level hardware configuration 
│   ├── ccic_hwreg.h
│   ├── ccic_vdev.c               // V4L2 video node implementation 
│   ├── ccic_vdev.h
│   ├── ccic_dma.c                // DMA channel management and binding from path to DMA
│   ├── ccic_dma.h
│   ├── ccic_iommu.c              // IOMMU-related implementation
│   ├── ccic_iommu.h
│   ├── ccic_reg_iommu.h
│   ├── dptc_drv.c
│   ├── dptc_drv.h
│   └── dptc_pll_setting.h
└── cam_sensor/
    ├── Kconfig
    ├── Makefile
    ├── imx219_sensor.c           // IMX219 sensor driver
    ├── imx415_sensor.c           // IMX415 sensor driver
    ├── ov5647_sensor.c           // OV5647 sensor driver
    ├── ov2735_sensor.c           // OV2735 sensor driver
    └── ov5640_max96724_max9295_sensor.c  // OV5640 GMSL sensor driver
```

### 2.2 Configuration Flow

The camera DTS configuration for the K3 platform follows this workflow:

1. Define `csiphy`, `ccic`, and `ccic_dma` in the SoC common `.dtsi`.
2. Enable the required `&csiphyX` and `&ccicX` nodes in the board-level `.dts`.
3. Add the specific sensor node under the corresponding I2C controller.
4. Bind the sensor to a CSI controller using `csi-id`.

### 2.3 SoC Common Nodes

The K3 SoC common camera nodes are located in `k3-camera.dtsi`:

- `csiphy0` ~ `csiphy2`
- `ccic0` ~ `ccic3`
- `ccic_dma`

Description:

- These common camera nodes in `k3-camera.dtsi` generally do not require board-level modification and should remain at their default SoC definitions.
- Board-level adaptation is completed in the `.dts`: enable or disable the corresponding `&csiphyX` and `&ccicX` nodes, and add or modify sensor nodes.

#### 2.3.1 `csiphy` Node

Typical usage example:

```dts
csiphy0: csiphy@d420a000 {
	compatible = "spacemit,csi-dphy";
	cell-index = <0>;
	spacemit,apmu = <&syscon_apmu>;
	spacemit,ctrl-offset = <0x37c>;
	reg = <0x0 0xd420a000 0x0 0x13f>;
	reg-names = "csiphy-regs";
	clocks = <&syscon_apmu CLK_APMU_CCIC1PHY>;
	clock-names = "csi_dphy";
	resets = <&syscon_apmu RESET_APMU_CCIC1_PHY>;
	reset-names = "csi_dphy_reset";
	status = "okay";
};
```

#### 2.3.2 `ccic` Node

Typical usage example:

```dts
ccic0: cam_ccic@d420a000 {
	compatible = "spacemit,ccic";
	cell-index = <0>;
	spacemit,csiphy = <&csiphy0>;
	reg = <0x0 0xd420a000 0x0 0x3ff>;
	reg-names = "ccic-regs";
	interrupt-parent = <&saplic>;
	interrupts = <81 4>;
	interrupt-names = "ccic-irq";
	clocks = <&syscon_apmu CLK_APMU_CSI>,
		 <&syscon_apmu CLK_APMU_CCIC_4X>,
		 <&syscon_apmu CLK_APMU_SC2_HCLK>,
		 <&syscon_apmu CLK_APMU_ISP_BUS>;
	clock-names = "csi_func", "ccic_func", "sc2_ahb", "sc2_axi";
	resets = <&syscon_apmu RESET_APMU_CSI>,
		 <&syscon_apmu RESET_APMU_CCIC_4X>,
		 <&syscon_apmu RESET_APMU_SC2_HCLK>,
		 <&syscon_apmu RESET_APMU_ISP_CIBUS>;
	reset-names = "csi_reset", "ccic_4x_reset",
		      "sc2_hclk_reset", "isp_cibus_reset";
	status = "okay";
};
```

**Note:** `ccic2` and `ccic3` share the same D-PHY and must be configured according to one of the following lane combinations:

- `2lane + 2lane` (when both `csi2` and `csi3` are used simultaneously)
  - The `spacemit,csiphy` property of both `ccic2` and `ccic3` must point to `&csiphy2`
  - Set `spacemit,bifmode-enable` of `&csiphy2` to `<1>`
  - Enable both `&ccic2` and `&ccic3`

```dts
&csiphy2 {
	spacemit,bifmode-enable = <1>;
	status = "okay";
};

&ccic2 {
	status = "okay";
};

&ccic3 {
	status = "okay";
};
```

- `4lane` (when only one of `csi2` or `csi3` is used)
  - The `ccic` controller in use still points to `&csiphy2`
  - Set `spacemit,bifmode-enable` of `&csiphy2` to `<0>`
  - Set the unused `ccic` controller to `disabled`

```dts
&csiphy2 {
	spacemit,bifmode-enable = <0>;
	status = "okay";
};

&ccic2 {
	status = "okay";
};

&ccic3 {
	status = "disabled";
};
```

#### 2.3.3 `ccic_dma` Node

Typical usage example:

```dts
ccic_dma: cam_ccic_dma@d420f000 {
	compatible = "spacemit,ccic-dma";
	cell-index = <0>;
	reg = <0x0 0xd420f000 0x0 0x1080>;
	reg-names = "ccic-dma-regs";
	interrupt-parent = <&saplic>;
	interrupts = <93 4>, <80 4>;
	interrupt-names = "ccic-dma-irq", "ccic-dma-mmu-irq";
	status = "okay";
};
```

## 3. Test Program
### 3.1 `csi-test`

The basic verification program for K3 MIPI-CSI is located at:

```text
package/csi-test/
```

The program entry is `csi_test.c`. Sensor cases are located in the `case/` subdirectory. Currently, the `CMakeLists.txt` includes:

```text
csi_test.c
csi_path.c
case/csi_test_case0.c
case/csi_test_case1.c
case/csi_test_case2.c
case/csi_test_case3.c
case/csi_test_case4.c
self_buffer_manager.c
```

This program performs the following main tasks:

- Controls the sensor power-up, initialization, and stream on/off via the sensor misc device node.
- Configures the CCIC path and receives RAW data through the `csiX_pathY` video device node.
- Scans `/dev/videoN` at startup and matches the video node to `csiX_pathY` through `VIDIOC_QUERYCAP`.

### 3.2 Capture Path

K3 MIPI-CSI capture path:

```shell
sensor -> CSI D-PHY -> CCIC -> CCIC DMA -> DDR
```

- User space captures RAW data through the V4L2 video node, without using the ISP or CPP image-processing path.

### 3.3 Software Flow

Refer to the test programs in `package/csi-test/`. The typical software flow is as follows:

1. Enable the sensor device node, for example `/dev/imx219-0`.
2. Perform `power on -> detect -> init regs` for the sensor.
3. Enable the corresponding `csiX_pathY` video node.
4. Set the image format through V4L2.
5. Configure private path parameters such as lane count, link frequency, work mode, VC/DT filter, and DMA channel through `VIDIOC_S_PARM`.
6. Request and queue capture buffers.
7. Enable the CSI path.
8. Execute `stream on` for the sensor.
9. The driver captures RAW data and returns it to user space through `DQBUF` or a callback.
10. When streaming stops, exit in the following order: `sensor stream off -> disable path -> release buffers -> power off sensor`.

This can be simplified as:

```shell
Sensor init
    -> CSI path set_attr
    -> REQBUFS / QBUF
    -> enable_csi_path
    -> Sensor stream on
    -> RAW frame capture
    -> stream off / disable path / release
```

### 3.4 Test Program Usage Guide
#### 3.4.1 Supported Test Scenarios

According to `package/csi-test/csi_test.c`, the following scenarios are supported:

- `case 0-3`: sensor single-channel raw stream output
- `case 4`: 4VC UYUV format capture for GMSL (`OV5640 + MAX96724 + MAX9295`)

#### 3.4.2 Basic Command Format

```shell
csi_test [options]
```

Common parameters:

- `-n, --case`: Test case number
- `-c, --csi`: CSI ID
- `-l, --lane`: Number of lanes
- `-b, --bit`: Bit depth
- `-r, --link-freq`: MIPI link frequency (Mbps)
- `-a, --autostart`: Whether to start streaming automatically (`1` indicates auto-start)
- `-f, --frame`: Number of frames to capture before auto-exit
- `-s, --sub-case`: Specifies the case for each child process in multi-instance concurrent scenarios

#### 3.4.3 Usage Examples

##### Single-Channel IMX219

```shell
csi_test -n 0 -c 0 -l 2 -b 10 -r 914 -a 1 -f 100
```

This command indicates:

- run case 0
- use `csi0`
- 2 lanes
- RAW10
- link frequency 914 Mbps
- auto-start streaming
- exit after capturing 100 frames

##### OV5640 GMSL 4VC

```shell
csi_test -n 4 -c 0 -l 4 -a 1 -f 100
```

This command indicates:

- run GMSL `case 4`
- use `csi0`
- 4 lanes
- 4 VCs mapped to `path0 ~ path3`
- DMA channels are bound by default according to a custom topology

##### Dual-channel Parallel Capture

```shell
csi_test -n 5 -c 0,1 -l 2,2 -b 10,10 -s 0,1 -a 1 -f 500
```

This command starts two child processes simultaneously. For example:

- One process runs case 0, bound to `csi0`
- One process runs case 1, bound to `csi1`

## 4. How to Adapt a New Sensor

### 4.1 Board-level DTS Configuration Overview

The board-level DTS performs three main tasks:

#### 4.1.1 Enable the `csiphy` corresponding to the CSI controller

For example:

```dts
&csiphy0 {
	status = "okay";
};
```

#### 4.1.2 Enable the corresponding `ccic`

For example:

```dts
&ccic0 {
	status = "okay";
};
```

If a `ccic` is not used on the board, it should remain disabled to avoid invalid probe attempts.

#### 4.1.3 Add the sensor node under the corresponding I2C


```dts
&i2c5 {
	status = "okay";

	imx219@10 {
		compatible = "sony,imx219";
		reg = <0x10>;
		csi-id = <0>;
		vdd-supply = <&aldo3>;
		pwdn-gpios = <&gpio 1 13 GPIO_ACTIVE_HIGH>;
		status = "okay";
	};
};
```

Here,

- `compatible`
  - Determines which sensor driver to match
- `reg`
  - The I2C address of the sensor
- `csi-id`
  - Specifies which CSI path the sensor is connected to
- `vdd-supply`
  - sensor supply
- `pwdn-gpios`
  - power-down control pin
- `reset-gpios`
  - Reset pin, can be added if required by the hardware
- `pinctrl-names` / `pinctrl-0`
  - The pinctrl configuration required by the sensor itself
  - If the interface where the sensor is connected also has a mux control pin, the corresponding mux pinctrl is usually referenced here
- `i2c-mux-gpios`
  - If an I2C mux controlled by GPIO is used on the board to switch the I2C path between two sensors, this attribute needs to be configured in the sensor node
  - This attribute should be understood together with `pinctrl-0`; the pinmux of the mux control pin is also part of the sensor node configuration
- `status`
  - Enables/disables this sensor

### 4.2 Add a New Sensor Driver in `cam_sensor`

In addition to DTS configuration, adapting a new sensor also requires implementing the driver code in the kernel `cam_sensor` directory.
The current sensor driver directory for the K3 platform is:

```text
linux-6.18/drivers/media/platform/spacemit/cam_sensor/
```

It is recommended to follow these steps:

1. Add the driver source file.
   - Create a new `<sensor>_sensor.c` (refer to `imx219_sensor.c`、`ov5647_sensor.c`)
2. Add a new sensor configuration option in `cam_sensor/Kconfig`.
   - Add `config SPACEMIT_K3_CAM_<SENSOR>`
   - Dependencies typically remain `SPACEMIT_K3_CCIC && I2C`
3. Add the build option in `cam_sensor/Makefile`.
   - Add `obj-$(CONFIG_SPACEMIT_K3_CAM_<SENSOR>) += <sensor>_sensor.o`
4. Align the `compatible` string in the DTS with the driver's `of_match_table`.
   - Ensure the device tree node can correctly match the new driver
5. Align the device node and `ioctl` definitions with `csi-test`.
   - The device node name must match what the test program actually opens, e.g., `/dev/imx219-<csi_id>`
   - Add or reuse the corresponding `<sensor>.h` ioctl macro definitions in `package/csi-test/case/`
6. If a new test case is needed:
   - Add `csi_test_caseX.c` in `package/csi-test/case/`
   - Supplement case dispatch and help information in `csi_test.c`
   - Add the new case source file to `CMakeLists.txt`
