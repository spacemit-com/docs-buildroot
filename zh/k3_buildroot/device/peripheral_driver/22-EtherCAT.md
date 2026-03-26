# EtherCAT

介绍 K3 平台 EtherCAT 主站相关驱动结构、DTS 配置方式、与 K3 GMAC 的关系，以及 bring-up / 调试时需要注意的关键点。

## 模块介绍

K3 SDK 里的 EtherCAT 不是简单依赖用户态软件包，而是**内核里已经集成了一套 EtherCAT 主站 + 设备接口实现**。

从当前 SDK 代码看，K3 这条线的基本结构是：

- EtherCAT 总开关：`CONFIG_ETHERCAT`
- IgH EtherCAT Master：`CONFIG_EC_MASTER`
- K3 GMAC 设备接口：`CONFIG_EC_K3_GMAC`

也就是说，K3 当前文档应该围绕：

1. **EtherCAT Master 子系统**
2. **K3 GMAC 专用 EtherCAT device 驱动**
3. **DTS 中 `ethercat_master` 与 `eth0` 的绑定关系**

来理解，而不是写成“普通网卡如何跑以太网”。

## 功能介绍

EtherCAT 是一个运行在二层以太网帧上的工业实时总线协议。K3 当前 SDK 提供的是：

- **主站（Master）侧能力**
- 基于 **K3 GMAC** 的低时延设备接口
- 在主站和 K3 GMAC 之间建立直接的数据收发链路

### K3 当前实现的总体拓扑

从 `k3.dtsi` 和 `drivers/net/ethercat/` 可以对出这样一条主线：

```text
IgH EtherCAT Master
        |
        v
K3 EtherCAT device interface (EC_K3_GMAC)
        |
        v
K3 GMAC controller (`spacemit,k3-gmac`)
        |
        v
PHY / RJ45 / EtherCAT slave network
```

所以这套实现不是“单独一颗 EtherCAT 控制器 IP”，而是：

- 用 K3 SoC 自带的 GMAC
- 配上 EtherCAT device 适配层
- 再由 IgH EtherCAT Master 来完成总线主站功能

## 源码结构介绍

K3 EtherCAT 相关代码位于：

```text
linux-6.18/
|-- drivers/net/ethercat/
|   |-- Kconfig
|   |-- Makefile
|   |-- master/
|   |   `-- Kconfig
|   `-- device/
|       |-- Kconfig
|       `-- K3/
|           |-- Makefile
|           |-- stmmac_main-ethercat.c
|           |-- stmmac_platform-ethercat.c
|           |-- dwmac-spacemit-ethqos-ethercat.c
|           `-- ...
`-- arch/riscv/boot/dts/spacemit/
    `-- k3.dtsi
```

### 1. 总入口

顶层 Kconfig：

```text
menuconfig ETHERCAT
    bool "EtherCAT support"
    depends on NET && NETDEVICES && ETHERNET
```

这说明 EtherCAT 在 K3 SDK 中是**内核内建的一套独立功能树**。

### 2. Master 侧

Master Kconfig 位于：

```text
drivers/net/ethercat/master/Kconfig
```

关键项：

```text
config EC_MASTER
    tristate "EtherCAT master (IgH)"
```

并且还能看到：

- `EC_MASTER_RUN_ON_CPU`
- `EC_MASTER_DEBUG_LEVEL`

这说明当前 SDK 不只是把主站代码塞进去了，还提供了：

- 主站运行 CPU 绑定
- 调试级别配置

### 3. Device 侧

设备接口 Kconfig 位于：

```text
drivers/net/ethercat/device/Kconfig
```

关键项：

```text
menuconfig EC_DEVICE
    bool "EtherCAT device"

config EC_GENERIC
    tristate "Generic EtherCAT device support"

config EC_K3_GMAC
    tristate "Spacemit K3 GMAC EtherCAT device support"
    depends on SOC_SPACEMIT_K3 && STMMAC_ETH!=y
    select PAGE_POOL
```

这里最关键的是：

- **K3 有明确的 `EC_K3_GMAC` 配置项**
- 它依赖 `SOC_SPACEMIT_K3`
- 并且要求 `STMMAC_ETH != y`

这个依赖非常关键，意味着：

- 你不能让普通 STMMAC 主驱动和 EtherCAT 版 K3 GMAC 驱动同时以冲突方式占同一块硬件

### 4. K3 专用 device 驱动目录

K3 的 EtherCAT 设备侧实现不在普通 `stmmac` 目录，而在：

```text
drivers/net/ethercat/device/K3/
```

当前能直接看到：

- `stmmac_main-ethercat.c`
- `stmmac_platform-ethercat.c`
- `dwmac-spacemit-ethqos-ethercat.c`
- 以及一整套 `dwmac*` / `stmmac*` 的 EtherCAT 变体文件

Makefile 里也能明确看到：

```text
obj-$(CONFIG_EC_K3_GMAC) += stmmac.o
```

并由大量 `*-ethercat.c` 文件组成一个专门的 K3 EtherCAT STMMAC 变体。

这说明 K3 不是简单“复用普通网卡驱动”，而是为 EtherCAT 场景做了**专门裁剪/分支化适配**。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| 支持 EtherCAT 总开关 | `CONFIG_ETHERCAT` |
| 支持 IgH 主站 | `CONFIG_EC_MASTER` |
| 支持 K3 GMAC 专用设备接口 | `CONFIG_EC_K3_GMAC` |
| DTS 中有独立 `ethercat_master` 节点 | 用来定义主站实例与主设备绑定 |
| 主设备可直接绑定到 `eth0` | `main-device = <&eth0>;` |
| K3 设备侧基于 STMMAC EtherCAT 变体实现 | `drivers/net/ethercat/device/K3/` |
| 设备接口与普通 STMMAC 存在互斥关系 | `depends on ... STMMAC_ETH!=y` |

### 当前 DTS 中已确认的核心主线

`k3.dtsi` 里可以直接看到：

```dts
ec_master: ethercat_master {
        compatible = "spacemit,igh-ec-master";
        master-count = <1>;
        status = "disabled";

        master0 {
                main-device = <&eth0>;
        };
};
```

这段内容非常关键，说明当前 K3 的 EtherCAT Master 配置模型是：

- 有一个主站节点：`ethercat_master`
- 兼容串：`spacemit,igh-ec-master`
- 当前默认主站实例数：`master-count = <1>`
- `master0` 绑定主设备：`main-device = <&eth0>`

也就是说，**K3 当前主站默认是挂在 `eth0` 上跑的**。

### 与 K3 GMAC 的关系

同一份 `k3.dtsi` 里，`eth0` 节点为：

```dts
eth0: ethernet@cac80000 {
    compatible = "spacemit,k3-gmac", "snps,dwmac-5.10a";
    ...
};
```

而 `drivers/net/ethercat/device/K3/dwmac-spacemit-ethqos-ethercat.c` 中又能直接看到：

- `compatible = "spacemit,k3-gmac"`
- `spacemit_ethqos_probe()`
- `module_platform_driver(spacemit_ethqos_driver)`

这就把 DTS 和驱动完全对上了：

- DTS 里 `eth0` / `eth1` 使用 `spacemit,k3-gmac`
- EtherCAT 设备驱动同样匹配 `spacemit,k3-gmac`
- 设备侧底座是 K3 的 GMAC + STMMAC EtherCAT 变体

## 配置介绍

主要包括：

1. EtherCAT 总开关
2. Master 配置
3. K3 GMAC 设备接口配置
4. DTS 主站与主设备绑定

### 1. CONFIG 配置

关键配置项包括：

- `CONFIG_ETHERCAT`
- `CONFIG_EC_MASTER`
- `CONFIG_EC_DEVICE`
- `CONFIG_EC_K3_GMAC`

其中：

- `CONFIG_EC_MASTER` 对应 IgH 主站
- `CONFIG_EC_K3_GMAC` 对应 K3 GMAC 专用 EtherCAT 设备接口

### 2. 一个很重要的互斥约束

`EC_K3_GMAC` 的 Kconfig 中明确写了：

```text
depends on SOC_SPACEMIT_K3 && STMMAC_ETH!=y
```

这条约束的实际含义是：

- 如果你要让 K3 的 GMAC 跑 EtherCAT 专用设备接口，普通 STMMAC 主驱动不能以冲突方式抢占同一硬件
- bring-up 时一定要先确认：你当前要的是**普通以太网**，还是 **EtherCAT 主站专用链路**

这在文档里必须强调，不然很容易配出“编译能过、运行谁都起不来”的状态。

### 3. DTS 配置

当前 K3 默认 DTS 里主站节点是 disabled：

```dts
ec_master: ethercat_master {
        compatible = "spacemit,igh-ec-master";
        master-count = <1>;
        status = "disabled";

        master0 {
                main-device = <&eth0>;
        };
};
```

所以如果板子要实际启用 EtherCAT，至少要在板级 DTS 中把它打开，例如：

```dts
&ec_master {
	status = "okay";
};
```

同时要确认 `main-device` 绑定的是你真正想拿来跑 EtherCAT 的 GMAC。

### 4. 主设备选择

当前默认是：

```dts
main-device = <&eth0>;
```

这不是唯一逻辑上可能的写法，但在当前 K3 默认 DTS 里，**已经明确给出的就是 `eth0`**。

如果后续板级设计想改成别的 GMAC 口，也要确保：

- 对应口的 `compatible`、clock、reset、irq、phy 配置都完整；
- EtherCAT 主站节点与实际 GMAC 口绑定一致。

## 驱动实现要点

### 1. K3 设备侧基于 STMMAC EtherCAT 分支

从 `drivers/net/ethercat/device/K3/Makefile` 可以看出：

- 它不是一个小薄封装
- 而是带了一整套 `stmmac_*_ethercat.c` / `dwmac*_ethercat.c`

这表明 K3 当前 EtherCAT 是沿着 **GMAC/STMMAC 深度适配** 的思路做的。

### 2. probe 入口

在 `dwmac-spacemit-ethqos-ethercat.c` 中可以直接看到：

- `spacemit_ethqos_probe()`
- `devm_stmmac_probe_config_dt()`
- `devm_stmmac_pltfr_probe()`
- `compatible = "spacemit,k3-gmac"`

所以它本质上还是：

- 平台 glue driver
- 读取 DTS
- 再进入 STMMAC EtherCAT 设备实现

### 3. EtherCAT 收发链路已经在网卡驱动里打通

在 `stmmac_main-ethercat.c` 中可以直接看到：

- `priv->ecdev_`
- `ecdev_set_link()`
- `ecdev_receive()`
- `get_ecdev(priv)`

这说明 K3 GMAC 设备侧驱动已经把：

- 链路状态
- RX 接收
- 主站设备对象

都接到了 EtherCAT 框架里，而不是用户自己在外面再套一层。

## 使用与调试介绍

### 1. 先确认你是在走“EtherCAT 模式”还是“普通以太网模式”

这是第一步。

如果目标是 EtherCAT，就要优先确认：

- `CONFIG_ETHERCAT`
- `CONFIG_EC_MASTER`
- `CONFIG_EC_K3_GMAC`
- `STMMAC_ETH` 没和它打架

### 2. 先看 DTS 里的 `ethercat_master`

优先检查：

- `compatible = "spacemit,igh-ec-master"`
- `master-count = <1>`
- `main-device = <&eth0>`
- `status = "okay"` 是否在板级被打开

如果主站节点还是 disabled，那后面用户态工具基本也不可能正常起来。

### 3. 再看 K3 GMAC 口是不是按 EtherCAT 目标口配置好的

这里至少要看：

- `eth0` 是否启用
- PHY 是否接对
- `phy-mode` 是否正确
- clock / reset / irq 是否完整
- 网口实际链路是否能建立

因为当前主站最终还是依赖 K3 GMAC 口收发帧。

### 4. 看启动日志

先查：

```shell
dmesg | grep -i ethercat
```

再查：

```shell
dmesg | grep -i -e stmmac -e gmac
```

重点确认：

- EtherCAT master 是否注册成功
- K3 GMAC EtherCAT device driver 是否 probe 成功
- `eth0` 相关链路日志是否正常

### 5. 关注链路状态同步

因为驱动里显式走了：

- `ecdev_set_link()`

所以如果你遇到“主站起来了但总线一直不工作”，除了看主站本身，还要看：

- 网口链路状态有没有被正确感知
- PHY 协商是否正常
- 物理连线 / 从站供电是否正常

## FAQ

### 1. K3 当前 EtherCAT 是真有内核实现，还是只有文档占位？

是**真有实现**。

当前能直接确认到：

- `drivers/net/ethercat/`
- `CONFIG_EC_MASTER`
- `CONFIG_EC_K3_GMAC`
- `k3.dtsi` 里的 `ethercat_master`

所以这不是空占位文档。

### 2. K3 EtherCAT 当前是基于哪块硬件口？

从默认 DTS 看，当前主站默认绑定：

- `main-device = <&eth0>`

也就是默认主口是 `eth0`。

### 3. K3 EtherCAT 是不是独立控制器？

不是当前文档里这种理解方式。当前实现更像是：

- **IgH Master + K3 GMAC EtherCAT 设备接口 + K3 GMAC 硬件**

### 4. 这篇最该记住哪条经验？

**先把“普通网卡模式”和“EtherCAT 专用模式”分清楚，再配内核和 DTS。**

K3 这里最大的坑，不是“主站软件不会用”，而是前面配置阶段就把普通 STMMAC 和 EtherCAT K3 GMAC 关系配乱了。
