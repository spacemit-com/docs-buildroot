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

从 `k3.dtsi` 可以确认 SoC 默认定义为：

- `usb2_host`：固定 Host，`maximum-speed = "high-speed"`
- `usb3_porta`：带双 PHY 的可配置端口，板级上通常拿来做 DRD/OTG
- `usb3_portb`：SoC 默认 `dr_mode = "host"`
- `usb3_portc`：SoC 默认 `dr_mode = "host"`
- `usb3_portd`：SoC 默认 `dr_mode = "host"`

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

## 基于原理图进行 USB 方案配置

这一部分是这次补充的重点。K3 的 USB 文档如果只写控制器和 DTS 语法，其实还不够，因为很多板级问题根本不在控制器本身，而在 **原理图怎么把 USB 控制器、PHY、Type-C 芯片、板载 HUB、M.2/BT 模组和 GPIO sideband 信号接起来**。

建议实际做板级移植时，固定按下面这个顺序看：

1. **先看 Block Diagram**：确认每个 USB 控制器最终接到了什么器件；
2. **再看 USB 复合端口关系**：确认某个 USB3 控制器是不是只用了 USB2 部分；
3. **再看外围器件**：例如 Type-C CC 控制器、HUB、电源开关、M.2 模组、BT 模组；
4. **最后再写 DTS / U-Boot / EDK2 配置**。

### 1. 先理解 K3 的 USB 复合端口

K3 上每一个 USB3 控制器，本质上都要按 **复合端口（Companion Port）** 理解：

- `1 * USB3 Companion Port = 1 * USB3 SuperSpeed PHY + 1 * USB2 PHY`

这点非常关键。因为很多板子虽然挂的是 `usb3_portb/c/d` 这种 USB3 控制器，但 **实际原理图只把 USB2 D+/D- 这半边引出来**，SuperSpeed TX/RX 根本没接，或者和 PCIe PHY 复用。

这时 DTS 就不能还按完整 SuperSpeed 端口去写，而应该显式限制：

```dts
maximum-speed = "high-speed";
```

必要时还要删掉默认的双 PHY 绑定，只保留 `usb2-phy`：

```dts
/delete-property/ phys;
/delete-property/ phy-names;
phys = <&usb3_portb_u2phy>;
phy-names = "usb2-phy";
```

这类写法在 `k3_evb.dts`、`k3_deb1.dts`、`k3_gemini_c0.dts` 等板级里都能看到。

### 2. 通过 Block Diagram 判断哪些控制器要使能

这次补看的飞书文档里，这部分写得很对，我把它提炼成 K3 文档里更通用的判断方法。

以 DEB1 这类板子为例，通常可以先从 Block Diagram 得到下面这些结论：

- **USB2.0 Host**：如果接了板载 USB HUB，就需要使能 `usb2_host`，并继续确认 HUB 的复位/上电/VBUS 控制；
- **USB3 PortA**：如果接的是 Type-C DRD 口，就要重点确认 `usb3_porta` + `FUSB301` + role switch + orientation switch；
- **USB3 PortB**：如果接的是 M.2/WWAN 插槽，而且只用了 USB2 通道，就要限制 `maximum-speed = "high-speed"`，同时检查 WWAN 的 `RESET` / `MODULE_TURN_ON` / `W_DISABLE#` 等 GPIO；
- **USB3 PortC**：如果接的是板载蓝牙，通常也只是 USB2 设备，需要限制 high-speed，并补上 BT enable / reset GPIO；
- **USB3 PortD**：如果接的是另一个完整 Type-C / DP Alt Mode 接口，就要继续确认对应 Type-C/PD 芯片（例如 ANX7447）以及相关 GPIO / I2C / mux。

换句话说，**K3 的 USB 方案配置首先是“读原理图 + 列待办”问题，不是先写 DTS 问题。**

### 3. K3 原理图阅读时最容易漏掉的几个点

#### 3.1 某个 USB3 控制器可能只有 USB2 有效

这是 K3 最常见的板级差异之一。

- 可能 SuperSpeed PHY 没引出；
- 可能 SuperSpeed PHY 和 PCIe 共用；
- 可能最终只接了板载 USB2 器件。

这种场景下文档必须强调两件事：

1. **不是所有 `usb3_portX` 都真能跑 SuperSpeed**；
2. **不能默认保留 `usb2-phy + usb3-phy` 的双 PHY 组合**。

#### 3.2 板载器件通常带有 sideband GPIO

比如：

- Type-C 芯片：`irq-gpios`、方向/翻转选择、I2C；
- 板载 HUB：复位脚、电源使能脚、下游端口 VBUS 开关；
- M.2 WWAN：`shutdown-gpios`、`reset-gpios`、`W_DISABLE#`；
- 板载 BT：`shutdown-gpios` 或 `reg_on` / `reset`。

这些 GPIO 如果不配，现象通常不是“驱动 probe 失败”，而是：

- 设备枚举不到；
- 外设不启动；
- 端口没供电；
- Type-C 角色不对；
- 模组上电后没出 USB 设备。

#### 3.3 pinctrl 的 `power-source` 不能忽略

飞书文档里专门提到了某些 Type-C / DIR 引脚的电压域问题，这个提醒很有价值。

例如某些 pinctrl 子节点会显式写：

```dts
power-source = <1800>;
```

这类配置要回到原理图确认实际 IO 电压，不要只看功能复用。

### 4. 以 `usb3_porta` Type-C DRD 为例

如果原理图显示 `usb3_porta` 外接的是 Type-C 接口，而且 CC 芯片是 `FUSB301`，那板级 DTS 一般至少要具备下面几部分：

#### 4.1 FUSB301 所在 I2C 总线

```dts
&i2c1 {
        pinctrl-names = "default";
        pinctrl-0 = <&i2c1_3_cfg>;
        status = "okay";

        tcpc@25 {
                compatible = "onsemi,fusb301";
                reg = <0x25>;
                pinctrl-names = "default";
                pinctrl-0 = <&fusb301_cfg>;
                irq-gpios = <&gpio 2 21 GPIO_ACTIVE_LOW>;
                wakeup-source;
                status = "okay";
        };
};
```

#### 4.2 Type-C connector graph

```dts
connector@0 {
        compatible = "usb-c-connector";
        data-role = "dual";
        power-role = "dual";
        try-power-role = "sink";
        pd-disable;
};
```

#### 4.3 `usb3_porta` role switch

```dts
&usb3_porta {
        dr_mode = "otg";
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        monitor-vbus;
        status = "okay";
};
```

#### 4.4 必要时还要补 SS 方向切换

在 `k3_com260.dts` / `k3_com260_kit_v02.dts` 中还能看到：

- `usb3_phy_switch`
- `fusb301_sw_ep`

这说明如果板级真想把 Type-C 的 SuperSpeed 也做完整，除了 role switch 之外，还要处理 **插头翻转导致的 SS lane 方向切换**。

### 5. 以 `usb2_host` + 板载 HUB 为例

如果 `usb2_host` 接的是板载 HUB，单纯把 Host 控制器开起来通常还不够。还要确认：

- HUB 是否需要复位脚；
- HUB 下游 VBUS 是否需要受控；
- VBUS 开关是主控 GPIO 控，还是 HUB 自己控制。

如果某个 VBUS 开关需要主控 GPIO 拉高才能供电，一个常见 DTS 方案是增加 `regulator-fixed`：

```dts
hub_vbus: regulator-hub-vbus-5v {
        compatible = "regulator-fixed";
        regulator-name = "HUB_VBUS_5V";
        regulator-min-microvolt = <5000000>;
        regulator-max-microvolt = <5000000>;
        gpio = <&gpio 0 18 GPIO_ACTIVE_HIGH>;
        regulator-always-on;
        regulator-boot-on;
        enable-active-high;
};
```

K3 当前 USB 文档之前对这类“板载 HUB/板载供电”的强调还不够，这次一起补上。

### 6. 以 `usb3_portb` 接 M.2 WWAN 为例

飞书文档给的这个例子也很典型，值得吸收到 K3 通用文档里。

如果 `usb3_portb` 接的是 M.2 Key B / WWAN 模组，常见特点是：

- 只用 USB2 通道；
- 需要限制 `maximum-speed = "high-speed"`；
- 需要额外配置 WWAN 的 `MODULE_TURN_ON`、`RESET` 等 sideband GPIO。

典型 DTS 可能会补一个 `rfkill-gpio` 节点：

```dts
rfkill-usb-wwan {
        compatible = "rfkill-gpio";
        label = "m.2 WWAN";
        radio-type = "wwan";
        shutdown-gpios = <&gpio 0 18 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&gpio 0 19 GPIO_ACTIVE_HIGH>;
};
```

这类内容之前我的 USB 通用文档里写得偏“控制器视角”，这次补完以后会更贴近实际板级移植。

### 7. Linux / U-Boot / EDK2 三套配置思路要区分

飞书文档还有一个有用点：把 Linux DTS、U-Boot DTS、EDK2 DSC 三套配置分开讲了。这个对 K3 很重要。

#### Linux DTS

Linux 侧重点是：

- 控制器节点
- PHY 绑定
- role switch graph
- Type-C 芯片
- 板载 GPIO / regulator / rfkill

#### U-Boot DTS

U-Boot 侧当前仍有一些 **nested node** 结构差异，某些属性需要写到控制器内部的 `dwc3@...` 子节点里，例如 `dr_mode`。

#### EDK2 DSC

EDK2 则偏固件配置表方式，例如：

- `PcdUsbHostConfigs.Controller[x].Enable`
- `PcdUsbHostConfigs.Controller[x].MaxSpeed`
- `PcdUsbHostVbusConfigs`
- `PcdUsbHostPinctrlConfigs`

所以 K3 做完整 USB 板级方案时，不能只看 Linux DTS。很多量产板最终还得同步考虑 U-Boot 和 EDK2。

### 8. K3 USB 方案配置的推荐工作流

建议在文档里明确给使用者一个固定流程：

1. **看 Block Diagram**：确认 5 个控制器谁启用；
2. **标记每个端口最终接到什么**：Type-C / HUB / M.2 / BT / 标准 A 口；
3. **判断是否只有 USB2 有效**：决定是否限速为 `high-speed`；
4. **列 sideband GPIO 待办**：reset / enable / power / INT / DIR；
5. **补 pinctrl 和 power-source**；
6. **再写 Linux DTS**；
7. **需要时同步写 U-Boot DTS / EDK2 DSC**；
8. **最后做 Host / Gadget / OTG 联调**。


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
