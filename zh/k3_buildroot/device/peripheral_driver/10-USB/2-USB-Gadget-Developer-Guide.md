sidebar_position: 2

# USB Gadget 开发指南

适用范围：SpacemiT Linux 6.18，K3 平台。

本文重点介绍 K3 平台在 **USB Device / USB Gadget** 模式下的实际使用方式。写这篇时要特别注意：K3 不是“所有 USB 口都能随便做 gadget”，当前从 DTS 和 USB 拓扑看，**真正适合做 Device / OTG 的核心端口是 `usb3_porta`**，其余端口大多按固定 Host 使用。

## Linux USB Gadget API 框架

### 概述

USB Gadget 允许开发板作为一个 USB 外设接入上位机，例如：

- U 盘
- 网卡
- 串口
- ADB 设备
- UVC 摄像头
- 自定义协议设备

K3 平台中，这条链路自底向上大致如下：

- **USB Device Controller Driver（UDC）**：底层由 DWC3 Gadget/UDC 提供；
- **UDC Core**：向上抽象 USB Device 控制器；
- **Composite Layer**：组合多个 USB function；
- **Function Driver**：如 ACM、RNDIS、NCM、Mass Storage、UVC、HID、FunctionFS；
- **Configfs**：用户空间通过 configfs 组装 gadget；
- **Userspace**：如 ADB、MTP、UVC 应用、定制协议应用等。

K3 这里仍然是标准 Linux USB Gadget 思路，不需要引入私有框架；真正和 K1/K2/K3 差异更大的部分，主要是：

- **哪个控制器能做 gadget**；
- **Type-C / OTG role switch 如何配合**；
- **板级 DTS 是否真的把该端口布成 device/otg**。

### K3 上的实际 Gadget 入口

从 `k3.dtsi` 与各板级 DTS 来看：

- `usb2_host`：固定 Host，不适合按 gadget 文档去用；
- `usb3_portb/c/d`：大多固定 Host；
- `usb3_porta`：K3 上最核心的 **DRD / OTG / Device** 入口。

也就是说，K3 的 gadget 开发，实际应优先围绕：

```dts
&usb3_porta {
        dr_mode = "peripheral";
};
```

或者：

```dts
&usb3_porta {
        dr_mode = "otg";
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        monitor-vbus;
};
```

这点要先写清楚，不然后面用户会误以为任意一个 USB3 口都能直接拿来做 gadget。

## K3 的 UDC / Gadget 基础

### 控制器与驱动基础

K3 当前 USB 控制器节点仍复用：

```dts
compatible = "spacemit,k1-dwc3";
```

但底层 Gadget 能力来自 Linux 标准 **DWC3 Gadget**：

- `drivers/usb/dwc3/gadget.c`
- `drivers/usb/dwc3/drd.c`
- `drivers/usb/dwc3/core.c`

在 `drivers/usb/dwc3/gadget.c` 中可以看到标准的 gadget ops，例如：

- `dwc3_gadget_start`
- `dwc3_gadget_stop`
- `dwc3_gadget_set_speed`
- `usb_gadget_udc_reset()`

因此从软件栈角度看，K3 gadget 并不是一套 SpacemiT 私有 UDC，而是 **基于 DWC3 的标准 Linux UDC**。

### 内核 menuconfig 配置

K3 Gadget 侧至少建议关注这些配置：

```text
CONFIG_USB_GADGET
CONFIG_USB_DWC3
CONFIG_USB_DWC3_GADGET
CONFIG_USB_DWC3_DUAL_ROLE
CONFIG_USB_CONFIGFS
CONFIG_USB_ROLE_SWITCH
```

常见 function 还包括：

```text
CONFIG_USB_CONFIGFS_ACM
CONFIG_USB_CONFIGFS_NCM
CONFIG_USB_CONFIGFS_RNDIS
CONFIG_USB_CONFIGFS_MASS_STORAGE
CONFIG_USB_CONFIGFS_UVC
CONFIG_USB_CONFIGFS_HID
CONFIG_USB_F_FS
```

如果希望调试时多一点信息，也可以打开：

```text
CONFIG_USB_GADGET_DEBUG=y
CONFIG_USB_GADGET_VERBOSE=y
CONFIG_USB_GADGET_DEBUG_FILES=y
CONFIG_USB_GADGET_DEBUG_FS=y
```

### DWC3 mode 选择

`drivers/usb/dwc3/Kconfig` 中 DWC3 的模式选择很明确：

- `USB_DWC3_HOST`
- `USB_DWC3_GADGET`
- `USB_DWC3_DUAL_ROLE`

如果 K3 板级要兼顾 Host 与 Gadget，建议选：

```text
CONFIG_USB_DWC3_DUAL_ROLE=y
```

如果某产品就是固定做 device 口，也可以只选 gadget mode，但对 K3 来说多数开发阶段还是 **Dual Role 更实用**。

## K3 DTS 配置

K3 gadget 文档里最重要的不是 function 本身，而是 **先把板级 USB 角色配对**。

### 1. 固定 Peripheral 场景

`k3_fpga_1x1.dts` 提供了一个很典型的固定 device 配置：

```dts
&usb3_porta {
        status = "disabled";
        maximum-speed = "high-speed";
        dr_mode = "peripheral";
        /delete-property/ phy_type;
        phy_type = "utmi_wide";
};
```

这里至少说明几点：

1. `usb3_porta` 可以直接设成 `peripheral`；
2. 某些平台会把 gadget 口限制为 `high-speed`；
3. 某些平台会改 `phy_type`，例如设成 `utmi_wide`；
4. 这类板级一般更像“固定下载口 / 固定调试口”。

如果你的产品不需要 OTG/Type-C 动态切换，只需要一个稳定的 device 口，这种方式最简单。

### 2. OTG / Dual Role 场景

K3 的真实产品板更常见的是 `usb3_porta` 配成 OTG：

```dts
&usb3_porta {
        dr_mode = "otg";
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        monitor-vbus;
        status = "okay";
};
```

这几个属性分别意味着：

- `dr_mode = "otg"`：支持角色切换；
- `usb-role-switch`：启用 role switch 框架；
- `role-switch-default-mode = "peripheral"`：默认先按 device 看待；
- `monitor-vbus`：根据 VBUS 状态协助判定角色。

对于 K3 来说，这通常才是 Type-C DRD 口的标准写法。

### 3. 与 FUSB301 的 graph 连接

在 `k3_dc_board.dts`、`k3_deb1.dts`、`k3_com260.dts`、`k3_com260_kit_v02.dts` 里，可以看到 K3 用 `FUSB301` 做 Type-C 角色检测，并通过 graph endpoint 和 DWC3 role switch 连接。

例如：

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

对应的 Type-C 控制器：

```dts
tcpc@25 {
        compatible = "onsemi,fusb301";
        reg = <0x25>;
        irq-gpios = <...>;
        wakeup-source;
        status = "okay";

        typec_0: connector@0 {
                compatible = "usb-c-connector";
                data-role = "dual";
                power-role = "dual";
                try-power-role = "sink";
                typec-power-opmode = "default";
                pd-disable;

                ports {
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

这说明 K3 的 gadget / OTG 并不只是改一个 `dr_mode`，而是完整依赖：

- `usb-role-switch`
- `usb-c-connector`
- `onsemi,fusb301`
- endpoint graph

### 4. SuperSpeed 方向切换

在 `k3_com260.dts` / `k3_com260_kit_v02.dts` 中，还能看到：

- `fusb301_sw_ep`
- `usb3_phy_switch`

这表示 Type-C 的插头翻转方向还会影响 SS PHY 路径选择。对 gadget 用户来说，这意味着：

- USB2 gadget 可能能通；
- 但如果板级还想跑 SuperSpeed gadget，则 SS orientation switch 也必须正确接好。

## K3 Gadget bring-up 流程

建议按下面顺序 bring-up。

### 第一步：确认板级角色真的支持 device

先看 DTS：

- 是不是 `usb3_porta`
- 是不是 `dr_mode = "peripheral"` 或 `"otg"`
- 若是 Type-C，是否有 `usb-role-switch`
- 是否有 `fusb301`
- endpoint graph 是否完整

如果这些没配对，后面的 configfs 再正确也没用。

### 第二步：确认 UDC 是否出现

启动系统后查看：

```bash
ls /sys/class/udc
```

如果 UDC 没有出现，优先检查：

- DWC3 是否 probe 成功；
- `usb3_porta` 是否 `status = "okay"`；
- PHY 是否正常起来；
- role switch 是否把当前口切到了 device。

### 第三步：看日志

```bash
dmesg | grep -Ei "usb|dwc3|udc|typec|fusb"
```

重点看：

- dwc3 probe 成功没有；
- gadget/udc 是否注册成功；
- FUSB301 是否探测成功；
- role switch 是否切换到 device；
- 是否有 PHY / link / VBUS 相关错误。

### 第四步：再做 configfs gadget

只有在 UDC 出来以后，再继续配 configfs。否则就是反着调试，容易误判。

## Configfs 配置方法

K3 这里继续使用标准 Linux configfs 用法，和 K1 基本一致。

先挂载 configfs：

```bash
mount -t configfs none /sys/kernel/config
cd /sys/kernel/config/usb_gadget
mkdir k3g
cd k3g
```

### 1. 设置设备描述符

```bash
echo 0x1d6b > idVendor
echo 0x0104 > idProduct
mkdir strings/0x409
echo "0123456789" > strings/0x409/serialnumber
echo "SpacemiT" > strings/0x409/manufacturer
echo "K3 USB Gadget" > strings/0x409/product
```

### 2. 创建 configuration

```bash
mkdir configs/c.1
mkdir configs/c.1/strings/0x409
echo "Config 1" > configs/c.1/strings/0x409/configuration
```

### 3. 添加 function

#### ACM 串口示例

```bash
mkdir functions/acm.usb0
ln -s functions/acm.usb0 configs/c.1/
```

#### NCM 网卡示例

```bash
mkdir functions/ncm.usb0
ln -s functions/ncm.usb0 configs/c.1/
```

#### Mass Storage 示例

```bash
mkdir functions/mass_storage.usb0
echo /root/udisk.img > functions/mass_storage.usb0/lun.0/file
ln -s functions/mass_storage.usb0 configs/c.1/
```

### 4. 绑定 UDC

先看可用 UDC：

```bash
ls /sys/class/udc
```

再绑定：

```bash
echo <udc-name> > UDC
```

例如：

```bash
cat UDC
```

可以确认当前是否已经成功绑定。

## 常见 gadget 场景

### 1. USB 串口（ACM）

适合最先打通：

- 配置简单；
- 日志清晰；
- 容易判断 host/device 是否真正联通。

推荐作为 K3 gadget bring-up 的第一步。

### 2. USB 网卡（NCM / RNDIS）

适合开发联机调试、批量刷机、内网通信等场景。

如果使用 Type-C DRD 口做这个功能，注意：

- role switch 必须稳定；
- 插线方向和 VBUS 检测必须正常；
- PC 侧驱动也要匹配。

### 3. USB U 盘（Mass Storage）

适合验证 bulk 传输是否稳定。

### 4. UVC / FunctionFS / ADB

这些功能更适合在基础 gadget 打通之后再做；否则当底层 role / UDC / PHY 还没稳定时，应用层看起来会“像协议问题”，其实根因还在底层。

## K3 上使用 Gadget 时的实际注意事项

### 1. 不是所有 USB 口都能拿来做 gadget

这是 K3 最重要的结论之一。

当前从 DTS 使用方式看：

- 绝大多数 gadget / OTG 场景围绕 `usb3_porta`
- `usb2_host`、`usb3_portb/c/d` 大多是 host-only 使用方式

所以写产品 DTS 时，不要默认把任意 USB 节点改成 `peripheral`。

### 2. OTG 场景里，Type-C 链路比 configfs 更容易先出问题

很多人做 gadget 时会先怀疑：

- function 配错了
- UVC/ADB/MTP 脚本错了

但 K3 上更常见的根因其实是：

- FUSB301 没起来；
- `usb-role-switch` graph 断了；
- `monitor-vbus` 不对；
- PHY switch/orientation graph 没接好；
- 口其实没切到 device。

### 3. 某些板级只支持 High-Speed gadget

例如 `k3_fpga_1x1.dts` 中直接写：

```dts
maximum-speed = "high-speed";
```

所以即便控制器是 DWC3，也不要默认期望它一定跑到 SuperSpeed gadget。

### 4. `phy_type` 可能需要板级调整

例如：

```dts
/delete-property/ phy_type;
phy_type = "utmi_wide";
```

这类写法说明某些板级 USB device 方案对 PHY 接口类型有专门要求。碰到 gadget 不出枚举时，这也是一个必须检查的点。

## 调试建议

### 1. 先看 role

```bash
ls /sys/class/usb_role/
cat /sys/class/usb_role/*/role
```

如果当前不是 `device`，gadget 通常不会正常工作。

### 2. 再看 UDC

```bash
ls /sys/class/udc
```

### 3. 再看 gadget 是否绑定成功

```bash
cat /sys/kernel/config/usb_gadget/k3g/UDC
```

### 4. 看 debugfs / 更详细日志

如果内核打开了 `CONFIG_USB_GADGET_DEBUG_FS`，可以进一步结合 debugfs 看 DWC3/gadget 相关状态。

### 5. Type-C 板级优先看 FUSB301

```bash
dmesg | grep -i fusb
```

如果 FUSB301 没起来，role switch 往往也不会正常。

## FAQ

### 1. K3 的 gadget 口到底优先用哪个？

优先看 `usb3_porta`。从现有 DTS 使用方式看，这是 K3 上真正用于 `peripheral` / `otg` / Type-C DRD 的核心端口。

### 2. 为什么我 configfs 都配好了，但上位机没枚举？

先别急着改 function，先检查：

- 当前 role 是不是 `device`
- UDC 有没有出来
- `usb3_porta` 是否真的 `status = "okay"`
- `dr_mode` 是否正确
- Type-C / FUSB301 / endpoint graph 是否正确

### 3. 为什么 gadget 只能跑 USB2 速度？

可能原因包括：

- 板级 DTS 写了 `maximum-speed = "high-speed"`
- 板级没有把 SS 通道接出来
- Type-C SuperSpeed 方向切换链路没配好
- PHY 侧只启用了 USB2 通道

### 4. 为什么 OTG 口默认是 peripheral？

因为部分 K3 板级明确写了：

```dts
role-switch-default-mode = "peripheral";
```

这样更适合把该口当下载口、调试口或 gadget 口来使用。
