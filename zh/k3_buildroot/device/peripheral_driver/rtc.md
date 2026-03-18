# RTC

本文介绍 K3 平台 RTC（实时时钟）的功能、驱动实现、DTS 配置方式以及常见使用与调试方法。

## 模块介绍

RTC（Real-Time Clock，实时时钟）主要用于：

- 维护系统时间；
- 提供 1Hz / alarm 中断；
- 在系统掉电后，依靠备用电源或 PMIC 内部 RTC 继续保存时间；
- 在部分场景下为唤醒、闹钟、掉电后时间保持提供基础能力。

K3 这块不能只按“K1 那套 PMIC RTC”去理解。当前 SDK 里实际上能看到 **至少三条 RTC 路径**：

1. SoC 内部 RTC：`rtc@d4010000`
2. RPMI RTC：`rpmi_rtc@0`
3. P1 PMIC RTC：`drivers/rtc/rtc-spacemit-p1.c`

所以对用户来说，最重要的不是只记一个驱动名，而是先分清：

- 你现在板子上到底启用的是哪一路 RTC；
- `/dev/rtc0` 最后绑定到的是哪个设备；
- 是否依赖 PMIC / mailbox / backup 电源；
- 你要验证的是“系统时间可读写”，还是“闹钟/唤醒/掉电保持”。

### 功能介绍

![](static/rtc.png)

Linux RTC 子系统通常分三层：

1. **rtc-core**  
   提供 `rtc_device` 注册、标准 ioctl、sysfs 和 `/dev/rtcN` 接口；
2. **RTC 驱动层**  
   实现具体硬件的读时间、写时间、闹钟、中断等；
3. **用户空间接口层**  
   用户通过 `hwclock`、`date`、`ioctl()`、`/sys/class/rtc/rtcN/` 等方式访问。

在 K3 当前 SDK 里，这三类 RTC 来源分别对应：

- **SoC RTC**：DTS 中 `compatible = "mrvl,mmp-rtc"` 的内部 RTC
- **RPMI RTC**：DTS 中 `compatible = "riscv,rpmi-rtc"` 的 mailbox 型 RTC
- **PMIC RTC**：`RTC_DRV_SPACEMIT_P1` 对应的 P1 PMIC RTC

## 源码结构介绍

K3 RTC 相关内容主要位于：

```text
linux-6.18/
|-- drivers/rtc/
|   |-- class.c
|   |-- dev.c
|   |-- interface.c
|   |-- Kconfig
|   |-- Makefile
|   |-- rtc-spacemit-p1.c          # SpacemiT P1 PMIC RTC
|   |-- rtc-rpmi.c                 # RPMI RTC（通用框架）
|   `-- ...
|-- drivers/mfd/
|   `-- simple-mfd-i2c.c           # P1 PMIC 通过 MFD 拆出 regulator / rtc 子设备
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    `-- k3*.dts
```

### K3 当前可见的 RTC 节点

从 `k3.dtsi` 里可以直接看到两类 SoC 侧 RTC 节点：

#### 1. SoC 内部 RTC

```dts
rtc: rtc@d4010000 {
	compatible = "mrvl,mmp-rtc";
	reg = <0x0 0xd4010000 0x0 0x100>;
	interrupt-parent = <&saplic>;
	interrupts = <21 IRQ_TYPE_LEVEL_HIGH>, <22 IRQ_TYPE_LEVEL_HIGH>;
	interrupt-names = "rtc 1Hz", "rtc alarm";
	resets = <&syscon_apbc RESET_APBC_RTC>;
	status = "disabled";
};
```

#### 2. RPMI RTC

```dts
rpmi_rtc: rpmi_rtc@0 {
	compatible = "riscv,rpmi-rtc";
	mboxes = <&mpxy_mbox 0xe 0x0>;
	interrupt-parent = <&saplic>;
	interrupts = <64 IRQ_TYPE_LEVEL_HIGH>;
	interrupt-names = "rpmi rtc";
	status = "okay";
};
```

也就是说，K3 当前平台里，**并不是只有一个 RTC 节点**。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| 多 RTC 来源并存 | K3 当前可见 SoC RTC、RPMI RTC、P1 PMIC RTC |
| 标准 Linux RTC 接口 | 驱动注册后通过 `/dev/rtcN` 和 rtc class 提供访问 |
| SoC RTC 带 1Hz / alarm 中断 | `rtc@d4010000` DTS 中有两路中断 |
| PMIC RTC 通过 MFD 派生 | P1 PMIC 由 `simple-mfd-i2c.c` 拆出 `spacemit-p1-rtc` 子设备 |
| P1 RTC 只实现基础时间读写 | 当前 `rtc-spacemit-p1.c` 没有提供 alarm 功能 |
| clock/reset 仍然重要 | SoC 内部 RTC 依赖 `RESET_APBC_RTC`，属于 APBC 域资源 |

### 1. SoC 内部 RTC 走 APBC 域

从 `k3.dtsi` 可以看到，内部 RTC 节点：

- 地址：`0xd4010000`
- 兼容串：`mrvl,mmp-rtc`
- reset：`<&syscon_apbc RESET_APBC_RTC>`
- 中断：
  - `rtc 1Hz`
  - `rtc alarm`

这说明它是 **SoC 内建 RTC 控制器**，而不是外部 I2C RTC。

### 2. RPMI RTC 是另一条独立路径

`rpmi_rtc@0` 这一路不是 MMIO RTC，而是通过：

- `mboxes = <&mpxy_mbox 0xe 0x0>`

来和底层固件/管理单元交互。

所以如果系统里最后枚举出来的主 RTC 是 RPMI 路径，那调试方法就和传统 MMIO RTC 不完全一样，重点要查：

- mailbox 是否正常；
- RPMI 通道是否可用；
- 中断是否正常上报。

### 3. P1 PMIC RTC 是 MFD 子设备

K3 SDK 中的 P1 PMIC RTC 驱动为：

```text
drivers/rtc/rtc-spacemit-p1.c
```

对应 Kconfig：

```text
config RTC_DRV_SPACEMIT_P1
	tristate "SpacemiT P1 RTC"
	depends on ARCH_SPACEMIT || COMPILE_TEST
	select MFD_SPACEMIT_P1
	default ARCH_SPACEMIT
```

P1 不是单独一个“RTC 总线设备”，而是通过：

```text
drivers/mfd/simple-mfd-i2c.c
```

里的 `simple-mfd-i2c` 先注册成 MFD，再拆出两个子设备：

- `spacemit-p1-regulator`
- `spacemit-p1-rtc`

对应代码里可以直接看到：

```c
static const struct mfd_cell spacemit_p1_cells[] = {
	{ .name = "spacemit-p1-regulator", },
	{ .name = "spacemit-p1-rtc", },
};
```

## 配置介绍

RTC 配置要分三种情况来看。

### 1. SoC 内部 RTC

如果使用 K3 SoC 内部 RTC，核心 DTS 节点来自 `k3.dtsi`：

```dts
rtc: rtc@d4010000 {
	compatible = "mrvl,mmp-rtc";
	reg = <0x0 0xd4010000 0x0 0x100>;
	interrupt-parent = <&saplic>;
	interrupts = <21 IRQ_TYPE_LEVEL_HIGH>, <22 IRQ_TYPE_LEVEL_HIGH>;
	interrupt-names = "rtc 1Hz", "rtc alarm";
	resets = <&syscon_apbc RESET_APBC_RTC>;
	status = "disabled";
};
```

如果板级要启用，通常是在 board DTS 里把它改成：

```dts
&rtc {
	status = "okay";
};
```

### 2. RPMI RTC

RPMI RTC 在 `k3.dtsi` 当前已经是：

```dts
status = "okay";
```

如果系统实际使用这一路 RTC，需要确认：

- `mpxy_mbox` 已正常工作；
- `riscv,rpmi-rtc` 驱动已使能；
- 中断和 mailbox 通信都正常。

### 3. P1 PMIC RTC

P1 PMIC RTC 不是直接在 DTS 里声明一个 `spacemit-p1-rtc` 节点，而是先声明 P1 PMIC 本体，例如板级 DTS 里能看到：

```dts
p1@41 {
	compatible = "spacemit,p1";
	reg = <0x41>;
	status = "disabled";
	...
};
```

当 `simple-mfd-i2c` 识别到：

```dts
compatible = "spacemit,p1";
```

就会在内核里自动派生出：

- `spacemit-p1-regulator`
- `spacemit-p1-rtc`

所以用户在 DTS 层真正要做的是：

- 把 P1 PMIC 所在的 I2C 控制器配好；
- 把 `p1@41` 节点本身配对；
- 确保 PMIC 节点状态正确、供电关系正常；
- 然后再让 MFD 自动拆出 RTC 子设备。

## P1 PMIC RTC 驱动细节

`rtc-spacemit-p1.c` 当前实现比较克制，核心只有：

- `read_time`
- `set_time`
- `devm_rtc_register_device`

### 1. 只实现基础时间读写

代码里当前提供的 `rtc_class_ops` 为：

```c
static const struct rtc_class_ops p1_rtc_class_ops = {
	.read_time = p1_rtc_read_time,
	.set_time = p1_rtc_set_time,
};
```

没有看到：

- `read_alarm`
- `set_alarm`
- `alarm_irq_enable`

所以这一路当前更适合描述为：

- 支持基础 RTC 时间读写
- 不应文档化成“支持完整闹钟功能”

### 2. P1 RTC 有硬件一致性问题，驱动做了规避

这点挺关键，值得写进文档。

驱动注释明确说明：

- 硬件文档说读时间寄存器会锁存一致快照；
- 但实际硬件存在 bug；
- 所以驱动要循环读取，直到连续两次结果一致。

代码里通过：

- `RTC_READ_TRIES = 20`
- 连续 `regmap_bulk_read()`
- 比较秒寄存器是否一致

来保证读出的时间快照稳定。

### 3. 写时间时会先关闭 RTC，再写回，再重新使能

驱动里还明确做了：

- `regmap_clear_bits(... RTC_EN)`
- `regmap_bulk_write(...)`
- `regmap_set_bits(... RTC_EN)`

原因同样是为了规避硬件实现上“不保证写入时天然锁存一致”的问题。

### 4. 时间范围有限

P1 RTC 驱动里直接设置了：

```c
rtc->range_min = RTC_TIMESTAMP_BEGIN_2000;
rtc->range_max = RTC_TIMESTAMP_END_2063;
```

因此这一路 RTC 的可表示时间范围是：

- **2000 ~ 2063**

文档里最好把这个限制说清，不然用户会误以为是通用 RTC 无限覆盖更大时间范围。

## 接口描述

Linux RTC 驱动注册成功后，会生成标准字符设备：

```text
/dev/rtc0
/dev/rtc1
...
```

同时也会在：

```text
/sys/class/rtc/
```

下暴露 class 节点。

### 常用节点

例如：

- `/sys/class/rtc/rtc0/name`
- `/sys/class/rtc/rtc0/date`
- `/sys/class/rtc/rtc0/time`
- `/sys/class/rtc/rtc0/since_epoch`
- `/sys/class/rtc/rtc0/wakealarm`

### 常用用户态命令

#### 1. 查看系统中有哪些 RTC

```shell
ls /sys/class/rtc/
```

#### 2. 查看 RTC 名称

```shell
cat /sys/class/rtc/rtc0/name
```

这一步很重要，因为它能帮助你判断当前 `/dev/rtc0` 到底对应：

- SoC RTC
- RPMI RTC
- 还是 PMIC RTC

#### 3. 读取 RTC 时间

```shell
hwclock -r
```

或者：

```shell
cat /sys/class/rtc/rtc0/date
cat /sys/class/rtc/rtc0/time
```

#### 4. 设置 RTC 时间

```shell
hwclock --set --date "2026-03-18 16:30:00"
hwclock -w
```

#### 5. 将 RTC 时间同步到系统时间

```shell
hwclock -s
```

## 测试介绍

### 1. 先确认系统枚举了哪个 RTC

```shell
ls /sys/class/rtc/
cat /sys/class/rtc/rtc0/name
```

如果系统里不止一个 RTC，要先分清你测的是哪一个。

### 2. 做基础读写验证

```shell
hwclock -r
hwclock --set --date "2026-03-18 16:30:00"
hwclock -r
```

### 3. 验证掉电保持能力

如果板子具备：

- RTC backup 电源
- 或 PMIC RTC 维持能力

则可以：

1. 设置 RTC 时间；
2. 正常断主电；
3. 保留 backup 电源；
4. 重新上电后读取 RTC；
5. 确认时间是否连续增长。

这一步对 PMIC RTC 尤其重要。

### 4. 闹钟功能测试要先确认是哪一路 RTC

不是所有 K3 RTC 路径都等价。

- SoC 内部 RTC 从 DTS 看有 `rtc alarm` 中断，适合进一步验证 alarm；
- P1 PMIC RTC 当前驱动代码没有实现 alarm ops，不应直接按“支持 alarm”来测；
- RPMI RTC 是否支持完整 alarm，要结合实际驱动行为和系统暴露节点确认。

所以别把 K1 那种“RTC 一定带完整 alarm”默认套进来。

## Debug 介绍

### 1. RTC 节点有了，但 `/dev/rtc0` 不对

先看：

```shell
cat /sys/class/rtc/rtc0/name
```

很多 RTC 调试第一步不是“驱动有没有起来”，而是“系统把哪一路排成 rtc0 了”。

### 2. SoC RTC 起不来，优先检查 reset / status

SoC 内部 RTC 当前 DTS 节点有：

- `resets = <&syscon_apbc RESET_APBC_RTC>`
- `status = "disabled"`

所以如果这一路没起来，先查：

- board DTS 是否真的 `status = "okay"`
- APBC reset 是否正确释放
- 中断号是否匹配

### 3. P1 PMIC RTC 起不来，先查 MFD 链路

因为 P1 RTC 不是直接枚举，而是从：

- `simple-mfd-i2c`
- `compatible = "spacemit,p1"`

这条链路里拆出来的。

所以调试顺序应该是：

1. I2C 控制器是否正常；
2. `p1@41` 是否探测成功；
3. MFD 是否创建了 `spacemit-p1-rtc` 子设备；
4. 之后再看 RTC 驱动本身是否注册成功。

### 4. P1 RTC 读时间偶发异常，要记得硬件有一致性问题

这不是纯软件层面的小概率 bug。当前驱动已经明确为硬件一致性问题做了规避。

如果后续仍看到异常：

- 优先怀疑底层 PMIC 通信/寄存器访问稳定性；
- 再看是否有并发访问或上电时序问题；
- 不要先入为主地把问题归因到 `hwclock` 工具本身。

## FAQ

### 1. K3 到底该写内部 RTC，还是 PMIC RTC？

这取决于你的板级方案，不是 SoC 单方面决定。

- 如果板级主要依赖 SoC 内部 RTC，就看 `rtc@d4010000`；
- 如果方案依赖 PMIC 保持时间，就要看 P1 PMIC RTC；
- 如果系统主时间来源走固件/管理单元，也可能最终用的是 RPMI RTC。

### 2. 为什么 `/dev/rtc0` 不一定是我想要的那一路？

因为系统里可能同时存在多路 RTC，枚举顺序会影响 `rtc0/rtc1` 编号。最稳的办法始终是看：

```shell
cat /sys/class/rtc/rtc0/name
```

### 3. P1 PMIC RTC 支持闹钟吗？

就当前 `drivers/rtc/rtc-spacemit-p1.c` 来看，**没有看到 alarm 相关 rtc ops**，所以文档里不应把它写成“支持完整闹钟”。

### 4. 文档里最该记住哪条经验？

**先分清当前系统实际使用的是哪一路 RTC，再谈功能和调试。**

这件事在 K3 上尤其重要，因为这里不是单一 RTC 方案，而是 SoC / RPMI / PMIC 多条路径并存。
