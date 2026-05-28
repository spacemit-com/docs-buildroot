# BT

This document introduces the K3 platform Bluetooth porting and notes.

## Overview

The K3 platform requires an external BT module to implement Bluetooth functionality, and supports interfaces such as UART / USB / SDIO.

## Functional Description

The BT software stack used on the K3 platform is `BlueZ`. The software framework based on `BlueZ` can be divided into the following layers from top to bottom:

![](static/bt.png)

1. **Bluetooth Application Layer**  
        Mainly implements application-layer logic and interacts with the protocol stack through the `DBus` interface.
2. **BlueZ Protocol Stack User Space**  
        Responsible for protocol management, application interfaces, service scheduling, and related functions.
3. **BlueZ Protocol Stack Kernel Space**  
        Mainly handles L2CAP segmentation and reassembly, ACL/SCO link management, HCI core scheduling, and related functions.
4. **HCI Transport Driver**  
        Responsible for transmitting HCI data packets between the host and the controller through physical interfaces such as UART / USB / SDIO.
5. **Interface Controller**  
        Mainly implements the BT data transmission interface functions.

## Source Code Structure

The BT-related kernel codes include:

```text
linux-6.18/
|-- net/bluetooth/          # Bluetooth core protocol stack
|-- drivers/bluetooth/      # HCI transport and vendor drivers
|-- drivers/rfkill/         # rfkill generic framework 
`-- arch/riscv/boot/dts/spacemit/       # Board-level BT power / GPIO configuration
```

The BT driver-related codes are located in `drivers/bluetooth/`:

- `hci_h4.c`
- `hci_h5.c`
- `hci_ldisc.c`
- `hci_serdev.c`
- `btusb.c`
- `btbcm.c`
- `btrtl.c`

The K3 BT driver is integrated with the serdev framework. Before the serdev framework was introduced, the kernel only provided the `tty_ldisc` device for UART-based Bluetooth. It was necessary to use commands such as `hciattach` to attach the device and download firmware. After adopting the serdev framework, these operations can be completed in the HCI driver.

When porting BT on K3, it is recommended to adapt at the `hci` layer. Different vendors need to adapt according to the specific transport protocol. For example, `rtl8852bs` uses H5, so the following content needs to be added to `hci_h5.c`:
```c
static const struct of_device_id rtl_bluetooth_of_match[] = {
#ifdef CONFIG_BT_HCIUART_RTL
        { .compatible = "realtek,rtl8852bs-bt",
                .data = (const void *)&h5_data_rtl8723bs },
#endif
        { },
};
```

## Key Features

### Platform UART Interface Features

| Feature | Description |
| :----- | :---- |
| 4-wire flow control support | Supports CTS/RTS hardware flow control |
| Maximum Baud Rate | Supports up to 3.6Mbps |
| DMA support | Supports DMA transfer mode |

### Module Performance Specifications

| Module Model | Specifications |
| :----- | :---- |
| rtl8852bs | Supports Bluetooth 5.2 |
|           | Supports Bluetooth 2.0 UART HCI H4/H5 |
| aic8800d80 | Supports Bluetooth 5.3 |
|            | Supports Bluetooth 2.0 UART HCI H4 |

## Configuration

It mainly includes **driver enablement configuration** and **DTS configuration**.

### Kconfig Configuration

#### Protocol Stack Configuration

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

#### UART HCI Configuration

```text
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=m])
                Bluetooth device drivers
                        HCI UART driver (BT_HCIUART [=m])
                                UART (H4) protocol support (BT_HCIUART_H4 [=y])
                                Three-wire UART (H5) protocol support (BT_HCIUART_3WIRE [=y])
                                Realtek protocol support (BT_HCIUART_RTL [=y])
```

By default, H4 and H5 are supported, where the Realtek BT serial port uses the H5 protocol.

#### USB HCI Configuration

```text
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=m])
                Bluetooth device drivers
                        HCI USB driver (BT_HCIBTUSB [=m])
                                Broadcom protocol support (BT_HCIBTUSB_BCM [=y])
                                Realtek protocol support (BT_HCIBTUSB_RTL [=y])
```

`BT_HCIBTUSB_BCM` and `BT_HCIBTUSB_RTL` correspond to USB interface support for Broadcom and Realtek devices, respectively.

#### AVRCP Configuration

```text
Device Drivers
        Input device support
                Generic input layer (needed for keyboard, mouse, ...) (INPUT [=y])
                        Miscellaneous devices (INPUT_MISC [=y])
                                User level driver support (INPUT_UINPUT [=y])
```

To deliver AVRCP key values and related information to user-space applications via the input device, the `INPUT_UINPUT` option must be enabled.

#### HOGP Configuration

```text
Device Drivers
        HID bus support (HID_SUPPORT [=y])
                HID bus core support (HID[=y])
                        User-space I/O driver support for HID subsystem (UHID [=y])
```

To deliver HoGP key values (e.g., `KEY_1`, `KEY_2`, `KEY_ESC`) to user-space applications via the input device, the `UHID` option must be enabled.

### DTS Configuration

#### USB Interface Configuration

Configure the BT node information according to the actual hardware configuration. Taking `k3_deb1.dts` as an example, if BT is connected to `portc`, the corresponding `&usb3_portc` configuration is:

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

- `maximum-speed` specifies the maximum speed. The Bluetooth module only requires high-speed mode.
- `phys` specifies the PHY corresponding to the controller, and only the USB 2.0 PHY is enabled.

BT power control is encapsulated in RFKILL, and the upper layer needs to operate RFKILL to enable BT.

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

#### UART Interface Configuration

Configure the BT node information according to the actual hardware configuration. Taking `&uart2` in `k3_evb.dts` as an example:

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

- `device-wake-gpios`: GPIO used to wake up the BT module. The active level depends on the actual hardware configuration.
- `enable-gpios`: GPIO used to enable the BT module. The active level depends on the actual hardware configuration.
- `host-wake-gpios`: GPIO used by the BT module to wake up the host. The active level depends on the actual hardware configuration.

The Bluetooth `pinctrl` configuration depends on the actual hardware. Flow control is enabled by default:

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

## Interface

### UART attach tools

In early kernel versions, UART Bluetooth usually required a user-space tool to bring up the HCI interface on the serial side, for example:

- `hciattach`
- Or a vendor-provided attach tool, such as `rtk_hciattach`

### USB rfkill control

USB Bluetooth uses `rfkill-gpio`. From user space, you can directly run:

```bash
rfkill list
rfkill block bluetooth
rfkill unblock bluetooth
```

### bluetoothctl

`bluetoothctl` is the core interactive management tool for the `BlueZ` Bluetooth protocol stack on Linux systems. It is used for scanning, pairing, connecting, configuring Bluetooth devices, and related operations, replacing older tools such as `hcitool` and `hciconfig`.

`bluetoothctl` depends on the `bluetoothd` daemon. First make sure the service is running:

```bash
systemctl start bluetooth
systemctl status bluetooth
```

Common operations after entering `bluetoothctl` include:

```text
power on
scan on
pair <MAC>
connect <MAC>
trust <MAC>
```

## Testing

### BT basic function test

First, make sure the `bluetoothd` service is running properly, then enter the command line by typing `bluetoothctl`:

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

### 1. Cannot find the `hci0` device node?

When running `hciconfig`, there is no output similar to the following:

```bash
hci0:Type: Primary  Bus: USB
        BD Address: C0:09:25:A7:F4:2D  ACL MTU: 1021:6  SCO MTU: 255:12
        UP RUNNING
        RX bytes:3505 acl:0 sco:0 events:352 errors:0
        TX bytes:63142 acl:0 sco:0 commands:352 errors:0
```

Common causes include:

- Whether the controller is enabled in the corresponding board DTS
- Whether the module power supply is normal
- For UART-based Bluetooth, whether the `enable-gpios` / `device-wake-gpios` states are correct
- For USB-based Bluetooth, whether the `rfkill` state is correct
- Whether the corresponding firmware exists

### 2. BLE device cannot be scanned (BR/EDR works, but BLE does not)?

Use `bluetoothctl show` to confirm whether the controller supports BLE. Under normal conditions, the output should include `Roles: central` and `Roles: peripheral`:

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

If the output does not include `Roles: central` or `Roles: peripheral`, it indicates that the controller does not support BLE or that the relevant kernel configuration is not enabled.

Common causes:

- Confirm that the controller supports BLE
- Check whether the BLE configuration is enabled in the kernel

### 3. Pairing failed: `Authentication failed`?

Common causes:

- Incorrect device PIN code
- The device is already paired with another device. Remove the existing pairing on the target device first.

### 4. A2DP connection succeeds but there is no sound?

You can use `bluetoothctl info <MAC>` to confirm whether the peer device supports A2DP Sink:

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

If the output does not include the `Audio Sink (0000110b...)` UUID, it indicates that the peer device does not support A2DP Sink. If the output only includes the `Audio Source (0000110a...)` UUID, it indicates that the peer device only supports A2DP Source.

Common causes:

- Confirm that PulseAudio or PipeWire is running and that the Bluetooth audio module is loaded
- Confirm that the audio output device has been switched to the Bluetooth device
- Check whether `bluetoothctl info <MAC>` includes the `Audio Sink (0000110b...)` UUID. If not, the peer device does not support A2DP Sink
- Confirm that `BT_BREDR` and `BT_LE` are enabled in the kernel
- Check `dmesg` for logs indicating A2DP codec negotiation failures
