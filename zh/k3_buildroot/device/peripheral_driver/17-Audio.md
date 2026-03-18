# Audio

介绍 K3 平台 Audio 子系统的功能、内核配置方法、设备树配置方法以及调试方式。

## 模块介绍

K3 的音频子系统已经不适合简单按 K1 的“2 路 I2S + 1 路 HDMI 音频”来描述。根据 K3 SDK 驱动、`k3.dtsi` / `k3-rdomain.dtsi` / 各板级 DTS 的实际情况，K3 当前至少存在下面几类真实音频路径：

- **A-domain I2S / SSPA 控制器**，用于板级外接 Codec；
- **R-domain RI2S 控制器**，主要用于 DP 音频链路；
- **DP 音频**，由 `simple-audio-card` 把 `ri2s2/ri2s3` 和 `dp0/dp1` 连接成声卡；
- **I2S + 外部 ES8326 Codec 声卡**，支持播放与录制；
- **DMIC 相关板级供电与 pinctrl**，在部分板级 DTS 中可见；
- **多板型差异**：不同板子启用的是 `sound_card_dp0`、`sound_card_dp1` 或 `sound_card1`。

也就是说，K3 的 Audio 文档必须按 **真实板级方案** 来写，而不能只按 K1 老的 HDMI/I2S 模型写。

### 功能介绍

K3 基于 Linux **ALSA + ASoC** 架构实现音频功能。整体可以分为以下几层：

- **ALSA Library**  
  提供用户空间音频 API；
- **ALSA Core**  
  提供 PCM、Control、Mixer 等通用接口；
- **ASoC Core**  
  把 SoC 音频驱动拆分为 CPU DAI / Codec / Machine；
- **Hardware Driver**  
  包括 K3 I2S/RI2S 控制器、外部 Codec、板级声卡 Machine。

在 K3 上，用户实际看到的“声卡”并不一定是单独某个 I2S 控制器，而往往是由 `simple-audio-card` 把：

- CPU DAI（如 `i2s1` / `ri2s2` / `ri2s3`）
- Codec DAI（如 `es8326` / `dp0` / `dp1`）

组合成一个完整声卡节点。

### K3 的实际音频方案

从板级 DTS 可以确认，K3 当前至少有下面几类实际方案：

#### 方案一：I2S + ES8326 Codec

典型出现在：

- `k3_dc_board.dts`
- `k3_deb1.dts`

链路大致为：

```text
I2S1 -> ES8326 Codec -> Speaker / Headphone / Mic
```

特点：

- 支持播放和录制；
- 通过 I2C 挂载 ES8326；
- 需要 I2S pinctrl、Codec I2C 节点、`simple-audio-card` 共同配合。

#### 方案二：DP0 音频

典型出现在：

- `k3_evb.dts`
- `k3_gemini_c0.dts`

链路大致为：

```text
RI2S2 -> DP0 -> DP 显示器音频输出
```

特点：

- `simple-audio-card,name = "snd-dp0"`；
- `format = "left_j"`；
- `playback-only`；
- 依赖 DP0 显示输出已启用。

#### 方案三：DP1 音频

典型出现在：

- `k3_com260.dts`
- `k3_com260_kit_v02.dts`
- `k3_deb1.dts`

链路大致为：

```text
RI2S3 -> DP1 -> DP 显示器音频输出
```

特点与 DP0 类似，但 CPU DAI 和 DP 端口不同。

## 源码结构介绍

K3 音频相关代码主要位于：

```text
linux-6.18/sound/soc/spacemit/
├── Kconfig
├── Makefile
├── k1_i2s.c      # A-domain I2S/SSPA 控制器驱动
└── k3_ri2s.c     # R-domain RI2S 控制器驱动
```

另外，板级声卡大多复用标准 ASoC 的 `simple-audio-card` 方式，不再像旧方案那样一定需要一套单独的私有 machine 驱动。

外部 Codec 例如 ES8326 仍由通用 Codec 驱动承担，位于：

```text
sound/soc/codecs/
├── es8326.c
└── es8326.h
```

### 驱动实现差异

K3 实际上同时存在两套音频控制器驱动：

#### `k1_i2s.c`

用于 A-domain I2S/SSPA 控制器，特点包括：

- 支持 DMAengine PCM；
- 支持播放和录制；
- 驱动中可见 `SND_SOC_DAIFMT_I2S`、`DSP_A`、`DSP_B`；
- 采样格式包含 `S16_LE` / `S32_LE`；
- 运行参数中可见最大 2 声道；
- 用于典型的 SoC + 外部 Codec 方案。

#### `k3_ri2s.c`

用于 R-domain RI2S 控制器，特点包括：

- `CONFIG_SND_SOC_K3_RI2S`；
- 也走 DMAengine PCM；
- 支持 `I2S` / `LEFT_J` / `RIGHT_J` / `DSP_A` / `DSP_B`；
- 在常规 I2S / Left-justified / Right-justified 模式下约束为 2 声道；
- 在 DSP A/B 模式下可支持 1~4 声道；
- K3 板级目前主要把它用于 DP 音频。

这点是 K3 相对 K1 一个明显变化：**K3 把 DP 音频链路专门落到了 R-domain RI2S 上。**

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 音频框架 | ALSA + ASoC |
| A-domain I2S | `k1_i2s.c`，用于外接 Codec |
| R-domain RI2S | `k3_ri2s.c`，用于 K3 R-domain 音频场景 |
| 声卡组织 | 主要通过 `simple-audio-card` 建模 |
| 外部 Codec | 支持 ES8326 |
| DP 音频 | 支持 `dp0` / `dp1` 音频输出 |
| 录音能力 | I2S + ES8326 方案支持录制 |
| DP 音频方向 | `playback-only` |
| 支持格式 | I2S / LEFT_J / RIGHT_J / DSP_A / DSP_B（取决于控制器） |
| 支持采样格式 | S16_LE / S32_LE |

### K3 相比 K1 值得单独强调的点

1. **新增 RI2S 路径**  
   K3 `Kconfig` 中新增了：

   - `SND_SOC_K3_RI2S`

   对应 DTS 节点：

   - `ri2s0`
   - `ri2s1`
   - `ri2s2`
   - `ri2s3`

2. **DP 音频已成为真实板级方案**  
   而且不是抽象支持，是多个板子都在用：

   - EVB / Gemini：`sound_card_dp0`
   - COM260 / DEB1：`sound_card_dp1`

3. **声卡建模以 simple-audio-card 为主**  
   K3 当前板级 DTS 里，很多声卡都是标准 `simple-audio-card` 节点，不需要引入单独私有 machine 驱动说明。

4. **板级差异更明显**  
   不同板子选用不同：

   - CPU DAI：`i2s1` / `ri2s2` / `ri2s3`
   - Codec DAI：`es8326` / `dp0` / `dp1`
   - pinctrl：`sspa1_1_cfg` / 各种 DP HPD pinctrl

## 配置介绍

主要包括：

- ALSA / ASoC 基础支持；
- K3 I2S / RI2S 控制器支持；
- 板级 Codec / simple-audio-card DTS 配置；
- DP 音频依赖的显示侧配置。

### CONFIG 配置

#### 基础音频功能支持

```text
Device Drivers
    Sound card support (SOUND [=y])
        Advanced Linux Sound Architecture (SND [=y])
            ALSA for SoC audio support (SND_SOC [=y])
```

#### K3 音频控制器支持

`drivers/sound/soc/spacemit/Kconfig` 中可确认：

- `CONFIG_SND_SOC_K1_I2S`
- `CONFIG_SND_SOC_K3_RI2S`

菜单大致如下：

```text
Device Drivers
    Sound card support
        Advanced Linux Sound Architecture
            ALSA for SoC audio support
                SpacemiT
                    K1 I2S Device Driver (SND_SOC_K1_I2S)
                    K3 RI2S Device Driver (SND_SOC_K3_RI2S)
```

其中：

- `SND_SOC_K1_I2S`：用于 K3 A-domain I2S 控制器；
- `SND_SOC_K3_RI2S`：用于 K3 R-domain RI2S 控制器。

#### 外部 Codec 支持

如果板级使用 ES8326，需要打开：

```text
Device Drivers
    Sound card support
        Advanced Linux Sound Architecture
            ALSA for SoC audio support
                CODEC drivers
                    Everest Semi ES8326 CODEC (SND_SOC_ES8326)
```

## DTS 配置

### 1. A-domain I2S 控制器

K3 的板级 I2S 方案中，常用的是 `i2s1`。例如在 `k3_dc_board.dts` / `k3_deb1.dts` 中：

```dts
&i2s1 {
        status = "okay";
        pinctrl-0 = <&sspa1_1_cfg>;
        pinctrl-names = "default";
};
```

这说明用户至少需要配置：

- 控制器节点 `status = "okay"`；
- 选对 pinctrl；
- 板级再通过 `simple-audio-card` 把它和 Codec 连起来。

### 2. I2S pinctrl 配置

K3 `k3-pinctrl.dtsi` 中定义了多组 `sspa` 引脚，例如：

- `sspa0_0_cfg`
- `sspa0_1_cfg`
- `sspa1_0_cfg`
- `sspa1_1_cfg`
- `sspa2_0_cfg`
- `sspa3_0_cfg`
- `sspa4_0_cfg`
- `sspa5_0_cfg`

例如：

```dts
sspa1_1_cfg: sspa1-1-cfg {
        sspa1-0-pins {
                pinmux = <K3_PADCONF(122, 2)>, /* sspa1 clk */
                         <K3_PADCONF(123, 2)>, /* sspa1 frm */
                         <K3_PADCONF(124, 2)>, /* sspa1 tx */
                         <K3_PADCONF(125, 2)>, /* sspa1 rx */
                         <K3_PADCONF(126, 2)>; /* sspa1 sysclk */
        };
};
```

和前面的 Display / PCIe 一样，K3 的 pinmux 方案很多，迁移 DTS 时必须按原理图选正确那一组，不能机械复用别的板子的配置。

### 3. ES8326 Codec 配置

以 `k3_dc_board.dts` 为例：

```dts
es8326: es8326@19 {
        compatible = "everest,es8326";
        reg = <0x19>;
        #sound-dai-cells = <0>;
        interrupt-parent = <&gpio>;
        interrupts = <3 31 IRQ_TYPE_EDGE_RISING>;
        spk-ctl-gpio = <&gpio 3 23 0>;
        everest,mic1-src = [44];
        everest,mic2-src = [66];
        everest,jack-detect-inverted;
        status = "okay";
};
```

这说明板级 Codec 还依赖：

- I2C 总线；
- 耳机插拔检测中断；
- 扬声器 GPIO；
- 麦克风源配置；
- jack detect 极性配置。

在 `k3_deb1.dts` 中，`spk-ctl-gpio` 被注释掉了，这也说明 **不同板型的板级音频外围并不完全相同**。

### 4. I2S + Codec 声卡配置

K3 典型 I2S + ES8326 声卡配置如下：

```dts
&sound_card1 {
        status = "okay";
        simple-audio-card,mclk-fs = <256>;
        simple-audio-card,dai-link@0 {
                format = "i2s";
                frame-master = <&link1_cpu>;
                bitclock-master = <&link1_cpu>;

                link1_cpu: cpu {
                        sound-dai = <&i2s1>;
                };

                link1_codec: codec {
                        sound-dai = <&es8326>;
                };
        };
};
```

几个关键点：

- `format = "i2s"`；
- `mclk-fs = <256>`；
- CPU 端用 `i2s1`；
- Codec 端用 `es8326`。

这种方案对应的是完整本地音频声卡，用户可用于：

- 扬声器播放；
- 耳机输出；
- 麦克风录音。

### 5. R-domain RI2S 控制器

K3 `k3-rdomain.dtsi` 中定义了：

- `ri2s0`
- `ri2s1`
- `ri2s2`
- `ri2s3`

例如：

```dts
ri2s2: ri2s2@c0883900 {
        compatible = "spacemit,k3-ri2s";
        #sound-dai-cells = <0>;
        status = "disabled";
};
```

这类节点在当前板级主要用于 DP 音频，不是传统外接模拟 Codec 方案。

### 6. DP0 音频声卡配置

K3 EVB / Gemini 上典型配置如下：

```dts
&ri2s2 {
        status = "okay";
};

&sound_card_dp0 {
        status = "okay";
        simple-audio-card,name = "snd-dp0";
        simple-audio-card,mclk-fs = <512>;
        simple-audio-card,dai-link@0 {
                format = "left_j";
                frame-master = <&dp0_link_cpu>;
                bitclock-master = <&dp0_link_cpu>;
                dp0_link_codec: codec {
                        playback-only;
                        sound-dai = <&dp0>;
                };
        };
};
```

几个关键点：

- CPU 端是 `ri2s2`；
- Codec 端并不是传统音频 Codec，而是 `dp0`；
- `playback-only` 明确表示只支持播放；
- `format = "left_j"`。

### 7. DP1 音频声卡配置

K3 COM260 / DEB1 上常见：

```dts
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

与 DP0 类似，只是切换到了：

- `ri2s3`
- `dp1`
- `snd-dp1`

### 8. DP 音频依赖关系

DP 音频并不是单独一块能工作的功能，它依赖显示链路本身已正常起来。因此要启用 DP 音频，通常还要保证：

- `dp0` / `dp1` 显示输出已启用；
- 对应 HPD pinctrl 正确；
- 外接 DP 显示器已正常枚举；
- DRM / DP 显示侧已经工作正常。

也就是说，K3 上 **Display 和 Audio 在 DP 场景里是联动的**。

## 接口与使用方法

### 1. 查看声卡

```bash
cat /proc/asound/cards
aplay -l
arecord -l
```

通过这些命令可以确认：

- 系统枚举出了哪些声卡；
- 是 `snd-es8326`、`snd-dp0` 还是 `snd-dp1`；
- 哪些设备支持播放，哪些支持录制。

### 2. 播放测试

对于 I2S + ES8326 或 DP 音频，可用：

```bash
aplay -D hw:0,0 test.wav
```

或者：

```bash
speaker-test -D hw:0,0 -c 2 -r 48000 -F S16_LE -t sine
```

### 3. 录音测试

若板级为 ES8326 方案，可测试：

```bash
arecord -D hw:0,0 -f S16_LE -r 48000 -c 2 -d 5 test.wav
```

如果是 DP 音频方案，一般不支持录音，因为 DTS 已标明 `playback-only`。

### 4. Mixer / 控件查看

```bash
amixer controls
amixer contents
alsamixer
```

适合查看：

- Codec 控件是否存在；
- 耳机/扬声器通路是否打开；
- Capture 开关是否打开。

## Debug 介绍

### 1. 先确认声卡是否注册

```bash
dmesg | grep -Ei "asoc|snd|i2s|ri2s|es8326|dp0|dp1"
cat /proc/asound/cards
```

重点看：

- ASoC card 是否注册成功；
- CPU DAI / Codec DAI 是否绑定成功；
- 是否出现 deferred probe 或 clock/reset 相关错误。

### 2. I2S + ES8326 不出声时怎么查

优先顺序：

1. `i2s1` 是否启用；
2. `sspa1_1_cfg` 是否正确；
3. `es8326@19` 是否探测成功；
4. `sound_card1` 是否注册成功；
5. `amixer` 里通路是否打开；
6. 耳机插拔中断和 `spk-ctl-gpio` 是否正常。

### 3. DP 音频不出声时怎么查

优先顺序：

1. 先确认 DP 显示本身已经正常；
2. 再确认 `ri2s2/ri2s3` 是否启用；
3. 再确认 `sound_card_dp0` / `sound_card_dp1` 是否注册成功；
4. 检查 `aplay -l` 是否能看到 `snd-dp0` / `snd-dp1`；
5. 检查外接显示器是否真的支持音频。

### 4. pinctrl 迁移问题

K3 的 `sspa` / `dp hpd` pin 组很多，迁移 DTS 时非常容易把 pinctrl 用错。若出现：

- Codec 正常探测但无 BCLK/LRCLK；
- DP 显示正常但音频链路不稳定；

建议回到：

- `k3-pinctrl.dtsi`
- 板级原理图
- 实际波形

逐项核对。

## FAQ

### 1. K3 的 Audio 为什么不能只沿用 K1 文档？

因为 K3 实际已经多了：

- R-domain `ri2s0~ri2s3`；
- DP0 / DP1 音频声卡；
- 多块板子真实使用 `simple-audio-card` + DP codec 模型；
- 不同板型间音频链路差异明显。

### 2. K3 的 DP 音频为什么写在 Audio 文档里，而不是只写在 Display？

因为从 ASoC 角度看，DP0/DP1 在板级 DTS 里已经被当作 `sound-dai` 参与声卡建模，用户最终也是通过 ALSA 设备来播放音频，所以它属于 Display 与 Audio 的交叉功能，但必须在 Audio 文档中专门说明。

### 3. 为什么 `sound_card_dp0` / `sound_card_dp1` 里用的是 `left_j`？

因为这是当前 K3 板级 DTS 的真实配置。文档应以实际 DTS 为准，而不是按通用习惯默认写成 `i2s`。

### 4. 为什么 DP 音频通常没有录音？

因为板级 DTS 里对 DP codec 侧已经使用了 `playback-only`，说明当前设计只用于向外部显示器输出音频，不用于采集。
