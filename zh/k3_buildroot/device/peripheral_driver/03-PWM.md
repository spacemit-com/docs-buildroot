# PWM

介绍 PWM 的功能和使用方法。

## 模块介绍

PWM（Pulse Width Modulation，脉宽调制）用于输出周期和占空比可配置的脉冲信号，常用于背光调光、电机调速、蜂鸣器驱动等场景。

### 功能介绍

Linux PWM 子系统通常由两部分组成：

- **PWM 控制器驱动**：负责访问硬件寄存器，完成周期、占空比、使能等配置
- **PWM consumer 驱动**：其它功能模块通过 `pwms` 属性申请 PWM，并按自身业务场景设置输出参数

K3 上的 PWM 控制器驱动复用了 PXA 系列 PWM 驱动框架，实际由 SpacemiT 在该驱动上做了扩展，以支持 K1/K3 SoC 的 PWM 控制器。

### 源码结构介绍

K3 PWM 相关代码和描述文件主要位于以下位置：

```text
Linux-6.18/
|-- drivers/pwm/pwm-pxa.c
|-- drivers/pwm/Kconfig
|-- drivers/pwm/Makefile
|-- Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3-pinctrl.dtsi
```

其中：

- `drivers/pwm/pwm-pxa.c` 是实际驱动文件
- `Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml` 是当前使用的 dt-binding
- `k3.dtsi` 定义 AP 域 PWM 控制器节点
- `k3-rdomain.dtsi` 定义 RCPU 域 RPWM 控制器节点
- `k3-pinctrl.dtsi` 给出了 PWM/RPWM 对应的复用管脚配置

## 关键特性

| 特性 | 特性说明 |
| :----- | :----- |
| 支持周期配置 | 支持按照纳秒单位设置 PWM period |
| 支持占空比配置 | 支持设置 duty cycle |
| 支持使能/关闭 | 通过 PWM framework 统一控制输出 |
| 支持多路 PWM 控制器 | K3 在 AP 域和 RCPU 域都提供多路 PWM |
| 支持标准 consumer 用法 | 其它设备可通过 `pwms = <...>` 申请 PWM |
| 仅支持正常极性 | 驱动中仅接受 `PWM_POLARITY_NORMAL` |

## 配置介绍

主要包括驱动 **CONFIG 使能配置** 和 **dts 配置**。

### CONFIG 配置

K3 PWM 依赖的核心配置项为：

- `CONFIG_PWM`
- `CONFIG_PWM_SYSFS`（如需 sysfs 方式调试）
- `CONFIG_PWM_PXA`

其中 `drivers/pwm/Kconfig` 中可看到：

```text
config PWM_PXA
	tristate "PXA PWM support"
	depends on ARCH_PXA || ARCH_MMP || ARCH_SPACEMIT || COMPILE_TEST
```

说明 SpacemiT 平台上的 PWM 使用的是 `PWM_PXA` 这套驱动。

### DTS 配置

#### 1. PWM 控制器节点

K3 的 AP 域 PWM 节点定义在 `arch/riscv/boot/dts/spacemit/k3.dtsi` 中，例如：

```dts
pwm0: pwm@d401a000 {
    compatible = "spacemit,k1-pwm", "marvell,pxa910-pwm";
    reg = <0x0 0xd401a000 0x0 0x10>;
    clocks = <&syscon_apbc CLK_APBC_PWM0>,<&syscon_apbc CLK_APBC_PWM0_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_apbc RESET_APBC_PWM0>;
    #pwm-cells = <3>;
    k1,pwm-disable-fd;
    status = "disabled";
};
```

RCPU 域 RPWM 节点定义在 `arch/riscv/boot/dts/spacemit/k3-rdomain.dtsi` 中，例如：

```dts
rpwm0: pwm@c088d100 {
    compatible = "spacemit,k1-pwm", "marvell,pxa910-pwm";
    reg = <0x0 0xc088d100 0x0 0x10>;
    clocks = <&syscon_rcpu_pwmctrl CLK_RCPU_PWMCTRL_RPWM0>,
             <&syscon_rcpu_pwmctrl CLK_RCPU_PWMCTRL_RPWM0_BUS>;
    clock-names = "func", "bus";
    resets = <&syscon_rcpu_pwmctrl RESET_RCPU_PWMCTRL_PWM0>;
    #pwm-cells = <3>;
    k1,pwm-disable-fd;
    status = "disabled";
};
```

#### 2. dt-binding 说明

当前 binding 文件是：

```text
Documentation/devicetree/bindings/pwm/marvell,pxa-pwm.yaml
```

其中对 SpacemiT 的兼容项定义为：

```yaml
compatible:
  oneOf:
    - items:
        - const: spacemit,k1-pwm
        - const: marvell,pxa910-pwm
```

并且 binding 还说明：

当 compatible 包含 `spacemit,k1-pwm` 时, `#pwm-cells` 必须为 `3`

这也是 K3 dtsi 中实际写法的来源。

#### 3. `#pwm-cells = <3>` 的含义

K3 PWM 控制器节点使用：

```dts
#pwm-cells = <3>;
```

因此 consumer 侧 `pwms` 通常由 3 个 cell 组成：

```dts
pwms = <&pwmX channel period_ns polarity>;
```

对于当前这类单通道 PWM 控制器来说，`channel` 一般为 `0`。

因此一个典型写法可以理解为：

```dts
pwms = <&pwm0 0 50000 PWM_POLARITY_NORMAL>;
```

各 cell 分别表示：

1. `&pwm0`：引用哪个 PWM 控制器
2. `0`：PWM 通道号
3. `50000`：PWM 周期，单位 ns
4. `PWM_POLARITY_NORMAL`：PWM 极性, 驱动仅支持该值

其中注意PWM周期约束条件:
- 最小值：必须大于 0。
- 最大值限制：受限于输入时钟频率 f_clk。其最大周期计算公式为：T_max = (10^9 * 64 * 1024) / f_clk (单位: ns)
- 溢出检查：驱动内部会计算总周期计数（Period Cycles），若 (period_ns * f_clk) / 10^9 的结果超过 65536 (64 * 1024)，驱动将返回 -EINVAL 错误。

#### 4. `k1,pwm-disable-fd` 属性说明

K3 dtsi 中 PWM 节点普遍带有：

```dts
k1,pwm-disable-fd;
```

对应驱动 `drivers/pwm/pwm-pxa.c` 中的解析逻辑：

```c
if (of_get_property(pdev->dev.of_node, "k1,pwm-disable-fd", NULL))
    pc->dcr_fd = 0;
else
    pc->dcr_fd = 1;
```

因此该属性用于控制驱动是否关闭 `PWMDCR_FD` 相关特性。当前 K3 dtsi 中默认带上该属性，说明平台默认选择关闭这一行为，并按照平台适配后的方式配置 PWM。

这一点也说明：虽然基础驱动来自 `pwm-pxa.c`，但 SpacemiT 对该驱动做了定制扩展，文档应以当前平台的实际 dts 用法为准。

## 管脚配置

PWM 想输出到外部引脚，除了打开 PWM 控制器节点，还必须在 pinctrl 中把对应管脚切换成 PWM 功能。

K3 的 PWM/RPWM pinmux 配置主要定义在：

```text
arch/riscv/boot/dts/spacemit/k3-pinctrl.dtsi
```

例如：

```dts
pwm0_0_cfg: pwm0-0-cfg {
    pwm0-0-pins {
        pinmux = <K3_PADCONF(0, 3)>;    /* pwm0 */
    };
};

pwm0_1_cfg: pwm0-1-cfg {
    pwm0-0-pins {
        pinmux = <K3_PADCONF(42, 6)>;   /* pwm0 */
    };
};
```

这说明同一个 PWM 控制器在 K3 上可能有多组可选引脚复用方案。

再例如 RCPU 域：

```dts
rpwm0_0_cfg: rpwm0-0-cfg {
    rpwm0-0-pins {
        pinmux = <K3_PADCONF(15, 3)>;   /* rpwm0 */
    };
};

rpwm0_1_cfg: rpwm0-1-cfg {
    rpwm0-0-pins {
        pinmux = <K3_PADCONF(27, 3)>;   /* rpwm0 */
    };
};
```

因此实际使用 PWM 时，必须同时完成：

1. 使能对应 PWM 控制器节点
2. 在设备或板级 dts 中选择正确的 pinctrl 配置

## 使用说明

### 使能 PWM 控制器

如果要启用某一路 PWM，通常需要先在板级 dts 中把对应节点改为 `okay`，例如：

```dts
&pwm0 {
    status = "okay";
};
```

如果使用的是 RCPU 域 PWM，则类似：

```dts
&rpwm0 {
    status = "okay";
};
```

### 选择管脚复用

同时需要给对应外设或板级节点选择正确的 pinctrl 配置。例如如果板级设计使用的是 `pwm0_0_cfg` 这组复用，则需要把相关 pinctrl state 配到对应位置。

如果只是作为单独输出口使用，常见做法是在板级中直接给 PWM 所在功能节点或相关外围节点引用对应 pinctrl 状态。

### 作为 consumer 使用

K3 板级 dts 中当前能直接检索到的 `pwms` 实例主要来自 EC 提供的 PWM，而不是 SoC 内建 PWM，例如：

```dts
pwms = <&cros_ec_pwm 0>;
```

这说明：

- SoC 内建 PWM 控制器节点已经在 `k3.dtsi` / `k3-rdomain.dtsi` 中定义好
- 但板级 dts 是否真正使用某一路 SoC PWM，还要取决于具体产品设计

因此客户在新增 SoC PWM 功能时，通常需要自己在板级 dts 中补充 consumer 节点的 `pwms` 引用。

一个概念性示例如下：

```dts
pwm-test {
    compatible = "pwm-backlight";
    pwms = <&pwm0 0 50000 PWM_POLARITY_NORMAL>;
    brightness-levels = <0 64 128 255>;
    default-brightness-level = <3>;
    status = "okay";
};
```

如果该场景还需要把 PWM0 输出到外部引脚，则还需要保证对应 pinmux 已经切到 `pwm0` 功能。

## 驱动实现说明

K3 PWM 驱动核心在 `drivers/pwm/pwm-pxa.c` 中。

### 1. 周期和占空比计算

驱动注释中给出了计算公式：

```c
/*
 * period_ns = 10^9 * (PRESCALE + 1) * (PV + 1) / PWM_CLK_RATE
 * duty_ns   = 10^9 * (PRESCALE + 1) * DC / PWM_CLK_RATE
 */
```

驱动会根据用户请求的 `period` 和 `duty_cycle` 计算：

- `prescale`
- `pv`
- `dc`

然后写入：

- `PWMCR`
- `PWMDCR`
- `PWMPCR`

### 2. 仅支持正常极性

在 `pxa_pwm_apply()` 中：

```c
if (state->polarity != PWM_POLARITY_NORMAL)
    return -EINVAL;
```

所以如果 consumer 侧尝试配置反向极性，驱动会直接返回错误。

### 3. 时钟使能流程

PWM 参数配置前，驱动会先：

```c
clk_prepare_enable(pc->clk);
```

配置完成后，再按当前状态决定是否关闭时钟。因此如果 DTS 中时钟资源没有配正确，PWM 无法正常输出。

### 4. 占空比为 100% 的处理

当 `duty_ns == period_ns` 时，驱动会走特殊路径处理 `PWMDCR_FD` 相关逻辑。SpacemiT 平台又增加了 `k1,pwm-disable-fd` 这个 dts 属性来控制该行为，因此这一点和传统 PXA 平台文档写法并不完全一样。

## K3 PWM 节点概览

从 `k3.dtsi` 和 `k3-rdomain.dtsi` 可以看到，K3 上至少包含两类 PWM：

- **AP 域 PWM**：`pwm0` ~ `pwm19`
- **RCPU 域 PWM**：`rpwm0` ~ `rpwm9`

这说明 K3 的 PWM 资源数量较多，实际使用时要特别注意：

- 当前启用的是 AP 域还是 RCPU 域控制器
- 对应的时钟/复位资源来自哪个 syscon
- 对应 pinmux 是否可引出到板级所需管脚

## 调试建议

如果 PWM 没有输出，建议按以下顺序排查：

1. 确认内核已使能 `CONFIG_PWM_PXA`
2. 确认对应 `pwmX` 或 `rpwmX` 节点状态为 `okay`
3. 确认 `clocks`、`clock-names`、`resets` 配置正确
4. 确认对应 pinctrl 已切换到 PWM 功能
5. 确认 consumer 节点中的 `pwms` 参数格式符合 `#pwm-cells = <3>`
6. 确认未使用 `PWM_POLARITY_INVERSED`
7. 如涉及 100% 占空比场景，注意 `k1,pwm-disable-fd` 对行为的影响
