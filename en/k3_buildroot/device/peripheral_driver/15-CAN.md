# CAN

This document describes CAN configuration, debugging, and validation methods.

## Module Overview

CAN (Controller Area Network) is a serial communication protocol for communication between controllers and peripheral devices.
It is widely used in automotive systems, industrial automation, medical equipment, aerospace systems, robotics, and similar applications.

### Functional Overview

The CAN controller supports both CAN 2.0 and CAN FD and can handle multiple frame types, including:

- Standard data frames
- Standard remote frames
- Extended data frames

The CAN driver is registered as a network device in the Linux networking framework. In user space, CAN message transmission and reception can be handled through standard network tools and interfaces.

### Source Tree Overview

The CAN controller driver code is located under `drivers/net/can`:

```
drivers/net/can
|--dev.c                     # Kernel CAN framework code, including bitrate calculation and CAN device registration
|--flexcan/                  # K3 CAN driver
 |--flexcan-core.c
 |--flexcan.h
```

## Key Features

### Feature Summary

| Feature | Description |
| :-- | :-- |
| CAN FD support | Supports the CAN FD protocol and is compatible with CAN 2.0 |
| Up to 64-byte payload | CAN FD supports payload sizes of 8, 16, 32, and 64 bytes |
| Multiple controllers | The K3 platform provides 10 CAN controllers: 5 main-domain CAN controllers (CAN0 to CAN4) and 5 R-domain CAN controllers (RCAN0 to RCAN4) |
| Operating modes | Configured through the `compatible` property: <br>`spacemit,k1-flexcan` enables CAN FD mode <br>`spacemit,k1-flexcan-can2.0` supports CAN 2.0 mode only |

### Performance

- CAN FD mode: supports data-phase bitrates up to 8 Mbps
- CAN 2.0 mode: supports bitrates up to 1 Mbps

## Configuration

Configuration mainly includes **driver enablement** and **DTS configuration**.

### CONFIG Options

`CONFIG_CAN_DEV`: enables support for the kernel CAN framework. This option should be set to `Y` when the K3 CAN driver is enabled.

```shell
Symbol: CAN_DEV [=y]
Device Drivers
    -> Network device support (NETDEVICES [=y]) 
  -> CAN Device Drivers (CAN_DEV [=y])
```

After the platform CAN framework is enabled, set `CONFIG_CAN_FLEXCAN` to `Y` to enable the K3 CAN driver.

```shell
Symbol: CAN_FLEXCAN [=y]
    -> CAN device drivers with Netlink support (CAN_NETLINK [=y])
  -> Support for Freescale FLEXCAN based chips (CAN_FLEXCAN [=y])
```

### DTS Configuration

The K3 platform **does not integrate a CAN transceiver**. Only the TX and RX signals are provided, so an external transceiver is required.

#### `pinctrl` Configuration Example

Refer to `arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi` in the Linux source tree for preconfigured CAN node examples, as shown below:

```dts
can0_0_cfg: can0-0-cfg {
    can0-0-pins {
        pinmux = <K3_PADCONF(11, 3)>,     /* can0 tx */
                 <K3_PADCONF(12, 3)>;     /* can0 rx */
        bias-pull-up;
        drive-strength = <25>;
        power-source = <3300>;
    };
};

can1_0_cfg: can1-0-cfg {
    can1-0-pins {
        pinmux = <K3_PADCONF(48, 3)>,     /* can1 rx */
                 <K3_PADCONF(49, 3)>;     /* can1 tx */
        bias-pull-up;
        drive-strength = <25>;
        power-source = <3300>;
    };
};
```

#### `dtsi` Configuration Example

The `dtsi` file defines the CAN controller base addresses, clocks, and reset resources. In most cases, this section **does not need to be modified**.

The K3 platform provides 10 CAN controllers, divided between the main domain and the R-domain.

**`compatible` property description:**

- `compatible = "spacemit,k1-flexcan"`: supports CAN FD mode and is compatible with CAN 2.0
- `compatible = "spacemit,k1-flexcan-can2.0"`: supports CAN 2.0 mode only

**Main-domain CAN controllers (`k3.dtsi`)**

```dts
flexcan0: fdcan@d4028000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xd4028000 0x0 0x4000>;
    interrupts = <161 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN0>,<&syscon_apbc CLK_APBC_CAN0_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN0>;
    status = "disabled";
};

flexcan1: fdcan@d402c000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xd402c000 0x0 0x4000>;
    interrupts = <163 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN1>,<&syscon_apbc CLK_APBC_CAN1_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN1>;
    status = "disabled";
};

flexcan2: fdcan@d4034000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xd4034000 0x0 0x4000>;
    interrupts = <165 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN2>,<&syscon_apbc CLK_APBC_CAN2_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN2>;
    status = "disabled";
};

flexcan3: fdcan@d4038000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xd4038000 0x0 0x4000>;
    interrupts = <167 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN3>,<&syscon_apbc CLK_APBC_CAN3_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN3>;
    status = "disabled";
};

flexcan4: fdcan@d403c000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xd403c000 0x0 0x4000>;
    interrupts = <169 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    clocks = <&syscon_apbc CLK_APBC_CAN4>,<&syscon_apbc CLK_APBC_CAN4_BUS>;
    clock-names = "per","ipg";
    resets = <&syscon_apbc RESET_APBC_CAN4>;
    status = "disabled";
};
```

**R-domain CAN controllers (`k3-rdomain.dtsi`)**

The R-domain provides 5 additional CAN controllers for real-time domain applications:

```dts
r_flexcan0: fdcan@c0710000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xc0710000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN0_BUS>;
    clock-names = "per","ipg";
    interrupts = <241 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN0>;
    status = "disabled";
};

r_flexcan1: fdcan@c0720000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xc0720000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN1>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN1_BUS>;
    clock-names = "per","ipg";
    interrupts = <243 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN1>;
    status = "disabled";
};

r_flexcan2: fdcan@c0730000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xc0730000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN2>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN2_BUS>;
    clock-names = "per","ipg";
    interrupts = <245 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN2>;
    status = "disabled";
};

r_flexcan3: fdcan@c0740000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xc0740000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN3>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN3_BUS>;
    clock-names = "per","ipg";
    interrupts = <247 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN3>;
    status = "disabled";
};

r_flexcan4: fdcan@c0750000 {
    compatible = "spacemit,k1-flexcan";          /* CAN FD mode */
    reg = <0x0 0xc0750000 0x0 0x4000>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN4>,
             <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RCAN4_BUS>;
    clock-names = "per","ipg";
    interrupts = <249 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    fsl,clk-source = <0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RCAN4>;
    status = "disabled";
};
```

**To use CAN 2.0 mode, change the `compatible` property as follows:**

```dts
/* Example: configure flexcan0 for CAN 2.0 mode */
&flexcan0 {
    compatible = "spacemit,k1-flexcan-can2.0";   /* CAN 2.0 only */
    /* Keep other settings unchanged */
};
```

#### DTS Configuration Example

The complete DTS configuration is shown below. Clock frequencies of 20 MHz, 40 MHz, or 80 MHz can be selected according to the required bitrate.

**Main-domain CAN configuration example:**

```dts
/* can0 */
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};

/* can1 */
&flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&can1_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

**R-domain CAN configuration example:**

```dts
/* rcan0 */
&r_flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};

/* rcan1 */
&r_flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan1_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

## Interface Overview

### API Overview

The CAN driver mainly provides interfaces for **message transmission and reception**.

- Common device-open interface:

    ```c
    static int flexcan_open(struct net_device *dev)  
    ```

- Message transmit interface, called after the CAN device is opened:

    ```c
    static netdev_tx_t flexcan_start_xmit(struct sk_buff *skb, struct net_device *dev) 
    ```

During driver initialization, bitrate information is read from the Device Tree and stored in the driver-private data structure.

## Advanced Configuration

### Mailbox Configuration in CAN FD Mode

**Important:** In CAN FD mode, FlexCAN uses only one mailbox for frame reception by default. In high-load scenarios, this may cause occasional packet loss.

#### Solutions

**Option 1: Use mailbox filtering for fixed frame IDs (recommended)**

If the application uses fixed CAN frame IDs, multiple mailboxes can be configured in DTS with ID filtering to improve reception reliability.

Add the following properties in DTS:

```dts
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    
    /* Configure mailboxes for fixed frame IDs */
    flexcan-mailbox-id = <0x123 0x456 0x789>;           /* CAN IDs to receive */
    flexcan-mailbox-id-bits = /bits/ 8 <11 11 29>;      /* ID width: 11-bit standard frame or 29-bit extended frame */
    
    status = "okay";
};
```

**Configuration notes:**

- `flexcan-mailbox-id`: specifies the list of CAN frame IDs to receive, up to `MAX_RX_MAILBOX`
- `flexcan-mailbox-id-bits`: specifies the bit width for each ID
    - `11`: standard frame (11-bit ID)
    - `29`: extended frame (29-bit ID)

**Examples:**

```dts
/* Example 1: receive three standard frame IDs */
flexcan-mailbox-id = <0x100 0x200 0x300>;
flexcan-mailbox-id-bits = /bits/ 8 <11 11 11>;

/* Example 2: mixed standard and extended frames */
flexcan-mailbox-id = <0x123 0x18FF1234>;
flexcan-mailbox-id-bits = /bits/ 8 <11 29>;
```

**Option 2: Use CAN 2.0 mode**

If the application uses only CAN 2.0 frames and does not require CAN FD, CAN 2.0 mode is recommended.
In this mode, RX FIFO can be used to improve receive performance:

```dts
&flexcan0 {
    compatible = "spacemit,k1-flexcan-can2.0";    /* Use CAN 2.0 mode */
    pinctrl-names = "default";
    pinctrl-0 = <&can0_0_cfg>;
    clock-frequency = <80000000>;
    status = "okay";
};
```

In CAN 2.0 mode, the driver automatically enables RX FIFO to provide better receive buffering capability.

## Debugging

1. Check whether the CAN device has been loaded successfully.

   ```shell
   ifconfig -a
   ```

2. Configure the arbitration-phase and data-phase bitrates on the K3 platform.

   ```shell
   ip link set can0 type can bitrate 125000 dbitrate 250000 berr-reporting on fd on  
   ```

3. Bring up the CAN device after the peer side is ready to receive.

   ```shell
   ip link set can0 up  
   ```

4. Send a frame from the K3 side.

    `cansend` format: `cansend can-dev id##data`

   ```shell
   cansend can0 123##3.11223344556677881122334455667788aabbccdd  
   ```

5. Receive frames on the K3 side while the peer side is transmitting.

   ```shell
   candump can0
   ```

## Testing

On the K3 platform, testing can be performed with an external CAN transceiver. The peer side is typically a USB CAN analyzer connected to a PC to emulate a CAN device.
Because peer devices and usage models may vary, this section focuses on test procedures on the K3 side.

The following example uses a K3 development board running Buildroot. For DTS settings, refer to the **[DTS Configuration Example](#dts-configuration-example)** section.

### Test Procedure

1. Connect the K3 development board to a CAN transceiver.

    Connect the CAN TX and RX signals to an external CAN transceiver, such as TJA1050 or SN65HVD230.

2. Install CAN tools on the PC side and connect the PC CAN interface.

    Two CAN devices can also be connected to transmit and receive frames with each other.

    The PEAK PC CAN tools are recommended: [PEAK website](https://www.peak-system.com)

3. Check whether the CAN device is loaded successfully.

   ```shell
   # ifconfig -a
   can0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
             NOARP  MTU:16  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:10 
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
             Interrupt:161
   ```

4. Configure the CAN bitrate and bring up the device.

   ```shell
   # ip link set can0 type can bitrate 500000 dbitrate 2000000 berr-reporting on fd on
   # ip link set can0 up
   ```

5. Send a CAN FD frame from the K3 side.

   ```shell
   # cansend can0 123##3.11223344556677881122334455667788aabbccdd
   ```

    The CAN tool on the PC side should receive the frame.

6. Receive frames on the K3 side.

   ```shell
   # candump can0
   ```

    When the PC side transmits a frame, the K3 side should receive and display it.

7. Stop the CAN device.

   ```shell
   # ip link set can0 down
   ```

## Controller Resource Summary

CAN controller resources on the K3 platform are distributed as follows:

| Controller | Base address | IRQ | Clock source | Domain |
| :-----| :----| :----| :----| :----|
| flexcan0 | 0xd4028000 | 161 | CLK_APBC_CAN0 | Main domain |
| flexcan1 | 0xd402c000 | 163 | CLK_APBC_CAN1 | Main domain |
| flexcan2 | 0xd4034000 | 165 | CLK_APBC_CAN2 | Main domain |
| flexcan3 | 0xd4038000 | 167 | CLK_APBC_CAN3 | Main domain |
| flexcan4 | 0xd403c000 | 169 | CLK_APBC_CAN4 | Main domain |
| r_flexcan0 | 0xc0710000 | 241 | CLK_RCPU_SYSCTRL_RCAN0 | R-domain |
| r_flexcan1 | 0xc0720000 | 243 | CLK_RCPU_SYSCTRL_RCAN1 | R-domain |
| r_flexcan2 | 0xc0730000 | 245 | CLK_RCPU_SYSCTRL_RCAN2 | R-domain |
| r_flexcan3 | 0xc0740000 | 247 | CLK_RCPU_SYSCTRL_RCAN3 | R-domain |
| r_flexcan4 | 0xc0750000 | 249 | CLK_RCPU_SYSCTRL_RCAN4 | R-domain |

## FAQ

### CAN Device Cannot Be Started

Check the following items:

1. Confirm that `CONFIG_CAN` and `CONFIG_CAN_FLEXCAN` are enabled in the kernel configuration.
2. Confirm that the CAN node in DTS has `status = "okay"`.
3. Confirm that the `pinctrl` configuration is correct.
4. Check the external CAN transceiver power supply and wiring.

### CAN Communication Is Abnormal

1. Check whether the bitrate configuration matches the peer side.
2. Check whether CAN_H and CAN_L are wired correctly.
3. Check whether the 120 $\Omega$ bus termination resistor is connected correctly.
4. Use `ip -details -statistics link show can0` to inspect error statistics.

### How to Choose Between CAN FD and CAN 2.0

- **CAN FD mode** (`compatible = "spacemit,k1-flexcan"`):
  - Supports higher data rates, up to 8 Mbps
  - Supports larger payloads, up to 64 bytes
  - Is backward compatible with CAN 2.0
  - Is recommended for new designs
  - **Note:** Only one mailbox is used by default. In high-load scenarios, packet loss may occur, so fixed frame ID filtering is recommended

- **CAN 2.0 mode** (`compatible = "spacemit,k1-flexcan-can2.0"`):
  - Supports only the legacy CAN 2.0 protocol
  - Supports bitrates up to 1 Mbps
  - Supports payloads up to 8 bytes
  - Uses RX FIFO for better receive performance
  - Is suitable for legacy compatibility or pure CAN 2.0 applications

### What to Do If Packet Loss Occurs Under High Load

In CAN FD mode, only one mailbox is used for reception by default, so packet loss may occur under high load. Possible solutions are listed below:

1. **Configure fixed frame ID filtering** (recommended):
    - Add `flexcan-mailbox-id` and `flexcan-mailbox-id-bits` properties in DTS
    - Assign a dedicated mailbox to each fixed ID that must be received
    - Refer to the **[Advanced Configuration](#advanced-configuration)** section for details

2. **Switch to CAN 2.0 mode**:
    - If the application uses only CAN 2.0 frames, change `compatible` to `spacemit,k1-flexcan-can2.0`
    - CAN 2.0 mode uses RX FIFO and provides stronger buffering capability

3. **Optimize application-level processing**:
    - Improve the application receive-processing speed
    - Use higher-priority threads for CAN receive handling

### How to View CAN Error Information

```shell
# View detailed statistics
ip -details -statistics link show can0

# View kernel logs
dmesg | grep -i can
```