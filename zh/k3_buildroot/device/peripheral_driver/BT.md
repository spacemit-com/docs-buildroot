# BT

介绍 K3 平台 Bluetooth 模组的常见移植方法以及注意事项。

## 模块介绍

K3 平台需要外接**外部 BT 模组**实现 BT 功能，支持 UART / USB / SDIO 等接口。

## 功能介绍

K3 平台上使用的 BT 软件栈为 `BlueZ` ，基于 `BlueZ` 的软件框架从上到下可以分为以下几层：

![](static/bt.png)

1. **蓝牙应用层**
   主要实现应用层的相关逻辑，通过 `DBus` 接口与协议栈交互；
2. **BlueZ 协议栈用户空间**
   负责管理协议，提供应用接口以及服务调度等；
3. **BlueZ 协议栈内核空间**
   主要处理 L2CAP 分片重组、ACL/SCO 链路管理、HCI 核心调度等；
4. **HCI 传输驱动**
   负责通过 UART / USB / SDIO 等物理接口在主机与控制器之间传输 HCI 数据包；
5. **接口控制器**
   主要实现 BT 数据传输接口功能；

## 源码结构介绍

BT 内核相关代码主要有：

```text
linux-6.18/
|-- net/bluetooth/          # 蓝牙核心协议栈
|-- drivers/bluetooth/      # HCI 传输与厂商驱动
|-- drivers/rfkill/         # rfkill 通用框架
`-- arch/riscv/boot/dts/spacemit/       # 板级 BT 电源/ GPIO 配置
```

其中 BT 驱动相关的代码在 `drivers/bluetooth/` ：

- `hci_h4.c`
- `hci_h5.c`
- `hci_ldisc.c`
- `hci_serdev.c`
- `btusb.c`
- `btbcm.c`
- `btrtl.c`

K3 BT 的驱动接入了 serdev 框架，在 serdev 框架之前，对于 UART 接口的蓝牙，内核只提供 tty_ldisc 设备，需要通过 `hciattach` 等命令来加载设备并下载固件，使用 serdev 框架后，可以将对应的操作放到 hci 驱动中完成。

K3 BT 导入时建议适配 `hci` 层，不同的厂商需要根据具体的传输协议进行适配，比如 rtl8852bs 使用的 H5 需要在 `hci_h5.c` 增加以下内容:

```c
static const struct of_device_id rtl_bluetooth_of_match[] = {
#ifdef CONFIG_BT_HCIUART_RTL
        { .compatible = "realtek,rtl8852bs-bt",
                .data = (const void *)&h5_data_rtl8723bs },
#endif
        { },
};
```

## 关键特性

### 平台 UART 接口特性

| 特性 | 特性说明 |
| :----- | :---- |
| 支持 4 线流控 | 支持 CTS/RTS 硬件流控 |
| 最高波特率 | 最高支持 3.6Mbps |
| 支持 DMA | 支持 DMA 传输模式 |

### 模组性能规格

| 模组型号 | 规格 |
| :----- | :---- |
| rtl8852bs | 支持 Bluetooth 5.2 |
|           | 支持 Bluetooth 2.0 UART HCI H4/H5 |
| aic8800d80 | 支持 Bluetooth 5.3 |
|            | 支持 Bluetooth 2.0 UART HCI H4 |

## 配置介绍

主要包括驱动使能配置和 DTS 配置。

### CONFIG 配置

#### 协议栈配置

```text
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=m])
                Bluetooth Classic (BR/EDR) features (BT_BREDR [=y])
                        RFCOMM protocol support (BT_RFCOMM [=m])
                                RFCOMM TTY support (BT_RFCOMM_TTY [=y])
                        BNEP protocol support (BT_BNEP [=y])
                        HIDP protocol support (BT_HIDP [=y])
                Bluetooth Low Energy (LE) features (BT_LE [=y])
        Export Bluetooth internals in debugfs (BT_DEBUGFS [=y])
```

#### UART HCI 配置

```text
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=m])
                Bluetooth device drivers
                        HCI UART driver (BT_HCIUART [=m])
                                UART (H4) protocol support (BT_HCIUART_H4 [=y])
                                Three-wire UART (H5) protocol support (BT_HCIUART_3WIRE [=y])
                                Realtek protocol support (BT_HCIUART_RTL [=y])
```

默认支持 H4 和 H5 ，其中 Realtek 的蓝牙串口走的是 H5 协议。

#### USB HCI 配置

```text
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=m])
                Bluetooth device drivers
                        HCI USB driver (BT_HCIBTUSB [=m])
                                Broadcom protocol support (BT_HCIBTUSB_BCM [=y])
                                Realtek protocol support (BT_HCIBTUSB_RTL [=y])
```

BT_HCIBTUSB_BCM 和 BT_HCIBTUSB_RTL 分别对应 Broadcom 和 Realtek 的 USB 接口的支持。

#### AVRCP 配置

```text
Device Drivers
        Input device support
                Generic input layer (needed for keyboard, mouse, ...) (INPUT [=y])
                        Miscellaneous devices (INPUT_MISC [=y])
                                User level driver support (INPUT_UINPUT [=y])
```

如果要把 AVRCP 的按键值等信息通过 input device 送给用户态程序，则需要打开 INPUT_UINPUT 。

#### HOGP 配置

```text
Device Drivers
        HID bus support (HID_SUPPORT [=y])
                HID bus core support (HID[=y])
                        User-space I/O driver support for HID subsystem (UHID [=y])
```

如果要把 HoG 的 KEY_1, KEY_2, KEY_ESC 等按键值通过 input device 送给用户态程序，则需要打开 UHID 。

### DTS 配置

#### USB 接口配置

根据实际硬件配置对应的 BT 节点信息，以 `k3_deb1.dts` 为例，BT 接到 portc 上，对应的 `&usb3_portc` 示例为：

```dts
&usb3_portc_u2phy {
        status = "okay";
};

&usb3_portc_u3phy {
        status = "disabled";
};

&usb3_portc {
        /* Bluetooth, only enable USB2 phy */
        /delete-property/ phys;
        /delete-property/ phy-names;
        maximum-speed = "high-speed";
        phys = <&usb3_portc_u2phy>;
        phy-names = "usb2-phy";
        status = "okay";
};
```

- `maximum-speed` 指定最大速度， 蓝牙模组只需要高速模式。
- `phys` 指定控制器对应的 phy ，只使能 USB2.0 PHY。

BT 的供电控制封装在 RFKILL 中，上层需要操作 RFKILL 给 BT 使能。

```dts
rfkill-usb-bt {
        compatible = "rfkill-gpio";
        label = "rfkill-usb-bt";
        pinctrl-names = "default";
        pinctrl-0 = <&bt_en_cfg>;
        radio-type = "bluetooth";
        shutdown-gpios = <&gpio 0 30 GPIO_ACTIVE_HIGH>;
};
```

#### UART 接口配置

根据实际硬件配置对应的 BT 节点信息，以 `k3_evb.dts` 中的 `&uart2` 示例为：

```dts
&uart2 {
        pinctrl-names = "default";
        pinctrl-0 = <&uart2_0_cfg>;
        status = "okay";

        bluetooth {
                compatible = "realtek,rtl8852bs-bt";
                device-wake-gpios = <&gpio 2 30 GPIO_ACTIVE_HIGH>;
                enable-gpios = <&gpio 2 29 GPIO_ACTIVE_HIGH>;
                host-wake-gpios = <&gpio 2 28 GPIO_ACTIVE_HIGH>;
        };
};
```

- `device-wake-gpios` 对应唤醒 BT 模块的 GPIO ，有效电平按实际硬件配置。
- `enable-gpios` 对应使能 BT 模块的 GPIO ，有效电平按实际硬件配置。
- `host-wake-gpios` 对应 BT 模块唤醒 HOST 的 GPIO ，有效电平按实际硬件配置。

蓝牙 pinctrl 配置以实际硬件为准，默认开启流控：

```dts
uart2_0_cfg: uart2-0-cfg {
        uart2-0-pins {
                pinmux = <K3_PADCONF(134, 2)>,
                        <K3_PADCONF(135, 2)>,
                        <K3_PADCONF(136, 2)>,
                        <K3_PADCONF(137, 2)>;

                bias-pull-up;
                drive-strength = <25>;
        };
};
```

## 接口介绍

### UART attach 工具

早期的内核版本使用 UART 蓝牙，通常需要先通过用户态工具把串口侧 HCI 拉起来，例如：

- `hciattach`
- 或者厂商自带 attach 工具，比如 `rtk_hciattach`

### USB rfkill 控制

USB 蓝牙使用了 `rfkill-gpio`，用户态可以直接通过：

```bash
rfkill list
rfkill block bluetooth
rfkill unblock bluetooth
```

### bluetoothctl

`bluetoothctl` 是 `Linux` 系统中 `BlueZ` 蓝牙协议栈的核心交互式管理工具，用于蓝牙设备的扫描、配对、连接、配置等相关操作，替代了旧版的 `hcitool` / `hciconfig` 。

`bluetoothctl` 依赖 `bluetoothd` 守护进程，先确保服务运行：

```bash
systemctl start bluetooth
systemctl status bluetooth
```

进入 `bluetoothctl` 后常见操作：

```text
power on
scan on
pair <MAC>
connect <MAC>
trust <MAC>
```

## 测试介绍

### BT 基本功能测试

首先确保 bluetoothd 服务正常运行，输入 `bluetoothctl` 进入命令行：

```bash
[bluetooth]# power on
[bluetooth]# Changing power on succeeded
[bluetooth]# scan on
[bluetooth]# SetDiscoveryFilter success
[bluetooth]# Discovery started
[bluetooth]# [CHG] Controller 5C:8A:AE:67:62:04 Discovering: yes
[bluetooth]# [NEW] Device 45:DC:1E:BC:2C:77 45-DC-1E-BC-2C-77
[bluetooth]# [NEW] Device 4C:30:B8:02:7F:7A 4C-30-B8-02-7F-7A
[bluetooth]# [NEW] Device DC:28:67:9A:70:8E DC-28-67-9A-70-8E
[bluetooth]# [NEW] Device 58:FB:F1:17:D4:19 58-FB-F1-17-D4-19
[bluetooth]# [NEW] Device 84:7B:57:FB:20:8D 84-7B-57-FB-20-8D
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D TxPower: 0x000c (12)
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D Name: LT-ZHENGHONG
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D Alias: LT-ZHENGHONG
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110a-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110b-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000111f-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000111e-0000-1000-8000-00805f9b34fb
[bluetooth]#
[bluetooth]# pair 84:7B:57:FB:20:8D
Attempting to pair with 84:7B:57:FB:20:8D
[CHG] Device 84:7B:57:FB:20:8D Connected: yes
[LT-ZHENGHONG]# Request confirmation
[LT-ZHENGHONG]#   [agent] Confirm passkey 947781 (yes/no): yes
[DEL] Device 58:FB:F1:17:D4:19 58-FB-F1-17-D4-19
[bluetooth]# info 84:7B:57:FB:20:8D
Device 84:7B:57:FB:20:8D (public)
        Name: LT-ZHENGHONG
        Alias: LT-ZHENGHONG
        Class: 0x002a010c (2752780)
        Icon: computer
        Paired: no
        Bonded: no
        Trusted: no
        Blocked: no
        Connected: yes
        LegacyPairing: no
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree                 (0000111e-0000-1000-8000-00805f9b34fb)
        RSSI: 0xffffffae (-82)
        TxPower: 0x000c (12)
[LT-ZHENGHONG]# [DEL] Device DC:28:67:9A:70:8E DC-28-67-9A-70-8E
[LT-ZHENGHONG]# [DEL] Device 45:DC:1E:BC:2C:77 45-DC-1E-BC-2C-77
[LT-ZHENGHONG]# [DEL] Device 53:84:3E:02:79:84 53-84-3E-02-79-84
[LT-ZHENGHONG]# [CHG] Device 84:7B:57:FB:20:8D Bonded: yes
[LT-ZHENGHONG]# info 84:7B:57:FB:20:8D
Device 84:7B:57:FB:20:8D (public)
        Name: LT-ZHENGHONG
        Alias: LT-ZHENGHONG
        Class: 0x002a010c (2752780)
        Icon: computer
        Paired: no
        Bonded: yes
        Trusted: no
        Blocked: no
        Connected: yes
        LegacyPairing: no
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree                 (0000111e-0000-1000-8000-00805f9b34fb)
        RSSI: 0xffffffae (-82)
        TxPower: 0x000c (12)
```

## FAQ

### 1. 找不到 `hci0` 设备节点？

执行 `hciconfig` 无以下类似打印：

```bash
hci0:Type: Primary  Bus: USB
        BD Address: C0:09:25:A7:F4:2D  ACL MTU: 1021:6  SCO MTU: 255:12
        UP RUNNING
        RX bytes:3505 acl:0 sco:0 events:352 errors:0
        TX bytes:63142 acl:0 sco:0 commands:352 errors:0
```

常见原因包括：

- 对应方案的控制器 DTS 是否有使能；
- 模组供电是否正常；
- UART 接口的蓝牙确认 `enable-gpios` / `device-wake-gpios` 状态是否正常；
- USB 接口的蓝牙确认 `rfkill` 状态是否正常；
- 对应的固件是否存在。

### 2. BLE 设备扫描不到（BR/EDR 能扫描，BLE 不行）？

可以通过 `bluetoothctl show` 确认控制器是否支持 BLE，正常情况下输出应包含 `Roles: central` 和 `Roles: peripheral`：

```bash
[bluetoothctl]> show
Controller C0:09:25:A7:F4:2D (public)
        Manufacturer: 0x005d (93)
        Version: 0x0c (12)
        Name: k3
        Alias: k3
        Class: 0x006c0000 (7077888)
        Powered: yes
        PowerState: on
        Discoverable: no
        DiscoverableTimeout: 0x000000b4 (180)
        Pairable: yes
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
        UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
        UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
        UUID: Device Information        (0000180a-0000-1000-8000-00805f9b34fb)
        UUID: Vendor specific           (03b80e5a-ede8-4b33-a751-6ce34ec4c700)
        UUID: Handsfree                 (0000111e-0000-1000-8000-00805f9b34fb)
        Modalias: usb:v1D6Bp0246d0555
        Discovering: no
        Roles: central
        Roles: peripheral
Advertising Features:
        ActiveInstances: 0x00 (0)
        SupportedInstances: 0x0a (10)
        SupportedIncludes: tx-power
        SupportedIncludes: appearance
        SupportedIncludes: local-name
        SupportedSecondaryChannels: 1M
        SupportedSecondaryChannels: 2M
        SupportedSecondaryChannels: Coded
        SupportedCapabilities.MinTxPower: 0x0001 (1)
        SupportedCapabilities.MaxTxPower: 0x001d (29)
        SupportedCapabilities.MaxAdvLen: 0xfb (251)
        SupportedCapabilities.MaxScnRspLen: 0xfb (251)
        SupportedFeatures: CanSetTxPower
        SupportedFeatures: HardwareOffload
```

如果输出中没有 `Roles: central` 或 `Roles: peripheral`，说明控制器不支持 BLE 或内核未开启相关配置。

常见原因包括：

- 确认控制器支持 BLE；
- 内核是否有开启 BLE 配置。

### 3. 配对失败: Authentication failed？

常见原因包括：

- 设备 PIN 码错误；
- 设备已被其他设备配对，先在目标设备上取消原有配对。

### 4. A2DP 连接成功但没有声音？

可以通过 `bluetoothctl info <MAC>` 确认对端是否支持 A2DP Sink：

```bash
Device 90:F0:52:D3:FA:37 (public)
        Name: xxpx
        Alias: xxpx
        Class: 0x005a020c (5898764)
        Icon: phone
        Paired: no
        Bonded: no
        Trusted: no
        Blocked: no
        Connected: yes
        LegacyPairing: yes
        CablePairing: no
        UUID: Vendor specific           (00000000-0000-0000-0000-000000000000)
        UUID: Unknown                   (00000bd1-0000-1000-8000-00805f9b34fb)
        UUID: OBEX Object Push          (00001105-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Headset AG                (00001112-0000-1000-8000-00805f9b34fb)
        UUID: PANU                      (00001115-0000-1000-8000-00805f9b34fb)
        UUID: NAP                       (00001116-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: SIM Access                (0000112d-0000-1000-8000-00805f9b34fb)
        UUID: Phonebook Access Server   (0000112f-0000-1000-8000-00805f9b34fb)
        UUID: Message Access Server     (00001132-0000-1000-8000-00805f9b34fb)
        UUID: PnP Information           (00001200-0000-1000-8000-00805f9b34fb)
        UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
        UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
        Modalias: bluetooth:v001Dp1200d1436
        RSSI: 0xffffffc0 (-64)
```

如果输出中没有 `Audio Sink (0000110b...)` UUID，说明对端不支持 A2DP Sink，上面的输出中只有 `Audio Source (0000110a...)` UUID 表示对端只支持 A2DP Source。

常见原因包括：

- 确认 PulseAudio 或 PipeWire 已启动，且加载了蓝牙音频模块；
- 确认音频输出设备已切换到蓝牙设备；
- 检查 `bluetoothctl info <MAC>` 中是否包含 `Audio Sink (0000110b...)` UUID，若没有说明对端不支持 A2DP Sink；
- 确认内核已开启 `BT_BREDR` 和 `BT_LE` 配置；
- 查看 `dmesg` 是否有 A2DP codec 协商失败的日志。
