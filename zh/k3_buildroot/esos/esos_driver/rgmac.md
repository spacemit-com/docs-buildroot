# RGMAC

介绍 RGMAC 的功能和使用方法。

## 模块介绍

K3 RGMAC 模块采用 Synopsys DesignWare Ethernet QoS 控制器（版本 5.40a），符合 IEEE 802.3-2015 标准。在 K3 平台该模块既可由 ACPU 访问，基于 Linux GMAC 驱动实现完整的以太网接口功能；也可由 RCPU 访问，基于 RT-Thread GMAC 驱动满足实时通信需求。本文仅针对基于 RT-Thread GMAC 驱动的 RGMAC 模块进行功能和使用方法介绍。

### 功能介绍
![](static/rgmac.png)
- **应用层：** 面向用户提供应用服务。
- **协议栈层：** 实现网络协议，并为应用层提供系统调用接口，当前小核系统仅支持 EtherCAT。
- **设备驱动层：** 负责实现数据传输和设备管理。
- **物理层：** 网络硬件设备。

### 源码结构介绍
驱动源码位于 esos/bsp/spacemit/drivers/gmac 目录，主要文件如下：

```bash
esos/bsp/spacemit/drivers/gmac
.
|-- dwc_eth_qos.c               # EQoS 控制器核心层驱动
|-- dwc_eth_qos.h
|-- dwc_eth_qos_spacemit.c      # EQoS 控制器平台驱动
|-- dwc_eth_qos_tool.c          # macctl 工具实现，用于 debug 和基本功能测试
|-- genphy.c                    # 通用 PHY 驱动
|-- genphy.h
`-- SConscript
```

## 关键特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 RGMII / RMII 接口 | 支持多种 PHY 接口类型 |
| 支持 10/100/1000 Mbps | 支持多种链路速率 |
| 支持自动协商 | PHY 端支持速率及双工模式自动协商 |
| 支持回环测试 | 支持 MAC 回环和 PHY 回环测试 |
| 仅支持轮询模式传输数据 | 数据收发需由上层主动轮询触发 |

> **注：** 此处所列特性仅针对基于 RT-Thread GMAC/PHY 驱动实现的功能，未涵盖硬件本身支持的全部特性。

## 配置介绍

主要包括 **Kconfig 配置** 和 **DTS 配置**

### Kconfig 配置

1. `BSP_USING_GMAC`：启用 RGMAC 驱动

```bash
config BSP_USING_GMAC
    bool "Enable GMAC"
    default n
```

2. `BSP_USING_MACCTL_TOOL`：启用 `macctl` 工具

```bash
if BSP_USING_GMAC
    config BSP_USING_MACCTL_TOOL
        bool "Enable MAC Control Tool"
        default n
endif
```

3. `RT_USING_ETHERCAT`：启用 RT-Thread 版本的 Igh EtherCAT 主站

```bash
config RT_USING_ETHERCAT
    bool "Enable IGH EtherCAT support"
    default n
```
> **注：** 注意如果想使用 `macctl` 工具进行功能测试，必须关闭 `RT_USING_ETHERCAT`。
### DTS 配置

#### pinctrl

在 `k3-pinctrl.dtsi` 中，RGMAC 相关的 pin 给出一组默认配置：
```c
  rgmac0_cfg: rgmac0-cfg {
      pinctrl-single,pins = <
          K3_PADCONF(GPIO_59, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rxdv */
          K3_PADCONF(GPIO_60, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rx d0 */
          K3_PADCONF(GPIO_61, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rx d1 */
          K3_PADCONF(GPIO_62, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rx clk */
          K3_PADCONF(GPIO_63, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rx d2 */
          K3_PADCONF(GPIO_64, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 rx d3 */
          K3_PADCONF(GPIO_65, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx d0 */
          K3_PADCONF(GPIO_66, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx d1 */
          K3_PADCONF(GPIO_67, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx clk */
          K3_PADCONF(GPIO_68, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx d2 */
          K3_PADCONF(GPIO_69, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx d3 */
          K3_PADCONF(GPIO_70, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 tx en */
          K3_PADCONF(GPIO_71, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 mdc */
          K3_PADCONF(GPIO_72, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 mdio */
          K3_PADCONF(GPIO_73, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 int */
          K3_PADCONF(GPIO_74, (MUX_MODE1 | EDGE_NONE | PULL_DIS | PAD_DS8))    /* rgmac0 clk ref */
      >;
  };
```
若无需更改驱动强度，板级 DTS 以太网节点只需引用这组 pin 配置
```c
&eth0 {
	pinctrl-names = "default";
	pinctrl-0 = <&rgmac0_cfg>;
	...
}
```
#### gpio

查看开发板原理图，确认 RGMAC PHY 复位信号对应的 GPIO。例如若为 GPIO75，则配置如下
```c
&eth0 {
	...
	phy-reset-pin = <75>;
	...
}
```
#### phy

由于 RT-Thread 版本中的 PHY 驱动为通用型 PHY 驱动，实现相对简单，只需配置接口类型和 PHY address 即可

```c
&eth0 {
	...
	phy-mode = "rgmii";
	phy-handle = <&rgmii>;
	...
	mdio {
		#address-cells = <1>;
		#size-cells = <0>;

		rgmii: phy@0 {
			reg = <0x1>;
		};
	};
	...
};
```

#### TX/RX phase

在 RGMII 接口下，时钟与数据信号的相位偏差会受板级 PCB 布线影响，严重时可能导致数据采样出错。K3 平台支持配置时钟相位偏移，以优化采样窗口，确保满足严苛的时序约束（Setup/Hold Margin）。以 K3 EVB 为例，配置如下：
```c
&eth0 {
	...
	clk-tuning-enable = <1>;
	clk-tuning-by-delayline = <1>;
	tx-phase = <51>;
	rx-phase = <54>;
	...
};
```
其中，`spacemit,clk-tuning-enable` 用于使能 K3 平台的时钟相位调节功能。对于 RMII 接口类型 PHY，通常无需启用该功能；另外若由 PHY 侧提供 delay，也无需在 K3 侧重复配置。当相位调节由 K3 侧承担时，进一步可分为以下三种方式：
```c
spacemit,clk-tuning-by-reg
spacemit,clk-tuning-by-delayline
spacemit,clk-tuning-by-clk-revert
```

注意 `spacemit,tx-phase` 和 `spacemit,rx-phase` 的取值均对应一定的实际调节量，但该调节量受板级供电因素影响较大，无法给出绝对精确对应关系，因此这里将其视为调节档位。

采用 `spacemit,clk-tuning-by-reg` 方式调节时，`tx-phase` 和 `rx-phase` 的取值范围为 0～7，即提供 8 个调节档位。
采用 `spacemit,clk-tuning-by-delayline` 方式调节时，`tx-phase` 和 `rx-phase` 的取值范围为 0～254，即提供 255 个调节档位。
采用 `spacemit,clk-tuning-by-clk-revert` 方式调节时，时钟相位会被反转 180°，仅在 RMII 模式下使用这种调节方式。

#### max-speed
用于设置平台支持的最大速率
```c
&eth0 {
	...
	max-speed = <1000>;
	...
};
```
#### 完整 DTS 配置

综上所述，完整配置如下。

```c
&eth0 {
	pinctrl-names = "default";
	pinctrl-0 = <&rgmac0_cfg>;
	phy-reset-pin = <75>;
	max-speed = <1000>;
	clk-tuning-enable = <1>;
	clk-tuning-by-delayline = <1>;
	tx-phase = <51>;
	rx-phase = <54>;
	phy-mode = "rgmii";
	phy-handle = <&rgmii>;

	status = "okay";

	mdio {
		#address-cells = <1>;
		#size-cells = <0>;

		rgmii: phy@0 {
			reg = <0x1>;
		};
	};
};
```
## 示例使用

在小核串口可以通过 `macctl` 工具对 RGMAC 进行基本功能测试，基本命令如下：

1. 寄存器访问测试
```bash
macctl eqos0 test reg-access
```

2. mac 回环测试
```bash
macctl eqos0 test mac-loopback
```

3. phy 回环测试
```bash
macctl eqos0 test phy-loopback
```

4. 开关老化测试
```bash
macctl eqos0 test up-down -c <count>
```

5. 循环收发包压力测试
```bash
macctl eqos0 test polling -c <count>
```

## 应用开发

当前 RGMAC 驱动仅支持 EtherCAT 协议通信，并未对接 RT-Thread 网络协议栈，下面对 RT-Thread 版本 EtherCAT 主要接口进行介绍，这些接口与 Linux 版本 EtherCAT 主站接口完全一致

- 请求主站实例

```c
ec_master_t *ecrt_request_master(unsigned int master_id);
```

- 创建过程数据域

```c
ec_domain_t *ecrt_master_create_domain(ec_master_t *master);
```

- 激活主站

```c
int ecrt_master_activate(ec_master_t *master);
```

- 同步主站参考时钟

```c
int ecrt_master_sync_reference_clock_to(ec_master_t *master, uint64_t ref_time);
```

- 同步所有从站时钟

```c
void ecrt_master_sync_slave_clocks(ec_master_t *master);
```

- 配置从站

```c
ec_slave_config_t *ecrt_master_slave_config(ec_master_t *master, uint16_t alias, uint16_t position, uint32_t vendor_id, uint32_t product_code);

```

- 配置从站 PDO 映射

```c
int ecrt_slave_config_pdos(ec_slave_config_t *sc, uint16_t sync_index, const ec_sync_info_t *syncs);
```

- 注册 PDO 条目到指定数据域

```c
int ecrt_slave_config_reg_pdo_entry(ec_slave_config_t *sc, uint16_t index, uint8_t subindex， ec_domain_t *domain, unsigned int *offset);

```

- 为从站配置分布式时钟

```c
int ecrt_slave_config_dc(ec_slave_config_t *sc, uint16_t assign_activate, uint32_t sync0_cycle_time, int32_t sync0_shift, uint32_t sync1_cycle_time, int32_t sync1_shift);
```

## Debug 介绍
## 附录
## FAQ
主要通过 `macctl` 工具进行调试

### 网卡激活失败
当运行 `macctl eqos0 up` 命令时遇到失败，常见原有有四种：

**1. PHY 设备未正常 work**

此时若 RJ45 口灯不闪烁，需检查 PHY 的输入信号如工作时钟、工作电压等是否符合要求。如 RJ45 LED 无明显异常，可以通过下述命令查看 PHY 寄存器进一步确认 PHY 是否正常。
```bash
macctl eqos0 phy-regs           #查看 PHY 通用寄存器
```

**2. RGMAC RXC 无稳定时钟信号输入**

先检查 PHY RXC 端是否输出正常时钟；若该时钟正常，再重点核查板级走线及相关连接的连通性。

**3. PHY 复位时间配置不当**

阅读手册设置合适复位时间。

**4. 识别不到 PHY 设备**

检查 PHY address 是否配置正确。

### 设备收发包存在误码
该现象可能与 TX/RX phase 配置不当有关，建议尝试不同的 phase 参数组合，并结合下述命令确认合适的配置值。
```bash
macctl eqos0 send               #本地手动发送一个特定包，同时在对端抓包，看当前 phase 设置下包是否正常
macctl eqos0 recv               #对端发包，本地手动收包，看当前 phase 设置下收包是否正常
```

### 设备收不到数据包
可先进行 `mac loopback` 测试排除控制器本身故障：

```bash
macctl eqos0 mac-loopback on

macctl eqos0 send

macctl eqos0 recv               #应当收到带 0x20211102 magic number 的包
```

若 `mac loopback` 测试成功，则重点检查 MAC 与 PHY 之间 RXD 信号的连通性，以及采样时序是否满足接口的 setup/hold 约束。

