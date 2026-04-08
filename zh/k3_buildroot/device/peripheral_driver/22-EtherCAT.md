# EtherCAT

介绍 K3 EtherCAT 主站功能和使用方法。

## 模块介绍

K3 SDK 集成 IGH EtherCAT 1.6.8 主站协议栈和定制化实时网卡驱动，提供面向高性能实时通信的 EtherCAT 主站能力，适用于运动控制、伺服驱动和工业机器人等高实时性应用场景。

### 功能介绍

![](static/EtherCAT.png)

EtherCAT 通信系统架构如上图所示，由四个部分构成：
- **应用层：** 用户应用程序，负责实现工业控制逻辑，并通过主站提供的接口与其交互。
- **EtherCAT 主站层：** 负责协议处理、总线拓扑管理、从站配置以及分布式时钟同步等核心功能。
- **EtherCAT 设备驱动层：** 由实时网卡驱动构成，负责 EtherCAT 数据帧的收发。
- **硬件层：** 底层网络接口及相关物理设备。

### 源码结构介绍

EtherCAT 相关代码位于 `drivers/net/ethercat` 目录下：

```bash
|-- device                              # EtherCAT device driver
|   |-- ecdev.h
|   |-- generic
|   |   |-- ec_generic.c                # 通用 EtherCAT 网卡驱动
|   |   `-- Makefile
|   |-- K1
|   |   |-- ec_k1x_emac.c               # K1 芯片千兆以太网 EtherCAT 设备驱动
|   |   |-- ec_k1x_emac.h
|   |   `-- Makefile
|   |-- K3                              # K3 芯片千兆以太网 EtherCAT 设备驱动
|   |   |-- chain_mode-ethercat.c
|   |   |-- common-ethercat.h
|   |   |-- descs_com-ethercat.h
|   |   |-- descs-ethercat.h
|   |   |-- dwmac1000_core-ethercat.c
|   |   |-- dwmac1000_dma-ethercat.c
|   |   |-- dwmac1000-ethercat.h
|   |   |-- dwmac100_core-ethercat.c
|   |   |-- dwmac100_dma-ethercat.c
|   |   |-- dwmac100-ethercat.h
|   |   |-- dwmac4_core-ethercat.c
|   |   |-- dwmac4_descs-ethercat.c
|   |   |-- dwmac4_descs-ethercat.h
|   |   |-- dwmac4_dma-ethercat.c
|   |   |-- dwmac4_dma-ethercat.h
|   |   |-- dwmac4-ethercat.h
|   |   |-- dwmac4_lib-ethercat.c
|   |   |-- dwmac5-ethercat.c
|   |   |-- dwmac5-ethercat.h
|   |   |-- dwmac_dma-ethercat.h
|   |   |-- dwmac_lib-ethercat.c
|   |   |-- dwmac-spacemit-ethqos-ethercat.c
|   |   |-- dwxgmac2_core-ethercat.c
|   |   |-- dwxgmac2_descs-ethercat.c
|   |   |-- dwxgmac2_dma-ethercat.c
|   |   |-- dwxgmac2-ethercat.h
|   |   |-- dwxlgmac2-ethercat.h
|   |   |-- enh_desc-ethercat.c
|   |   |-- hwif-ethercat.c
|   |   |-- hwif-ethercat.h
|   |   |-- Makefile
|   |   |-- mmc_core-ethercat.c
|   |   |-- mmc-ethercat.h
|   |   |-- norm_desc-ethercat.c
|   |   |-- ring_mode-ethercat.c
|   |   |-- stmmac_est-ethercat.c
|   |   |-- stmmac_est-ethercat.h
|   |   |-- stmmac-ethercat.h
|   |   |-- stmmac_ethtool-ethercat.c
|   |   |-- stmmac_fpe-ethercat.c
|   |   |-- stmmac_fpe-ethercat.h
|   |   |-- stmmac_hwtstamp-ethercat.c
|   |   |-- stmmac_main-ethercat.c
|   |   |-- stmmac_mdio-ethercat.c
|   |   |-- stmmac_pci-ethercat.c
|   |   |-- stmmac_pcs-ethercat.h
|   |   |-- stmmac_platform-ethercat.c
|   |   |-- stmmac_platform-ethercat.h
|   |   |-- stmmac_ptp-ethercat.c
|   |   |-- stmmac_ptp-ethercat.h
|   |   |-- stmmac_tc-ethercat.c
|   |   |-- stmmac_vlan-ethercat.c
|   |   |-- stmmac_vlan-ethercat.h
|   |   |-- stmmac_xdp-ethercat.c
|   |   `-- stmmac_xdp-ethercat.h
|   |-- Kconfig
|   `-- Makefile
|-- include
|   |-- config.h                        # 全局配置项和宏定义
|   |-- ecrt.h                          # 用户程序接口
|   |-- ectty.h
|   `-- globals.h                       # 全局变量
|-- Kconfig
|-- Makefile
`-- master                              # IGH EtherCAT 主站实现
    |-- cdev.c
    |-- cdev.h
    |-- coe_emerg_ring.c
    |-- coe_emerg_ring.h
    |-- datagram.c
    |-- datagram.h
    |-- datagram_pair.c
    |-- datagram_pair.h
    |-- debug.c
    |-- debug.h
    |-- device.c
    |-- device.h
    |-- domain.c
    |-- domain.h
    |-- doxygen.c
    |-- eoe_request.c
    |-- eoe_request.h
    |-- ethernet.c
    |-- ethernet.h
    |-- flag.c
    |-- flag.h
    |-- fmmu_config.c
    |-- fmmu_config.h
    |-- foe.h
    |-- foe_request.c
    |-- foe_request.h
    |-- fsm_change.c
    |-- fsm_change.h
    |-- fsm_coe.c
    |-- fsm_coe.h
    |-- fsm_eoe.c
    |-- fsm_eoe.h
    |-- fsm_foe.c
    |-- fsm_foe.h
    |-- fsm_master.c
    |-- fsm_master.h
    |-- fsm_pdo.c
    |-- fsm_pdo_entry.c
    |-- fsm_pdo_entry.h
    |-- fsm_pdo.h
    |-- fsm_sii.c
    |-- fsm_sii.h
    |-- fsm_slave.c
    |-- fsm_slave_config.c
    |-- fsm_slave_config.h
    |-- fsm_slave.h
    |-- fsm_slave_scan.c
    |-- fsm_slave_scan.h
    |-- fsm_soe.c
    |-- fsm_soe.h
    |-- globals.h
    |-- ioctl.c
    |-- ioctl.h
    |-- Kconfig
    |-- mailbox.c
    |-- mailbox.h
    |-- Makefile
    |-- master.c
    |-- master.h
    |-- module.c
    |-- module_of.c
    |-- pdo.c
    |-- pdo_entry.c
    |-- pdo_entry.h
    |-- pdo.h
    |-- pdo_list.c
    |-- pdo_list.h
    |-- reg_request.c
    |-- reg_request.h
    |-- rtdm.c
    |-- rtdm_details.h
    |-- rtdm.h
    |-- rtdm-ioctl.c
    |-- rtdm_xenomai_v3.c
    |-- rt_locks.h
    |-- sdo.c
    |-- sdo_entry.c
    |-- sdo_entry.h
    |-- sdo.h
    |-- sdo_request.c
    |-- sdo_request.h
    |-- slave.c
    |-- slave_config.c
    |-- slave_config.h
    |-- slave.h
    |-- soe_errors.c
    |-- soe_request.c
    |-- soe_request.h
    |-- sync.c
    |-- sync_config.c
    |-- sync_config.h
    |-- sync.h
    |-- voe_handler.c
    `-- voe_handler.h
```

## 关键特性

| 特性 | 特性说明 |
| :-----| :----|
| 自动从站配置 | 支持自动扫描并配置连接的从站设备，简化网络配置 |
| 分布式时钟同步 | 实现小于 1 µs 精度的分布式时钟（DC）同步 |
| 多协议支持 | 支持 CoE、SoE、FoE 等协议 |
| 高实时性能 | 支持 1ms DC 周期，满足大部分工业应用的实时性要求 |
| 多主站组合 | 支持配置多个主站，每个主站可管理两个网络设备：主设备和备用设备 |

## 配置介绍

主要包括 **Kconfig 配置** 和 **DTS 配置**。

### Kconfig 配置

- `ETHERCAT`：在 K3 平台如果要启用 EtherCAT 服务，首先将此选项配置为 `Y`。

```c
menuconfig ETHERCAT
        bool "EtherCAT support"
        depends on NET && NETDEVICES && ETHERNET
        help
          EtherCAT is an industrial real-time fieldbus protocol that runs over
          Ethernet frames (Layer 2). Selecting this option enables the EtherCAT
          master subsystem and the corresponding network device interfaces.
```

- `EC_MASTER`：启用 IgH EtherCAT 主站。
- `EC_MASTER_OF`：启用基于 Device Tree 的主站配置方式，若关闭则回退为通过传统模块参数传递主站配置。
- `EC_MASTER_RUN_ON_CPU`：指定 EtherCAT 主站运行的 CPU 核。
- `EC_MASTER_DEBUG_LEVEL`：设置主站调试信息输出级别。

```c
config EC_MASTER
	tristate "EtherCAT master (IgH)"
	depends on ETHERCAT
	help
	  Say Y or M here to build the IgH EtherCAT master driver.
	  The master is controlled via a character device interface.
	  If unsure, say N.

if EC_MASTER

config EC_MASTER_OF
	bool "Use Device Tree for EtherCAT master setup"
	depends on OF
	default y
	help
	  Use Device Tree to provide EtherCAT master configuration,
	  including master instances and bound Ethernet devices.

	  This mode is intended for platform-integrated setups where
	  EtherCAT master parameters are fixed by the board description,
	  so runtime module parameters are not needed.

	  If disabled, the original upstream module-parameter based
	  configuration path is used instead.

if EC_MASTER_OF

config EC_MASTER_RUN_ON_CPU
	int "Run EtherCAT master on CPU"
	default 1
	range 0 7
	help
	  Set the CPU (0-7) on which to run the EtherCAT master.
	  If unsure, the default is 1.

config EC_MASTER_DEBUG_LEVEL
	int "Debug level for EtherCAT master"
	default 0
	range 0 2
	help
	  Set the debug level for the EtherCAT master driver.
	  0: No debug information.
	  1: Error messages.
	  2: Full debug information.

endif # EC_MASTER_OF

endif # EC_MASTER
```

- `EC_DEVICE`：启用 EtherCAT 设备驱动。
- `EC_GENERIC`：启用通用 EtherCAT 设备驱动。
- `EC_K3_GMAC`：启用 K3 平台专用 EtherCAT 设备驱动。

```c
menuconfig EC_DEVICE
        tristate "EtherCAT device"
        depends on ETHERCAT && EC_MASTER
        help
          Enable EtherCAT device support.

if EC_DEVICE

config EC_GENERIC
        tristate "Generic EtherCAT device support"
        depends on EC_MASTER
        help
          Generic EtherCAT device support using the standard Linux networking
          stack. This is portable but may have higher latency/jitter.

config EC_K1X_GMAC
        tristate "Spacemit K1 GMAC EtherCAT device support"
        depends on EC_MASTER && (SOC_SPACEMIT_K1 || COMPILE_TEST)
        help
          EtherCAT device support for the Spacemit K1X GMAC controller.

config EC_K3_GMAC
        tristate "Spacemit K3 GMAC EtherCAT device support"
        depends on EC_MASTER && SOC_SPACEMIT_K3
        select PAGE_POOL
        help
          EtherCAT device support for Spacemit K3 GMAC controller.

endif # EC_DEVICE
```
> **注：** 对于 K3 平台，默认开启 `ETHERCAT`、`EC_MASTER`、`EC_DEVICE` ，`EC_K3_GMAC` 和 `EC_MASTER_OF`。

另外，为获得更好的实时性能，建议在内核配置中启用 `CONFIG_PREEMPT_RT`。
```c
config PREEMPT_RT
        bool "Fully Preemptible Kernel (Real-Time)"
        depends on EXPERT && ARCH_SUPPORTS_RT && !COMPILE_TEST
        select PREEMPTION
        help
          This option turns the kernel into a real-time kernel by replacing
          various locking primitives (spinlocks, rwlocks, etc.) with
          preemptible priority-inheritance aware variants, enforcing
          interrupt threading and introducing mechanisms to break up long
          non-preemptible sections. This makes the kernel, except for very
          low level and critical code paths (entry code, scheduler, low
          level interrupt handling) fully preemptible and brings most
          execution contexts under scheduler control.

          Select this if you are building a kernel for systems which
          require real-time guarantees.
```
### DTS 配置

在 DTS 中，可通过 `master-count` 配置主站数量，并定义对应的 `master` 子节点，二者需保持一致。每个 `master` 子节点中可通过 `main-device` 和 `backup-device` 分别指定主用和备用设备。
以 `k3.dtsi` 中的默认配置为例，默认定义了 1 个 EtherCAT 主站实例 `master0`，其 `main-device` 绑定为 `eth0`，当前节点状态为 `disabled`：
```c
ec_master: ethercat_master {
        compatible = "spacemit,igh-ec-master";
        master-count = <1>;
        status = "disabled";

        master0 {
                main-device = <&eth0>;
        };
};
```

用户若需使用 EtherCAT，最简单的方式是在方案 DTS 中直接使能 `ec_master` 节点，并将对应网口的 `compatible` 修改为`spacemit,k3-ec-gmac`，配置如下：

```c
&ec_master {
	status = "okay";
};

&eth0 {
	compatible = "spacemit,k3-ec-gmac", "snps,dwmac-5.10a";
}
```

如需修改配置，可以在方案 DTS 中覆盖 `ec_master` 中其他属性，例如配置两个主站：`master0` 和 `master1`，`master0` 使用 `eth1`、 `master1` 使用 `eth0`：
```c
&ec_master {
        master-count = <2>;

        master0 {
                main-device = <&eth1>;
        };

        master1 {
                main-device = <&eth0>;
        };
};

&eth0 {
	compatible = "spacemit,k3-ec-gmac", "snps,dwmac-5.10a";
}

&eth1 {
	compatible = "spacemit,k3-ec-gmac", "snps,dwmac-5.10a";
}
```

## 接口介绍

### API 介绍
下面对一些常用 API 进行介绍：
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

### sysfs

可通过 `/sys/class/EtherCAT/EtherCAT0` 查看主站信息：

```c
/sys/class/EtherCAT/EtherCAT0
.
|-- dev
|-- power
|   |-- autosuspend_delay_ms
|   |-- control
|   |-- runtime_active_time
|   |-- runtime_status
|   `-- runtime_suspended_time
|-- subsystem -> ../../../../class/EtherCAT
`-- uevent

```

- dev：提供主站设备号信息。
- power：管理设备的电源状态。
- subsystem：子系统链接，表明设备属于 EtherCAT 子系统。
- uevent：主站设备号与设备名。

## 测试介绍

EtherCAT 主站测试步骤：
1. 连接从站设备 至主站网口。
2. 上电启动系统，内核自动加载 EtherCAT 主站和实时网卡设备驱动。
3. 主站自动扫描从站，识别成功后输出日志。
4. 主站进入 `PREOP` 状态，等待用户应用程序运行。

启动日志示例如下：

```c
[  966.525910] k1x_ec_emac cac80000.ethernet ecm0 (uninitialized): Link is Up - 100Mbps/Full - flow control off
[  966.535906] EtherCAT 0: Link state of ecm0 changed to UP.
[  966.552545] EtherCAT 0: 1 slave(s) responding on main device.
[  966.558389] EtherCAT 0: Slave states on main device: INIT.
[  966.564036] EtherCAT 0: Scanning bus.
[  966.739197] EtherCAT 0: Bus scanning completed in 176 ms.
[  966.745275] EtherCAT 0: Using slave 0 as DC reference clock.
[  966.756564] EtherCAT 0: Slave states on main device: PREOP.

```

测试用例可基于 [EtherLab 官方示例](https://gitlab.com/etherlab.org/ethercat) 中的 `examples/dc_user/main.c` 进行开发，以下示例在 1 ms DC 通讯周期下，连接 1 个从站进行测试，输出如下：

```c
period         995180 ...    1004620
exec             4680 ...      48215
latency          3125 ...       7421

period         995420 ...    1004875
exec             4755 ...      47106
latency          3268 ...       7354

period         995060 ...    1004510
exec             4898 ...      46892
latency          3415 ...       7288

period         995330 ...    1004790
exec             4528 ...      47936
latency          3372 ...       7410

period         995210 ...    1004685
exec             4680 ...      48524
latency          3291 ...       7398

period         995560 ...    1004980
exec             4712 ...      47680
latency          3446 ...       7305

period         995140 ...    1004565
exec             4586 ...      48312
latency          3218 ...       7442

period         995470 ...    1004890
exec             4864 ...      46975
latency          3364 ...       7386

period         995090 ...    1004470
exec             4638 ...      48744
latency          3482 ...       7468

period         995380 ...    1004760
exec             4781 ...      47296
latency          3337 ...       7349
```

**注：**
- `period`：给出的数值是每秒内通讯周期波动范围。
- `exec`：给出的数值是每秒内主站周期任务执行时间波动范围。
- `latency`：给出的数值是每秒内主站唤醒延迟波动范围。

## FAQ

