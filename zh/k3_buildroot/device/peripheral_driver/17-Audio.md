# Audio

介绍K3 Audio的功能和使用方法。

## 模块介绍

K3 Audio 模块包含 6 路 I2S 音频接口、 4 路 RI2S 音频接口和 2 路 DP 音频接口，其中有两路 RI2S 接口用于 DP 音频输出。

### 功能介绍

系统基于 ALSA（Advanced Linux Sound Architecture）音频架构，整体框架如下：

![](static/AUDIO.png)

ALSA音频框架可以分为以下几个层次：

- **ALSA Library**
向应用程序提供统一的 API 接口，应用程序可调用 alsa-lib 接口实现音频播放、录制和控制。目前提供两套常用库：标准 Linux 系统使用 alsa-lib，Android 系统通常使用 tinyalsa 作为简化的 ALSA 库。

- **ALSA Core**
ALSA 核心层，向上提供 PCM、CTL、MIDI、TIMER 等逻辑设备接口，向下驱动底层硬件（如 I2S、DMA、Codec）。

- **ASoC Core （ALSA System on Chip）**
ALSA 的标准框架，ASoC 是 ALSA 的 SoC 驱动框架核心部分，提供音频设备驱动的通用接口与数据结构。

- **Hardware Driver**
音频硬件驱动主要由三部分组成：Machine、Platform 和 Codec。它们共同提供 ALSA 驱动接口（ALSA Driver API）以及音频设备的初始化和运行流程。这部分内容由驱动开发人员具体实现，功能较为底层。

   - **Machine（板级驱动）**：通常指一款具体的开发板，集成了特定外设，为 CPU 和 Codec 提供运行载体。由于硬件配置不同，Machine 驱动几乎无法复用。它的主要作用是将 Platform 驱动与 Codec 驱动关联起来，并完成与具体板卡相关的初始化工作。例如，可以通过 `snd_soc_dai_link` 指定要使用的 Platform 驱动、SoC 端的 DAI（Digital Audio Interface）接口、对应的 Codec 驱动及其 DAI 接口，同时还会处理一些和硬件设计紧密相关的逻辑。

   - **Platform（平台驱动）**：通常指某款 SoC 平台，包含 I2S、AC97 等音频接口，具备音频所需的时钟、DMA 等资源。Platform 驱动仅与特定 SoC 相关，负责实现 SoC 的 DMA 传输和 DAI 接口逻辑。它与 Machine 无关，因此可以被多个 Machine 驱动复用。得益于这种抽象设计，同一款 SoC 可在不同的开发板上直接复用 Platform 驱动而无需修改。

   - **Codec（编解码器驱动）**：Codec 芯片通常集成了 I2S 接口、D/A（数模转换）、A/D（模数转换）、混音器（Mixer）、功放（PA）等模块，支持多种输入（如麦克风 Mic、线路输入 Line-in、I2S、PCM）和输出（如耳机、喇叭、听筒、线路输出 Line-out）。一般通过 I2C 与 SoC 通信进行控制。Codec 驱动仅与具体的 Codec 芯片相关，和 SoC 或 Machine 无关，也具有较高的复用性。同一个 Codec 驱动可以在多个不同的 Machine 上使用。

#### 音频接口
K3 Audio 模块包含3种音频接口：

- **I2S 接口**
6 路 I2S 接口
- **RI2S 接口**
4 路 RI2S 接口，其中有两路用于 DP 音频输出
- **DP 音频接口**
2 路 DP 音频接口

#### 音频方案

K3 目前支持的音频方案：

- **方案一**：I2S1 + Codec 芯片 ES8326B（I2C 外接），支持音频播放与录制功能（deb板型）
- **方案二**：RI2S2 + DP0 接口进行音频输出，仅支持播放（com260板型）
- **方案三**：RI2S3 + DP1 接口进行音频输出，仅支持播放（deb板型）

### 源码结构

I2S / RI2S 控制器驱动代码在 `sound/soc/spacemit` 目录下：

```text
sound/soc/spacemit/
|-- k1_i2s.c               # I2S 控制器驱动
|-- k3_ri2s.c              # RI2S 控制器驱动
|-- Kconfig
`-- Makefile
```

Codec ES8326B 驱动代码在 `sound/soc/codecs` 目录下：

```text
sound/soc/codecs
|-- es8326.c              # ES8326B 驱动
|-- es8326.h              # ES8326B 驱动头文件
```

DP 音频驱动代码
```text
drivers/gpu/drm/spacemit/
|-- Kconfig
|-- Makefile
|-- spacemit_inno_dp.c      # DP 驱动
|-- spacemit_inno_dp.h      # DP 驱动头文件
```

## I2S

### 关键特性
- 支持播放和录制功能
- 支持全双工
- 支持 DMA 传输
- 支持 标准I2S、DSP_A、DSP_B 格式
  - 标准I2S：支持 8/16/48KHz 采样率，16 bits 采样深度，2 channels
  - DSP_A/DSP_B：支持 8/16/48KHz 采样率，32 bits 采样深度，1 channel

### 配置介绍

主要包括 **CONFIG 配置** 和 **DTS 配置**

#### CONFIG 配置

- 基础音频功能支持

`CONFIG_SOUND`、`CONFIG_SND`、`CONFIG_SND_SOC` 为 ALSA 音频驱动框架提供基础支持，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])
```

- I2S 音频功能支持

`CONFIG_SND_SOC_K1_I2S` 为 I2S 音频功能提供支持，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])│
  │             -> SpacemiT│
  │               -> K1 I2S Device Driver (SND_SOC_K1_I2S [=y])
```

#### DTS 配置
主要是 pinctrl 配置，根据实际的硬件设计进行配置。

##### pinctrl 配置

- I2S0 pinctrl 配置
有两组 pinctrl 配置 `sspa0_0_cfg` 和 `sspa0_1_cfg`，根据实际的硬件设计进行选择。

```dts
        sspa0_0_cfg: sspa0-0-cfg {
                sspa0-0-pins {
                        pinmux = <K3_PADCONF(81, 2)>,   /* sspa0 clk */
                                 <K3_PADCONF(82, 2)>,   /* sspa0 frm */
                                 <K3_PADCONF(83, 2)>,   /* sspa0 tx */
                                 <K3_PADCONF(84, 2)>,   /* sspa0 rx */
                                 <K3_PADCONF(85, 2)>;   /* sspa0 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };

        sspa0_1_cfg: sspa0-1-cfg {
                sspa0-0-pins {
                        pinmux = <K3_PADCONF(111, 2)>,  /* sspa0 clk */
                                 <K3_PADCONF(112, 2)>,  /* sspa0 frm */
                                 <K3_PADCONF(113, 2)>,  /* sspa0 tx */
                                 <K3_PADCONF(114, 2)>,  /* sspa0 rx */
                                 <K3_PADCONF(115, 2)>;  /* sspa0 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s0 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa0_0_cfg>;         /* 使用sspa0_0_cfg这组pin */
        status = "okay";
};
```

- I2S1 pinctrl 配置
有两组 pinctrl 配置 `sspa1_0_cfg` 和 `sspa1_1_cfg`，根据实际的硬件设计进行选择。

```dts
        sspa1_0_cfg: sspa1-0-cfg {
                sspa1-0-pins {
                        pinmux = <K3_PADCONF(15, 2)>,   /* sspa1 clk */
                                 <K3_PADCONF(16, 2)>,   /* sspa1 frm */
                                 <K3_PADCONF(17, 2)>,   /* sspa1 tx */
                                 <K3_PADCONF(18, 2)>,   /* sspa1 rx */
                                 <K3_PADCONF(19, 2)>;   /* sspa1 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };

        sspa1_1_cfg: sspa1-1-cfg {
                sspa1-0-pins {
                        pinmux = <K3_PADCONF(122, 2)>,  /* sspa1 clk */
                                 <K3_PADCONF(123, 2)>,  /* sspa1 frm */
                                 <K3_PADCONF(124, 2)>,  /* sspa1 tx */
                                 <K3_PADCONF(125, 2)>,  /* sspa1 rx */
                                 <K3_PADCONF(126, 2)>;  /* sspa1 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s1 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa1_0_cfg>;          /* 使用sspa1_0_cfg这组pin */
        status = "okay";
};
```

- I2S2 pinctrl 配置
只有一组 pinctrl 配置

```dts
        sspa2_0_cfg: sspa2-0-cfg {
                sspa2-0-pins {
                        pinmux = <K3_PADCONF(76, 2)>,   /* sspa2 clk */
                                 <K3_PADCONF(77, 2)>,   /* sspa2 frm */
                                 <K3_PADCONF(78, 2)>,   /* sspa2 tx */
                                 <K3_PADCONF(79, 2)>,   /* sspa2 rx */
                                 <K3_PADCONF(80, 2)>;   /* sspa2 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s2 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa2_0_cfg>;          /* 使用sspa2_0_cfg这组pin */
        status = "okay";
};
```

- I2S3 pinctrl 配置
只有一组 pinctrl 配置

```dts
        sspa3_0_cfg: sspa3-0-cfg {
                sspa3-0-pins {
                        pinmux = <K3_PADCONF(99, 2)>,   /* sspa3 clk */
                                 <K3_PADCONF(100, 2)>,  /* sspa3 frm */
                                 <K3_PADCONF(101, 2)>,  /* sspa3 tx */
                                 <K3_PADCONF(102, 2)>,  /* sspa3 rx */
                                 <K3_PADCONF(103, 2)>;  /* sspa3 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s3 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa3_0_cfg>;          /* 使用sspa3_0_cfg这组pin */
        status = "okay";
};
```

- I2S4 pinctrl 配置
只有一组 pinctrl 配置

```dts
        sspa4_0_cfg: sspa4-0-cfg {
                sspa4-0-pins {
                        pinmux = <K3_PADCONF(69, 2)>,   /* sspa4 clk */
                                 <K3_PADCONF(70, 2)>,   /* sspa4 frm */
                                 <K3_PADCONF(71, 2)>,   /* sspa4 tx */
                                 <K3_PADCONF(72, 2)>,   /* sspa4 rx */
                                 <K3_PADCONF(73, 2)>;   /* sspa4 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s4 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa4_0_cfg>;          /* 使用sspa4_0_cfg这组pin */
        status = "okay";
};
```

- I2S5 pinctrl 配置
只有一组 pinctrl 配置

```dts
        sspa5_0_cfg: sspa5-0-cfg {
                sspa5-0-pins {
                        pinmux = <K3_PADCONF(0, 2)>,    /* sspa5 clk */
                                 <K3_PADCONF(1, 2)>,    /* sspa5 frm */
                                 <K3_PADCONF(2, 2)>,    /* sspa5 tx */
                                 <K3_PADCONF(3, 2)>,    /* sspa5 rx */
                                 <K3_PADCONF(4, 2)>;    /* sspa5 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&i2s5 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa5_0_cfg>;          /* 使用sspa5_0_cfg这组pin */
        status = "okay";
};
```

##### 固定采样率配置
如果希望 I2S 固定支持某个采样率，可以通过 `spacemit,fixed-sample-rate` 进行配置，取值有 8000/16000/48000，单位是Hz。
需要注意的是：因为多个 I2S 之间有共用的时钟，所以多个 I2S 同时使能时，需要配置这些 I2S 只支持某个采样率，保持频率一致，否则会导致数据出错，也是通过 `spacemit,fixed-sample-rate` 进行配置。
例如，I2S0 和 I2S1 同时使能，采样率可配置为 8000/16000/48000Hz 中的一种，但必须相同。

```dts
&i2s0 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa0_0_cfg>;
        /* must set to same value when enable multi i2s: 8000/16000/48000Hz */
        spacemit,fixed-sample-rate = <48000>;
        status = "okay";
};

&i2s1 {
        pinctrl-names = "default";
        pinctrl-0 = <&sspa1_0_cfg>;
        /* must set to same value when enable multi i2s: 8000/16000/48000Hz */
        spacemit,fixed-sample-rate = <48000>;
        status = "okay";
};
```

## RI2S

### 关键特性

- 支持播放和录制功能
- 支持半双工
- 支持 DMA 传输（专用DMA）
- 支持标准I2S、左对齐、右对齐、DSP_A、DSP_B 格式
  - 标准I2S/左对齐/右对齐：支持 8/16/48KHz 采样率，16/32 bits 采样深度，2 channels
  - DSP_A/DSP_B：支持 8/16/48KHz 采样率，16/32 bits 采样深度，up to 4 channels

### 配置介绍

主要包括 **CONFIG 配置** 和 **DTS 配置**。

#### CONFIG 配置

- 基础音频功能支持

`CONFIG_SOUND`、`CONFIG_SND`、`CONFIG_SND_SOC`为 ALSA 音频驱动框架提供支持，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])
```

- RI2S 音频功能支持

`CONFIG_SND_SOC_K3_RI2S` 为 RI2S 音频功能提供支持，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])│
  │             -> SpacemiT│
  │               -> K3 RI2S Device Driver (SND_SOC_K3_RI2S [=y])
```

#### DTS 配置
主要是 pinctrl 配置，要根据实际的硬件设计进行配置。

##### pinctrl 配置

- RI2S0 pinctrl 配置
有两组 pinctrl 配置 `rsspa0_0_cfg` 和 `rsspa0_1_cfg`，根据实际的硬件设计进行选择。

```dts
        rsspa0_0_cfg: rsspa0-0-cfg {
                rsspa0-0-pins {
                        pinmux = <K3_PADCONF(6, 2)>,    /* rsspa0 clk */
                                 <K3_PADCONF(7, 2)>,    /* rsspa0 frm */
                                 <K3_PADCONF(8, 2)>,    /* rsspa0 tx */
                                 <K3_PADCONF(9, 2)>,    /* rsspa0 rx */
                                 <K3_PADCONF(10, 2)>;   /* rsspa0 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };

        rsspa0_1_cfg: rsspa0-1-cfg {
                rsspa0-0-pins {
                        pinmux = <K3_PADCONF(76, 1)>,   /* rsspa0 clk */
                                 <K3_PADCONF(77, 1)>,   /* rsspa0 frm */
                                 <K3_PADCONF(78, 1)>,   /* rsspa0 tx */
                                 <K3_PADCONF(79, 1)>,   /* rsspa0 rx */
                                 <K3_PADCONF(80, 1)>;   /* rsspa0 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&ri2s0 {
        pinctrl-names = "default";
        pinctrl-0 = <&rsspa0_0_cfg>;         /* 使用rsspa0_0_cfg这组pin */
        status = "okay";
};
```

- RI2S1 pinctrl 配置
只有一组 pinctrl 配置

```dts
        rsspa1_0_cfg: rsspa1-0-cfg {
                rsspa1-0-pins {
                        pinmux = <K3_PADCONF(36, 2)>,   /* rsspa1 clk */
                                 <K3_PADCONF(37, 2)>,   /* rsspa1 frm */
                                 <K3_PADCONF(38, 2)>,   /* rsspa1 tx */
                                 <K3_PADCONF(39, 2)>,   /* rsspa1 rx */
                                 <K3_PADCONF(40, 2)>;   /* rsspa1 sysclk */

                        bias-pull-up;                   /* normal pull-up */
                        drive-strength = <25>;          /* DS8 */
                };
        };
```
配置示例：
```dts
&ri2s1 {
        pinctrl-names = "default";
        pinctrl-0 = <&rsspa1_0_cfg>;         /* 使用rsspa1_0_cfg这组pin */
        status = "okay";
};
```

- RI2S2 pinctrl 配置

RI2S2 默认与 DP0 连接，无须配置Pinctrl。

- RI2S3 pinctrl 配置

RI2S3 默认与 DP1 连接，无须配置Pinctrl。


## DP 音频接口

### 关键特性
- 支持播放功能
- 支持标准I2S、左对齐、右对齐 格式
  - 支持 16/20/24 bits 采样深度，2 channels，采样率最高 192KHz
- 要求：BCLK = 64 * fs， MCLK = 512 * fs

### 配置介绍

主要包括 **CONFIG 配置** 和 **DTS 配置**。

#### CONFIG 配置

- 基础音频功能支持

`CONFIG_SOUND`、`CONFIG_SND`、`CONFIG_SND_SOC`为 ALSA 音频驱动框架提供支持，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])
```

因为 DP 音频功能 依赖于 DP 显示功能，需要确保 `DP 显示相关配置已启用`，请参考 DP 显示相关文档。

#### DTS 配置
```dts
        dp0: dp0@cac84000 {
                compatible = "spacemit,inno-dp0";
                reg = <0x0 0xcac84000 0x0 0x4000>;
                interrupt-parent = <&saplic>;
                interrupts = <132 IRQ_TYPE_LEVEL_HIGH>;
                clocks = <&syscon_apmu CLK_APMU_EDP0_PXCLK>;
                clock-names = "pxclk";
                resets = <&syscon_apmu RESET_APMU_EDP0>;
                reset-names = "reset";
                color_format = <1>;
                ref_clock = <24000000>;
                dp-id = <0>;
                #sound-dai-cells = <0>;               /* 配置 dp0 支持数字音频接口 */
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

        dp1: dp1@cac88000 {
                compatible = "spacemit,inno-dp1";
                reg = <0x0 0xcac88000 0x0 0x4000>;
                interrupt-parent = <&saplic>;
                interrupts = <140 IRQ_TYPE_LEVEL_HIGH>;
                clocks = <&syscon_apmu CLK_APMU_EDP1_PXCLK>;
                clock-names = "pxclk";
                resets = <&syscon_apmu RESET_APMU_EDP1>;
                reset-names = "reset";
                color_format = <1>;
                ref_clock = <24000000>;
                dp-id = <1>;
                #sound-dai-cells = <0>;               /* 配置 dp1 支持数字音频接口 */
                status = "disabled";

                port {
                        #address-cells = <1>;
                        #size-cells = <0>;

                        dp1_in: endpoint@0 {
                                reg = <0>;
                                remote-endpoint = <&dpu1_crtc0_out0>;
                        };
                };
        };
```

## 声卡配置

### I2S-Codec 声卡配置

#### Simple-Card 配置

##### CONFIG 配置
`CONFIG_SND_SIMPLE_CARD` 为 ALSA 音频声卡 Machine 驱动，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])│
  │             -> Generic drivers│
  │               -> ASoC Simple sound card support (SND_SIMPLE_CARD [=y])
```

##### DTS 配置
```dts
                sound_card0: sound-card@0 {
                        status = "disabled";
                        compatible = "simple-audio-card";
                };

                sound_card1: sound-card@1 {
                        status = "disabled";
                        compatible = "simple-audio-card";
                };
```

#### Codec 配置
以 Codec ES8326B 为例，进行配置

##### CONFIG 配置

`CONFIG_SND_SOC_ES8326` 为 Everest Semi ES8326 CODEC 驱动配置，需要配置为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])│
  │             -> CODEC drivers│
  │               -> Everest Semi ES8326 CODEC (SND_SOC_ES8326 [=y])
```

##### DTS 配置

deb板上 Codec ES8326B 的完整方案配置如下：

```dts
        es8326: es8326@19{
                compatible = "everest,es8326";
                reg = <0x19>;
                #sound-dai-cells = <0>;
                interrupt-parent = <&gpio>;
                interrupts = <3 31 IRQ_TYPE_EDGE_RISING>;
                /* spk-ctl-gpio = <&gpio 127 0>; */
                everest,mic1-src = [44];
                everest,mic2-src = [66];
                everest,jack-detect-inverted;
                status = "okay";
        };
```
以下配置需要根据实际硬件设计进行调整。

- GPIO

```dts
                spk-ctl-gpio = <&gpio 127 0>;         /* 板级扬声器控制gpio */
```
表示通过 gpio127 控制板级扬声器。

- 中断
```dts
                interrupt-parent = <&gpio>;
                interrupts = <3 31 IRQ_TYPE_EDGE_RISING>;     /* 耳机插拔检测 */
```
表示通过 gpio31 监听耳机插拔检测中断，耳机插入或者拔出都会触发中断，ES8326驱动会根据寄存器值来判断耳机是插入或拔出，并切换到对应通路，实现耳机与板级扬声器的播放切换以及麦克风输入源的切换。

- ADC 输入配置
```dts
                everest,mic1-src = [44];
                everest,mic2-src = [66];                /* ADC 默认输入源是 板级数字硅麦 */
```
```dts
                everest,mic1-src = [00];
                everest,mic2-src = [00];                /* ADC 默认输入源是 耳机mic */
```

- 耳机检测电平配置
```
                everest,jack-detect-inverted;
```
表示耳机插拔检测引脚为低电平有效，即插入耳机时引脚为低电平，拔出耳机时引脚为高电平。没有这个配置，默认为高电平有效。需根据实际硬件设计进行配置。

##### MCLK 配置

Codec ES8326B 的 MCLK 由 I2S 提供，在声卡节点中的 `simple-audio-card,mclk-fs` 进行配置

```dts
&sound_card1 {
        status = "okay";
        simple-audio-card,name = "snd-es8326";             /* 声卡名称 */
        simple-audio-card,mclk-fs = <256>;                 /* 配置mclk = 256 * fs，I2S 控制器只提供 64/128/256 * fs 的 MCLK */
        simple-audio-card,dai-link@0 {
                format = "i2s";                            /* 配置 format 为 标准i2s 格式，I2S 控制器只支持 标准I2S/DSP_A/DSP_B 模式 */
                frame-master = <&link1_cpu>;
                bitclock-master = <&link1_cpu>;            /* 配置 i2s 为 master，提供 BCLK 和 LRCK 时钟 */

                link1_cpu: cpu {
                        sound-dai = <&i2s1>;               /* 配置 i2s1 为 cpu dai */
                };

                link1_codec: codec {
                        sound-dai = <&es8326>;             /* 配置 es8326 为 codec dai */
                };
        };
};
```

注意：I2S 控制器只提供 64/128/256 * fs 的 MCLK，所以 `simple-audio-card,mclk-fs` 只支持配置成 64/128/256；I2S 控制器只支持 标准I2S/DSP_A/DSP_B 模式，所以 `format` 只支持配置成 "i2s"、"dsp_a"、"dsp_b"。

#### 完整声卡配置

```dts
&sound_card1 {
        status = "okay";
        simple-audio-card,name = "snd-es8326";             /* 声卡名称 */
        simple-audio-card,mclk-fs = <256>;                 /* 配置mclk = 256 * fs，I2S 控制器只提供 64/128/256 * fs 的 MCLK */
        simple-audio-card,dai-link@0 {
                format = "i2s";                            /* 配置 format 为 标准i2s 格式，I2S 控制器只支持 标准I2S/DSP_A/DSP_B 模式 */
                frame-master = <&link1_cpu>;
                bitclock-master = <&link1_cpu>;            /* 配置 i2s 为 master，提供 BCLK 和 LRCK 时钟 */

                link1_cpu: cpu {
                        sound-dai = <&i2s1>;               /* 配置 i2s1 为 cpu dai */
                };

                link1_codec: codec {
                        sound-dai = <&es8326>;             /* 配置 es8326 为 codec dai */
                };
        };
};
```

注意：
I2S 控制器只支持 标准I2S/DSP_A/DSP_B 模式，所以 `format` 只支持配置成 "i2s"、"dsp_a"、"dsp_b"；`simple-audio-card,mclk-fs` 只支持配置成 64/128/256。


### RI2S-DP 声卡配置

#### Simple-Card 配置

##### CONFIG 配置
`CONFIG_SND_SIMPLE_CARD` 为 ALSA 音频声卡 Machine 驱动，默认情况下，选项为 `Y`

```text
     -> Device Drivers│
  │       -> Sound card support (SOUND [=y])│
  │         -> Advanced Linux Sound Architecture (SND [=y])│
  │           -> ALSA for SoC audio support (SND_SOC [=y])│
  │             -> Generic drivers│
  │               -> ASoC Simple sound card support (SND_SIMPLE_CARD [=y])
```

##### DTS 配置
```dts
        sound_card_dp0: sound-card-dp0@0 {
                compatible = "simple-audio-card";
                simple-audio-card,mclk-fs = <512>;
                status = "disabled";
                simple-audio-card,dai-link@0 {
                        format = "left_j";
                        frame-master = <&dp0_link_cpu>;
                        bitclock-master = <&dp0_link_cpu>;
                        dp0_link_cpu: cpu {
                                sound-dai = <&ri2s2>;        /* 配置 ri2s2 为 DP0 声卡 cpu dai */
                        };
                };
        };

        sound_card_dp1: sound-card-dp1@1 {
                compatible = "simple-audio-card";
                simple-audio-card,mclk-fs = <512>;
                status = "disabled";
                simple-audio-card,dai-link@0 {
                        format = "left_j";
                        frame-master = <&dp1_link_cpu>;
                        bitclock-master = <&dp1_link_cpu>;
                        dp1_link_cpu: cpu {
                                sound-dai = <&ri2s3>;        /* 配置 ri2s3 为 DP1 声卡cpu dai */
                        };
                };
        };
```

#### 完整声卡配置

RI2S2 硬件上与 DP0 连接，RI2S3 硬件上与 DP1 连接，所以 sound_card_dp0 与 sound_card_dp1 分别对应 DP0 与 DP1，cpu dai分别配置为 RI2S2 与 RI2S3。
```dts
        sound_card_dp0: sound-card-dp0@0 {
                compatible = "simple-audio-card";
                simple-audio-card,mclk-fs = <512>;
                status = "disabled";
                simple-audio-card,dai-link@0 {
                        format = "left_j";
                        frame-master = <&dp0_link_cpu>;
                        bitclock-master = <&dp0_link_cpu>;
                        dp0_link_cpu: cpu {
                                sound-dai = <&ri2s2>;         /* 配置 ri2s2 为 DP0 声卡 cpu dai */
                        };
                };
        };

        sound_card_dp1: sound-card-dp1@1 {
                compatible = "simple-audio-card";
                simple-audio-card,mclk-fs = <512>;
                status = "disabled";
                simple-audio-card,dai-link@0 {
                        format = "left_j";
                        frame-master = <&dp1_link_cpu>;
                        bitclock-master = <&dp1_link_cpu>;
                        dp1_link_cpu: cpu {
                                sound-dai = <&ri2s3>;        /* 配置 ri2s3 为 DP1 声卡 cpu dai */
                        };
                };
        };
```

deb 支持 DP1，所以在 deb 板级配置中，需要配置 `adma3`、`ri2s3`、`sound_card_dp1`。
```dts
&adma3 {
        status = "okay";                                  /* 使能 adma3 */
};

&ri2s3 {
        status = "okay";                                  /* 使能 ri2s3 */
};

&sound_card_dp1 {
        status = "okay";
        simple-audio-card,name = "snd-dp1";               /* 配置声卡名称为 snd-dp1 */
        simple-audio-card,mclk-fs = <512>;                /* 配置 MCLK = 512 * fs，因为 DP 音频接口要求 BCLK = 64 * fs， MCLK = 512 * fs */
        simple-audio-card,dai-link@0 {
                format = "left_j";                        /* 配置 format 为 left_j 左对齐格式 */
                frame-master = <&dp1_link_cpu>;
                bitclock-master = <&dp1_link_cpu>;        /* 配置 ri2s3 为 master，提供 BCLK 和 LRCK 时钟 */
                dp1_link_codec: codec {
                        playback-only;                    /* 配置 dp1 为只支持播放 */
                        sound-dai = <&dp1>;               /* 配置 dp1 为 DP1 声卡 codec dai */
                };
        };
};
```

注意：
DP 音频接口只支持 标准I2S/左对齐/右对齐 模式，所以 `format` 只支持配置成 "i2s"、"left_j"、"right_j"，但因为DP 音频接口还要求 BCLK = 64 * fs，MCLK = 512 * fs，所以 `format` 只支持配置成 "left_j"、"right_j"， `simple-audio-card,mclk-fs` 必须配置成 512。

## 接口介绍
### API 介绍

请参阅相关的 Linux 官方文档。

## Debug 介绍

可以通过 `/proc/asound/` 下的节点进行 debug

### 查看声卡设备（`/proc/asound/pcm`）

```shell
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/pcm
00-00: d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0 : d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0 : playback 1 : capture 1
01-00: c0883d00.ri2s3-dp audio dp audio-0 : c0883d00.ri2s3-dp audio dp audio-0 : playback 1
bianbu@bianbu-spacemitk3deb1:~$
```

### 查看声卡状态等信息

- `closed`：声卡处于关闭状态
```shell
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/card0/pcm0p/sub0/status
closed
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/card0/pcm0p/sub0/hw_params
closed
```

- `RUNNING`：声卡处于播放或录制状态，可以查看声卡状态和参数
```shell
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/card0/pcm0p/sub0/status
state: RUNNING
owner_pid   : 3767
trigger_time: 224110.719883196
tstamp      : 224164.735391138
delay       : 2048
avail       : 2048
avail_max   : 2048
-----
hw_ptr      : 2592768
appl_ptr    : 2594816
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/card0/pcm0p/sub0/status
state: RUNNING
owner_pid   : 3767
trigger_time: 224110.719883196
tstamp      : 224166.975406348
delay       : 3072
avail       : 1024
avail_max   : 2048
-----
hw_ptr      : 2700288
appl_ptr    : 2703360
bianbu@bianbu-spacemitk3deb1:~$ cat /proc/asound/card0/pcm0p/sub0/hw_params
access: RW_INTERLEAVED
format: S16_LE
subformat: STD
channels: 2
rate: 48000 (48000/1)
period_size: 1024
buffer_size: 4096
root:/#
```

## 测试介绍

音频功能可以通过 `alsa-utils/tinyalsa` 工具进行功能测试，Bianbu/Buildroot 已集成 alsa-utils 工具。

### 播放测试

#### 查看 playback 设备 （`aplay -l`）

`aplay -l`：查看播放设备
以 deb 板为例：
```shell
bianbu@bianbu-spacemitk3deb1:~$ aplay -l
**** PLAYBACK 硬體裝置清單 ****
card 0: sndes8326 [snd-es8326], device 0: d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0 [d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0]
  子设备: 1/1
  子设备 #0: subdevice #0
card 1: snddp1 [snd-dp1], device 0: c0883d00.ri2s3-dp audio dp audio-0 [c0883d00.ri2s3-dp audio dp audio-0]
  子设备: 1/1
  子设备 #0: subdevice #0
bianbu@bianbu-spacemitk3deb1:~$
```
可以看到有两个播放设备：
I2S-Codec 播放设备，`cardid` 为 `0`，`deviceid` 为 `0`
RI2S-DP1 播放设备，`cardid` 为 `1`，`deviceid` 为 `0`

#### 播放命令示例

选择设备进行播放，通过 `cardid` 和 `deviceid` 进行指定

例：选择 I2S-Codec 声卡进行播放
```shell
aplay -Dhw:0,0 -r 48000 -f S16_LE --period-size=1024 --buffer-size=4096 test.wav
```
例：选择 RI2S-DP1 声卡进行播放
```shell
aplay -Dhw:1,0 -r 48000 -f S16_LE --period-size=1024 --buffer-size=4096 test.wav
```

### 录音测试

#### 查看 capture 设备（`arecord -l`）

`arecord -l`：查看录制设备
以 deb 板为例：

```shell
bianbu@bianbu-spacemitk3deb1:~$ arecord -l
**** CAPTURE 硬體裝置清單 ****
card 0: sndes8326 [snd-es8326], device 0: d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0 [d4026800.i2s1-ES8326 HiFi ES8326 HiFi-0]
  子设备: 1/1
  子设备 #0: subdevice #0
bianbu@bianbu-spacemitk3deb1:~$
```
可以看到有一个录制设备：
I2S-Codec 录制设备，`cardid` 为 `0`，`deviceid` 为 `0`

#### 录音命令示例

选择设备进行录制，通过 `cardid` 和 `deviceid` 进行指定

例：选择 I2S-Codec 声卡进行录制
```shell
arecord -Dhw:0,0 -r 48000 -c 2 -f S16_LE --period-size=1024 --buffer-size=4096 test_record.wav
```

## FAQ
### 如何确认声卡是否注册

```bash
cat /proc/asound/pcm
```
查看已注册的声卡设备，确认是否有对应声卡存在，如果没有，说明声卡未注册成功。
未注册成功需重点确认：
1. DTS 相关配置是否打开；
2. CONFIG 相关配置是否打开；
3. 是否出现 deferred probe 、 clock/reset 或 pinctrl 相关错误。

### I2S + Codec 不出声怎么查

1. `cat /proc/asound/pcm` 查看声卡是否注册成功；
2. DTS I2S pinctrl 配置是否正确；
3. `i2cdetect` 是否能看到 Codec 设备；
4. `i2cdump` 读取 Codec 寄存器是否成功；
5. 播放时选择设备是否为 Codec 对应声卡；
6. `amixer` 查看 Codec 的 playback 通路是否打开；
7. `amixer` 查看 Codec 的 DAC 音量配置是否正常；
8. 插拔耳机，确认耳机和板级扬声器是否都无声；
9. `i2cdump` 读取 Codec 寄存器看其他配置是否正常，需结合硬件设计和寄存器定义进行分析。

### I2S + Codec 录不到音怎么查

1. `cat /proc/asound/pcm` 查看声卡是否注册成功；
2. DTS I2S pinctrl 配置是否正确；
3. `i2cdetect` 是否能看到 Codec 设备；
4. `i2cdump` 读取 Codec 寄存器是否成功；
5. 录音时选择设备是否为 Codec 对应声卡；
6. `amixer` 查看 Codec 的 capture 通路是否打开；
7. `amixer` 查看 Codec 的 ADC 音量配置是否正常；
8. 插拔耳机，确认耳机 mic 和板级 mic 是否都录不到音；
9. `i2cdump` 读取 Codec 寄存器看其他配置是否正常，需结合硬件设计和寄存器定义进行分析。

### DP 音频不出声时怎么查

1. 先确认 DP 显示是否正常；
2. `cat /proc/asound/pcm` 查看 DP 声卡是否注册成功；
3. 播放时选择设备是否为 DP 对应声卡；
4. 检查外接显示器是否支持音频；
5. 读取外接显示器的EDID，确认是否支持当前音频格式。
