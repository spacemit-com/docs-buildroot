# IR-RX

IR Functionality and Usage Guide.

## Overview

The Infrared Receiver (IR-RX) module captures and processes IR signals from remotes or other sources.

### Functional Description

![](static/ir.jpg)
On the K1 platform, an external infrared receiver (demodulator) receives the demodulated electrical signal, which is then decoded and reported as events within the driver and the kernel IR framework.

### Source Code Structure
  
The IR-RX controller driver code is located in the `drivers/media/rc` directory:

```  
drivers/media/rc  
|--rc-ir-raw.c            # Kernel IR framework interface code
|--ir-nec-decoder.c       # Kernel IR decoder for NEC signal protocol  
|--ir-spacemit.c          # K1 IR driver 
```  

## Key Features

- Configurable noise threshold
- 32Bytes RX FIFO size

## Configuration

It mainly includes driver enablement configuration and dts configuration.

### CONFIG Configuration

CONFIG_IR_SPACEMIT=y

```
Symbol: IR_SPACEMIT [=y]
Device Drivers
    -> Remote Controller support (RC_CORE [=y])
  -> Remote Controller devices (RC_DEVICES [=y])
   -> SPACEMIT IR remote Recriver control (IR_SPACEMIT [=y])
```

### DTS Configuration

#### pinctrl

You can refer to the PWM node configuration already set up in the Linux repository at:
`arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi` as shown below:

```dts
 pinctrl_ir_rx_1: ir_rx_1_grp {
  pinctrl-single,pins = <
   K1X_PADCONF(GPIO_79, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* ir_rx */
  >;
 };
```

#### dtsi Configuration Example

Configure the base address of the IR controller and the clock reset resources in the dtsi file. Typically, these settings stay unchanged.

```dts
 ircrx: irc-rx@d4017f00 {
  compatible = "spacemit,k1x-irc";
  reg = <0x0 0xd4017f00 0x0 0x100>;
  interrupts = <69>;
  interrupt-parent = <&intc>;
  clocks = <&ccu CLK_IR>;
  resets = <&reset RESET_IR>;
  clock-frequency = <102400000>;
  status = "disabled";
 };
```

#### dts Configuration Example

The complete DTS configuration is shown below.

```dts
 &ircrx {
  pinctrl-names = "default";
  pinctrl-0 = <&pinctrl_ir_rx_1>;
  status = "okay";
 };
```

## Interface

### API

Commonly used:

```
int ir_raw_event_store_with_filter(struct rc_dev *dev, struct ir_raw_event *ev)
```

The driver calls the interface implemented by the IR framework to complete signal storage, decoding, and event reporting in the interrupt callback function.

## Testing

You can use an external infrared demodulator connected to the K1 platform, link it to the IR-configured pin as mentioned above, send signals to the demodulator using a remote control, and receive the code values at the application layer.

## FAQ
