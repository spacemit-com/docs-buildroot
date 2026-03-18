# WIFI

介绍 K3 平台 WIFI 模组的常见接入方式、当前 SDK 中可见的软件栈位置、DTS 配置重点以及用户态基本使用方法。

## 模块介绍

K3 平台当前仍然主要通过**外部 WiFi 模组**实现无线连接能力，而不是 SoC 内建一套独立 WLAN MAC/PHY。

从当前 SDK 和 DTS 的情况看，K3 上最稳、最清晰的一条主线仍然是：

- **SDIO 接口挂载外部 WiFi 模组**
- 通过 `sdio` 控制器承载数据传输
- 通过 `regulator-fixed` / `mmc-pwrseq-simple` 做基础上电与复位控制
- 再由具体 WiFi 厂商驱动完成协议与射频功能

这意味着 K3 这篇 WIFI 文档不能像某些 SoC 一样写成“单一平台 WiFi 驱动说明”，而应该按三层来理解：

1. **接口层**：SDIO / PCIe / USB
2. **模组驱动层**：厂商提供或内核已有的 WiFi 驱动
3. **板级配电与复位层**：regulator / pwrseq / pinctrl / wakeup

## 功能介绍

WiFi 在 Linux 里的基本软件层次可以概括为：

![](static/wlan.png)

1. **接口控制器**  
   例如 SDIO host controller，负责总线传输；
2. **模组驱动**  
   负责与具体 WiFi 芯片/模组交互；
3. **cfg80211 / mac80211 / nl80211**  
   提供 Linux 无线协议栈与用户态控制接口；
4. **板级控制**  
   包括供电、复位、唤醒 GPIO、keep-power-in-suspend 等。

### K3 当前更值得写哪条主线

从当前 `k3_evb.dts` 来看，K3 已明确给出了一个比较典型的 **SDIO WiFi 板级接法**：

- `vmmc_sdio: regulator-vmmc-sdio`
- `sdio_pwrseq: sdio-pwrseq`
- `&sdio { ... mmc-pwrseq = <&sdio_pwrseq>; ... }`

所以当前文档最适合围绕：

- **SDIO WiFi bring-up**
- **供电 / reset / mmc-pwrseq**
- **SDIO host 的 DTS 约束**
- **用户态连接与调试**

来展开。

## 源码结构介绍

WIFI 相关代码一般分成三块：

```text
linux-6.18/
|-- drivers/net/wireless/          # 具体 WiFi 驱动（厂商/主线）
|-- drivers/mmc/                   # SDIO / MMC host 控制器
|-- drivers/regulator/             # 模组供电控制
|-- drivers/mmc/core/pwrseq*       # mmc-pwrseq 通用上电/复位逻辑
`-- arch/riscv/boot/dts/spacemit/  # 板级 DTS 配置
```

### 当前 K3 里能确认下来的几件事

#### 1. 无线驱动主要仍在通用内核目录

当前 SDK 的 `drivers/net/wireless/` 下能看到大量通用/厂商无线驱动目录，例如：

- `realtek/`
- `rtw88/`
- `rtw89/`
- `ti/`
- `silabs/`
- `ath/`
- `broadcom/`
- 等

这说明 K3 当前并没有表现出“唯一指定某一个平台私有 WiFi 驱动”的形态，而更像是：

- **平台提供接口与板级接法**
- **具体模组走各自驱动栈**

#### 2. 目前没看到 K1 文档那种明确的 `drivers/soc/spacemit/spacemit-rf` 主线

我这轮特意回查了当前 K3 SDK，没有直接找到 K1 文档里那种很明确的：

- `spacemit-pwrseq.c`
- `spacemit-wlan.c`
- `spacemit-bt.c`

所以当前 K3 文档不能照着 K1 那套“平台 rf 驱动一站式控制 WiFi/BT”去写。更稳的写法是：

- 按 **SDIO + regulator-fixed + mmc-pwrseq-simple + 厂商 WiFi 驱动** 这条实际能对上的链路来写。

#### 3. K3 当前可直接确认的 host 侧核心节点是 `sdio`

在 `k3.dtsi` 中可见：

```dts
sdio: mmc@d4280800 {
    ...
};
```

在 `k3_evb.dts` 中则能看到对应板级配置：

- `bus-width = <4>;`
- `non-removable;`
- `vmmc-supply = <&vmmc_sdio>;`
- `vqmmc-supply = <&p1v8>;`
- `mmc-pwrseq = <&sdio_pwrseq>;`
- `no-mmc;`
- `no-sd;`

这已经构成了一条很完整的 SDIO WiFi 接入示例。

## 关键特性

### SDIO 接口相关特性

| 特性 | 特性说明 |
| :--- | :--- |
| 基于 K3 `sdio` 控制器接模组 | 当前 DTS 已给出板级示例 |
| 4-bit SDIO 总线 | `bus-width = <4>` |
| 非插拔模组场景 | `non-removable` |
| 模组独立上电控制 | `vmmc-supply` + `regulator-fixed` |
| 模组复位/上电时序 | `mmc-pwrseq-simple` + `reset-gpios` |
| 可保留 suspend 供电策略 | 需要时可用 `keep-power-in-suspend` |
| 通过厂商无线驱动接入 cfg80211 | 平台侧不限定唯一 WiFi 芯片驱动 |

### 当前板级例子里的关键含义

K3 EVB 这套写法里最重要的不是“某个具体模组型号”，而是这一整条 bring-up 关系：

- `vmmc_sdio` 负责模组主供电
- `sdio_pwrseq` 负责 reset 时序
- `&sdio` 节点声明这是一个 **non-removable 的 SDIO 外设**
- `no-mmc` / `no-sd` 明确这个口就是给 SDIO 模组用，不拿来跑 eMMC / SD 卡

这类写法对大部分板级 WiFi 方案都很有参考价值。

## 配置介绍

主要包括：

1. SDIO host 控制器配置
2. WiFi 模组供电配置
3. mmc-pwrseq 配置
4. 用户态网络配置

### 1. 模组供电配置

在 `k3_evb.dts` 中能看到一个固定电源示例：

```dts
vmmc_sdio: regulator-vmmc-sdio {
	compatible = "regulator-fixed";
	regulator-name = "vmmc-sdio";
	regulator-min-microvolt = <3300000>;
	regulator-max-microvolt = <3300000>;
	enable-active-high;
	gpio = <&gpio 3 6 GPIO_ACTIVE_HIGH>;
};
```

这表示：

- WiFi 模组主供电由一个固定 3.3V rail 提供；
- 通过 GPIO 控制上电使能；
- 这一路后面被 `&sdio` 的 `vmmc-supply` 引用。

### 2. pwrseq 配置

板级 DTS 里还定义了：

```dts
sdio_pwrseq: sdio-pwrseq {
	compatible = "mmc-pwrseq-simple";
	reset-gpios = <&gpio 3 4 GPIO_ACTIVE_LOW>;
};
```

这说明当前 K3 EVB 是用通用的 `mmc-pwrseq-simple` 来做：

- 模组 reset
- 上电后的初始化时序辅助

对于很多 SDIO WiFi 模组来说，这已经够用了，不一定非要平台私有 pwrseq 驱动。

### 3. SDIO host 节点配置

`k3_evb.dts` 中的 `&sdio` 示例为：

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
	clock-frequency = <375000000>;
	status = "disabled";
};
```

这些字段里最关键的是：

| 属性 | 作用 |
| :--- | :--- |
| `bus-width = <4>` | SDIO 4bit 总线 |
| `non-removable` | 表示模组为板载不可热插拔设备 |
| `vmmc-supply` | 模组主供电 |
| `vqmmc-supply` | SDIO IO 电压，通常是 1.8V |
| `mmc-pwrseq` | 指向模组 reset / 上电时序控制节点 |
| `no-mmc` / `no-sd` | 明确该 host 口不用于 MMC/SD 卡 |
| `status` | 板级启用时应改为 `okay` |

### 4. 和 SDIO binding 的关系

`mmc-controller-common.yaml` 里可以确认这些都是标准或通用的 MMC/SDIO 属性：

- `non-removable`
- `cap-sdio-irq`
- `keep-power-in-suspend`
- `vmmc-supply`
- `vqmmc-supply`
- `mmc-pwrseq`

所以这条 K3 WiFi 配置路线并不是“拍脑袋定制”，而是：

- K3 SDIO host
- 配合 Linux 通用 MMC/SDIO binding
- 再挂具体模组和驱动

### 5. 板级 bring-up 时的建议配置思路

如果你要在 K3 新板子上接一颗 SDIO WiFi 模组，建议按这个顺序想：

1. 先确认硬件上 SDIO 接的是哪一路 host；
2. 确认模组主电源是否需要独立 enable GPIO；
3. 确认模组是否有 reset 脚，并挂到 `mmc-pwrseq-simple`；
4. 确认 IO 电压是 1.8V 还是 3.3V，对应 `vqmmc-supply`；
5. 如果需要 suspend 保持供电，再补 `keep-power-in-suspend`；
6. 最后再接入具体厂商 WiFi 驱动。

## 接口介绍

### 用户态控制接口

当前 K3 这条 WiFi 主线到用户态后，仍是标准 Linux 无线网络接口，例如：

- `wlan0`
- `wpa_supplicant`
- `wpa_cli`
- `iw`
- `ip`

也就是说，平台差异主要集中在：

- SDIO / 供电 / reset / DTS bring-up

而不是用户态无线工具本身。

## Debug 介绍

### 1. 先看 SDIO host 有没有起来

优先看：

```shell
dmesg | grep -i mmc
```

以及：

```shell
ls /sys/kernel/debug/mmc/
```

如果连 SDIO host 都没起来，先别急着看 WiFi 驱动。

### 2. 再看 WiFi 模组有没有枚举成功

如果是 SDIO 模组，一般会在 `dmesg` 中看到：

- SDIO card/function 枚举日志
- 后续厂商 WiFi 驱动 probe 日志

如果停在 card init 阶段，就先查：

- `vmmc-supply`
- `vqmmc-supply`
- `reset-gpios`
- `status = "okay"`
- `non-removable`
- `no-mmc` / `no-sd`

### 3. 看 SDIO 工作状态

可以参考 MMC debugfs：

```shell
cat /sys/kernel/debug/mmc1/ios
```

重点看：

- `clock`
- `bus width`
- `timing spec`
- `signal voltage`

如果这里电压和总线模式都不对，WiFi 大概率也跑不稳。

### 4. 用户态网络连通性验证

常见流程仍然是：

```shell
wpa_supplicant -iwlan0 -Dnl80211 -c/wpa_supplicant.conf -B
wpa_cli -iwlan0 -p/var/run/wpa_supplicant
```

扫描网络：

```shell
scan
scan_results
```

连接 AP 后再配合：

```shell
ip addr
ping
iperf3
```

做吞吐和稳定性验证。

## 测试介绍

### 基础连接测试

1. 启动 `wpa_supplicant`
2. 使用 `wpa_cli` 扫描 AP
3. 配置目标 AP 的 SSID / 密码
4. 获取 IP 并测试联通

示例：

```shell
wpa_supplicant -iwlan0 -Dnl80211 -c/wpa_supplicant.conf -B
wpa_cli -iwlan0 -p/var/run/wpa_supplicant
```

### 吞吐测试

同一局域网中可使用：

```shell
# 服务端
iperf3 -s

# 客户端
iperf3 -c <server-ip> -t 60
```

如果你在做板级 bring-up，建议先别一上来就盯峰值吞吐，先把下面三件事确认稳：

- 能稳定枚举
- 能稳定关联 AP
- 长时间传输不掉线

## FAQ

### 1. K3 当前 WiFi 是不是有一条很明确的平台私有驱动？

从这轮已确认到的 SDK 情况看，**没有看到像 K1 文档里那样明确的一整套 `spacemit-rf` 平台私有主线**。更稳的结论是：

- K3 主要提供 host + DTS + 供电/reset 方案
- 具体 WiFi 功能仍由对应厂商驱动承担

### 2. K3 当前最稳的 WiFi 文档主线是什么？

**SDIO WiFi 主线。**

因为它在 `k3_evb.dts` 中已经有明确板级例子，比较适合做 bring-up 参考。

### 3. 如果以后换成 PCIe / USB WiFi，这篇还值不值看？

值，尤其是“外部模组 + 板级供电/复位/唤醒”的思路仍然成立。但具体的数据通路和驱动，会从 `sdio` 换成对应的 PCIe/USB 路线。

### 4. 这篇最该记住哪条经验？

**先把 WiFi 问题拆成两半：先看 host 和板级上电/reset，再看厂商无线驱动。**

别一看到 `wlan0` 不起来，就直接往上层网络配置上扑。很多问题其实在 SDIO / 电源 / pwrseq 这一步就已经埋下了。
