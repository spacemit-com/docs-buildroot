sidebar_position: 5

# MIPI-CSI

K3 平台在摄像头输入侧仅保留了 **MIPI-CSI 采集链路**，不包含 K1 平台中的 ISP 和 CPP 硬件模块。
因此本章聚焦于 K3 的 `MIPI D-PHY + CCIC + CCIC DMA + Sensor`驱动架构、设备树配置方式以及测试程序使用说明。

## 1 规格介绍

### MIPI CSI (CSI-2 V1.1) 4 lane (x3)

- 支持多种 Lane 组合模式：
  - 4 Lane + 4 Lane + 4 Lane
  - 4 Lane + 4 Lane + 2 Lane + 2 Lane
- DPHY V1.1，最高速率可达 1.5 Gbps/lane
- 支持 RAW8/RAW10/RAW12 格式，以及传统的 YUV420 8-bit 输入格式


## 2 DTS 配置说明

### 2.1 驱动代码结构

K3 平台 MIPI-CSI 相关代码位于 `linux-6.18/drivers/media/platform/spacemit/`，
其中 `cam_ccic/` 负责 CSI PHY、CCIC、DMA、视频节点和 IOMMU，
`cam_sensor/` 负责各类 sensor 的上电、寄存器初始化和用户态控制接口。

代码目录结构如下：

```text
linux-6.18/drivers/media/platform/spacemit/
├── cam_ccic/
│   ├── Kconfig
│   ├── Makefile
│   ├── csiphy.c                  // D-PHY 初始化与链路参数配置
│   ├── csiphy.h
│   ├── ccic_drv.c                // CCIC 主控驱动，负责探测、资源初始化和 path 管理
│   ├── ccic_drv.h
│   ├── ccic_hwreg.c              // CCIC 寄存器读写和底层硬件配置
│   ├── ccic_hwreg.h
│   ├── ccic_vdev.c               // V4L2 视频节点实现
│   ├── ccic_vdev.h
│   ├── ccic_dma.c                // DMA 通道管理和 path 到 DMA 的绑定
│   ├── ccic_dma.h
│   ├── ccic_iommu.c              // IOMMU 相关实现
│   ├── ccic_iommu.h
│   ├── ccic_reg_iommu.h
│   ├── dptc_drv.c
│   ├── dptc_drv.h
│   └── dptc_pll_setting.h
└── cam_sensor/
    ├── Kconfig
    ├── Makefile
    ├── imx219_sensor.c           // IMX219 sensor 驱动
    ├── imx415_sensor.c           // IMX415 sensor 驱动
    ├── ov5647_sensor.c           // OV5647 sensor 驱动
    ├── ov2735_sensor.c           // OV2735 sensor 驱动
    └── ov5640_max96724_max9295_sensor.c  // OV5640 GMSL sensor 驱动
```

### 2.2 配置思路

K3 的摄像头 DTS 配置可以按下面的顺序理解：

1. SoC 公共 `.dtsi` 中先定义好 `csiphy`、`ccic`、`ccic_dma`
2. 板级 `.dts` 中打开需要使用的 `&csiphyX` 和 `&ccicX`
3. 在对应的 I2C 控制器下添加具体 sensor 节点
4. 用 `csi-id` 把 sensor 绑定到某一路 CSI 控制器

### 2.3 SoC 公共节点

K3 SoC 公共摄像头节点位于 `k3-camera.dtsi`，主要包含三类节点：

- `csiphy0` ~ `csiphy2`
- `ccic0` ~ `ccic3`
- `ccic_dma`

说明：

- `k3-camera.dtsi` 中这些 camera 公共节点一般不做板级修改，通常保持 SoC 公共定义。
- 板级适配主要在 `.dts` 中完成：使能/关闭对应 `&csiphyX`、`&ccicX`，并新增/调整 sensor 节点。

#### 2.3.1 `csiphy` 节点

典型写法如下：

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

#### 2.3.2 `ccic` 节点

典型写法如下：

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

特别的，对于 `ccic2` 和 `ccic3`，两者共用同一个D-PHY，需要按两种 lane 组合来配置：

- `2lane + 2lane`（`csi2` 与 `csi3` 同时使用）
  - `ccic2` 与 `ccic3` 的 `spacemit,csiphy` 都指向 `&csiphy2`
  - `&csiphy2` 的 `spacemit,bifmode-enable` 设为 `<1>`
  - 同时使能 `&ccic2` 和 `&ccic3`

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

- `4lane`（只使用 `csi2` 或 `csi3` 其中一路）
  - 使用的那一路 `ccic` 仍指向 `&csiphy2`
  - `&csiphy2` 的 `spacemit,bifmode-enable` 设为 `<0>`
  - 未使用的那一路 `ccic` 设为 `disabled`
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

#### 2.3.3 `ccic_dma` 节点

典型写法如下：

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



## 3 测试程序
### 3.1 `csi-test`

K3 的 MIPI-CSI 基础验证程序位于：

```text
package/csi-test/
```

程序入口为 `csi_test.c`，各 sensor case 位于 `case/` 子目录，当前 `CMakeLists.txt`
包含：

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

该程序本质上完成两部分工作：

- 通过 sensor misc 设备节点控制具体传感器上电、初始化、开关流
- 通过 `csiX_pathY` 视频节点配置 CCIC path 并接收 RAW 数据
- 启动时扫描 `/dev/videoN`，通过 `VIDIOC_QUERYCAP` 将视频节点匹配为 `csiX_pathY`

### 3.2 采集链路

K3 的 MIPI-CSI 采集流程如下：

```shell
sensor -> CSI D-PHY -> CCIC -> CCIC DMA -> DDR
```

- 用户空间通过 V4L2 视频节点获取 RAW 数据，没有ISP和CPP 图像处理链路。

### 3.3 软件流程

参考 `package/csi-test/` 中测试程序，典型软件流程如下：

1. 打开 sensor 设备节点，例如 `/dev/imx219-0`
2. 对 sensor 依次执行 `power on -> detect -> init regs`
3. 打开对应的 `csiX_pathY` 视频节点
4. 通过 V4L2 设置图像格式
5. 通过 `VIDIOC_S_PARM` 下发 lane、link frequency、work mode、VC/DT filter、dma channel 等私有路径参数
6. 申请并 queue 采集 buffer
7. 使能 CSI path
8. 对 sensor 执行 `stream on`
9. 驱动采集 RAW 数据并通过 `DQBUF` / callback 返回用户空间
10. 停流时按 `sensor stream off -> disable path -> 释放 buffer -> power off sensor` 的顺序退出

可简化表示为：

```shell
Sensor init
    -> CSI path set_attr
    -> REQBUFS / QBUF
    -> enable_csi_path
    -> Sensor stream on
    -> RAW frame capture
    -> stream off / disable path / release
```

### 3.4 测试程序使用说明
#### 3.4.1 支持的测试场景

从 `package/csi-test/csi_test.c` 的帮助信息可知，当前包含如下场景：

- `case 0-3`：sensor 单路raw出流
- `case 4`：GMSL (OV5640+MAX96724+MAX9295) 的 4VC UYUV格式采集

#### 3.4.2 基本命令格式

```shell
csi_test [options]
```

常用参数如下：

- `-n, --case`：测试用例编号
- `-c, --csi`：CSI ID
- `-l, --lane`：lane 数
- `-b, --bit`：bit depth
- `-r, --link-freq`：MIPI link frequency，单位 Mbps
- `-a, --autostart`：是否自动开流，`1` 表示自动开流
- `-f, --frame`：采集多少帧后自动退出
- `-s, --sub-case`：多实例并发时为每个子进程指定 case

#### 3.4.3 使用示例

##### 单路 IMX219

```shell
csi_test -n 0 -c 0 -l 2 -b 10 -r 914 -a 1 -f 100
```

表示：

- 运行 case 0
- 使用 `csi0`
- 2 lane
- RAW10
- link frequency 914 mbps
- 自动开流
- 采集 100 帧后退出

##### OV5640 GMSL 4VC

```shell
csi_test -n 4 -c 0 -l 4 -a 1 -f 100
```

表示：

- 运行 GMSL `case 4`
- 使用 `csi0`
- 4 lane
- 4 个 VC 映射到 `path0 ~ path3`
- 默认按自定义 topo 绑定 DMA channel

##### 双路并行

```shell
csi_test -n 5 -c 0,1 -l 2,2 -b 10,10 -s 0,1 -a 1 -f 500
```

表示同时启动两个子进程，例如：

- 一个执行 case 0，绑定 `csi0`
- 一个执行 case 1，绑定 `csi1`

## 4 如何适配一个新的sensor

### 4.1 板级 DTS 需要配置哪些内容

板级 DTS 主要做三件事：

#### 4.1.1 使能csi控制器对应的 `csiphy`

例如：

```dts
&csiphy0 {
	status = "okay";
};
```

#### 4.1.2 使能对应的 `ccic`

例如：

```dts
&ccic0 {
	status = "okay";
};
```

如果该板子没有用到某一路 CCIC，应将其保持为 `disabled`，避免无效探测。

#### 4.1.3 在 I2C 下增加 sensor 节点

例如：

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

其中：

- `compatible`
  - 决定匹配哪颗 sensor 驱动
- `reg`
  - sensor 的 I2C 地址
- `csi-id`
  - 指定这颗 sensor 接到哪一路 CSI
- `vdd-supply`
  - sensor 供电
- `pwdn-gpios`
  - power-down 控制脚
- `reset-gpios`
  - 若硬件需要，可增加 reset 脚
- `pinctrl-names` / `pinctrl-0`
  - sensor 本身需要的 pinctrl 配置
  - 如果这颗 sensor 所在接口还挂了一个 mux 控制脚，也通常在这里引用对应的 mux pinctrl
- `i2c-mux-gpios`
  - 若板上通过 GPIO 控制 I2C mux，用于在两颗 sensor 之间切换 I2C 通路，需要在 sensor 节点里配置这个属性
  - 这个属性应和 `pinctrl-0` 一起理解，mux 控制脚的 pinmux 也属于 sensor 节点配置的一部分
- `status`
  - 是否使能该 sensor

### 4.2 在 `cam_sensor` 中添加 sensor 驱动

除 DTS 配置外，适配新 sensor 还需要在内核 `cam_sensor` 中补齐驱动代码。  
K3 当前 sensor 驱动目录为：

```text
linux-6.18/drivers/media/platform/spacemit/cam_sensor/
```

建议按以下步骤进行：

1. 新增驱动源文件
   - 参考 `imx219_sensor.c`、`ov5647_sensor.c` 等新建 `<sensor>_sensor.c`
2. 在 `cam_sensor/Kconfig` 增加新 sensor 配置项
   - 新增 `config SPACEMIT_K3_CAM_<SENSOR>`
   - 依赖一般保持 `SPACEMIT_K3_CCIC && I2C`
3. 在 `cam_sensor/Makefile` 增加编译项
   - 增加 `obj-$(CONFIG_SPACEMIT_K3_CAM_<SENSOR>) += <sensor>_sensor.o`
4. 对齐 DTS 的 `compatible` 与驱动 `of_match_table`
   - 确保设备树能正确匹配到新增 driver
5. 对齐 `csi-test` 的设备节点与 ioctl 定义
   - 设备节点命名需与测试程序实际打开的节点一致，例如 `/dev/imx219-<csi_id>`
   - 在 `package/csi-test/case/` 新增或复用对应 `<sensor>.h` 的 ioctl 宏定义
6. 若需要新增测试 case
   - 在 `package/csi-test/case/` 新增 `csi_test_caseX.c`
   - 在 `csi_test.c` 中补充 case 分发和帮助信息
   - 在 `CMakeLists.txt` 中加入新的 case 源文件
