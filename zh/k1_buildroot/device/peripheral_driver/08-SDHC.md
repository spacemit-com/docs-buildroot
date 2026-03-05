# SDHC

介绍SDHC的功能和使用方法。

## 模块介绍

SDHC 是多媒体卡（MMC）、安全数字卡（SD）和安全数字输入输出（SDIO）模块的控制器。

### 功能介绍

![](static/MMC.png)

MMC 框架图可以分为以下几个层次：  
- **MMC Host：** 这是MMC控制器驱动层，负责初始化 MMC 控制器以及底层的数据收发操作，直接控制底层寄存器。  
- **MMC Core：** 这是核心层，负责抽象出虚拟的 card 设备，并提供接口供上层使用。  
- **MMC Block：** 这是块设备层，负责实现块设备驱动程序，对接内核其他框架（如块设备、TTY、wifi等）。  
这些层次结构共同构成了 Linux 系统中 MMC 子系统的完整框架，确保了 MMC 设备在系统中的正常运行和数据传输。

### 源码结构介绍

控制器驱动代码位于 `drivers/mmc/host` 目录下：

```
drivers/mmc/host
|-- sdhci.c          #sdhci标准代码
|-- sdhci-pltfm.c               #sdhci平台层
|-- sdhci-of-k1x.c  #k1 sdhci驱动
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持eMMC5.1 | 支持eMMC5.1协议，包括HS400,HS200 |
| 支持sd3.0 | 支持sd3.0协议的卡，兼容sd2.0协议 |
| 支持DMA | 支持DMA数据传输 |

### 性能参数

| eMMC型号 | 顺序读(MB/s) | 顺序写(MB/s) | 随机读(MB/s) | 随机写(MB/s) |
| :-----| :----| :----: | :----: |:----: |
| KLMAG1JETD-B041 | 295 | 53.3 | 65.4 | 45.2 |
| FEMDME008G-A8A39 | 304 | 107 | 32.3 | 44 |

测试方法

```
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/1
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/1
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=512k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/1
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=512k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/1
```

***默认配置HS400 200M***

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_MMC 为MMC总线协议提供支持，默认值为 `Y`

```
Device Drivers
        MMC/SD/SDIO card support (MMC [=y])     
```

CONFIG_MMC_BLOCK为安装文件系统的MMC块设备驱动提供支持，默认值为 `Y`

```
Device Drivers
 MMC/SD/SDIO card support (MMC [=y])
     HW reset support for eMMC (PWRSEQ_EMMC [=y])
        Simple HW reset support for MMC (PWRSEQ_SIMPLE [=y])
     MMC block device driver (MMC_BLOCK [=y])
```

CONFIG_MMC_SDHCI 为MMC控制器驱动提供支持，默认情况下，此选型为Y

```
Device Drivers
 MMC/SD/SDIO card support (MMC [=y])
     Secure Digital Host Controller Interface support (MMC_SDHCI [=y])
         SDHCI platform and OF driver helper (MMC_SDHCI_PLTFM [=y])
       SDHCI OF support for the Spacemit K1X SDHCI controllers (MMC_SDHCI_OF_K1X [=y])
```

### dts配置

#### pinctrl

SDHC 一共有三个 slots：
- slot1 支持 SD/SDIO (1/4 bit)
- slot2 支持 SDIO/eMMC (1/4 bit)
- slot3 只支持 eMMC (1/4/8 bit)

方案上一般 slot1 用于 SD，slot2 用于 SDIO，slot3 用于 eMMC。

SD 和 SDIO 都需要配置卡的信号线对应的 pinctl 为 mode0 模式，分别对应 pinctrl_mmc1 和 pinctrl_mmc2。

mmc1 的 pinctl 还有一个 fast 模式，在时钟高于 100M 时需要切换到 pinctrl_mmc1_fast 模式。

```c
    pinctrl_mmc1: mmc1_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d3 */
            K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d2 */
            K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d1 */
            K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d0 */
            K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_cmd */
            K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_3V_DS4))       /* mmc1_clk */
        >;
    };

    pinctrl_mmc1_fast: mmc1_fast_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d3 */
            K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d2 */
            K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d1 */
            K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d0 */
            K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_cmd */
            K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS3))       /* mmc1_clk */
        >;
    };
```

#### gpio

SD 的检测是通过 GPIO 完成的，需要按实际原理图来配置卡检测的 GPIO。

```c
&sdhci0 {
        cd-gpios = <&gpio 80 0>;
        cd-inverted;
};
```

比如方案使用 gpio80 来做卡的检测，还需要配置 gpio80 的 pintcl 功能。

```c
&pinctrl {
        pinctrl-single,gpio-range = <
                &range GPIO_80  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
        >;
};

&gpio{
        gpio-ranges = <
                &pinctrl 80  GPIO_80  4
        >;
};
```

#### 电源配置

SD 和 SDIO 需要配置两个电源，分别是 **vmmc-supply** 和 **vqmmc-supply**，分别对应 **卡的功能** 和 **IO 供电**，vqmmc-supply 会根据卡的运行模式动态切换电源，硬件设计上需要确保能支持 3.3V 和 1.8V。

eMMC 设计上会保证供电，不需要配置电源。

```c
&sdhci0 {
        vmmc-supply = <&dcdc_4>;
        vqmmc-supply = <&ldo_1>;
};
```

#### tuning 配置

在 SD 高速模式下需要进行调试（Tuning），不同硬件版本需适配 TX 和 RX 的相关参数。

#### dts 配置示例

SD 的完整方案配置如下：

```c
&sdhci0 {
        pinctrl-names = "default","fast";
        pinctrl-0 = <&pinctrl_mmc1>;
        pinctrl-1 = <&pinctrl_mmc1_fast>;
        bus-width = <4>;
        cd-gpios = <&gpio 80 0>;
        cd-inverted;
        vmmc-supply = <&dcdc_4>;
        vqmmc-supply = <&ldo_1>;
        no-mmc;
        no-sdio;
        spacemit,sdh-host-caps-disable = <(
                        MMC_CAP_UHS_SDR12 |
                        MMC_CAP_UHS_SDR25
                        )>;
        spacemit,sdh-quirks = <(
                        SDHCI_QUIRK_BROKEN_CARD_DETECTION |
                        SDHCI_QUIRK_INVERTED_WRITE_PROTECT |
                        SDHCI_QUIRK_BROKEN_TIMEOUT_VAL
                        )>;
        spacemit,sdh-quirks2 = <(
                        SDHCI_QUIRK2_PRESET_VALUE_BROKEN |
                        SDHCI_QUIRK2_BROKEN_PHY_MODULE |
                        SDHCI_QUIRK2_SET_AIB_MMC
                        )>;
        spacemit,aib_mmc1_io_reg = <0xD401E81C>;
        spacemit,apbc_asfar_reg = <0xD4015050>;
        spacemit,apbc_assar_reg = <0xD4015054>;
        spacemit,rx_dline_reg = <0x0>;
        spacemit,tx_dline_reg = <0x0>;
        spacemit,tx_delaycode = <0xA0>;
        spacemit,rx_tuning_limit = <50>;
        spacemit,sdh-freq = <204800000>;
        status = "okay";
};
```

eMMC 的完整方案配置如下：

```c
/* eMMC */
&sdhci2 {
        bus-width = <8>;
        non-removable;
        mmc-hs400-1_8v;
        mmc-hs400-enhanced-strobe;
        no-sd;
        no-sdio;
        spacemit,sdh-quirks = <(
                        SDHCI_QUIRK_BROKEN_CARD_DETECTION |
                        SDHCI_QUIRK_BROKEN_TIMEOUT_VAL
                        )>;
        spacemit,sdh-quirks2 = <(
                        SDHCI_QUIRK2_PRESET_VALUE_BROKEN
                        )>;
        spacemit,sdh-freq = <375000000>;
        status = "okay";
};
```

## 接口介绍

### API介绍

Linux操作系统包括一个实施 MMC 总线协议的 MMC 总线驱动、MMC 块驱动处理文件系统读/写调用，并使用 MMC 主控制器接口驱动向 uSDHC 发送命令。
K1 MMC 控制器驱动实现了init，exit，request，resume，suspend 和 set_ios 的接口，主要有：

- init 函数 sdhci_pltfm_init() 初始化平台硬件并注册 sdhci_k1x_pdata 结构
- exit 函数 spacemit_sdhci_remove() 取消平台硬件初始化，并释放分配的存储器

### Debug介绍

#### sysfs

```
sd_card_pmux
该节点用于切换 SD 卡的 pin 为 jtag 功能，0 表示 SD 卡功能，1 表示 jtag 功能。
tx_delaycode
tx_delaycode的值默认是在方案 dts 中指定，可以通过 sysfs 下的该节点进行动态修改，方便调试阶段的验证。
```

#### debugfs

```
常用于查询 MMC 的工作状态，包括频率，位宽，模式等信息。
cat /sys/kernel/debug/mmc0/ios
clock:          204800000 Hz
actual clock:   204800000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    6 (sd uhs SDR104)
signal voltage: 1 (1.80 V)
driver type:    0 (driver type B)
```

## 测试介绍

MMC/SD等存储可以通过第三方工具完成性能和功能测试，例如：FIO，bonnie++，目前 buildroot 上已集成 FIO 工具。可使用 FIO 工具进行读写性能评估与老化压力测试。

## FAQ
