# IR-RX

本文介绍 IR-RX（红外接收）的配置和调试方式。

## 模块介绍  

**IR-RX（Infrared Receiver，红外接收）**模块主要功能是接收红外信号。

### 功能介绍  

在 K3 平台中，外接红外接收头（解调器）接收解调后的电信号，在驱动和内核 IR 框架中进行解码并上报事件。

**IR-RX 工作流程：**

```
红外遥控器
    ↓ (红外信号)
红外接收头（解调器）
    ↓ (电信号)
IR-RX 控制器
    ↓
IR-RX 驱动
    ↓
内核 RC 框架
    ↓
IR 解码器（NEC/RC5/RC6 等）
    ↓
输入子系统
    ↓
用户空间应用
```

### 源码结构介绍

IR-RX 控制器驱动代码在 `drivers/media/rc` 目录下：

```
drivers/media/rc
|--rc-core.c              # RC 框架核心代码
|--rc-ir-raw.c            # 内核 IR 框架接口代码
|--ir-nec-decoder.c       # NEC 协议解码器
|--ir-rc5-decoder.c       # RC5 协议解码器
|--ir-rc6-decoder.c       # RC6 协议解码器
|--ir-spacemit-k1.c       # K3 IR-RX 驱动
```

### 硬件特性

K3 IR-RX 控制器特性：

- 支持 4 个 IR-RX 控制器：2 个主域（ircrx0、ircrx1）+ 2 个 R-domain（r_ircrx0、r_ircrx1）
- 32 Bytes 大小 RX FIFO
- 可配置噪声阈值
- 工作频率：100 kHz
- 基础时钟频率：102.4 MHz
- 支持多种红外协议（NEC、RC5、RC6 等）

## 关键特性  

- 支持 4 路红外接收通道（2 路主域 + 2 路 R-domain）
- 32 Bytes RX FIFO
- 可配置噪声阈值
- 支持多种红外协议解码
- 中断驱动方式
- 集成到 Linux RC 框架

## 配置介绍

主要包括 **内核 CONFIG 配置** 和 **DTS 配置**

### 内核 CONFIG 配置

#### 启用 RC 框架

`CONFIG_RC_CORE` 为内核 RC（Remote Controller）框架提供支持。

```
Symbol: RC_CORE [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
```

#### 启用 IR 解码器

需要启用相应的 IR 协议解码器，例如 NEC 协议：

```
Symbol: IR_NEC_DECODER [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
            -> Enable IR raw decoder for the NEC protocol (IR_NEC_DECODER [=y])
```

#### 启用 K3 IR-RX 驱动

需要启用 `CONFIG_IR_SPACEMIT_K1` 以支持 K3 的 IR-RX 驱动。

```
Symbol: IR_SPACEMIT_K1 [=y]
Device Drivers
      -> Remote Controller support (RC_CORE [=y])
            -> Remote Controller devices (RC_DEVICES [=y])
                  -> Spacemit K1/K3 IR remote receiver control (IR_SPACEMIT_K1 [=y])
```

### DTS 配置

#### DTSI 配置示例

在 `dtsi` 文件中定义 IR-RX 控制器的寄存器地址、时钟和复位资源。
通常情况下，该部分**无需修改**。

**IR-RX 控制器节点（k3.dtsi）**

```dts
ircrx0: irc-rx0@d4017e00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017e00 0x0 0x100>;
    interrupts = <69 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR0>;
    resets = <&syscon_apbc RESET_APBC_IR0>;
    clock-frequency = <102400000>;
    status = "disabled";
};

ircrx1: irc-rx1@d4017f00 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xd4017f00 0x0 0x100>;
    interrupts = <20 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_apbc CLK_APBC_IR1>;
    resets = <&syscon_apbc RESET_APBC_IR1>;
    clock-frequency = <102400000>;
    status = "disabled";
};
```

**R-domain IR-RX 控制器节点（k3-rdomain.dtsi）**

K3 平台在 R-domain（RCPU 域）还提供 2 个额外的 IR-RX 控制器，定义在 `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi` 中：

```dts
r_ircrx0: r-irc-rx0@c0887000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc0887000 0x0 0x100>;
    interrupts = <274 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC0>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC0>;
    status = "disabled";
};

r_ircrx1: r-irc-rx1@c088e000 {
    compatible = "spacemit-k1,irc";
    reg = <0x0 0xc088e000 0x0 0x100>;
    interrupts = <275 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&saplic>;
    clocks = <&syscon_rcpu_sysctrl CLK_RCPU_SYSCTRL_RIRC1>;
    resets = <&syscon_rcpu_sysctrl RESET_RCPU_SYSCTRL_RIRC1>;
    status = "disabled";
};
```

与主域节点的主要区别：

- 寄存器基地址位于 R-domain 地址空间（`0xc08xxxxx`）
- 时钟和复位资源来自 `syscon_rcpu_sysctrl`，而非 `syscon_apbc`
- 无 `clock-frequency` 属性
- 中断号不同（274、275）

**属性说明：**

- `compatible`：兼容字符串，标识为 Spacemit K1/K3 IR 控制器
- `reg`：寄存器基地址和大小
- `interrupts`：中断号
- `clocks`：时钟资源
- `resets`：复位资源
- `clock-frequency`：基础时钟频率（102.4 MHz，仅主域节点）

#### DTS 板级配置示例

在板级 DTS 中启用 IR-RX 并配置 pinctrl：

**主域 IR-RX：**

```dts
&ircrx0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ir0_0_cfg>;  /* 使用预定义的 pinctrl 配置 */
    linux,rc-map-name = "rc-empty";  /* 或指定具体的遥控器映射 */
    status = "okay";
};
```

**R-domain IR-RX：**

```dts
&r_ircrx0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rir0_0_cfg>;  /* 使用 R-domain 预定义的 pinctrl 配置 */
    linux,rc-map-name = "rc-empty";
    status = "okay";
};
```

**可选属性：**

- `linux,rc-map-name`：指定遥控器按键映射表
  - `rc-empty`：空映射（默认）
  - `rc-nec-terratec-cinergy-xs`：特定遥控器映射
  - 可在 `drivers/media/rc/keymaps/` 中查看支持的映射表

#### Pinctrl 配置

K3 平台已在 `k3-pinctrl.dtsi` 中预定义了多组 IR-RX 的 pinctrl 配置：

**ircrx0 可选配置：**

```dts
/* 配置 1：GPIO 41，功能 5 */
ir0_0_cfg: ir0-0-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(41, 5)>;  /* ir0 rx */
    };
};

/* 配置 2：GPIO 108，功能 4 */
ir0_1_cfg: ir0-1-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(108, 4)>;  /* ir0 rx */
    };
};
```

**ircrx1 可选配置：**

```dts
/* 配置 1：GPIO 0，功能 4 */
ir1_0_cfg: ir1-0-cfg {
    ir1-0-pins {
        pinmux = <K3_PADCONF(0, 4)>;  /* ir1 rx */
    };
};

/* 配置 2：GPIO 70，功能 4 */
ir1_1_cfg: ir1-1-cfg {
    ir1-0-pins {
        pinmux = <K3_PADCONF(70, 4)>;  /* ir1 rx */
    };
};
```

**R-domain ircrx 可选配置：**

```dts
/* rir0 配置 1：GPIO 40，功能 5 */
rir0_0_cfg: rir0-0-cfg {
    rir0-0-pins {
        pinmux = <K3_PADCONF(40, 5)>;  /* rir0 rx */
    };
};

/* rir0 配置 2：GPIO 71，功能 4 */
rir0_1_cfg: rir0-1-cfg {
    rir0-0-pins {
        pinmux = <K3_PADCONF(71, 4)>;  /* rir0 rx */
    };
};
```

**K3_PADCONF 宏定义：**

```c
#define K3_PADCONF(pin, func) (((pin) << 16) | (func))
```

- `pin`：GPIO 引脚号
- `func`：功能复用编号

**可选的 pinctrl 属性：**

```dts
ir0_0_cfg: ir0-0-cfg {
    ir0-0-pins {
        pinmux = <K3_PADCONF(41, 5)>;
        bias-pull-up;           /* 上拉 */
        drive-strength = <25>;  /* 驱动强度 DS8 */
        power-source = <1800>;  /* 电源电压 1.8V */
    };
};
```

> **注意**：根据硬件原理图选择合适的 pinctrl 配置组（ir0_0_cfg、ir0_1_cfg、ir1_0_cfg、ir1_1_cfg 等）。

## 接口说明

### 内核 API

IR-RX 驱动使用 Linux RC 框架提供的接口：

#### 设备注册

```c
struct rc_dev *rc_allocate_device(enum rc_driver_type type);
int rc_register_device(struct rc_dev *dev);
void rc_unregister_device(struct rc_dev *dev);
void rc_free_device(struct rc_dev *dev);
```

#### 事件上报

```c
/* 存储并过滤 IR 原始事件 */
int ir_raw_event_store_with_filter(struct rc_dev *dev, struct ir_raw_event *ev);

/* 处理 IR 原始事件（触发解码） */
void ir_raw_event_handle(struct rc_dev *dev);
```

#### IR 原始事件结构

```c
struct ir_raw_event {
    unsigned pulse:1;      /* 1 = 脉冲，0 = 空闲 */
    unsigned duration:31;  /* 持续时间（微秒） */
};
```

### 用户空间接口

#### 输入设备节点

IR-RX 驱动注册为输入设备，可通过标准输入子系统访问：

```bash
# 查看输入设备
cat /proc/bus/input/devices

# 查看 IR 设备事件
evtest /dev/input/eventX
```

#### sysfs 接口

通过 sysfs 查看和配置 RC 设备：

```bash
# 查看 RC 设备
ls /sys/class/rc/

# 查看支持的协议
cat /sys/class/rc/rc0/protocols

# 启用特定协议
echo nec > /sys/class/rc/rc0/protocols

# 查看当前按键映射
cat /sys/class/rc/rc0/keymap
```

## Debug 说明

### 基本测试

1. 检查 IR-RX 设备是否注册

   ```bash
   ls /sys/class/rc/
   cat /sys/class/rc/rc0/name
   ```

2. 查看驱动加载情况

   ```bash
   dmesg | grep -i "ir\|spacemit"
   ```

3. 查看输入设备

   ```bash
   cat /proc/bus/input/devices | grep -A 10 "spacemit-ir"
   ```

### 红外信号测试

使用 `ir-keytable` 工具测试红外接收：

```bash
# 安装 ir-keytable（如果未安装）
# apt-get install ir-keytable

# 查看 RC 设备信息
ir-keytable

# 测试红外接收（按遥控器按键）
ir-keytable -t

# 清除按键映射
ir-keytable -c

# 加载按键映射
ir-keytable -w /etc/rc_keymaps/nec.toml
```

### 使用 evtest 测试

```bash
# 安装 evtest
# apt-get install evtest

# 列出输入设备
evtest

# 测试 IR 设备（选择对应的 event 编号）
evtest /dev/input/event0

# 按遥控器按键，观察输出
```

### 测试程序示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <linux/input.h>

int main(void)
{
    int fd;
    struct input_event ev;

    fd = open("/dev/input/event0", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    printf("Waiting for IR events...\n");

    while (1) {
        if (read(fd, &ev, sizeof(ev)) == sizeof(ev)) {
            if (ev.type == EV_KEY) {
                printf("Key %s: code=%d, value=%d\n",
                       ev.value ? "pressed" : "released",
                       ev.code, ev.value);
            }
        }
    }

    close(fd);
    return 0;
}
```

## 测试说明

### 硬件连接

1. 准备红外接收头（如 VS1838B、IRM-3638T 等）
2. 连接红外接收头到 K3 开发板：
   - VCC → 3.3V
   - GND → GND
   - OUT → IR-RX GPIO 引脚
3. 准备红外遥控器（NEC 协议）

### 功能测试

1. **基本接收测试**

   ```bash
   # 启用 NEC 协议
   echo nec > /sys/class/rc/rc0/protocols
   
   # 测试接收
   ir-keytable -t
   
   # 按遥控器按键，观察输出
   ```

2. **按键映射测试**

   ```bash
   # 使用 evtest 测试
   evtest /dev/input/event0
   
   # 按遥控器按键，查看键值
   ```

3. **协议切换测试**

   ```bash
   # 查看支持的协议
   cat /sys/class/rc/rc0/protocols
   
   # 切换到 RC5 协议
   echo rc5 > /sys/class/rc/rc0/protocols
   
   # 测试 RC5 遥控器
   ir-keytable -t
   ```

## FAQ

### IR-RX 设备不存在

检查以下几点：

1. 确认内核配置已启用 `CONFIG_RC_CORE`、`CONFIG_IR_NEC_DECODER` 和 `CONFIG_IR_SPACEMIT_K1`
2. 确认 DTS 中 ircrx 节点 status 为 "okay"
3. 检查 pinctrl 配置是否正确
4. 查看内核日志：`dmesg | grep -i "ir\|spacemit"`

### 无法接收红外信号

可能的原因：

1. 红外接收头连接错误或损坏
2. GPIO 引脚配置错误
3. 协议不匹配（遥控器使用的协议与驱动启用的协议不同）
4. 红外接收头供电不足

解决方法：

```bash
# 检查协议配置
cat /sys/class/rc/rc0/protocols

# 启用所有协议
echo "nec rc5 rc6" > /sys/class/rc/rc0/protocols

# 查看中断计数
cat /proc/interrupts | grep ir

# 检查硬件连接
```

### 接收到的键值不正确

可能的原因：

1. 按键映射表不匹配
2. 协议解码错误

解决方法：

```bash
# 清除现有映射
ir-keytable -c

# 使用 ir-keytable 学习模式
ir-keytable -t

# 根据输出的扫描码创建自定义映射表
```

### 如何创建自定义按键映射

1. 使用 `ir-keytable -t` 获取遥控器的扫描码
2. 创建映射文件（TOML 格式）：

   ```toml
   [[protocols]]
   name = "my_remote"
   protocol = "nec"
   
   [protocols.scancodes]
   0x00 = "KEY_POWER"
   0x01 = "KEY_VOLUMEUP"
   0x02 = "KEY_VOLUMEDOWN"
   0x03 = "KEY_MUTE"
   ```

3. 加载映射：

   ```bash
   ir-keytable -w /path/to/my_remote.toml
   ```

### 支持哪些红外协议

K3 IR-RX 驱动支持 Linux RC 框架的所有协议，常用的包括：

- NEC
- RC5
- RC6
- Sony SIRC
- JVC
- Sanyo
- Sharp

查看支持的协议：

```bash
cat /sys/class/rc/rc0/protocols
```

### 如何调试红外信号

启用内核调试信息：

```bash
# 启用 RC 框架调试
echo 8 > /proc/sys/kernel/printk
echo "file drivers/media/rc/* +p" > /sys/kernel/debug/dynamic_debug/control

# 查看调试信息
dmesg -w
```

## 注意事项

1. **硬件连接**：确保红外接收头正确连接，注意 VCC、GND、OUT 引脚
2. **协议匹配**：遥控器使用的协议必须与驱动启用的协议匹配
3. **GPIO 配置**：IR-RX 的 GPIO 引脚需要正确配置 pinctrl
4. **中断冲突**：确保 IR-RX 的中断号没有与其他设备冲突
5. **电源干扰**：红外接收头对电源噪声敏感，建议使用滤波电容
6. **距离和角度**：红外接收有效距离和角度有限，测试时注意位置
7. **环境光干扰**：强光可能影响红外接收，避免在强光下测试

## 参考资料

- Linux RC 框架文档：`Documentation/media/rc-core.rst`
- IR 协议说明：`Documentation/media/rc-protocols.rst`
- 按键映射表：`drivers/media/rc/keymaps/`
