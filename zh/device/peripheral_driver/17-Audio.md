# Audio

介绍Audio的功能和使用方法。

## 模块介绍

Audio 模块包含 2 路 I2S 接口和 1 路 HDMI 音频接口。

### 功能介绍

系统基于 ALSA（Advanced Linux Sound Architecture）音频架构，整体框架如下：

![](static/AUDIO.png)

ALSA音频框架可以分为以下几个层次：

- **ALSA Library**  
向应用程序上提供统一的 API 接口，应用程序可调用 alsa-lib 接口实现音频播放、录制和控制。现在提供了两套基本的库。而 Android 系统通常使用 tinyalsa 作为简化的 ALSA 库。

- **ALSA Core**
ALSA 核心层，向上提供 PCM、CTL、MIDI、TIMER 等逻辑设备接口，向下驱动底层硬件（如 I2S、DMA、Codec）。

- **ASoC Core （ALSA System on Chip）**  
ALSA 的标准框架，ASoC 是 ALSA 的 SoC 驱动框架核心部分，提供音频设备驱动的通用接口与数据结构。

- **Hardware Driver**  
音频硬件驱动主要由三部分组成：Machine、Platform 和 Codec。它们共同提供 ALSA 驱动接口（ALSA Driver API）以及音频设备的初始化和运行流程。这部分内容由驱动开发人员具体实现，功能较为底层。

   - **Machine（板级驱动）**：通常指一款具体的开发板，集成了特定外设，为 CPU 和 Codec 提供运行载体。由于硬件配置不同，Machine 驱动几乎无法复用。它的主要作用是将 Platform 驱动与 Codec 驱动关联起来，并完成与具体板卡相关的初始化工作。例如，可以通过 `snd_soc_dai_link` 指定要使用的 Platform 驱动、SoC 端的 DAI（Digital Audio Interface）接口、对应的 Codec 驱动及其 DAI 接口，同时还会处理一些和硬件设计紧密相关的逻辑。

   - **Platform（平台驱动）**：通常指某款 SoC 平台，包含 I2S、AC97 等音频接口，具备音频所需的时钟、DMA 等资源。Platform 驱动仅与特定 SoC 相关，负责实现 SoC 的 DMA 传输和 DAI 接口逻辑。它与 Machine 无关，因此可以被多个 Machine 驱动复用。得益于这种抽象设计，同一款 SoC 可在不同的开发板上直接复用 Platform 驱动而无需修改。

   - **Codec（编解码器驱动）**：Codec 芯片通常集成了 I2S 接口、D/A（数模转换）、A/D（模数转换）、混音器（Mixer）、功放（PA）等模块，支持多种输入（如麦克风 Mic、线路输入 Line-in、I2S、PCM）和输出（如耳机、喇叭、听筒、线路输出 Line-out）。一般通过 I2C 与 SoC 通信进行控制。Codec 驱动仅与具体的 Codec 芯片相关，和 SoC 或 Machine 无关，也具有较高的复用性。同一个 Codec 驱动可以在多个不同的 Machine 上使用。

### 音频方案介绍

K1 目前支持两种音频声卡方案：

- **方案一**：通过 HDMI 接口进行音频输出，仅支持播放；

- **方案二**：I2S0 配合外接的 I2C 接口 Codec 芯片 ES8326B，支持音频的播放与录制功能。

### 源码结构介绍

I2S / HDMIAUDIO 控制器驱动代码在 `sound/soc/spacemit` 目录下：

```
sound/soc/spacemit
├── Kconfig
├── Makefile
├── spacemit-dummy-codec.c    #dummy codec，配合hdmiaudio创建声卡
├── spacemit-snd-card.c       #声卡驱动
├── spacemit-snd-i2s.c        #i2s驱动
├── spacemit-snd-i2s.h
├── spacemit-snd-pcm-dma.c    #platform驱动，主要是pcm相关
├── spacemit-snd-sspa.c       #hdmiaudio驱动
├── spacemit-snd-sspa.h
```

Codec ES8326B 驱动代码在 `sound/soc/codec` 目录下：

```
sound/soc/codec
├── es8326.c
├── es8326.h
```

## I2S

### 关键特性

- 支持 48kHz 采样率，16bit 采样深度，双声道 
- 支持播放和录制功能  
- 支持全双工工作模式  

### 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

#### CONFIG 配置

##### 基础音频功能支持

`CONFIG_SOUND`、`CONFIG_SND`、`CONFIG_SND_SOC` 为 ALSA 音频驱动框架提供基础支持，默认情况，选项都为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
```

##### K1 音频功能支持

`CONFIG_SND_SOC_SPACEMIT`、`CONFIG_SPACEMIT_CARD`、`CONFIG_SPACEMIT_PCM` 为 K1 音频功能提供支持，默认情况下，选项都为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
```

##### I2S 功能支持

`CONFIG_SPACEMIT_I2S` 为 I2S 功能提供支持，默认情况下，此选项为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
                                        Audio Cpudai I2S (SPACEMIT_I2S [=y])
```

#### DTS 配置

##### pinctrl

- I2S0 pinctrl 配置  
有两组 pinctrol 配置 `pinctrl_sspa0_0` 和 `pinctrl_sspa0_1`，根据实际情况硬件设计进行配置

```
        pinctrl_sspa0_0: sspa0_0_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_118, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_clk */
                        K1X_PADCONF(GPIO_119, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_frm */
                        K1X_PADCONF(GPIO_120, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_txd */
                        K1X_PADCONF(GPIO_121, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_rxd */
                        K1X_PADCONF(GPIO_122, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_sysclk */
                >;
        };

        pinctrl_sspa0_1: sspa0_1_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_58,  MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_sysclk */
                        K1X_PADCONF(GPIO_111, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_clk */
                        K1X_PADCONF(GPIO_112, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_frm */
                        K1X_PADCONF(GPIO_113, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_txd */
                        K1X_PADCONF(GPIO_114, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_rxd */
                >;
        }

&i2s0 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_sspa0_0>;         #使用pinctrl_sspa0_0这组pin
        status = "okay";
};

```

- I2S1 pinctrl配置  

```
        pinctrl_sspa1: sspa1_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_24, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_sysclk */
                        K1X_PADCONF(GPIO_25, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_sclk */
                        K1X_PADCONF(GPIO_26, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_frm */
                        K1X_PADCONF(GPIO_27, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_txd */
                        K1X_PADCONF(GPIO_28, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_rxd */
                >;
        };

&i2s1 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_sspa1>;          #使用pinctrl_sspa1这组pin
        status = "okay";
};

```

#### I2S-Codec 声卡配置

##### Codec 配置

以 Codec ES8326B 为例，进行完整声卡配置

###### Config 配置

打开 ES8326B 配置

```
Device Drivers│
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                CODEC drivers
                                        Everest Semi ES8326 CODEC (SND_SOC_ES8326 [=y])
```

###### DTS 配置

- GPIO
需要按实际硬件设计来配置，例如 K1 某些开发板上，ES8326B 通过 gpio129 完成耳机插拔检测，通过 gpio127 控制板级扬声器。

```
        es8326: es8326@19{
                interrupt-parent = <&gpio>;
                interrupts = <126 1>;                 #耳机插拔检测
                spk-ctl-gpio = <&gpio 127 0>;         #板级扬声器控制gpio
        };

```

###### DTS 配置示例

Codec ES8236B 的完整方案配置如下：

```
        es8326: es8326@19{
                compatible = "everest,es8326";
                reg = <0x19>;
                #sound-dai-cells = <0>;
                interrupt-parent = <&gpio>;
                interrupts = <126 1>;                 #耳机插拔检测
                spk-ctl-gpio = <&gpio 127 0>;         #板级扬声器控制gpio
                everest,mic1-src = [44];              #ADC sourcs配置
                everest,mic2-src = [66];
                status = "okay";
        };
```

###### MCLK 倍率配置

Codec ES8326B 的 MCLK 由 I2S0 提供，在声卡节点 `sound_codec` 中进行配置

```

&sound_codec {
        status = "okay";
        simple-audio-card,name = "snd-es8326";
        spacemit,mclk-fs = <64>;                      #配置mclk = 64 * fs，即3.072MHz
        simple-audio-card,codec {
                sound-dai = <&es8326>;
        };
};

```

注意：`spacemit,mclk-fs` 只支持配置成 64/128/256，即 3.072/6.144/12.288MHz

##### 声卡配置

###### DTS 配置

```
        sound_codec: snd-card@1 {
                compatible = "spacemit,simple-audio-card";
                simple-audio-card,format = "i2s";
                status = "disabled";
                interconnects = <&dram_range4>;
                interconnect-names = "dma-mem";
                spacemit,init-jack;
                simple-audio-card,cpu {                      #cpudai配置
                        sound-dai = <&i2s0>;
                };
                simple-audio-card,plat {
                        sound-dai = <&i2s0_dma>;             #platform pcm配置
                };
        };

&sound_codec {
        status = "okay";
        simple-audio-card,name = "snd-es8326";
        spacemit,mclk-fs = <64>;
        simple-audio-card,codec {
                sound-dai = <&es8326>;                       #codecdai配置
        };
};

```

## HDMIAUDIO

### 关键特性

- 支持 48kHz 采样率，16bit 采样深度，双声道  
- 只支持播放

### 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**。
因为 HDMIAUDIO 依赖于 HDMI 显示功能，需要**确保 HDMI 显示已启用**，请参考对应的文档。

#### CONFIG 配置

##### 基础音频功能支持

`CONFIG_SOUND`、`CONFIG_SND`、`CONFIG_SND_SOC`为 ALSA 音频驱动框架提供支持，默认情况，此选项都为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
```

##### K1 音频功能支持

`CONFIG_SND_SOC_SPACEMIT`、`CONFIG_SPACEMIT_CARD`、`CONFIG_SPACEMIT_PCM` 为 K1 音频功能提供支持，默认情况下，此选项都为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
```

##### HDMIAUDIO 功能支持

`CONFIG_SPACEMIT_HDMIAUDIO`、`CONFIG_SPACEMIT_DUMMYCODEC` 为 HDMIAUDIO 功能提供支持，默认情况下，此选项都为 `Y`

```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
                                        Audio Cpudai HDMI Audio (SPACEMIT_HDMIAUDIO [=y])
                                        Audio CodecDai Dummy Codec (SPACEMIT_DUMMYCODEC [=y]) 
```

#### DTS 配置

```
&hdmiaudio {
        status = "okay";
};
```

#### HDMIAUDIO 声卡配置

##### DTS 配置

```

        sound_hdmi: snd-card@0 {
                compatible = "spacemit,simple-audio-card";
                simple-audio-card,name = "snd-hdmi";
                status = "disabled";
                interconnects = <&dram_range4>;
                interconnect-names = "dma-mem";
                simple-audio-card,plat {                        #platform pcm配置
                        sound-dai = <&hdmi_dma>;
                };
                simple-audio-card,codec {                       #codecdai配置
                        sound-dai = <&dummy_codec>;
                };
        };

&sound_hdmi {
        status = "okay";
        simple-audio-card,cpu {
                sound-dai = <&hdmiaudio>;                       #cpudai配置
        };
};

```

## 接口介绍

### API 介绍

请参阅相关的 Linux 官方文档。

## Debug 介绍

可以通过 `/proc/asound/` 下的节点进行 debug

### 查看声卡设备（`/proc/asound/pcm`）

```
root:/# cat /proc/asound/pcm
00-00: SSPA2-dummy_codec dummy_codec-0 :  : playback 1
01-00: i2s-dai0-ES8326 HiFi ES8326 HiFi-0 :  : playback 1 : capture 1
root:/#
```

### 查看声卡状态等信息

- `ClOSED`：声卡处于关闭状态
```
root:/# cat /proc/asound/card1/pcm0p/sub0/status
closed
root:/# cat /proc/asound/card1/pcm0p/sub0/hw_params
closed
root:/#
```

- `RUNNING`：声卡处于播放或录制状态，可以查看声卡状态和参数
```
root:/# cat /proc/asound/card1/pcm0p/sub0/status
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
root:/# cat /proc/asound/card1/pcm0p/sub0/status
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
root:/# cat /proc/asound/card1/pcm0p/sub0/hw_params
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

音频功能可以通过 `alsa-utils/tinyalsa` 工具完成功能测试，目前 Buildroot 已集成alsa-utils 工具。

### 播放测试

#### 查看 playback 设备 （`aplay -l`）

`aplay-l`：查看播放设备

```
root:/# aplay -l
**** PLAYBACK 硬體裝置清單 ****
card 0: sndhdmi [snd-hdmi], device 0: SSPA2-dummy_codec dummy_codec-0 []
  子设备: 1/1
  子设备 #0: subdevice #0
card 1: sndes8326 [snd-es8326], device 0: i2s-dai0-ES8326 HiFi ES8326 HiFi-0 []
  子设备: 1/1
  子设备 #0: subdevice #0
root:/#
```
可以看到有两个播放设备：
HDMIAUDIO 播放设备，`cardid` 为 `0`，`deviceid` 为 `0`
I2S-Codec 播放设备，`cardid` 为 `1`，`deviceid` 为 `0`

#### 播放命令示例

选择设备进行播放，通过 `cardid` 和 `deviceid` 进行指定

例：选择 HDMIAUDIO 声卡进行播放
```
aplay -Dhw:0,0 -r 48000 -f S16_LE --period-size=480 --buffer-size=1920 xxx.wav
```
例：选择 I2S-Codec 声卡进行播放
```
aplay -Dhw:1,0 -r 48000 -f S16_LE --period-size=1024 --buffer-size=4096 xxx.wav
```

### 录音测试

#### 查看 capture 设备（`arecord -1`）

`arecord -l`：查看录制设备
```
root@spacemit-k1-x-deb1-board:~# arecord -1
****
CAPTURE硬體裝置清單
****
card 1: sndes8326 [snd-es8326], device 0: i2s-daai0-ES8326 HiFi ES8326 HiFi-@
子设备:1/1
子设备 #0: subdevice #0
root@spacemit-k1-x-deb1-board:~#
```
可以看到有一个录制设备：
I2S-Codec 录制设备，`cardid` 为 `1`，`deviceid` 为 `0`

#### 录音命令示例

选择设备进行录制，通过 `cardid` 和 `deviceid` 进行指定

例：选择 I2S-Codec 声卡进行录制
```
arecord -Dhw:1,0 -r 48000 -c 2 -f S16_LE --period-size=1024 --buffer-size=4096 xxx.wav
```

## FAQ
