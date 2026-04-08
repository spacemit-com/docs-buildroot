# WIFI

介绍 K3 平台 WiFi 模组的常见移植方法以及注意事项。

## 模块介绍

K3 平台需要外接**外部 WiFi 模组**实现 WiFi 功能，支持 SDIO / PCIe / USB 等接口。

## 功能介绍

WiFi 在 Linux 里主要有以下几层：

![](static/wlan.png)

1. **cfg80211 / mac80211 / nl80211** 
   提供 Linux 无线协议栈与用户态控制接口；
2. **模组驱动** 
   WiFi 模组驱动，由 WiFi 厂商提供，主要实现 WiFi 功能；
3. **接口控制器**
   主要实现 WiFi 数据传输接口功能，如 PCIe、SDIO 以及 USB 等接口；

## 源码结构介绍

涉及的源码主要有：

```text
linux-6.18/
|-- drivers/net/wireless/          # 具体 WiFi 驱动（厂商/主线）
|-- drivers/mmc/                   # SDIO / MMC host 控制器
|-- drivers/regulator/             # 模组供电控制
|-- drivers/mmc/core/pwrseq*       # mmc-pwrseq 通用上电/复位逻辑
`-- arch/riscv/boot/dts/spacemit/  # 板级 DTS 配置
```

其中 WiFi 驱动的源码一般放到以下目录：

```text
drivers/net/wireless
|-- aic8800             # aic 厂商驱动
|-- realtek             # realtek 厂商驱动
    |-- rtl8852bs       # rtl8852bs
    |-- rtw89           # rtl8852be
```

## 关键特性

### SDIO 接口相关特性

| 特性 | 特性说明 |
| :----- | :---- |
| 兼容 SDIO v4.10 | 兼容 4bit SDIO 4.10 规范 |
| 支持 SD 3.0 模式 | 支持 SDR12/SDR25/DDR50/SDR50/SDR104 模式 |
| 支持 PIO/DMA | 支持 PIO, SDMA, ADMA, ADMA2 传输模式 |

### 性能参数

| 模组型号 | TX(Mb/s) | RX(Mb/s) |
| :----- | :---- | :----: |
| rtl8852bs | 460 | 480 |
| aic8800d80 | 410 | 470 |

## SDIO 接口模组配置介绍

### CONFIG 配置

WiFi 相关配置如下：

`CONFIG_NET` / `CONFIG_WIRELESS` / `CONFIG_CFG80211` 为 WiFi 提供支持，需要设置为 `Y`：

```text
Networking support (NET [=y])
    Wireless (WIRELESS [=y])
        cfg80211 - wireless configuration API (CFG80211 [=y])
```

SDIO 相关配置如下：

`CONFIG_MMC` 为 MMC 总线协议提供支持，通常为 `Y`：

```text
Device Drivers
    MMC/SD/SDIO card support (MMC [=y])
```

`CONFIG_MMC_SDHCI` / `CONFIG_MMC_SDHCI_PLTFM` / `CONFIG_MMC_SDHCI_OF_K1` 为 SpacemiT SDHCI 控制器提供支持，需要设置为 `Y`：

```text
Device Drivers
    MMC/SD/SDIO card support (MMC [=y])
        Secure Digital Host Controller Interface support (MMC_SDHCI [=y])
            SDHCI platform and OF driver helper (MMC_SDHCI_PLTFM [=y])
                SDHCI OF support for the SpacemiT SDHCI controller (MMC_SDHCI_OF_K1 [=y])
```

### DTS 配置

#### SDIO 控制器配置

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
        keep-power-in-suspend;
        clock-frequency = <375000000>;
        spacemit,tx_delaycode = <0x7f>;
        status = "okay";
};
```

K3 上的 SDIO WiFi 驱动不再需要关注 regulator 和 GPIO 等板级相关信息， 统一放到总线层面进行管理。
供电相关的 regulator 和 GPIO 等配置放到 `vmmc-supply`、`vqmmc-supply` 中，WiFi REG_ON/RESET 放到 `sdio_pwrseq` 中进行配置。

#### SDIO 电源配置

`vmmc_sdio` 用于封装 WiFi 模组供电依赖的 regulator 和 GPIO，有的模组供电除了依赖 regulator 以外可能还需要依赖某些 GPIO 的状态， 建议使用 `regulator-fixed` 进行封装。

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

#### WiFi REG_ON 配置

`sdio_pwrseq` 定义 reset 引脚，对应 WiFi REG_ON ：

```dts
sdio_pwrseq: sdio-pwrseq {
        compatible = "mmc-pwrseq-simple";
        reset-gpios = <&gpio 3 4 GPIO_ACTIVE_LOW>;
};
```

## 接口介绍

### 用户态接口

用户态推荐采用 `nl80211` 接口访问 WiFi 设备，例如：

- `wpa_supplicant`
- `wpa_cli`
- `iw`
- `ip`

基于 `wext` 接口的访问方式默认不支持， 需要的话可以打开 `CONFIG_CFG80211_WEXT` ：

```text
cfg80211 wireless extensions compatibility (CFG80211_WEXT [=n])
```

## Debug 介绍

### 1. 确认控制器的状态

`SDIO` ：

```bash
dmesg | grep -i mmc1
```

```bash
ls /sys/kernel/debug/mmc/
```

`USB` ：

```bash
dmesg | grep -i usb
```

```bash
ls /sys/kernel/debug/usb/
```

### 2. 确认 WiFi 模组有没有识别

一般会在 `dmesg` 中看到：

- SDIO/USB/PCIe card/function 枚举日志
- 后续厂商 WiFi 驱动 probe 日志

如果 WiFi 模组没有识别到，需要排查：

- `vmmc-supply`
- `vqmmc-supply`
- `reset-gpios`
- `spacemit,tx_delaycode`
- `status = "okay"`

### 3. 确认总线工作状态

`SDIO` ：

```bash
cat /sys/kernel/debug/mmc1/ios
```

确认以下信息：

- `clock`
- `bus width`
- `timing spec`
- `signal voltage`

`USB` ：

```bash
cat /sys/kernel/debug/usb/devices
```

确认以下信息：

- `Driver`
- `Spd`
- `Vendor / ProdID`
- `Manufacturer / Product`

## 测试介绍

### 扫描连接测试

首先确认 wpa_supplicant 服务有正常运行。

```bash
wpa_supplicant -iwlan0 -Dnl80211 -c/etc/wpa_supplicant.conf -B
```

wpa_supplicant.conf 参考配置如下：

```txt
ctrl_interface=/var/run/wpa_supplicant
disable_scan_offload=1
update_config=1
filter_rssi=-75
pmf=1
#sae_pwe=2
wowlan_triggers=any
#bgscan="simple:11:-70:300"
gas_rand_addr_lifetime=0
gas_rand_mac_addr=1
```

- `ctrl_interface`: 指定 wpa_supplicant 与用户空间工具（如 wpa_cli）通信的控制接口路径，ctrl_interface 如果不是默认的 `/var/run/wpa_supplicant` ，则 wpa_cli 运行时需要使用 -p 进行显式指定；
- `disable_scan_offload`: 禁用硬件扫描（部分模组支持），强制使用软件扫描；
- `update_config`: 允许 wpa_supplicant 自动更新配置文件；
- `filter_rssi`: 过滤信号强度低于 -75 dBm 的 WiFi 热点，仅连接信号质量达标的 AP；
- `pmf`: 启用 PMF（802.11w）管理帧保护，防止管理帧（如解除认证帧）伪造，`1` 为可选，`2` 为强制；
- `sae_pwe`: 控制 SAE 密码元素推导方式，`2` 表示仅使用 H2E 方式；
- `wowlan_triggers`: 启用 WiFi 唤醒（WoWLAN）功能，指定唤醒触发条件为任意事件；
- `bgscan`: 设置后台扫描，`simple:11:-70:300` 是指当前信号高于或等于 -70 dBm 时每 300 秒扫描一次，信号低于 -70 dBm 每 11 秒扫描一次；
- `gas_rand_addr_lifetime`: 设置 GAS（Generic Advertisement Service）随机 MAC 地址的有效期，0 为永久；
- `gas_rand_mac_addr`: 启用 GAS 交互时的随机 MAC 地址功能，提升隐私性。

wpa_cli 扫描：

```bash
wpa_cli -iwlan0 -p/var/run/wpa_supplicant
scan
scan_results
```

正常扫描会有类似如下输出：

```bash
bssid / frequency / signal level / flags / ssid
f6:12:b3:d4:65:ef       2462    -37     [WPA2-PSK-CCMP][WPS][ESS][P2P]  wilson
78:85:f4:82:01:3c       2462    -66     [WPA2-PSK-CCMP][WPS][ESS]       HUAWEI-LX45AG_HiLink
02:0e:5e:76:a5:6e       2412    -69     [WPA-PSK-CCMP+TKIP][ESS]        ChinaNet-1mMr
30:8e:7a:2f:64:8c       2437    -69     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]    K03_1tlftb
dc:16:b2:57:9e:65       2437    -78     [WPA2-PSK-CCMP][ESS]    \x00\x00\x00\x00\x00\x00\x00\x00
dc:16:b2:57:9e:60       2437    -78     [WPA-PSK-CCMP][WPA2-PSK-CCMP][WPS][ESS] TK-ZJB
48:0e:ec:ad:52:4d       2462    -78     [WPA-PSK-CCMP][WPA2-PSK-CCMP][WPS][ESS] TP-LINK_524D
3c:d2:e5:c6:08:9b       2452    -83     [WPA2-PSK-CCMP][ESS]
3e:d2:e5:16:08:9b       2452    -83     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]    young
80:ea:07:dc:f2:be       2462    -88     [WPA-PSK-CCMP][WPA2-PSK-CCMP][ESS]      HZXF
9a:00:74:84:d1:60       2412    -85     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]   ChinaNet-ieR7
dc:f8:b9:46:ec:30       2472    -85     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]   ChinaNet-MiZK
```

选择需要连接的 AP 网络进行连接：

```bash
> add_network
0
> set_network 0 ssid "wilson"
OK
> set_network 0 key_mgmt WPA-PSK
OK
> set_network 0 psk "wilson2001"
OK
> enable_network 0
```

```bash
wpa_supplicant -iwlan0 -Dnl80211 -c/wpa_supplicant.conf -B
wpa_cli -iwlan0 -p/var/run/wpa_supplicant
```

### 吞吐量测试

同一局域网中可使用：

```bash
# 服务端
iperf3 -s

# 客户端
iperf3 -c <server-ip> -t 60
```

### 信号强度查看

连接成功后可通过以下命令查看当前信号强度和链路状态：

```bash
iw dev wlan0 link
```

查看更详细的统计信息：

```bash
iw dev wlan0 station dump
```

关注以下字段：

- `signal` — 当前 RSSI，单位 dBm，一般 -70 dBm 以上为正常；
- `tx bitrate` / `rx bitrate` — 当前协商速率；
- `tx failed` / `tx retries` — 发送失败和重传次数，数值持续增大说明信号质量差。

## FAQ

### 1. 为什么控制器起来了，但加载驱动后找不到 `wlan0` 设备节点？

常见原因包括：

- 对应方案的控制器 DTS 是否有使能；
- `vmmc-supply` / `vqmmc-supply` 配置错误或者电压异常；
- `mmc-pwrseq` 是否正确配置 WiFi 模组 REG_ON/RESET 引脚；
- 对应的 Wi‑Fi 固件是否存在。

### 2. WiFi 可以使用，但是使用过程中经常会出现异常打印？

例如出现如下错误打印：

```txt
[69686.314058] rtl8852bs mmc1:0001:1: rtw_sdio_raw_write: sdio write failed (-84)
[69686.314063] mmc1: set tx_delaycode: 127
[69686.314080] rtl8852bs mmc1:0001:1: RTW_SDIO: WRITE use CMD53
[69686.314085] rtl8852bs mmc1:0001:1: RTW_SDIO: WRITE to 0x1800a, 80 bytes
[69686.322783] mmc1: pretuned card, use select_delay[1]:200
[69686.328249] RTW_SDIO: WRITE 00000000: 00 64 48 00 00 00 00 00 1a 00 24 00 b9 23 00 00
[69686.341886] RTW_SDIO: WRITE 00000010: 00 00 00 00 00 00 00 00 00 00 00 40 00 00 00 00
[69686.349841] RTW: ERROR sdio_io: write FAIL! error(-2) addr=0x1800a 80 bytes, retry=0,0
[69686.349942] rtl8852bs mmc1:0001:1: rtw_sdio_raw_write: sdio write failed (-110)
```

常见原因包括：

- `-84` 表示 SDIO 的 tx 出现 crc 校验错误，需要调整 SDIO 的 `spacemit,tx_delaycode` 参数；
- `-110` 表示 SDIO 操作超时，通常是信号完整性问题或时序参数不匹配，同样可以尝试调整 `spacemit,tx_delaycode` 或降低 SDIO 时钟频率排查。

### 3. 连接 AP 成功但无法上网？

常见原因包括：

- 确认是否获取到 IP 地址，可通过 `ip addr show wlan0` 查看；
- 若未获取 IP，确认 DHCP 客户端（如 `udhcpc` / `dhclient`）是否正常运行；
- 确认 DNS 配置是否正确，检查 `/etc/resolv.conf` 是否有有效的 nameserver；
- 确认默认路由是否存在，可通过 `ip route` 查看。
