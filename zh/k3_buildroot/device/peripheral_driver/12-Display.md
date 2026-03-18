# Display

介绍 SpacemiT K3 平台 Display 子系统的功能、设备树组织方式、板级使用方式以及常见调试方法。

## 模块介绍

K3 的 Display 子系统明显比 K1 更复杂，不能只按 K1 现有章节照搬。结合 K3 SDK 的驱动、`k3.dtsi` / `k3-*.dtsi` / `k3*.dts` 可以看到，K3 显示链路至少包含下面这些实际硬件和软件模块：

- **DRM/KMS 核心驱动**；
- **DPU（Display Processing Unit）/ CRTC**；
- **MIPI DSI Host**；
- **MIPI D-PHY**；
- **DP / eDP 输出控制器**；
- **Writeback（WB）路径**；
- **MIPI 屏驱动 / Raspberry Pi 7 寸屏驱动**；
- **背光控制**，既有通用 `pwm-backlight`，也有 `spacemit-backlight`；
- **多套板级显示拓扑**，包括：
  - DSI 直连 MIPI 面板；
  - DSI + DPI/桥接屏；
  - eDP 输出；
  - DP 输出；
  - DP 音频。

所以 K3 的显示文档必须围绕 **真实链路** 来写，而不是只围绕单个控制器写。

### 功能介绍

K3 平台基于 Linux **DRM（Direct Rendering Manager）** 框架实现显示子系统。显示数据路径大致可以抽象为：

1. **用户空间**  
   通过 libdrm、Wayland、Weston、modetest、kmscube 等工具发起显示配置和缓冲区提交；
2. **DRM/KMS 驱动层**  
   管理 CRTC、plane、connector、encoder、framebuffer、atomic commit；
3. **SoC 显示硬件层**  
   包括 DPU、DSI Host、DP/eDP 控制器、D-PHY、writeback 等；
4. **面板 / Bridge / 背光 / 外部显示设备**  
   最终将像素输出到 LCD、eDP 面板、DP 显示器等终端设备。

K3 的一个关键点是：**它不是单一路显示输出**。从 DTS 可以确认至少存在两类主路径：

- **DSI 路径**：`DPU -> DSI Host -> DPHY -> MIPI panel / bridge panel`
- **DP/eDP 路径**：`DPU -> DP/eDP controller -> external monitor / eDP panel`

此外，K3 还存在：

- **LCD0 / LCD1 两个 power domain**；
- **多套 DPU/CRTC 定义**（`dpu_crtc0` / `dpu_crtc1` / `dpu0_crtc0` / `dpu1_crtc0`）；
- **writeback 通道**；
- **DP 音频节点**（如 `sound_card_dp0`）。

这几项都比 K1 文档里的模型更丰富。

### DRM 架构说明

DRM/KMS 里几个最关键的抽象如下：

- **Framebuffer**：显示缓冲区；
- **Plane**：图层；
- **CRTC**：负责时序与扫描输出；
- **Encoder**：负责把像素流转换为具体输出接口格式；
- **Connector**：代表外部连接器或面板；
- **Panel / Bridge**：表示真实显示终端或桥接器件。

K3 当前驱动实现使用 atomic 模式配置，`spacemit_drm.c` 中可以看到：

- 使用 `drm_atomic_helper_commit_*` 完成提交流程；
- 支持 framebuffer / GEM DMA helper；
- mode check 中对仅调整部分垂直时序（如 vfp）做了特殊处理。

## 源码结构介绍

K3 显示驱动主要位于：

```text
linux-6.18/drivers/gpu/drm/spacemit/
|-- Kconfig
|-- Makefile
|-- spacemit_drm.c                 # DRM core
|-- spacemit_crtc.c               # CRTC 相关
|-- spacemit_planes.c             # plane 相关
|-- spacemit_gem.c                # GEM / buffer 管理
|-- spacemit_dmmu.c               # DMMU
|-- spacemit_cmdlist.c            # cmdlist
|-- spacemit_wb.c                 # writeback
|-- spacemit_dsi.c                # DSI 逻辑封装
|-- spacemit_dphy.c               # DPHY 逻辑封装
|-- spacemit_inno_dp.c            # DP / eDP 输出驱动
|-- spacemit_mipi_panel.c         # MIPI panel 驱动
|-- raspberrypi_touchscreen.c     # RPi 7-inch touchscreen panel 驱动
|-- spacemit_bootloader.c         # bootloader 相关接口
|-- backlight/
|   `-- spacemit-backlight.c      # SpacemiT backlight 驱动
|-- dpu/
|   |-- dpu_saturn.c
|   |-- dpu_saturn_hee.c
|   `-- post_process_hee.c
|-- dsi/
|   |-- spacemit_dsi_drv.c
|   `-- spacemit_dptc_drv.c
|-- dphy/
|   `-- spacemit_dphy_drv.c
`-- sysfs/
    |-- sysfs_dpu.c
    |-- sysfs_dsi.c
    |-- sysfs_dphy.c
    `-- sysfs_mipi_panel.c
```

相比 K1，K3 源码结构里更值得注意的点有：

- 新增了 `dpu_saturn_hee.c` / `post_process_hee.c`，说明 DPU/后处理能力有演进；
- 使用 `spacemit_inno_dp.c` 同时承载 DP / eDP 路径；
- 单独存在 `backlight/spacemit-backlight.c`；
- 单独支持 `raspberrypi_touchscreen.c`，表明 K3 板级已支持树莓派 7 寸触摸屏这一真实场景；
- 存在 `writeback` 路径，不是只有最简单的 scanout。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| DRM/KMS | 基于 DRM + atomic helper |
| DPU | 使用 `spacemit,dpu-saturn` |
| DSI | 支持 `spacemit,dsi-host` |
| DPHY | 支持 `spacemit,dsi-phy` |
| DP/eDP | 支持 `spacemit,inno-dp0` / `spacemit,inno-dp1` / `spacemit,inno-edp0` |
| MIPI Panel | 支持 `spacemit,mipi-panel` |
| Raspberry Pi 屏 | 支持 `raspberrypi,dpi-panel` 与 `raspberrypi,7inch-touchscreen-panel` |
| 背光 | 支持 `pwm-backlight` 与 `spacemit-backlight` |
| 双显示域 | 存在 `K3_PMU_LCD0_PWR_DOMAIN`、`K3_PMU_LCD1_PWR_DOMAIN` |
| Writeback | 存在 `wb0` 节点 |
| DP Audio | 板级 DTS 中可见 `sound_card_dp0` |

### K3 实际比 K1 多出来的内容

这部分是 K3 文档必须显式讲清楚的，不能只照抄 K1：

1. **DP / eDP 拓扑更明确**  
   K3 有独立的：
   - `k3-edp0.dtsi`
   - `k3-dp0.dtsi`
   - `k3-dp1.dtsi`

2. **双 DPU/双显示域更清晰**  
   从 DTS 可见：
   - `dpu0_crtc0` 挂在 `LCD0` power domain；
   - `dpu1_crtc0` 挂在 `LCD1` power domain。

3. **板级显示场景明显更多**  
   K3 板级不止一种 LCD 方案，而是实际存在：
   - EVB / Gemini 上的 MIPI 面板；
   - COM260 上的 Raspberry Pi 7-inch 触摸屏方案；
   - DC board / DEB1 上的 eDP；
   - EVB / DC board / DEB1 上的 DP1 输出。

4. **DPU bitclk 可板级调参**  
   如 `k3_com260.dts` 中：

   ```dts
   spacemit-dpu-bitclk = <700000000>;
   ```

5. **Writeback 路径在 K3 DTS 中被建模**  
   `lcd_dsi_panel.dtsi` 中有 `wb0`，并且 `dpu_crtc0` / `dpu_crtc1` 都连接到 WB 输入。

6. **Panel 驱动场景更真实**  
   例如 `force-attached` 使用：
   - `lcd_icnl9911c_mipi`
   - `lcd_gc9503v_mipi`
   - `lcd_tc358762xbg_dpi_800x480`

7. **DP/eDP 配置里还带背光、电源 GPIO、使能 GPIO**  
   例如：
   - `backlight = <&backlight>;`
   - `gpios-power = <101>;`
   - `gpios-enable = <118>;`

## 配置介绍

主要包括：

- DRM / Display 驱动 CONFIG；
- DPU / DSI / DP(eDP) / panel / backlight 的 DTS 配置；
- 板级显示拓扑配置。

### CONFIG 配置

K3 显示常见内核配置包括：

- `CONFIG_DRM`
- `CONFIG_DRM_SPACEMIT`
- `CONFIG_SPACEMIT_INNO_DP`
- `CONFIG_SPACEMIT_MIPI_PANEL`
- `CONFIG_BACKLIGHT_CLASS_DEVICE`
- `CONFIG_DRM_PANEL`
- `CONFIG_DRM_MIPI_DSI`
- `CONFIG_DRM_DISPLAY_HELPER`
- `CONFIG_DRM_DISPLAY_DP_HELPER`
- `CONFIG_DRM_DISPLAY_DP_AUX_BUS`

菜单路径大致如下：

```text
Device Drivers
    Graphics support
        Direct Rendering Manager
            DRM Support for Spacemit (DRM_SPACEMIT)
            Spacemit specific extensions for inno dp/edp (SPACEMIT_INNO_DP)
            mipi panel support for SPACEMIT SOCs platform (SPACEMIT_MIPI_PANEL)
```

从 `drivers/gpu/drm/spacemit/Kconfig` 可以确认：

- `DRM_SPACEMIT` 选择了 `DRM_GEM_DMA_HELPER`、`DRM_FBDEV_EMULATION`、`DRM_MIPI_DSI`、`BACKLIGHT_CLASS_DEVICE`；
- `SPACEMIT_INNO_DP` 用于 DP / eDP；
- `SPACEMIT_MIPI_PANEL` 用于 MIPI 屏；
- 还提供了 `DRM_RASPBERRYPI_TOUCHSCREEN` 以支持树莓派 7 寸屏。

## DTS 配置

K3 的显示 DTS 组织建议按“**子系统 -> DPU -> 输出控制器 -> 外设/面板/背光**”理解。

### 1. DSI 显示路径

参考 `lcd_dsi_panel.dtsi`，K3 的 DSI 路径包含：

- `display-subsystem-dsi`
- `dpu_crtc0`
- `dpu_crtc1`
- `dsi0`
- `dphy0`
- `wb0`

#### DPU CRTC 节点

例如：

```dts
dpu_crtc0: dpu_crtc0 {
        compatible = "spacemit,dpu-saturn";
        clocks = <&syscon_apmu CLK_APMU_LCD_PXCLK>,
                 <&syscon_apmu CLK_APMU_LCD_MCLK>,
                 <&syscon_apmu CLK_APMU_LCD_HCLK>,
                 <&syscon_apmu CLK_APMU_DSI_ESC>,
                 <&syscon_apmu CLK_APMU_DPU_ACLK>,
                 <&syscon_apmu CLK_APMU_LCD_DSC>;
        power-domains = <&power K3_PMU_LCD0_PWR_DOMAIN>;
        pipeline-id = <1>;
        is_edp = <0>;
        dpu-id = <0>;
        spacemit-dpu-min-mclk = <40960000>;
        spacemit-dpu-dsipll;
        status = "disabled";
};
```

几个关键点：

- `compatible = "spacemit,dpu-saturn"`：K3 DPU 使用 Saturn 系列实现；
- `power-domains`：显示电源域必须匹配；
- `spacemit-dpu-min-mclk`：最小时钟门限；
- `spacemit-dpu-dsipll`：DSI 场景下启用 DSI PLL 相关逻辑；
- 板级还可能附加 `spacemit-dpu-bitclk`。

#### DSI Host 节点

```dts
dsi0: dsi0@d421a800 {
        compatible = "spacemit,dsi-host";
        reg = <0 0xd421a800 0 0x200>;
        interrupts = <79 IRQ_TYPE_LEVEL_HIGH>;
        ip = "synopsys-dhost";
        dev-id = <2>;
        status = "disabled";
};
```

#### DPHY 节点

```dts
dphy0: dphy0@d421a800 {
        compatible = "spacemit,dsi-phy";
        reg = <0 0xd421a800 0 0x200>;
        ip = "spacemit-dphy";
        dev-id = <2>;
        status = "ok";
};
```

#### graph 连接关系

DSI 方案不是简单 parent-child，而是通过 `ports/endpoint` 连接：

```dts
DPU -> DSI Host -> DPHY -> Panel
```

例如：

- `dpu_crtc0_out0 -> dsi0_in`
- `dsi0_out -> dphy0_in`
- 板级 `panel_in -> dsi1_output`

这是 K3 显示 DTS 的一个关键读法。

### 2. MIPI 面板方案

EVB / Gemini / FPGA 这类板级通常在 `&dsi0` 下挂 MIPI panel：

```dts
&dsi0 {
        status = "disabled";

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
```

这说明 K3 MIPI panel 实际会依赖：

- `reset-gpios`
- `bl-gpios`
- `dc0-gpios`
- `dc1-gpios`
- `force-attached`

而且不同板子实际挂的 panel 型号不同，例如：

- `lcd_icnl9911c_mipi`
- `lcd_gc9503v_mipi`

不能把 K3 面板写成“只有一个通用面板”。

### 3. Raspberry Pi 7-inch 触摸屏方案

在 `k3_com260.dts` / `k3_com260_kit_v02.dts` 中，DSI 显示路径不是传统 MIPI panel，而是树莓派 7 寸屏方案：

```dts
&dsi0 {
        status = "okay";

        ports {
                port@0 {
                        dsi1_output: endpoint@1 {
                                remote-endpoint = <&panel_in>;
                        };
                };
        };

        panel0: panel0@0 {
                status = "ok";
                compatible = "raspberrypi,dpi-panel";
                force-attached = "lcd_tc358762xbg_dpi_800x480";
        };
};
```

同时在 I2C 上还能看到：

```dts
raspits-panel@45 {
        compatible = "raspberrypi,7inch-touchscreen-panel";
        reg = <0x45>;

        port {
                panel_in: endpoint {
                        remote-endpoint = <&dsi1_output>;
                };
        };
};
```

这说明 K3 的真实显示场景已经覆盖：

- DSI 输出；
- 外部桥接 / DPI 屏；
- I2C 控制的触摸屏面板；
- 触摸 IC（如 `raspits_ft5426`）配合工作。

这是 K3 相对 K1 非常值得补充的硬件事实。

### 4. eDP 路径

参考 `k3-edp0.dtsi`：

```dts
dpu0_crtc0: dpu0_crtc0 {
        compatible = "spacemit,dpu-saturn";
        power-domains = <&power K3_PMU_LCD0_PWR_DOMAIN>;
        is_edp = <1>;
        dpu-id = <0>;
        status = "disabled";

        ports {
                port@0 {
                        dpu0_crtc0_out0: endpoint@0 {
                                remote-endpoint = <&edp0_in>;
                        };
                };
        };
};

edp0: edp0@cac84000 {
        compatible = "spacemit,inno-edp0";
        reg = <0x0 0xcac84000 0x0 0x4000>;
        clocks = <&syscon_apmu CLK_APMU_EDP0_PXCLK>;
        resets = <&syscon_apmu RESET_APMU_EDP0>;
        color_format = <1>;
        ref_clock = <24000000>;
        edp-id = <0>;
        status = "disabled";
};
```

在板级 `k3_dc_board.dts` / `k3_deb1.dts` 中，实际启用方式为：

```dts
&edp0 {
        pinctrl-names = "default";
        pinctrl-0 = <&dp0_1_cfg>;
        backlight = <&backlight>;
        gpios-power = <101>;
        gpios-enable = <118>;
        status = "okay";
};
```

这说明 eDP 方案的板级 bring-up 还必须关心：

- eDP pinctrl；
- 背光节点；
- power GPIO；
- enable GPIO。

### 5. DP 路径

K3 存在至少两路 DP 输出：

- `dp0`
- `dp1`

参考 `k3-dp0.dtsi` / `k3-dp1.dtsi`：

```dts
dp0: dp0@cac84000 {
        compatible = "spacemit,inno-dp0";
        ref_clock = <24000000>;
        dp-id = <0>;
        #sound-dai-cells = <0>;
        status = "disabled";
};
```

```dts
dp1: dp1@cac88000 {
        compatible = "spacemit,inno-dp1";
        ref_clock = <24000000>;
        dp-id = <1>;
        #sound-dai-cells = <0>;
        status = "disabled";
};
```

板级常见启用方式：

```dts
&dp0 {
        pinctrl-names = "default";
        pinctrl-0 = <&dp0_2_cfg>;
        status = "okay";
};

&dp1 {
        pinctrl-names = "default";
        pinctrl-0 = <&dp1_4_cfg>;
        status = "okay";
};
```

因此 K3 的 Display 文档里必须把 **dp0 / dp1 / edp0 的差异和板级连接方式** 单独说清楚。

### 6. 背光配置

K3 板级当前既能看到：

- `pwm-backlight`
- `spacemit,backlight`

例如 `k3_deb1.dts` / `k3_dc_board.dts` 中用了 `pwm-backlight`：

```dts
backlight: backlight {
        compatible = "pwm-backlight";
        ...
};
```

同时驱动树里又存在：

```c
compatible = "spacemit,backlight"
```

因此对用户来说，背光并不是只有一种做法，具体要看板级方案：

- 若是 PWM 背光，优先看 `pwm-backlight`；
- 若是 SoC 自定义背光路径，则看 `spacemit-backlight`；
- 对 MIPI panel，还要关注面板驱动内部是否通过 DCS brightness 控制亮度。

### 7. pinctrl 配置

K3 `k3-pinctrl.dtsi` 中可以看到多组 DSI TE pin：

- `dsi_0_cfg`
- `dsi_1_cfg`
- `dsi_2_cfg`
- `dsi_3_cfg`
- `dsi_4_cfg`
- `dsi_5_cfg`
- `dsi_6_cfg`

例如：

```dts
dsi_0_cfg: dsi-0-cfg {
        dsi-0-pins {
                pinmux = <K3_PADCONF(13, 6)>;  /* dsi te */
        };
};
```

这说明 K3 的 DSI TE 信号复用位置较多，板级迁移时不能只复制一个已有例子，必须对照原理图选择正确 pin group。

## 用户视角的几种典型显示拓扑

### 方案一：MIPI 屏（EVB / Gemini / FPGA）

```text
DPU(CRTC0) -> DSI Host -> DPHY -> spacemit,mipi-panel
```

典型关心点：

- `&dpu_crtc0` 是否启用；
- `&dsi0` 是否启用；
- panel 子节点是否存在；
- `force-attached` 是否匹配正确面板；
- reset/bl/dc GPIO 是否接对；
- `memory-region` 是否配置给 DPU。

### 方案二：DSI + Raspberry Pi 7-inch 屏（COM260）

```text
DPU(CRTC0) -> DSI Host -> endpoint graph -> raspberrypi,7inch-touchscreen-panel
                         \-> raspberrypi,dpi-panel
```

典型关心点：

- `ports` / `endpoint` 是否正确互连；
- I2C 面板地址是否正确；
- 触摸 IC 是否一并启用；
- `spacemit-dpu-bitclk` 是否按板级需求配置。

### 方案三：eDP 面板（DC board / DEB1）

```text
DPU0 -> edp0 -> eDP panel + backlight + enable/power gpio
```

典型关心点：

- `&dpu0_crtc0` 是否启用；
- `&edp0` pinctrl / backlight / GPIO 是否完整；
- 背光 PWM 是否打开；
- LCD0 power domain 是否正常。

### 方案四：DP 外接显示器

```text
DPU0/DPU1 -> dp0/dp1 -> DP connector -> monitor
```

典型关心点：

- `dp0` / `dp1` 哪一路实际接出；
- pinctrl 组是否正确；
- 是否需要同时打开 DP 音频卡；
- 是否能读到 EDID 并完成模式协商。

## 使用与调试

### 1. 查看 DRM 设备

```bash
ls /dev/dri/
```

一般会看到：

- `card0` / `card1` ...
- `renderD128` 等节点。

### 2. 查看 connector / encoder / mode

```bash
modetest -M spacemit -c
modetest -M spacemit -e
modetest -M spacemit -p
```

这些命令适合确认：

- connector 是否 `connected`；
- 支持哪些 mode；
- encoder / CRTC / plane 的关联关系。

### 3. 点亮测试

例如：

```bash
modetest -M spacemit -s <connector_id>@<crtc_id>:1920x1080
```

如果是 MIPI 屏或 eDP 屏，通常先用 `modetest -c` 找到正确 connector，再做模式设置。

### 4. 查看内核日志

```bash
dmesg | grep -Ei "drm|dpu|dsi|dp|edp|panel|backlight"
```

重点关注：

- DPU probe 是否成功；
- DSI host / DPHY 是否初始化成功；
- panel prepare / enable 是否成功；
- DP/eDP 是否检测到连接 / HPD；
- 背光是否注册。

### 5. 查看 sysfs / DRM 状态

```bash
ls /sys/class/drm/
```

可用于观察：

- connector 状态；
- EDID；
- mode 列表；
- DP 链路状态等。

### 6. 常见板级排查顺序

如果屏不亮，建议按下面顺序查：

1. **先看 DPU 是否启用**  
   `memory-region`、电源域、clock/reset 是否齐全；
2. **再看输出接口是否启用**  
   是 `dsi0`、`dp0`、`dp1` 还是 `edp0`；
3. **再看 graph 拓扑是否连通**  
   `ports/endpoint/remote-endpoint` 是否一一对应；
4. **再看 panel / bridge / backlight**  
   GPIO、I2C、PWM、背光节点是否完整；
5. **最后看板级特定参数**  
   如 `spacemit-dpu-bitclk`、`force-attached`、`gpios-power`、`gpios-enable`。

## FAQ

### 1. 为什么 K3 的 Display 文档不能只参考 K1？

因为 K3 的真实显示硬件拓扑比 K1 丰富得多。仅从 DTS 就能确认：

- 有 DSI；
- 有 DP0 / DP1；
- 有 eDP0；
- 有双 DPU / 双显示域；
- 有 writeback；
- 有树莓派 7 寸屏方案；
- 有板级 bitclk 调参；
- 有 DP 音频。

如果只按 K1 写，会遗漏 K3 真正在用的硬件功能。

### 2. 为什么同样是 K3，有的板子走 DSI，有的走 eDP / DP？

因为不同板级产品的显示器件和连接方式不同。K3 SoC 本身支持多种显示输出链路，具体启用哪一条由板级 DTS 决定。

### 3. `force-attached` 是干什么的？

它用于强制指定当前 panel 驱动所绑定的面板型号/配置，在没有标准自动识别机制时尤其常见。不同板子可能配置不同的 `force-attached` 字符串，不能混用。

### 4. 为什么有的板级要加 `spacemit-dpu-bitclk`？

这是板级对 DPU bit clock 的额外调参，用来适配某些实际屏幕或桥接方案。不是所有板子都需要，但在 COM260 这类方案里已经实际使用。

### 5. eDP/DP 屏不亮优先查什么？

优先查：

- `dpu0_crtc0` / `dpu1_crtc0` 是否启用；
- `edp0` / `dp0` / `dp1` 是否启用；
- pinctrl 是否正确；
- `backlight` / `gpios-power` / `gpios-enable` 是否完整；
- `dmesg` 里是否有 DP/eDP link 训练或 HPD 相关日志。
