# GMAC

介绍 K3 平台 GMAC 以太网控制器驱动的能力、设备树配置方法以及调试方式。

## 模块介绍

GMAC（Gigabit Media Access Controller）是 K3 平台用于千兆以太网通信的 MAC 控制器。它本身负责：

- 以太网 MAC 层收发；
- DMA 描述符和数据搬运；
- 中断处理；
- 与 PHY 通过 MDIO 交互；
- 与 Linux 网络栈对接。

从用户角度看，GMAC 文档最重要的不是 MAC 内部寄存器，而是：

- K3 上有几路网口；
- 控制器节点和 PHY 节点怎么写；
- `phy-mode`、`phy-handle`、`mdio`、`snps,reset-gpios` 分别是什么意思；
- RGMII 场景下时钟延时参数怎么理解；
- Linux 下怎么看链路是否起来、协商是否正常、吞吐是否正常。

### 功能介绍

Linux 以太网软件栈大致分为：

- **MAC 控制器驱动**：负责 SoC 内部以太网控制器；
- **PHY 驱动**：负责外部 PHY 芯片；
- **MDIO 总线**：MAC 通过它访问 PHY 寄存器；
- **Linux 网络栈**：通过 `eth0` / `eth1` 等接口对用户提供能力。

K3 上的 GMAC 使用的是 **Synopsys DesignWare MAC（stmmac）** 框架，K3 DTS 中控制器节点使用：

```dts
compatible = "spacemit,k3-gmac", "snps,dwmac-5.10a";
```

也就是说，K3 的 GMAC 不是一套完全独立的私有网络框架，而是基于主线常见的 `stmmac` / `dwmac` 体系，再叠加 SpacemiT 的 glue layer 和平台参数。

### 源码结构介绍

K3 GMAC 相关代码主要位于：

```text
linux-6.18/
|-- drivers/net/ethernet/stmicro/stmmac/
|   |-- stmmac_main.c                 # stmmac 主体驱动
|   |-- stmmac_platform.c             # 平台设备公共逻辑
|   |-- dwmac-spacemit.c / glue层     # SpacemiT 平台 glue layer
|   `-- Kconfig
|-- Documentation/devicetree/bindings/net/
|   `-- snps,dwmac.yaml               # DWMAC/stmmac 通用 binding
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                       # K3 GMAC 控制器节点
    `-- k3*.dts                       # 板级 GMAC / PHY / MDIO 实际用法
```

结合 `Kconfig` 可以看到 K3 平台使用：

```text
CONFIG_STMMAC_ETH
CONFIG_STMMAC_PLATFORM
CONFIG_DWMAC_SPACEMIT_ETHQOS
```

因此写 K3 GMAC 文档时，应该同时参考：

- K1 现有文档结构；
- K3 的 `k3.dtsi` 控制器定义；
- K3 板级 `k3_evb.dts` / `k3_deb1.dts` / `k3_gemini_c0.dts` 等真实用法；
- `snps,dwmac.yaml` 中的通用属性语义。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 控制器框架 | 基于 `stmmac` / `dwmac` |
| K3 compatible | `spacemit,k3-gmac` |
| MAC IP 版本 | `snps,dwmac-5.10a` |
| 网口数量 | K3 `k3.dtsi` 中定义 `eth0` ~ `eth3` |
| PHY 管理 | 通过 `mdio` 子节点管理外部 PHY |
| PHY 连接模式 | 板级当前主要使用 `rgmii` |
| 时间戳 | 提供 `ptp_ref` 时钟，支持 PTP/硬件时间戳能力 |
| DMA 优化 | 板级 DTS 中常使用 `snps,tso`、`snps,force_sf_dma_mode` |
| 时钟调优 | 支持 `spacemit,clk-tuning-enable`、`spacemit,tx-phase`、`spacemit,rx-phase` |
| 唤醒中断 | 支持 `eth_wake_irq`，可配合 `spacemit,wake-irq-enable` |

### 从用户角度最关键的几个点

#### 1. GMAC 不是单独就能通信的，必须配 PHY

K3 的 GMAC 是 MAC 控制器，真正接网线的通常是外部 PHY 芯片。要让网口正常工作，设备树里至少要有：

- `phy-mode`
- `phy-handle`
- `mdio` 子节点
- `ethernet-phy@X` 节点

否则即使 MAC 控制器 probe 成功，也可能没有链路。

#### 2. `phy-mode` 要和硬件走线一致

K3 板级实际用法里，常见为：

```dts
phy-mode = "rgmii";
```

这不是随便填的。如果板级原理图走的是 RGMII，就必须写 RGMII；如果板子实际上做了别的 PHY 接口模式，写错会导致：

- 链路起不来；
- 协商异常；
- 能 ping 但丢包严重；
- 千兆退化到百兆甚至不稳定。

#### 3. RGMII 延时参数是板级成败关键

K3 板级 DTS 中广泛使用：

```dts
spacemit,clk-tuning-enable;
spacemit,clk-tuning-by-delayline;
spacemit,tx-phase = <65>;
spacemit,rx-phase = <50>;
```

这说明 K3 的千兆网口不是简单“启用就行”，而是很依赖板级时钟和采样点调优。对于用户 bring-up 来说，这些参数非常关键。

## 配置介绍

### CONFIG 配置

K3 GMAC 常见配置包括：

- `CONFIG_NETDEVICES`
- `CONFIG_ETHERNET`
- `CONFIG_STMMAC_ETH`
- `CONFIG_STMMAC_PLATFORM`
- `CONFIG_DWMAC_SPACEMIT_ETHQOS`
- `CONFIG_PHYLIB`
- `CONFIG_PTP_1588_CLOCK`（若需要时间戳/PTP 功能）

菜单路径大致如下：

```text
Device Drivers
    Network device support
        Ethernet driver support
            STMicroelectronics devices
                STMMAC Ethernet driver support (STMMAC_ETH)
                STMMAC Platform bus support (STMMAC_PLATFORM)
                Spacemit ETHQOS support (DWMAC_SPACEMIT_ETHQOS)
```

### DTS 配置

#### 控制器节点

K3 `k3.dtsi` 中定义了 4 路以太网控制器：

- `eth0`
- `eth1`
- `eth2`
- `eth3`

典型控制器节点如下：

```dts
eth0: ethernet@cac80000 {
        compatible = "spacemit,k3-gmac", "snps,dwmac-5.10a";
        reg = <0x0 0xcac80000 0x0 0x2000>;
        spacemit,apmu = <&syscon_apmu>;
        spacemit,ctrl-offset = <0x3e4>;
        spacemit,dline-offset = <0x3e8>;
        clocks = <&syscon_apmu CLK_APMU_EMAC0_BUS>,
                 <&syscon_apmu CLK_APMU_EMAC0_1588>;
        clock-names = "stmmaceth", "ptp_ref";
        resets = <&syscon_apmu RESET_APMU_EMAC0>;
        reset-names = "stmmaceth";
        interrupts = <131 IRQ_TYPE_LEVEL_HIGH>, <276 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-names = "macirq", "eth_wake_irq";
        mac-address = [00 00 00 00 00 00];
        snps,axi-config = <&gmac_axi_setup>;
        status = "disabled";
};
```

各属性含义如下：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | K3 平台 GMAC 控制器匹配字符串 |
| `reg` | MAC 寄存器空间 |
| `spacemit,apmu` | 平台时钟/控制寄存器所在 syscon |
| `spacemit,ctrl-offset` | 平台控制寄存器偏移 |
| `spacemit,dline-offset` | 平台 delayline 控制寄存器偏移 |
| `clocks` / `clock-names` | MAC 主时钟和 PTP 参考时钟 |
| `resets` / `reset-names` | MAC 复位 |
| `interrupts` | MAC 中断和唤醒中断 |
| `mac-address` | MAC 地址，可由 bootloader 填充 |
| `snps,axi-config` | AXI 总线参数配置引用 |

#### AXI 配置

K3 `k3.dtsi` 中定义了：

```dts
gmac_axi_setup: stmmac-axi-config {
        snps,wr_osr_lmt = <4>;
        snps,rd_osr_lmt = <8>;
        snps,blen = <0 0 0 0 16 8 4>;
};
```

这部分用于配置 STMMAC 的 AXI 访问行为。对大多数用户来说一般不用改，但需要知道它是控制器节点的一部分，并且会影响 DMA 访问特性。

#### 板级网口启用示例

K3 EVB 上 `eth0` 的典型用法如下：

```dts
&eth0 {
        pinctrl-names = "default";
        pinctrl-0 = <&gmac0_cfg>;

        max-speed = <1000>;
        tx-fifo-depth = <8192>;
        rx-fifo-depth = <8192>;
        snps,tso;
        snps,force_sf_dma_mode;
        phy-mode = "rgmii";
        phy-handle = <&gmac0_phy>;
        snps,reset-gpios = <&gpio 0 15 GPIO_ACTIVE_LOW>;
        snps,reset-delays-us = <0 20000 100000>;

        spacemit,wake-irq-enable;

        spacemit,clk-tuning-enable;
        spacemit,clk-tuning-by-delayline;
        spacemit,tx-phase = <65>;
        spacemit,rx-phase = <50>;

        status = "okay";

        mdio {
                #address-cells = <1>;
                #size-cells = <0>;
                compatible = "snps,dwmac-mdio";

                gmac0_phy: ethernet-phy@1 {
                        compatible = "ethernet-phy-id001c.c916",
                                     "ethernet-phy-ieee802.3-c22";
                        reg = <1>;
                        device_type = "ethernet-phy";
                };
        };
};
```

这个例子基本涵盖了用户最关心的所有要点。

#### `phy-mode`

K3 当前板级 DTS 主要使用：

```dts
phy-mode = "rgmii";
```

它表示 MAC 和 PHY 之间使用 RGMII 接口。用户必须按原理图来写，不能凭经验乱填。

#### `phy-handle`

```dts
phy-handle = <&gmac0_phy>;
```

用于把 MAC 节点和 MDIO 总线下具体那个 PHY 节点绑定起来。没有它，MAC 可能找不到对应 PHY。

#### `mdio` 子节点

K3 板级中一般这样写：

```dts
mdio {
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "snps,dwmac-mdio";

        gmac0_phy: ethernet-phy@1 {
                reg = <1>;
        };
};
```

这里的 `reg = <1>` 表示 PHY 地址是 1。用户如果板上 PHY strap 地址不同，这里也必须同步改。

#### PHY reset GPIO

K3 板级大量使用：

```dts
snps,reset-gpios = <&gpio 0 15 GPIO_ACTIVE_LOW>;
snps,reset-delays-us = <0 20000 100000>;
```

这说明 PHY 的 reset 由 MAC 节点统一控制。其含义通常是：

- reset GPIO 是哪根脚；
- reset 是高有效还是低有效；
- assert / deassert 前后的延时要求。

如果 reset GPIO 配错，常见现象是：

- PHY ID 读不出来；
- 网口始终无 link；
- 偶发能通但重启后不稳定。

#### 时钟调优参数

K3 板级 DTS 里非常关键的一组属性：

```dts
spacemit,clk-tuning-enable;
spacemit,clk-tuning-by-delayline;
spacemit,tx-phase = <58>;
spacemit,rx-phase = <70>;
```

不同板子的 `tx-phase` / `rx-phase` 数值不同，说明这些参数与：

- 板级布线；
- PHY 型号；
- 时钟路径；
- 实际信号质量；

都有关系。

用户在新板 bring-up 时，应把这类参数看成 **和 pinctrl 一样重要的板级参数**，而不是普通可有可无的 tuning。

#### 其他常见属性

K3 板级常见还会使用：

- `max-speed = <1000>`
- `tx-fifo-depth`
- `rx-fifo-depth`
- `snps,tso`
- `snps,force_sf_dma_mode`
- `spacemit,wake-irq-enable`

这些属性主要影响：

- 目标链路速率；
- DMA/FIFO 路径；
- 分段卸载能力；
- 唤醒和省电场景。

## 接口介绍

### 用户空间接口

GMAC 在 Linux 中表现为标准网卡，例如：

- `eth0`
- `eth1`
- `eth2`
- `eth3`

用户通常通过以下工具使用：

- `ip`
- `ethtool`
- `ping`
- `iperf3`
- `mii-tool`（较旧）

### 常用命令

查看接口：

```bash
ip link
```

拉起接口：

```bash
ip link set eth0 up
ip addr add 192.168.1.10/24 dev eth0
```

查看链路和驱动信息：

```bash
ethtool eth0
ethtool -i eth0
```

查看统计信息：

```bash
ethtool -S eth0
ip -s link show eth0
```

查看 PHY 协商：

```bash
ethtool eth0
```

吞吐测试：

```bash
iperf3 -s
iperf3 -c <peer-ip>
```

## 调试介绍

### 1. 检查驱动是否 probe 成功

```bash
dmesg | grep -Ei "stmmac|dwmac|eth"
```

用户应重点看：

- 控制器是否成功注册为 `ethX`；
- PHY 是否成功识别；
- 是否有 reset / mdio / clock 相关报错。

### 2. 查看网口是否创建

```bash
ip link
```

如果 DTS 已启用且驱动 probe 正常，通常能看到：

- `eth0`
- `eth1`
- 等接口。

### 3. 查看链路状态和协商信息

```bash
ethtool eth0
```

重点关注：

- `Speed`
- `Duplex`
- `Auto-negotiation`
- `Link detected`

如果 `Link detected: no`，通常先不要怀疑网络栈，而要先排查：

- PHY reset 是否正常；
- PHY 地址是否写对；
- `phy-mode` 是否正确；
- 网线/交换机/对端是否正常。

### 4. 查看统计信息

```bash
ethtool -S eth0
ip -s link show eth0
```

统计信息可以帮助判断：

- 是否有大量 CRC 错误；
- 是否有 RX/TX drop；
- 是否有 FIFO / DMA 异常；
- 是否存在链路虽然能起但数据面不稳定的情况。

### 5. 吞吐和稳定性测试

```bash
iperf3 -s
iperf3 -c <peer-ip> -t 30
```

如果存在：

- 吞吐上不去；
- 大包丢包；
- 千兆协商异常；
- 长时间压测掉线；

建议重点回到：

- `tx-phase` / `rx-phase`
- PHY reset 时序
- RGMII 时钟和板级布线
- `phy-mode` 与硬件一致性

## FAQ

### 1. 为什么 `eth0` 出来了，但 `Link detected` 一直是 `no`？

常见原因：

- `phy-handle` 指向错了；
- `reg` 里的 PHY 地址不对；
- `snps,reset-gpios` 或极性错误；
- PHY 没供电或没出 reset；
- 网线、对端设备或交换机异常。

### 2. 为什么能通但只有百兆，或者千兆不稳定？

重点检查：

- `phy-mode` 是否正确；
- 板级是否真的是 RGMII；
- `tx-phase` / `rx-phase` 是否需要重新调优；
- PHY 侧 strap 和供电是否正确；
- 线材和对端是否满足千兆要求。

### 3. 为什么不同板子的 `tx-phase` / `rx-phase` 不一样？

因为这些参数和板级设计直接相关。不同板卡的：

- 走线长度；
- PHY 型号；
- 时钟路径；
- SI 裕量；

都可能不同，所以不能简单照搬别的板子数值。

### 4. 为什么 DTS 里 `mac-address` 是全 0？

这通常表示 MAC 地址预留由 bootloader 或上层系统在启动时填充。用户需要确认：

- bootloader 是否正确写入 MAC 地址；
- 最终 Linux 中 `ip link` 看到的地址是否有效；
- 多个网口是否存在 MAC 地址冲突。
