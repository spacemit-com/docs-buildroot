# RUART

This document describes the RUART module on the K3 RCPU platform (ESOS/RTT), including its role, software architecture, device tree usage, and application-side programming model.

## Module Overview

RUART (R-domain UART) is the serial communication controller on the K3 secondary-core side. It is based on the PXA UART hardware IP and exposes a unified character device interface through the RT-Thread Serial framework. 
On the K3 ESOS secondary-core platform, the RUART module provides the following functions:

- Sending and receiving serial data
- Configuring baud rate, data bits, stop bits, parity, and related parameters
- Receiving data through interrupts
- Registering as a character device through the RT-Thread Serial framework for upper-layer use
- Working with the pinctrl subsystem to multiplex the UART function onto the target PAD

### Functional Architecture

The following diagram illustrates the high-level call path from application or component driver code down to the hardware layer:

```text
Application / component drivers
├─ rt_device_find("uart0")
├─ rt_device_open()
├─ rt_device_read()
├─ rt_device_write()
└─ rt_device_control()

                │
                ▼
RT-Thread Serial framework
├─ components/drivers/serial/serial.c
└─ components/drivers/include/drivers/serial.h

                │
                ▼
SoC UART driver layer
├─ bsp/spacemit/drivers/uart/board_uart.c   # device registration and initialization
├─ bsp/spacemit/drivers/uart/pxa_uart.c     # PXA UART controller operations
└─ bsp/spacemit/drivers/uart/pxa_uart.h     # register definitions and data structures

                │
                ▼
UART controller hardware
└─ uart0 ~ uart4
```

### Source Layout

The driver source code is located in `bsp/spacemit/drivers/uart`:

```text
bsp/spacemit/drivers/uart/
├── board_uart.c      # device tree parsing, device registration, and initialization
├── pxa_uart.c        # low-level PXA UART controller operations (baud rate, data bits, and related configuration)
├── pxa_uart.h        # register bit definitions and private data structures
├── drv_uart.h        # UART enum types and low-level interface declarations
├── uart_service.c    # RPMSG UART service for heterogeneous communication scenarios
└── uart-test.c       # test command implementation
```

## Key Features

| Feature | Description |
| :---- | :---- |
| Number of controllers | Supports 5 controllers: `uart0` - `uart4` |
| Data bits | Supports 5 / 6 / 7 / 8 bits |
| Stop bits | Supports 1 or 2 stop bits |
| Parity | Supports none, odd, and even parity |
| Baud rate | Supports `9600` - `3686400` bps |
| Receive mode | Interrupt-driven receive, with an RX FIFO buffer size of 2048 bytes |
| Default baud rate | `115200` bps |

### Supported Baud Rates

| Baud Rate | Clock Source |
| :---- | :---- |
| `9600` ~ `460800` | 14.48 MHz low-speed clock |
| `921600` / `1843200` / `3686400` | baud rate × 16 clock |

## Configuration

Configuration mainly includes **Kconfig settings** and **DTS settings**.

### Kconfig Settings

**1. Enable the RT-Thread Serial framework:** `RT_USING_SERIAL` (automatically selected by `BSP_USING_UART`)

**2. Enable the UART driver:** `BSP_USING_UART` (default: `y`)

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    [*] Enable UART
```

**3. Enable the UART test command (optional):** `BSP_UART_TEST` (default: `n`)

```text
-> Hardware Drivers Config
  -> On-chip Peripheral Drivers
    -> Enable UART (BSP_USING_UART [=y])
      [ ] Enable uart test driver
```

### Device Tree Settings

#### pinctrl

In `k3-pinctrl.dtsi`, multiple default pin groups are already defined for RUART. For example, `ruart0` provides 3 selectable pin groups:

```dts
ruart0_0_cfg: ruart0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_tx */
        K3_PADCONF(GPIO_135, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_rx */
    >;
};

ruart0_1_cfg: ruart0-1-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_147, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_tx */
        K3_PADCONF(GPIO_148, (MUX_MODE6 | EDGE_NONE | PULL_UP | PAD_DS8))  /* r_uart0_rx */
    >;
};
```

Select the appropriate pin group according to the board schematic. For details, refer to [pinctrl](pinctrl.md).

#### DTSI Configuration Example

The UART controller nodes are defined in `k3.dtsi` and typically do not need to be modified:

```dts
uart1: serial1@c0881200 {
    compatible = "spacemit,pxa-uart1";
    reg = <0xc0881200 0x100>;
    interrupt-parent = <&intc>;
    interrupts = <0 19 0>;
    clocks = <&ccu CLK_RCPU_UART2>, <&ccu CLK_RST_RCPU_UART2>;
    status = "disabled";
};
```

#### Board-Level DTS Example

In the board-level DTS, enable the required UART and select the appropriate pin group:

```dts
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ruart0_0_cfg>;
    status = "okay";
};
```

## API Reference

RUART is accessed through the RT-Thread standard device interface. Device names range: `uart0` - `uart4`.

**Find a device**

```c
rt_device_t rt_device_find(const char *name);
```

- `name`: device name, for example `"uart0"`
- Return value: device handle; returns `RT_NULL` on failure

**Open a device**

```c
rt_err_t rt_device_open(rt_device_t dev, rt_uint16_t oflag);
```

- Common `oflag` values:
    - `RT_DEVICE_FLAG_RDWR`: read/write mode
    - `RT_DEVICE_FLAG_INT_RX`: interrupt-driven receive mode

**Write data**

```c
rt_size_t rt_device_write(rt_device_t dev, rt_off_t pos, const void *buffer, rt_size_t size);
```

- `pos`: always pass `0`
- Return value: actual number of bytes written

**Read data**

```c
rt_size_t rt_device_read(rt_device_t dev, rt_off_t pos, void *buffer, rt_size_t size);
```

- `pos`: always pass `0`
- Return value: actual number of bytes read

**Configure serial parameters**

```c
rt_err_t rt_device_control(rt_device_t dev, int cmd, void *arg);
```

- `cmd`: `RT_DEVICE_CTRL_CONFIG`
- `arg`: `struct serial_configure *`, used to configure baud rate, data bits, and related parameters

**Close a device**

```c
rt_err_t rt_device_close(rt_device_t dev);
```

## Usage Examples

### Write Data Example

```c
rt_device_t dev = rt_device_find("uart0");
rt_device_open(dev, RT_DEVICE_FLAG_RDWR);

char *msg = "Hello RUART!\r\n";
rt_device_write(dev, 0, msg, rt_strlen(msg));

rt_device_close(dev);
```

### Read Data Example

```c
rt_device_t dev = rt_device_find("uart0");
rt_device_open(dev, RT_DEVICE_FLAG_INT_RX);

char ch;
while (rt_device_read(dev, 0, &ch, 1) == 1) {
    rt_kprintf("recv: 0x%02x\n", ch);
}

rt_device_close(dev);
```

### Baud Rate Configuration Example

```c
rt_device_t dev = rt_device_find("uart0");

struct serial_configure config = RT_SERIAL_CONFIG_DEFAULT;
config.baud_rate = 921600;
rt_device_control(dev, RT_DEVICE_CTRL_CONFIG, &config);
```

## Application Development

For a complete test example, refer to `bsp/spacemit/drivers/uart/uart-test.c`.

## Debugging

### MSH Command Line

After `BSP_UART_TEST` is enabled, the `uart` command is available in MSH for debugging.

**Show help:**

```sh
msh > uart
```

**Initialize the target port:**

```sh
msh > uart init uart0
```

**Send test data:**

```sh
msh > uart send
```

**Receive data (default timeout: 5000 ms):**

```sh
msh > uart recv 3000
```

**Change the baud rate:**

```sh
msh > uart baud 921600
```

**Run the full test (requires a TX-RX loopback connection):**

```sh
msh > uart test
```

This test iterates through 9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600, 1843200, and 3686400 bps, verifies transmit and receive one byte at a time, and finally reports `PASS` or `FAIL`.

### Check Device Registration

```sh
msh > list_device
device           type         ref count
-------- -------------------- ----------
uart0    Character Device     0
uart1    Character Device     0
```

## FAQ

### UART Device Not Found

1. Confirm that `BSP_USING_UART` is enabled.
2. Confirm that the corresponding UART node in the board-level DTS is set to `status = "okay"`.
3. Check the kernel log to confirm driver initialization, for example: `dmesg | grep uart`.

### Garbled Receive Data

1. Confirm that both ends use the same baud rate.
2. Confirm that data bits, stop bits, and parity settings match.
3. Check whether the selected pinctrl group matches the board schematic.

### Baud Rate Change Fails

Before calling `rt_device_control` to change the baud rate, the device must be closed and not in use by another thread. Otherwise, `-RT_EBUSY` is returned.