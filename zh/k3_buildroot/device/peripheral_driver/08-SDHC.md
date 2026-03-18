# SDHC

介绍 K3 平台 SDHC 的功能、设备树配置方法以及调试方式。

## 模块介绍

SDHC 是 MMC/SD/SDIO 设备的主控制器。K3 平台的 SDHC 文档主要面向三类典型场景：

- **SD 卡**：可插拔存储卡；
- **SDIO**：常用于 Wi‑Fi 等板载外设；
- **eMMC**：板载固定存储，通常用于系统启动和根文件系统。

从用户角度看，最重要的不是控制器寄存器细节，而是：

- K3 上有几个 SDHC 控制器、分别适合什么用途；
- `sdcard` / `sdio` / `emmc` 节点怎么写；
- `bus-width`、`non-removable`、`cd-gpios`、`vmmc-supply`、`vqmmc-supply` 这些属性是什么意思；
- eMMC HS400、SD 卡 UHS、电压切换这些能力怎么体现；
- Linux 下怎么看设备有没有识别成功、工作在哪个模式。

### 功能介绍

Linux MMC 框架大致分为三层：

- **MMC Host**：主控制器驱动层，负责控制器初始化、命令发送、数据收发、电压切换和调谐；
- **MMC Core**：MMC 核心层，统一管理卡枚举、协议处理和状态机；
- **MMC Block / SDIO 功能层**：为块设备或 SDIO 外设驱动提供上层接口。

K3 上，用户通常能看到这几类结果：

- eMMC 枚举后形成 `/dev/mmcblkX`；
- SD 卡插入后形成 `/dev/mmcblkX`；
- SDIO 外设不会形成块设备，而是进一步枚举出 Wi‑Fi 等功能驱动。

### 源码结构介绍

控制器相关代码主要位于：

```text
linux-6.18/
|-- drivers/mmc/host/
|   |-- sdhci.c                # SDHCI 通用框架
|   |-- sdhci-pltfm.c          # 平台层辅助代码
|   `-- sdhci-of-k1.c          # SpacemiT K1/K3 SDHCI 驱动
|-- Documentation/devicetree/bindings/mmc/
|   `-- spacemit,sdhci.yaml    # SpacemiT SDHCI binding
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                # K3 控制器节点定义
    `-- k3*.dts                # 板级用法示例
```

K3 控制器节点在 DTS 中使用：

```dts
compatible = "spacemit,k3-sdhci";
```

而驱动 `drivers/mmc/host/sdhci-of-k1.c` 同时兼容：

- `spacemit,k1-sdhci`
- `spacemit,k3-sdhci`

这意味着 K3 继续沿用了同一套 SDHCI 驱动框架，但文档应以 K3 当前 DTS、binding 和驱动实际行为为准。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 控制器数量 | K3 提供 3 个主控制器：`sdcard`、`sdio`、`emmc` |
| 支持设备类型 | 支持 SD、SDIO、eMMC |
| eMMC 高速模式 | 支持 `mmc-hs400-1_8v` 和 `mmc-hs400-enhanced-strobe` |
| SD 卡电压切换 | 支持 UHS 电压切换，驱动会配合 pinctrl 切换到 `uhs` 状态 |
| SDIO 场景 | 支持 `mmc-pwrseq`、`keep-power-in-suspend` |
| 软件调谐 | 驱动支持 tuning，并允许通过 DTS / sysfs 调整 `tx_delaycode` |
| 调试接口 | 对 SD/SDIO 控制器暴露 `tx_delaycode` sysfs 调试节点 |

### K3 的三个 slot 用法

结合 `k3.dtsi` 和板级 DTS，可把 K3 的三个控制器理解为：

- **`sdcard`**：通常接外部 SD 卡槽；
- **`sdio`**：通常接板载 SDIO 设备，例如 Wi‑Fi 模组；
- **`emmc`**：通常接板载 eMMC。

这和 K1 文档中的整体思路是一致的，但 K3 命名更直接，用户更容易从 DTS 上看懂控制器用途。

## 配置介绍

主要包括驱动使能配置和 DTS 配置。

### CONFIG 配置

K3 SDHC 常见配置如下：

`CONFIG_MMC` 为 MMC 总线协议提供支持，通常为 `Y`：

```text
Device Drivers
    MMC/SD/SDIO card support (MMC [=y])
```

`CONFIG_MMC_BLOCK` 为 MMC 块设备提供支持，通常为 `Y`：

```text
Device Drivers
    MMC/SD/SDIO card support (MMC [=y])
        MMC block device driver (MMC_BLOCK [=y])
```

`CONFIG_MMC_SDHCI` / `CONFIG_MMC_SDHCI_PLTFM` / `CONFIG_MMC_SDHCI_OF_K1` 为 SpacemiT SDHCI 控制器提供支持：

```text
Device Drivers
    MMC/SD/SDIO card support (MMC [=y])
        Secure Digital Host Controller Interface support (MMC_SDHCI [=y])
            SDHCI platform and OF driver helper (MMC_SDHCI_PLTFM [=y])
            SDHCI OF support for the SpacemiT SDHCI controller (MMC_SDHCI_OF_K1 [=y])
```

> 注意：Kconfig 名称仍然是 `MMC_SDHCI_OF_K1`，但该驱动已经支持 `spacemit,k3-sdhci`。

### DTS 配置

#### 控制器节点

K3 的三个控制器节点位于 `k3.dtsi`：

```dts
sdcard: mmc@d4280000 {
        compatible = "spacemit,k3-sdhci";
        reg = <0x0 0xd4280000 0x0 0x200>;
        clocks = <&syscon_apmu CLK_APMU_SDH_AXI>,
                 <&syscon_apmu CLK_APMU_SDH0>;
        clock-names = "core", "io";
        interrupts = <99 IRQ_TYPE_LEVEL_HIGH>;
        resets = <&syscon_apmu RESET_APMU_SDH_AXI>,
                 <&syscon_apmu RESET_APMU_SDH0>;
        reset-names = "sdh_axi", "sdh0";
        status = "disabled";
};
```

`sdio` 与 `emmc` 节点写法类似，只是寄存器地址、中断号和时钟复位实例不同。

根据 `Documentation/devicetree/bindings/mmc/spacemit,sdhci.yaml`，用户至少需要理解这些基础资源：

- `compatible`
- `reg`
- `interrupts`
- `clocks`
- `clock-names`

K3 的实际 DTS 还增加了：

- `resets`
- `reset-names`

这部分是 K3 平台实际启用控制器所必须的资源描述。

#### pinctrl 配置

K3 板级中，SD 卡控制器通常会配置三组 pinctrl：

- `default`
- `uhs`
- `debug`

例如：

```dts
&sdcard {
        pinctrl-names = "default","uhs","debug";
        pinctrl-0 = <&mmc1_cfg>;
        pinctrl-1 = <&mmc1_uhs_cfg>;
        pinctrl-2 = <&mmc1_debug_cfg>;
        status = "okay";
};
```

其中：

- `default`：普通工作模式；
- `uhs`：在 1.8V / UHS 场景下由驱动切换使用；
- `debug`：用于调试或特殊板级场景。

驱动 `sdhci-of-k1.c` 中也能看到它会显式获取：

- `pins_default`
- `pins_uhs`
- `pins_debug`

并在电压切换时对 `pins_uhs` 做切换。

#### 电源配置

对 SD 卡和 SDIO 来说，通常需要配置：

- `vmmc-supply`：卡本体供电；
- `vqmmc-supply`：IO 供电，常参与 3.3V / 1.8V 切换。

例如：

```dts
&sdcard {
        vmmc-supply = <&p3v3>;
        vqmmc-supply = <&aldo1>;
};
```

对 eMMC，板级设计通常会保证供电稳定，实际是否在 DTS 中显式补充 regulator，要看板级电源设计；但从用户角度，最重要的是理解：

- **SD / SDIO 更依赖动态 IO 电压配置**；
- **eMMC 更关注总线位宽和高速模式配置**。

#### 卡检测配置

如果是外部 SD 卡槽，通常需要卡检测 GPIO，例如：

```dts
&sdcard {
        cd-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
};
```

K3 `k3_evb.dts` 中同时还配置了：

```dts
wp-inverted;
```

这说明该板级还对写保护相关极性做了约定。对用户来说，这类 GPIO 极性必须按原理图和插槽设计来配，配错了会表现为：

- 卡一直“插着”；
- 或一直“没插卡”；
- 或写保护状态异常。

#### SD 卡配置示例

K3 EVB 上的 `sdcard` 典型写法如下：

```dts
&sdcard {
        pinctrl-names = "default","uhs","debug";
        pinctrl-0 = <&mmc1_cfg>;
        pinctrl-1 = <&mmc1_uhs_cfg>;
        pinctrl-2 = <&mmc1_debug_cfg>;
        bus-width = <4>;
        wp-inverted;
        cd-gpios = <&gpio 2 22 GPIO_ACTIVE_LOW>;
        vmmc-supply = <&p3v3>;
        vqmmc-supply = <&aldo1>;
        no-mmc;
        no-sdio;
        clock-frequency = <204800000>;
        spacemit,tx_delaycode = <0x1f>;
        status = "okay";
};
```

其中关键点是：

- `bus-width = <4>`：SD 卡通常为 4-bit；
- `no-mmc` / `no-sdio`：明确这个 slot 只用于 SD 卡；
- `clock-frequency`：指定目标时钟；
- `spacemit,tx_delaycode`：指定调谐相关发送延时参数。

#### SDIO 配置示例

K3 EVB 中的 `sdio` 写法如下：

```dts
&sdio {
        pinctrl-names = "default";
        pinctrl-0 = <&mmc2_cfg>;
        bus-width = <4>;
        non-removable;
        vmmc-supply = <&vmmc_sdio>;
        vqmmc-supply = <&p1v8>;
        mmc-pwrseq = <&sdio_pwrseq>;
        no-mmc;
        no-sd;
        keep-power-in-suspend;
        clock-frequency = <375000000>;
        status = "disabled";
};
```

这类配置通常用于板载 SDIO Wi‑Fi，关键点是：

- `non-removable`：板载设备不可插拔；
- `mmc-pwrseq`：由电源时序节点控制上电/复位；
- `keep-power-in-suspend`：休眠时保持供电，常见于 Wi‑Fi 场景；
- `no-mmc` / `no-sd`：明确这个 slot 只用于 SDIO。

#### eMMC 配置示例

K3 EVB 上的 `emmc` 节点：

```dts
&emmc {
        bus-width = <8>;
        non-removable;
        mmc-hs400-1_8v;
        mmc-hs400-enhanced-strobe;
        no-sd;
        no-sdio;
        clock-frequency = <375000000>;
        status = "okay";
};
```

关键点如下：

- `bus-width = <8>`：eMMC 通常使用 8-bit；
- `non-removable`：板载固定器件；
- `mmc-hs400-1_8v`：启用 HS400 1.8V 模式；
- `mmc-hs400-enhanced-strobe`：启用 enhanced strobe；
- `no-sd` / `no-sdio`：明确该 slot 用于 eMMC。

### tuning 与时序相关配置

K3 驱动中保留了调谐相关逻辑，并支持：

- `spacemit,tx_delaycode`
- 软件 tuning
- HS400 / HS200 相关准备与切换流程
- enhanced strobe 控制

从驱动看，`spacemit,tx_delaycode` 若 DTS 未指定，会回落到默认值；而对 SD/SDIO 类型控制器，驱动还会创建 sysfs 节点，方便在线调整。

这意味着在用户调试高速 SD 卡或 SDIO 模组时，如果出现：

- 高速模式不稳定；
- 读写偶发报错；
- 某些板卡批次差异导致 timing margin 变化；

就可以围绕 `tx_delaycode` 做进一步验证。

## 接口介绍

### API 介绍

Linux 操作系统中，MMC/SD/SDIO 设备统一通过 MMC 子系统访问。K3 的 `sdhci-of-k1.c` 驱动在标准 SDHCI 基础上补充了以下平台能力：

- reset 控制；
- pinctrl 状态切换；
- 电压切换辅助；
- HS400 / HS200 相关流程；
- tuning 与延时参数设置；
- SDIO 重新扫描辅助接口 `spacemit_sdio_detect_change()`。

其中 `spacemit_sdio_detect_change()` 被导出，说明 K3 上有些场景可能需要触发 SDIO 设备重新扫描。

### Debug 介绍

#### sysfs

K3 驱动对 SD / SDIO 控制器会创建：

```text
tx_delaycode
```

该节点可用于查看和修改当前的 `tx_delaycode`，方便调试高速模式问题。

例如：

```bash
cat /sys/devices/platform/*.mmc/tx_delaycode 2>/dev/null
```

#### debugfs

MMC 子系统仍然支持通过 debugfs 查看 host 当前工作状态，例如：

```bash
cat /sys/kernel/debug/mmc0/ios
```

常可用于查看：

- 当前时钟；
- 实际工作频率；
- 总线位宽；
- 时序模式；
- 信号电压。

## 测试介绍

MMC/SD 等存储可以通过标准 Linux 工具完成功能和性能测试，例如：

- `fio`
- `dd`
- `hdparm`（针对块设备场景，谨慎使用）

### 识别与枚举检查

```bash
dmesg | grep -Ei "mmc|sdhci|sdio"
lsblk
cat /proc/partitions
```

### 查看当前 host 状态

```bash
cat /sys/kernel/debug/mmc0/ios
```

示例输出通常包括：

- `clock`
- `actual clock`
- `bus width`
- `timing spec`
- `signal voltage`

### eMMC / SD 卡性能测试

可参考如下 `fio` 方法：

```bash
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/testfile
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/testfile
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=512k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/testfile
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=512k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/testfile
```

### SDIO 设备检查

如果 `sdio` 用于 Wi‑Fi，则应进一步查看：

```bash
dmesg | grep -Ei "sdio|wifi|wlan"
ip link
```

此时是否出现块设备并不重要，关键是 SDIO 功能驱动是否成功枚举。

## FAQ

### 1. 为什么 `sdcard` 控制器起来了，但插卡后没有块设备？

常见原因包括：

- `cd-gpios` 极性错误；
- pinctrl 配置不对；
- `vmmc-supply` / `vqmmc-supply` 没有真正打开；
- 该 slot 被错误写成了 `no-mmc` / `no-sd` / `no-sdio` 组合；
- 卡本身或插槽硬件异常。

### 2. 为什么 eMMC 能识别，但速度不对或者高速模式不稳定？

重点检查：

- `mmc-hs400-1_8v` / `mmc-hs400-enhanced-strobe` 是否正确；
- 板级时钟与供电是否满足；
- 是否需要进一步调试 timing；
- 驱动日志里是否有 tuning 失败或 CRC 错误。

### 3. 为什么 SDIO 设备没起来？

常见排查点：

- `non-removable` 是否配置；
- `mmc-pwrseq` 是否正确控制模组上电/复位；
- `keep-power-in-suspend` 是否符合模组要求；
- 对应的 Wi‑Fi/SDIO 功能驱动和固件是否存在。
