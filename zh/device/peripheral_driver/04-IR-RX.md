# IR-RX

介绍IR的配置和调试方式

## 模块介绍  

红外接收模块主要功能是接收红外信号。  

### 功能介绍  

![](static/ir.jpg) 

在 K1 平台中外接红外接收头(解调器)收到解调后的电信号在驱动和内核IR框架中进行解码并上报事件。

### 源码结构介绍

IR-RX控制器驱动代码在 `drivers/media/rc` 目录下：  

```  
drivers/media/rc  
|--rc-ir-raw.c            #内核ir框架接口代码
|--ir-nec-decoder.c       #内核ir解码电信号代码
|--ir-spacemit.c          #k1 ir驱动  
```  

## 关键特性  

- 可配置噪声阈值 
- 32Bytes大小RX FIFO 

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_IR_SPACEMIT=y

```
Symbol: IR_SPACEMIT [=y]
Device Drivers
    -> Remote Controller support (RC_CORE [=y])
  -> Remote Controller devices (RC_DEVICES [=y])
   -> SPACEMIT IR remote Recriver control (IR_SPACEMIT [=y])
```

### dts配置

#### pinctrl

可查看 linux 仓库的`arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi`，参考已配置好的 ir 节点配置，如下：

```dts
 pinctrl_ir_rx_1: ir_rx_1_grp {
  pinctrl-single,pins = <
   K1X_PADCONF(GPIO_79, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* ir_rx */
  >;
 };
```

#### dtsi配置示例

dtsi中配置IR控制器基地址和时钟复位资源，正常情况无需改动

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

#### dts配置示例

dts完整配置，如下所示

```dts
 &ircrx {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_ir_rx_1>;
        status = "okay";
 };
```

## 接口描述

### API介绍

常用：

```
int ir_raw_event_store_with_filter(struct rc_dev *dev, struct ir_raw_event *ev)
驱动中调用ir框架实现的接口在中断回调函数中完成信号的存储解码和事件上报。
```

## 测试介绍

可基于K1平台外接红外解调器，连接到上述IR配置的pin上，通过遥控器向解调器发送信号，并在应用层接收码值。

## FAQ
