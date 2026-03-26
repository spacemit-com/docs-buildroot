# PCIe

介绍 K3 平台 PCIe 控制器驱动的能力、设备树配置方法以及调试方式。

## 模块介绍

PCIe（Peripheral Component Interconnect Express）是一种高速串行点对点扩展总线，常用于连接：

- NVMe SSD；
- PCIe Wi‑Fi 模组；
- 网卡、采集卡、AI/加速卡；
- 其他标准 PCIe 外设。

从用户角度看，PCIe 文档最需要回答的是：

- K3 上有几个 PCIe 控制器；
- 当前支持 RC 还是 EP；
- 每个控制器支持多少 lane、最高什么速率；
- DTS 里 `phys`、`phy-names`、`num-lanes`、`max-link-speed`、`pinctrl` 应该怎么配；
- PERST# / CLKREQ# / WAKE# 对应哪组 pin；
- Linux 下怎么看链路训练和设备枚举是否成功。

### 功能介绍

Linux PCIe 软件栈大致分为三层：

1. **PCIe 核心**  
   负责总线枚举、资源分配、配置空间访问和中断管理；
2. **PCIe 主机控制器驱动**  
   负责 SoC 内部 PCIe RC 控制器初始化和链路训练；
3. **PCIe 终端设备驱动**  
   如 NVMe、Wi‑Fi、网卡等具体设备驱动。

K3 当前 PCIe 方案基于 **Synopsys DesignWare PCIe** 框架实现，控制器驱动位于 `drivers/pci/controller/dwc/` 目录中。

### 源码结构介绍

K3 PCIe 相关代码主要位于：

```text
linux-6.18/
|-- drivers/pci/controller/dwc/
|   |-- pcie-designware.c            # DWC PCIe 公共代码
|   |-- pcie-designware-host.c       # DWC Host 公共逻辑
|   |-- pcie-spacemit-k1.c           # SpacemiT PCIe Host 驱动
|   |-- spacemit_pcie_phy.c          # SpacemiT PCIe PHY 驱动
|   `-- Kconfig
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi                      # K3 PCIe / PCIe PHY 节点定义
    |-- k3-pinctrl.dtsi              # PCIe pinctrl 组定义
    `-- k3*.dts                      # 各板级 PCIe 使用示例
```

K3 当前 PCIe 控制器在 DTS 中使用：

```dts
compatible = "spacemit,k1-pcie";
```

PCIe PHY 在 DTS 中使用：

```dts
compatible = "spacemit,k3-pcie-phy";
```

这说明 K3 控制器继续复用 `pcie-spacemit-k1.c` 这套主机驱动，而 PHY 侧则引入了 K3 平台自己的 `spacemit,k3-pcie-phy` 节点。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 工作模式 | 当前文档对应 **RC（Root Complex）模式** |
| 驱动框架 | 基于 DesignWare PCIe Host 框架 |
| 控制器数量 | `pcie0_rc` ~ `pcie4_rc` 共 5 路 |
| PHY 数量 | `phy0` ~ `phy5` 共 6 路 PCIe PHY |
| 最大速率 | `max-link-speed = <3>`，即 Gen3 |
| lane 规模 | 根据控制器不同支持 x8 / x2 / x1，板级可按实际裁剪 |
| 中断 | 支持 `pcie_irq`，同时接入 MSI |
| IOMMU | 节点中包含 `iommu-map` |
| 板级扩展 | 支持 `spacemit,bifurcation-gpios`、`spacemit,device-detect-gpios` 等私有属性 |

### K3 的控制器与 PHY 关系

结合 `k3.dtsi` 可看到：

- `pcie0_rc` 默认可配到更大的 lane 宽度；
- `pcie1_rc`、`pcie2_rc` 为中等规模控制器；
- `pcie3_rc`、`pcie4_rc` 为 x1 控制器；
- PHY 节点单独存在，通过 `phys` / `phy-names` 进行绑定。

也就是说，K3 的 PCIe 设备树配置不能只盯着 RC 节点，还必须同时理解：

- 这个 RC 对应哪几个 PHY；
- 板级 pinctrl 用的是哪一组 PERST#/WAKE#/CLKREQ#；
- 当前板子实际走了多少 lane。

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**。

### CONFIG 配置

K3 PCIe 常见配置包括：

- `CONFIG_PCI`
- `CONFIG_PCIEPORTBUS`
- `CONFIG_PCIE_DW_HOST`
- `CONFIG_PCIE_SPACEMIT_K1`
- `CONFIG_PCI_MSI`

对应菜单路径大致如下：

```text
Device Drivers
    PCI support (PCI [=y])
        PCI controller drivers
            DesignWare-based PCIe controllers
                SpacemiT K1 PCIe controller (host mode) (PCIE_SPACEMIT_K1 [=y])
```

> 注意：Kconfig 名称仍然是 `PCIE_SPACEMIT_K1`，但 K3 当前 DTS 仍在复用 `spacemit,k1-pcie` compatible。

### DTS 配置

#### PCIe PHY 节点

K3 `k3.dtsi` 中先定义了 PCIe PHY，例如：

```dts
phy0: phy-pcie@81d00000 {
        compatible = "spacemit,k3-pcie-phy";
        reg = <0x0 0x81d00000 0x0 0x1000>;
        spacemit,syscon-apb-spare = <&pll>;
        num-lanes = <2>;
        spacemit,phy-id = <0>;
        #phy-cells = <0>;
        status = "disabled";
};
```

几个关键点：

- `num-lanes` 表示这个 PHY 本身支持的 lane 数；
- `spacemit,phy-id` 表示 PHY 编号；
- RC 节点通过 `phys = <&phyX>` 把它们关联起来。

#### RC 控制器节点

以 `pcie1_rc` 为例：

```dts
pcie1_rc: pcie@80400000 {
        compatible = "spacemit,k1-pcie";
        reg = <0x0 0x80400000 0x0 0x00001000>,
              <0x0 0x80500000 0x0 0x00001000>,
              <0x0 0x80700000 0x0 0x00003f20>,
              <0x11 0x80000000 0x0 0x00010000>,
              <0x0 0x82c00000 0x0 0x00001000>;
        reg-names = "dbi", "dbi2", "atu", "config", "link";
        bus-range = <0x00 0xff>;
        max-link-speed = <3>;
        num-lanes = <2>;
        device_type = "pci";
        #address-cells = <3>;
        #size-cells = <2>;
        interrupts = <142 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-names = "pcie_irq";
        clocks = <&syscon_apmu CLK_APMU_PCIE_PORTB_BUS>,
                 <&syscon_apmu CLK_APMU_PCIE_PORTB_MSTR>,
                 <&syscon_apmu CLK_APMU_PCIE_PORTB_SLV>;
        clock-names = "dbi", "mstr", "slv";
        resets = <&syscon_apmu RESET_APMU_PCIE_PORTB_DBI>,
                 <&syscon_apmu RESET_APMU_PCIE_PORTB_MSTR>,
                 <&syscon_apmu RESET_APMU_PCIE_PORTB_SLV>;
        reset-names = "dbi", "mstr", "slv";
        linux,pci-domain = <1>;
        msi-parent = <&simsic>;
        spacemit,pcie-port = <1>;
        spacemit,apmu = <&syscon_apmu 0x1d0>;
        status = "disabled";
};
```

控制器节点的关键点如下：

| 属性 | 说明 |
| ---- | ---- |
| `compatible` | 当前 K3 使用 `spacemit,k1-pcie` |
| `reg` / `reg-names` | DBI、DBI2、ATU、config、link 等地址空间 |
| `max-link-speed` | 最大链路速率，`<3>` 表示 Gen3 |
| `num-lanes` | 链路 lane 数 |
| `ranges` | I/O、MEM、prefetchable MEM 地址空间映射 |
| `interrupts` | 控制器中断 |
| `clocks` / `resets` | 控制器时钟与复位 |
| `linux,pci-domain` | PCI domain 编号 |
| `msi-parent` | MSI 控制器 |
| `spacemit,pcie-port` | 平台私有端口号标识 |
| `spacemit,apmu` | 平台 PMU/APMU 配置寄存器引用 |
| `iommu-map` | 和 IOMMU 的地址映射关系 |

#### pinctrl 配置

K3 的 pinctrl 中为 PCIe 定义了多组复用，例如：

- `pcie0_1_cfg`
- `pcie1_1_cfg`
- `pcie2_2_cfg`
- `pcie3_1_cfg`
- `pcie4_1_cfg`

这些 pin 组一般包含：

- `PERST#`
- `WAKE#`
- `CLKREQ#`

例如：

```dts
pcie4_1_cfg: pcie4-1-cfg {
        pcie4-0-pins {
                pinmux = <K3_PADCONF(76, 5)>,   /* pcie4 perst */
                         <K3_PADCONF(77, 5)>,   /* pcie4 wake */
                         <K3_PADCONF(78, 5)>;   /* pcie4 clkreq */
        };
};
```

因此用户在新板 bring-up 时，一定要先根据原理图确认：

- 用的是哪一路 PCIe；
- PERST#/WAKE#/CLKREQ# 实际接到了哪组 pad；
- 该选哪组 `pcieX_Y_cfg`。

#### 板级启用示例

K3 EVB 上的 `pcie2_rc` 典型配置如下：

```dts
&pcie2_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie2_2_cfg>;
        phys = <&phy2>, <&phy3>;
        phy-names = "phy0", "phy1";
        status = "okay";
};
```

这说明：

- 该 RC 使用两路 PHY；
- PHY 通过 `phys` 和 `phy-names` 绑定；
- 板级 pinmux 选择了 `pcie2_2_cfg`。

#### lane 数裁剪示例

K3 不同板级会根据实际连接裁剪 `num-lanes`。例如：

```dts
&pcie0_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie0_1_cfg>;
        phys = <&phy0>;
        phy-names = "phy0";
        num-lanes = <2>;
        status = "okay";
};
```

而在另一些板级中，同一路控制器可能会配置成：

```dts
num-lanes = <4>;
```

这说明用户不能只看 `k3.dtsi` 默认值，还必须结合板级连接实际确定：

- 真实接了几个 lane；
- 是否使用了 bifurcation；
- 绑定了几个 PHY。

#### 板级私有扩展属性

在 `k3_deb1.dts` 中还能看到：

```dts
spacemit,bifurcation-gpios = <&gpio 2 25 GPIO_ACTIVE_HIGH>;
spacemit,device-detect-gpios = <&gpio 2 25 GPIO_ACTIVE_HIGH>;
```

这说明 K3 板级 PCIe 还会额外使用一些辅助 GPIO 来做：

- bifurcation 模式选择；
- 外设存在检测；
- 某些特定板级的控制逻辑。

这类属性不是所有板子都要配，但在用户迁移 DTS 时需要特别注意，不能简单漏掉。

## 驱动实现角度的几个关键信息

结合 `drivers/pci/controller/dwc/pcie-spacemit-k1.c`，可以确认：

- 驱动通过 `of_match` 匹配 `spacemit,k1-pcie`；
- 会读取 `spacemit,pcie-port`；
- 会读取 `num-lanes`，无效时回落到默认值；
- 使用 `dw_pcie_host_init()` 走标准 DWC Host 初始化流程；
- 通过 LTSSM 相关寄存器控制链路训练和状态机；
- 驱动日志里会输出与 LTSSM / L2 进入超时相关的错误信息。

这意味着在用户调试链路失败时，`dmesg` 中的 LTSSM 相关报错很有价值。

## 接口介绍

### 用户空间接口

PCIe 在 Linux 中不会表现成单一 `/dev/pcieX` 设备，而是通过 PCI 总线统一管理。用户最常用的观察入口是：

- `lspci`
- `lspci -vv`
- `/sys/bus/pci/devices`
- `dmesg`

### 常用命令

查看 PCIe 设备拓扑：

```bash
lspci
```

查看某个设备详细信息：

```bash
lspci -vvvs 0001:01:00.0
```

查看 sysfs：

```bash
ls /sys/bus/pci/devices
```

如果接的是 NVMe：

```bash
nvme list
lsblk
```

## Debug 介绍

### 1. 查看控制器是否枚举成功

```bash
dmesg | grep -Ei "pcie|pci"
lspci
```

如果 RC probe 成功且链路训练正常，`lspci` 中应能看到：

- PCI bridge；
- 终端设备（如 NVMe、Wi‑Fi 模组等）。

### 2. 检查链路训练与 LTSSM 相关报错

K3 驱动里有 LTSSM 相关状态检查，因此建议重点关注：

```bash
dmesg | grep -Ei "LTSSM|pcie|link"
```

若出现：

- link training timeout；
- LTSSM 异常；
- L2 entry timeout；

通常应优先回到硬件连接和 PHY 绑定侧排查。

### 3. 检查设备是否真正枚举到

即使 RC 起了，如果终端设备没枚举出来，也需要继续区分：

- 是链路根本没起来；
- 还是链路起来了，但对端设备没供电 / 没出 reset / 没被扫描到。

这时可重点检查：

- `phys` / `phy-names` 是否正确；
- `num-lanes` 是否和硬件一致；
- `pinctrl` 是否选对；
- PERST# / CLKREQ# / WAKE# 是否接对；
- 终端设备供电是否正常。

### 4. 典型功能测试

#### NVMe 设备检查

```bash
lspci | grep -i nvme
nvme list
lsblk
```

#### NVMe 顺序读测试

```bash
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

#### NVMe 顺序写测试

```bash
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

## FAQ

### 1. 为什么 `lspci` 里看不到终端设备？

常见原因包括：

- RC 控制器没启用；
- PHY 没有正确绑定；
- `num-lanes` 配错；
- pinctrl 选错，导致 PERST#/CLKREQ#/WAKE# 不对；
- 对端设备没供电；
- 对端设备一直处于 reset。

### 2. 为什么同一路控制器在不同板子上 `num-lanes` 不一样？

因为 K3 平台支持根据板级连线裁剪 lane 规模，不同板子可能把同一路控制器接成 x1 / x2 / x4。文档和 DTS 都必须以板级实际硬件为准。

### 3. 为什么 K3 里控制器 compatible 还是 `spacemit,k1-pcie`？

因为 K3 当前 SDK 继续复用了 `pcie-spacemit-k1.c` 这套控制器驱动。也就是说，驱动框架延续了 K1 的 compatible，但 PHY 和板级资源组织已经是 K3 的实现方式。

### 4. 用户调试 PCIe 时最优先看什么？

建议顺序：

1. `dmesg` 看 RC probe 是否成功；
2. `dmesg` 看 link / LTSSM 是否异常；
3. `lspci` 看是否枚举出桥和终端设备；
4. 再回到 pinctrl / PHY / lane / reset / 电源逐项排查。
