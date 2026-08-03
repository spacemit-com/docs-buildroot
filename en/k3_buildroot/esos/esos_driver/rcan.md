# RCAN

This document describes the RCAN module on the K3 RCPU (ESOS/RTT) platform, including its role, software architecture, device tree configuration, and application programming model.

## Module Overview

RCAN (R-domain CAN) is the CAN bus controller on the K3 small-core. It is based on the FlexCAN hardware IP and provides a unified CAN device interface to upper layers via the RT-Thread CAN framework. In the K3 small-core ESOS environment, RCAN is responsible for the following:

- Support for CAN 2.0A/B and CAN FD protocols
- Support for standard frames (11-bit ID) and extended frames (29-bit ID)
- Flexible receive filter configuration using ID + Mask
- Interrupt-driven CAN message transmission and reception
- Loopback mode for self-test
- Registration as a CAN device through the RT-Thread CAN framework for application use
- Integration with the pinctrl subsystem to mux CAN functionality onto the target PAD pins

### Functionality

The call flow from the application layer to the hardware is as follows:

```text
Application / Component driver
├─ rt_device_find("flexcan0")
├─ rt_device_open()
├─ rt_device_read()
├─ rt_device_write()
└─ rt_device_control()

                │
                ▼
RT-Thread CAN Framework
├─ components/drivers/can/can.c
└─ components/drivers/include/drivers/can.h

                │
                ▼
SoC CAN driver layer
├─ bsp/spacemit/drivers/can/drv_can.c       # Device registration and RT-Thread interface adaptation
├─ bsp/spacemit/drivers/can/fsl_flexcan.c   # FlexCAN controller low-level operations
├─ bsp/spacemit/drivers/can/fsl_flexcan.h   # Register definitions and data structures
└─ bsp/spacemit/drivers/can/flexcan-test.c  # Test command implementation

                │
                ▼
FlexCAN Controller Hardware
└─ flexcan0 ~ flexcan4
```

### Source Code Structure

The driver source code is located in the `bsp/spacemit/drivers/can` directory:

```text
bsp/spacemit/drivers/can/
├── drv_can.c         # Device tree parsing, device registration and RT-Thread CAN framework adaptation
├── drv_can.h         # CAN register bit definitions and macros
├── fsl_flexcan.c     # FlexCAN controller low-level operations (initialization, send/receive, filter configuration)
├── fsl_flexcan.h     # FlexCAN register definitions and data structures
├── flexcan-test.c    # Implementation of MSH test commands (loopback/send/recv/filter/stress)
└── SConscript        # Build script
```

## Key Features

| Feature | Description |
| :---- | :---- |
| Controller Count | 5 controllers: `flexcan0` ~ `flexcan4` |
| Supported Protocols | CAN 2.0A/B, CAN FD |
| Supported Frames | Standard frames (11-bit ID) and extended frames (29-bit ID) |
| Data Length | CAN 2.0: 0~8 bytes; CAN FD: 0~64 bytes |
| Baud Rate | Arbitration phase: up to 1 Mbps; data phase (FD): up to 5 Mbps |
| Receive Mailboxes | 12 independent RX mailboxes with ID + Mask filtering |
| Transmit Mailboxes | 1 TX mailbox (MB13) |
| Operating Modes | Normal mode, loopback mode, listen-only mode |
| Clock Source | 80 MHz (from PLL6_80) |

### Supported Baud Rate Configurations

| Arbitration Phase Baud Rate | Data Phase Baud Rate (FD) | Typical Application |
| :---- | :---- | :---- |
| 125 kbps | - | Low-speed CAN network |
| 250 kbps | - | Industrial control |
| 500 kbps | 1 Mbps | Automotive electronics (default configuration) |
| 1 Mbps | 2 Mbps | High-speed CAN FD |

## Configuration

This section covers both **Kconfig** and **DTS** configuration.

### Kconfig Configuration

**1. Enable RT-Thread CAN Framework:** `RT_USING_CAN` (automatically selected by `BSP_USING_CAN`)

**2. Enable CAN FD Support:** `RT_CAN_USING_CANFD` (automatically selected by `BSP_USING_CAN`)

**3. Enable CAN Driver:** `BSP_USING_CAN` (default: y)

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    [*] Enable CAN
```

### DTS Configuration

#### pinctrl

In `k3-pinctrl.dtsi`, multiple default pin configurations are provided for RCAN. For example, `rcan0` provides two selectable pin groups:

```dts
rcan0_0_cfg: rcan0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_57, (MUX_MODE3 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 rx */
        K3_PADCONF(GPIO_58, (MUX_MODE3 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 tx */
    >;
};

rcan0_1_cfg: rcan0-1-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_90, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 rx */
        K3_PADCONF(GPIO_91, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* rcan0 tx */
    >;
};
```

Select the appropriate pin group based on the board-level schematic. For details, refer to [pinctrl](https://www.spacemit.com/community/document/info?lang=en&nodepath=software/SDK/buildroot/k3_buildroot/esos/esos_driver/pinctrl.md).

#### Example: DTSI Configuration

The CAN controller nodes are defined in `k3.dtsi`, and usually do not need to be modified.

```dts
flexcan0: can0@c0710000 {
    compatible = "spacemit,flexcan0";
    reg = <0xc0710000 0x10000>;
    interrupt-parent = <&intc>;
    interrupts = <0 36 0>;
    clocks = <&ccu CLK_RCPU_CAN0>, <&ccu CLK_RST_RCPU_CAN0>;
    clock-frequency = <80000000>;
    status = "disabled";
};
```

#### Example: Board-Level DTS Configuration

Enable the required CAN controllers and select the pin groups in the board-level DTS:

```dts
&flexcan0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan0_0_cfg>;
    status = "okay";
};

&flexcan1 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan1_0_cfg>;
    status = "okay";
};

&flexcan2 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan2_0_cfg>;
    status = "okay";
};

&flexcan3 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan3_0_cfg>;
    status = "okay";
};

&flexcan4 {
    pinctrl-names = "default";
    pinctrl-0 = <&rcan4_0_cfg>;
    status = "okay";
};
```

## Interface

RCAN devices are accessed via the standard RT-Thread CAN device interface. Device names range from `flexcan0` to `flexcan4`.

**Find Device**

```c
rt_device_t rt_device_find(const char *name);
```

- `name`: Device name, for example `"flexcan0"`
- Return value: Device handle; returns `RT_NULL` on failure

**Configure CAN Parameters**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`: `RT_DEVICE_CTRL_CONFIG`
- `arg`: `struct can_configure *`, used to configure baud rate, operating mode, CAN FD enable, etc.

**Open Device**

```c
rt_err_t rt_device_open(rt_device_t dev, rt_uint16_t oflag);
```

- Common `oflag` values:
  - `RT_DEVICE_FLAG_INT_TX`: Interrupt-driven transmission mode
  - `RT_DEVICE_FLAG_INT_RX`: Interrupt-driven reception mode

**Send a CAN Message**

```c
rt_size_t rt_device_write(rt_device_t dev, rt_off_t pos, const void *buffer, rt_size_t size);
```

- `pos`: always `0`
- `buffer`: `struct rt_can_msg *`
- `size`: `sizeof(struct rt_can_msg)`
- Return value: Returns `sizeof(struct rt_can_msg)` on success

**Receive a CAN Message**

```c
rt_size_t rt_device_read(rt_device_t dev, rt_off_t pos, void *buffer, rt_size_t size);
```

- `pos`: always `0`
- `buffer`: `struct rt_can_msg *`
- `size`: `sizeof(struct rt_can_msg)`
- Return value: Returns `sizeof(struct rt_can_msg)` on success

**Set the Receive Filter**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`: `RT_CAN_CMD_SET_FILTER`
- `arg`: `struct rt_can_filter_config *`, used to configure the filter ID and mask

**Close Device**

```c
rt_err_t rt_device_close(rt_device_t dev);
```

## Usage Examples

### Example: Send standard frames

```c
rt_device_t dev = rt_device_find("flexcan0");

struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.mode = RT_CAN_MODE_NORMAL;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

rt_device_open(dev, RT_DEVICE_FLAG_INT_TX | RT_DEVICE_FLAG_INT_RX);

struct rt_can_msg msg = {0};
msg.ide = RT_CAN_STDID;
msg.rtr = RT_CAN_DTR;
msg.id = 0x123;
msg.len = 8;
for (int i = 0; i < 8; i++)
    msg.data[i] = i;

rt_device_write(dev, 0, &msg, sizeof(msg));
rt_device_close(dev);
```

### Example: Receive a Message

```c
rt_device_t dev = rt_device_find("flexcan0");

struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.mode = RT_CAN_MODE_NORMAL;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

rt_device_open(dev, RT_DEVICE_FLAG_INT_RX);

struct rt_can_msg msg;
if (rt_device_read(dev, 0, &msg, sizeof(msg)) == sizeof(msg))
{
    rt_kprintf("RX: ID=0x%X len=%d\n", msg.id, msg.len);
}

rt_device_close(dev);
```

### Example: Send a CAN FD Frame

```c
struct can_configure cfg = CANDEFAULTCONFIG;
cfg.baud_rate = CAN500kBaud;
cfg.baud_rate_fd = CAN1MBaud;
cfg.enable_canfd = 1;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &cfg);

struct rt_can_msg msg = {0};
msg.ide = RT_CAN_STDID;
msg.rtr = RT_CAN_DTR;
msg.id = 0x200;
msg.len = 12;
msg.fd_frame = 1;
msg.brs = 1;  /* data phase uses 1 Mbps */
for (int i = 0; i < 12; i++)
    msg.data[i] = i;

rt_device_write(dev, 0, &msg, sizeof(msg));
```

### Example: Configure a Receive Filter

```c
struct rt_can_filter_item filter;
filter.ide = RT_CAN_STDID;
filter.rtr = RT_CAN_DTR;
filter.id = 0x200;
filter.mask = 0x7FF;  /* exact match for 0x200 */
filter.mode = 0;
filter.hdr_bank = 0;

struct rt_can_filter_config cfg;
cfg.count = 1;
cfg.actived = 1;
cfg.items = &filter;

rt_device_control(dev, RT_CAN_CMD_SET_FILTER, &cfg);
```

## Application Development

Refer to `bsp/spacemit/drivers/can/flexcan-test.c` for complete test code.

## Debugging

### MSH Command Line

You can use the `can_test` command in MSH for debugging.

**View Help:**

```sh
msh > can_test
```

**Loopback Self-Test (Requires CAN FD Support):**

```sh
msh > can_test loopback flexcan0
```

This test sends three frames (classic CAN with 8 bytes, FD with 12 bytes and no BRS, and FD with 12 bytes and BRS enabled) and verifies that all frames are received successfully.

**Send Single Frame:**

```sh
msh > can_test send flexcan0 0x123 de ad be ef
```

Sends a standard frame with ID=0x123 and data `DE AD BE EF`.

**Receive Frames:**

```sh
msh > can_test recv flexcan0 4
```

Receives and prints 4 frames.

**Set Filter and Receive:**

```sh
msh > can_test filter flexcan0 0x200 0x7FF
```

Sets the filter to exactly match ID=0x200, then receives 1 frame.

**Stress Test (Requires TX-RX Short Connection):**

```sh
msh > can_test stress flexcan0 200
```

Sends 200 frames in loopback mode and verifies that the transmit and receive counts match.

### Check Device Registration Status

```sh
msh > list_device
device           type         ref count
-------- -------------------- ----------
flexcan0 CAN Device           0
flexcan1 CAN Device           0
```

## FAQ

### CAN Device Not Found

1. Confirm that `BSP_USING_CAN` is enabled.
2. Confirm that the corresponding CAN node in the board-level DTS is set to `status = "okay"`.
3. Check the kernel log to confirm driver initialization: `dmesg | grep can`

### Transmission Failed or No Data Received

1. Confirm that a 120Ω termination resistor is present on the CAN bus.
2. Confirm that the sender and receiver use the same baud rate.
3. Verify that the selected pinctrl group matches the board schematic.
4. Use loopback mode (`RT_CAN_MODE_LOOPBACK`) to isolate hardware issues.

### CAN FD Frame Transmission Failed

1. Confirm that `cfg.enable_canfd = 1` is set.
2. Confirm that `msg.fd_frame = 1` is set.
3. Confirm that the peer device supports the CAN FD protocol.
