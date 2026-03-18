sidebar_position: 1

# USB 通用开发指南

介绍 K3 USB 的基本功能、驱动组成、设备树配置方法以及常见调试方式。

适用范围：SpacemiT Linux 6.18（K3 SDK）

## 模块介绍

K3 的 USB 子系统比 K1 明显更复杂，不能只按 K1 的“三个控制器”来理解。根据 `k3.dtsi`、各板级 DTS 和实际驱动，可以确认 K3 当前至少包含：

- **1 路 USB2.0 Host**：`usb2_host`
- **4 路 USB3.x 控制器端口**：`usb3_porta`、`usb3_portb`、`usb3_portc`、`usb3_portd`
- **1 路可切换 DRD/OTG 端口**：通常是 `usb3_porta`
- **多路 Host 端口**：通常是 `usb3_portb/c/d`
- **独立 USB2 PHY**：`spacemit,k3-usb2-phy`
- **独立 USB3 PHY**：`spacemit,k3-usb3-phy`
- **Type-C / role switch / orientation switch 相关逻辑**：
  - `usb-role-switch`
  - `usb-c-connector`
  - `onsemi,fusb301`
  - `spacemit,k3-typec-switch`

也就是说，K3 上的 USB 不是简单“一个 USB3 控制器”，而是一个由 **DWC3 控制器 + USB2 PHY + USB3 PHY + Type-C 检测/切换 + 板级 GPIO/pinctrl** 组成的完整系统。

### K3 实际控制器分布

从 `k3.dtsi` 可以确认：

- `usb2_host`：固定 Host，`maximum-speed = "high-speed"`
- `usb3_porta`：默认作为 DRD/OTG 使用的主要端口
- `usb3_portb`：固定 Host
- `usb3_portc`：固定 Host
- `usb3_portd`：固定 Host

其中控制器节点当前仍使用：

```dts
compatible = "spacemit,k1-dwc3";
```

这说明 **K3 继续复用了 K1 的 DWC3 glue compatible**，但 PHY、Type-C 和板级连接方式已经是 K3 自己的实现。

### 功能介绍

Linux 中，USB 主要支持以下三种角色：

- **Host**：外接 U 盘、键盘、鼠标、摄像头、网卡等 USB 设备；
- **Device / Peripheral**：开发板作为 U 盘、网卡、ADB、UVC 摄像头等 USB 外设接入上位机；
- **OTG / DRD**：可在 Host 和 Device 之间切换。

K3 实际上覆盖了这三种场景：

- `usb2_host`、`usb3_portb/c/d` 主要是 Host；
- `usb3_porta` 可配置为 `host` / `peripheral` / `otg`；
- 板级若加上 Type-C CC 控制器（如 FUSB301），就能做真实的 Type-C DRD 角色切换。

### 功能框架说明

#### USB Host

USB Host 角色驱动框架可以分为：

- **Host Controller Driver**：如 DWC3 Host / xHCI；
- **USB Core**：负责设备枚举、URB、配置描述符处理；
- **Class Driver**：如 Storage、HID、UVC、CDC ECM/NCM 等。

#### USB Device

USB Device 角色驱动框架可以分为：

- **UDC Driver**：DWC3 gadget / UDC；
- **UDC Core**；
- **Composite / ConfigFS**；
- **Function Driver**：如 mass-storage、RNDIS、NCM、UVC、ADB 等。

#### Type-C / Role Switch

K3 的 Type-C 方案中，还会多出一层：

- **Type-C Port Controller（如 FUSB301）**；
- **usb-role-switch**；
- **orientation / SuperSpeed switch**；
- **USB-C connector graph**。

这层是 K3 相比 K1 很值得单独写清楚的点。

## 源码结构介绍

K3 USB 相关代码主要位于：

```text
linux-6.18/drivers/usb/
|-- dwc3/
|   |-- core.c
|   |-- drd.c
|   |-- gadget.c
|   |-- host.c
|   `-- dwc3-generic-plat.c
|-- host/
|   `-- xhci-plat.c
`-- typec/
    |-- fusb301.c
    |-- class.c
    |-- mux/
    `-- altmodes/
```

结合 DTS 可见，K3 当前主要依赖的是：

- **DWC3 Core**：Host / Gadget / DRD 主逻辑
- **平台 DWC3 glue compatible**：`spacemit,k1-dwc3`
- **Type-C Port Controller**：`fusb301.c`
- **Type-C switch / orientation 处理**：与 `spacemit,k3-typec-switch` 配合

K3 的 USB PHY 节点由 DTS 中的专用 compatible 描述：

- `spacemit,k3-usb2-phy`
- `spacemit,k3-usb3-phy`
- `spacemit,k3-typec-switch`（用于部分 DRD/Type-C 场景）

## 关键特性

### 特性

| 特性 | 特性说明 |
| :----- | :---- |
| 控制器数量 | `usb2_host` + `usb3_porta/b/c/d` |
| 角色支持 | Host / Device / OTG |
| USB2 Host | `usb2_host` 固定 Host |
| DRD 端口 | `usb3_porta` 可做 OTG / Device / Host |
| 固定 Host 端口 | `usb3_portb/c/d` |
| PHY 组织 | USB2 PHY 与 USB3 PHY 分离 |
| Type-C 支持 | `usb-role-switch` + `fusb301` + `usb-c-connector` |
| SuperSpeed 方向切换 | 部分板级通过 `spacemit,k3-typec-switch` + graph 端点连接 |
| 板级约束 | 大量依赖 `pinctrl` / `monitor-vbus` / role switch graph |

### K3 相比 K1 值得特别说明的点

1. **USB 端口数明显更多**  
   K3 不是 K1 那种“USB0/USB1/USB2-3”固定描述，而是明确分成：
   - `usb2_host`
   - `usb3_porta`
   - `usb3_portb`
   - `usb3_portc`
   - `usb3_portd`

2. **PHY 拆分更清晰**  
   每个 USB3 端口通常都包含：
   - 一个 `usb2-phy`
   - 一个 `usb3-phy`

3. **Type-C 建模更完整**  
   板级 DTS 中已经存在完整的：
   - `tcpc@25`
   - `usb-c-connector`
   - `ports/endpoint`
   - `usb-role-switch`

4. **SuperSpeed 切换图连接真实存在**  
   在 `k3_com260.dts` / `k3_com260_kit_v02.dts` 中，`fusb301` 甚至通过额外 endpoint 与 `usb3_porta_u3phy` 的 `usb3_phy_switch` 相连，说明板级还处理了 Type-C 插头方向与 SS lane 切换。

5. **板级大量使用 maximum-speed 限速**  
   多个板子把部分 USB3 端口限制为：

   ```dts
   maximum-speed = "high-speed";
   ```

   这说明实际产品并不一定把所有端口都做成 SuperSpeed，文档必须按板级事实说明。

## 配置介绍

主要包括：

- DWC3 / xHCI / Gadget / Type-C 内核配置；
- PHY 节点与控制器节点配置；
- Type-C role switch 图连接；
- 板级 Host / OTG / Device 具体使用方式。

### CONFIG 配置

K3 USB 常见配置包括：

- `CONFIG_USB`
- `CONFIG_USB_XHCI_HCD`
- `CONFIG_USB_XHCI_PLATFORM`
- `CONFIG_USB_DWC3`
- `CONFIG_USB_DWC3_HOST`
- `CONFIG_USB_DWC3_GADGET`
- `CONFIG_USB_DWC3_DUAL_ROLE`
- `CONFIG_USB_ROLE_SWITCH`
- `CONFIG_TYPEC`
- `CONFIG_TYPEC_FUSB301`
- `CONFIG_USB_GADGET`
- `CONFIG_USB_CONFIGFS`

如果板子需要 USB Gadget 功能，还需要打开对应的 configfs functions，例如：

- `CONFIG_USB_CONFIGFS_ACM`
- `CONFIG_USB_CONFIGFS_RNDIS`
- `CONFIG_USB_CONFIGFS_NCM`
- `CONFIG_USB_CONFIGFS_MASS_STORAGE`
- `CONFIG_USB_CONFIGFS_UVC`

## DTS 配置

K3 USB 建议按下面几个层次理解：

1. **USB PHY 节点**
2. **DWC3 控制器节点**
3. **role switch / Type-C graph**
4. **板级 Hub / 外设 / pinctrl / power**

### 1. USB2 Host 节点

K3 `usb2_host` 是固定 Host：

```dts
usb2_host_u2phy: phy@c0a20000 {
        compatible = "spacemit,k3-usb2-phy";
        reg = <0x0 0xc0a20000 0x0 0x200>;
        #phy-cells = <0>;
        status = "disabled";
};

usb2_host: usb2@c0a00000 {
        compatible = "spacemit,k1-dwc3";
        reg = <0x0 0xc0a00000 0x0 0x10000>;
        phys = <&usb2_host_u2phy>;
        phy-names = "usb2-phy";
        phy_type = "utmi";
        dr_mode = "host";
        maximum-speed = "high-speed";
        status = "disabled";
};
```

板级启用时通常很简单：

```dts
&usb2_host_u2phy {
        status = "okay";
};

&usb2_host {
        status = "okay";
};
```

### 2. USB3 DRD 端口 `usb3_porta`

K3 的 `usb3_porta` 是最值得重点写的端口。其基础定义为：

```dts
usb3_porta_u2phy: phy@cad20000 {
        compatible = "spacemit,k3-usb2-phy";
        ...
};

usb3_porta_u3phy: phy@cad30000 {
        compatible = "spacemit,k3-usb3-phy", "spacemit,k3-typec-switch";
        ...
};

usb3_porta: usb3@cad00000 {
        compatible = "spacemit,k1-dwc3";
        phys = <&usb3_porta_u3phy>, <&usb3_porta_u2phy>;
        phy-names = "usb3-phy", "usb2-phy";
        ...
};
```

几个关键点：

- `usb3_porta` 同时绑了 USB2/USB3 两个 PHY；
- `usb3_porta_u3phy` 额外带了 `spacemit,k3-typec-switch`；
- 这是板级做 DRD / Type-C 的核心端口。

### 3. 固定 Host 端口 `usb3_portb/c/d`

例如：

```dts
usb3_portb: usb3@81400000 {
        compatible = "spacemit,k1-dwc3";
        phys = <&usb3_portb_u2phy>, <&usb3_portb_u3phy>;
        phy-names = "usb2-phy", "usb3-phy";
        dr_mode = "host";
        status = "disabled";
};
```

`usb3_portc` / `usb3_portd` 结构类似。注意：

- 默认在 SoC dtsi 中就是 `dr_mode = "host"`；
- 某些板子会把它们限速为 `high-speed`；
- 某些板级还会在 Host 端口下面挂板载 HUB。

例如 `k3_com260.dts` 中：

```dts
&usb3_portb {
        status = "okay";
        #address-cells = <1>;
        #size-cells = <0>;

        hub_2_0: hub@1 {
                compatible = "usb2109,2817";
                reg = <1>;
        };
};
```

这说明文档里不能只停留在“控制器使能”，还要考虑真实板上挂的 downstream HUB / device。

### 4. OTG / role switch 配置

以 `k3_dc_board.dts` 为例：

```dts
&usb3_porta {
        dr_mode = "otg";
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        monitor-vbus;
        status = "okay";

        ports {
                #address-cells = <1>;
                #size-cells = <0>;
                port@0 {
                        reg = <0x0>;
                        porta_role_switch: endpoint {
                                remote-endpoint = <&fusb301_ep>;
                        };
                };
        };
};
```

这说明 DRD 不只是 `dr_mode = "otg"`，还需要：

- `usb-role-switch`
- `role-switch-default-mode`
- `monitor-vbus`
- 和 Type-C 控制器的 graph 连接

### 5. Type-C / FUSB301 配置

例如：

```dts
tcpc@25 {
        compatible = "onsemi,fusb301";
        reg = <0x25>;
        pinctrl-names = "default";
        pinctrl-0 = <&fusb301_cfg>;
        irq-gpios = <&gpio 2 21 GPIO_ACTIVE_LOW>;
        wakeup-source;
        status = "okay";

        typec_0: connector@0 {
                compatible = "usb-c-connector";
                label = "USB-C";
                data-role = "dual";
                power-role = "dual";
                try-power-role = "sink";
                typec-power-opmode = "default";
                pd-disable;

                ports {
                        #address-cells = <0x1>;
                        #size-cells = <0x0>;
                        port@0 {
                                reg = <0x0>;
                                fusb301_ep: endpoint {
                                        remote-endpoint = <&porta_role_switch>;
                                };
                        };
                };
        };
};
```

几个关键点：

- 当前板级使用 `FUSB301` 做 Type-C 角色检测；
- `data-role = "dual"` / `power-role = "dual"`；
- `pd-disable` 说明这里只做 Type-C/role，不做 USB PD 协议；
- 通过 endpoint 与 DWC3 role switch 相连。

### 6. SuperSpeed 方向切换

在 `k3_com260.dts` / `k3_com260_kit_v02.dts` 中还能看到：

```dts
&usb3_porta_u3phy {
        ports {
                port@0 {
                        usb3_phy_switch: endpoint {
                                remote-endpoint = <&fusb301_sw_ep>;
                        };
                };
        };
};
```

同时在 FUSB301 connector 里还有：

```dts
port@1 {
        reg = <0x1>;
        fusb301_sw_ep: endpoint {
                remote-endpoint = <&usb3_phy_switch>;
        };
};
```

这表示 K3 某些 Type-C 板级不只是做 USB2 role switch，还真正把 **Type-C 插头方向和 SuperSpeed PHY switch** 建模进了 DTS。

### 7. pinctrl 配置

K3 `k3-pinctrl.dtsi` 中有大量 USB 相关 pin group，例如：

- `usb30_drd_0_cfg`
- `usb30_drd_1_cfg`
- `usb30_drd_2_cfg`
- `usb30_drd_int_0_cfg`
- `usb30_drd_int_1_cfg`
- `usb30_drd_dir_0_cfg`
- `usb30_drd_dir_1_cfg`
- `usb30_b_drv_0_cfg`
- `usb30_c_drv_0_cfg`
- `usb30_d_drv_0_cfg`
- `usb30h1_drv_0_cfg`
- `usb30h2_drv_0_cfg`

比如：

```dts
usb30_drd_dir_0_cfg: usb30_drd_dir-0-cfg {
        usb30_drd_dir-pins {
                pinmux = <K3_PADCONF(86, 6)>; /* usb30 drd dir */
        };
};
```

USB bring-up 时，pinctrl 选择非常关键，尤其是：

- DRD 的 ID / VBUSON / DIR / INT；
- Host 端口的 `drv` 控制信号；
- Type-C 中断脚。

## 常见板级场景

### 场景一：普通 Host 口

例如 `usb2_host` / `usb3_portb/c/d`：

```text
DWC3(host) -> PHY -> USB-A / 板载 HUB / 外设
```

用户最关心的是：

- `status = "okay"`
- PHY 是否打开
- 是否需要 `maximum-speed = "high-speed"`
- 是否有板载 HUB

### 场景二：Type-C DRD 口

例如 `usb3_porta`：

```text
DWC3(DRD) <-> usb-role-switch <-> FUSB301 <-> usb-c-connector
                           \-> usb3 PHY orientation switch
```

用户最关心的是：

- `dr_mode = "otg"`
- `usb-role-switch`
- `monitor-vbus`
- FUSB301 I2C 节点
- endpoint graph 是否正确

### 场景三：USB Gadget / 烧录 / 调试口

如果某板级把 `usb3_porta` 设成：

```dts
dr_mode = "peripheral";
maximum-speed = "high-speed";
```

那么它就主要作为 USB Device / UDC 来用，可用于：

- ADB
- gadget configfs
- U 盘 gadget
- 网卡 gadget
- UVC gadget

## 调试方法

### 1. 查看控制器和总线枚举

```bash
dmesg | grep -Ei "usb|dwc3|xhci|typec|fusb"
lsusb
```

### 2. 查看 role switch / Type-C 状态

```bash
ls /sys/class/usb_role/
ls /sys/class/typec/
```

如果板级使用 role switch，可进一步查看：

```bash
cat /sys/class/usb_role/*/role
```

### 3. Host 口测试

插入 U 盘后：

```bash
dmesg | tail -n 50
lsusb
lsblk
```

如果是 SuperSpeed，通常会在枚举日志中看到 `5000M` 或 USB 3.x 相关信息；如果被板级限速为 `high-speed`，则只会看到 USB2.0 速率。

### 4. Gadget 口测试

先确认 UDC：

```bash
ls /sys/class/udc
```

然后使用 gadget 脚本或 configfs 配置功能。具体见下一篇《USB Gadget 开发指南》。

### 5. Type-C 相关排查

若 Type-C DRD 不工作，优先检查：

- FUSB301 是否探测成功；
- `irq-gpios` 是否正确；
- `fusb301_ep <-> porta_role_switch` 是否连接正确；
- 若用到了 SS 方向切换，`fusb301_sw_ep <-> usb3_phy_switch` 是否连接正确；
- `usb3_porta_u3phy` 的 pinctrl 是否正确。

## FAQ

### 1. 为什么 K3 USB 文档不能只照 K1 写？

因为 K3 的硬件组织已经明显更复杂：

- USB 端口更多；
- 每个端口的 USB2/USB3 PHY 分离；
- Type-C role switch / connector / orientation switch 已经真实进入 DTS；
- 不同板级对端口速度和角色限制差异很大。

### 2. 为什么 K3 的 DWC3 compatible 还是 `spacemit,k1-dwc3`？

因为 K3 当前继续复用了 K1 的 DWC3 glue compatible；但 USB2 PHY、USB3 PHY、Type-C switch 和板级连接是 K3 的实际实现。

### 3. 为什么有些 USB3 端口在板级 DTS 里却写 `maximum-speed = "high-speed"`？

因为实际产品可能只接出了 USB2 通道，或者板级方案为了兼容/成本/稳定性做了限速。文档必须以板级 DTS 和硬件为准，而不是看到“USB3 控制器”就默认写成 SuperSpeed。

### 4. Type-C 口只用了 FUSB301，为什么还写了 `pd-disable`？

因为 FUSB301 在这些板级里主要承担 CC 检测和角色判断，不承担完整 USB PD 协议协商，所以 DTS 中明确写了 `pd-disable`。
