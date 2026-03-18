# WDT

介绍 K3 平台 Watchdog（WDT）的驱动实现、DTS 配置方式、用户态接口以及常见调试方法。

## 模块介绍

Watchdog（看门狗）用于在系统软件失去响应、长期不喂狗或重启流程异常时，强制触发复位，避免设备永久卡死。

对用户来说，WDT 文档最重要的不是“看门狗概念”，而是下面这些问题：

- K3 实际使用的是哪份 watchdog 驱动；
- DTS 里怎么配置时钟、reset、寄存器和私有属性；
- 默认是自动喂狗、手动喂狗，还是仅注册重启处理器；
- 怎么判断一次重启是不是 WDT 触发的；
- 什么时候会用它做 restart handler。

### 功能介绍

![](static/watchdog.png)

Linux watchdog 子系统通常包括：

1. **watchdog core**  
   提供 `/dev/watchdog`、标准 ioctl 和超时管理；
2. **platform watchdog driver**  
   负责具体硬件的启动、停止、喂狗、超时和复位控制；
3. **board DTS 配置**  
   描述寄存器、时钟、reset、中断和平台私有控制；
4. **重启联动**  
   通过 restart handler 将 watchdog 作为系统重启路径的一部分。

K3 当前 SDK 里的实际 watchdog 主线是：

- DTS compatible：`"spacemit-k1,wdt"`
- 驱动文件：`drivers/watchdog/spacemit-k1-wdt.c`
- Kconfig：`CONFIG_SPACEMIT_K1_WATCHDOG`

也就是说，和 I2C / DMA 一样，**平台虽然是 K3，但 watchdog 这条通用 IP 仍沿用了 K1 的命名体系**。

## 源码结构介绍

K3 WDT 相关代码主要位于：

```text
linux-6.18/
|-- drivers/watchdog/
|   |-- watchdog_dev.c
|   |-- watchdog_core.c
|   |-- spacemit-k1-wdt.c         # K3 当前实际使用的 WDT 驱动
|   |-- Kconfig
|   `-- Makefile
`-- arch/riscv/boot/dts/spacemit/
    `-- k3.dtsi
```

本轮最关键的几个对象是：

- `drivers/watchdog/spacemit-k1-wdt.c`
- `drivers/watchdog/Kconfig`
- `arch/riscv/boot/dts/spacemit/k3.dtsi`

### `spacemit-k1-wdt.c` 做了什么

这份驱动实现的核心能力包括：

- `start`
- `stop`
- `ping`
- `set_timeout`
- restart handler
- 通过 hrtimer 自动喂狗
- 启动时检查上次是否由 WDT 复位引起

源码中可以直接看到：

- `spa_wdt_start`
- `spa_wdt_stop`
- `spa_wdt_ping`
- `spa_wdt_set_timeout`
- `spa_wdt_restart_handler`
- `spa_wdt_feed`

这说明它不是一个最小实现，而是把：

- 正常 watchdog 运行
- 自动续命
- 系统重启托管
- 上次复位原因判断

都串起来了。

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| K3 当前复用 K1 WDT IP | DTS compatible 和驱动文件都沿用 K1 命名 |
| 支持标准 watchdog 核心接口 | `start` / `stop` / `ping` / `set_timeout` |
| 支持 restart handler | 可把 WDT 作为系统重启通路 |
| 支持自动喂狗 | 驱动内部用 hrtimer 周期性 `ping` |
| 可判断是否由 WDT 复位启动 | 通过状态寄存器读取复位原因 |
| 依赖 clock + reset | `clk` / `clk-bus` + `RESET_APBC_TIMERS0` |
| 含平台私有 DTS 属性 | `spa,wdt-disabled`、`spa,wdt-enable-restart-handler` |

### 默认时间参数

驱动里直接定义了几组关键时间值：

- `CONFIG_SPACEMIT_WATCHDOG_DEFAULT_TIME = 60`
- `SPACEMIT_WATCHDOG_MAX_TIMEOUT = 255`
- `SPACEMIT_WATCHDOG_EXPIRE_TIME = 100`
- `SPACEMIT_WATCHDOG_FEED_TIMEOUT = 30`

其中：

- watchdog 硬件计数频率按驱动注释是 `256Hz`
- `set_timeout()` 最终通过 `timeout << 8` 转成 tick
- `start()` 里默认会先把超时设置成 `100s`
- 自动喂狗周期是 `30s`

所以从当前实现看，这个驱动更像是“平台侧自己管理一只长超时 watchdog”，而不是只把 `/dev/watchdog` 暴露给用户后完全不管。

## 配置介绍

### CONFIG 配置

Kconfig 中的开关为：

```text
config SPACEMIT_K1_WATCHDOG
	tristate "Spacemit k1 Watchdog"
	depends on SOC_SPACEMIT
	select WATCHDOG_CORE
	help
	  Spacemit k1 SoC Watchdog timer. This will reboot your system when
	  the timeout is reached.
```

Makefile 对应：

```text
obj-$(CONFIG_SPACEMIT_K1_WATCHDOG) += spacemit-k1-wdt.o
```

也就是说，K3 当前 watchdog 这条主线真正要开的就是：

- `CONFIG_SPACEMIT_K1_WATCHDOG`

### DTS 配置

K3 当前 watchdog 节点位于 `k3.dtsi`：

```dts
watchdog: watchdog@d4014000 {
	compatible = "spacemit-k1,wdt";
	clocks = <&syscon_apbc CLK_APBC_TIMERS0>,
		 <&syscon_apbc CLK_APBC_TIMERS0_BUS>;
	clock-names = "clk", "clk-bus";
	resets = <&syscon_apbc RESET_APBC_TIMERS0>;
	reg = <0x0 0xd4014000 0x0 0xff>,
	      <0x0 0xd4050000 0x0 0x1024>;
	interrupts = <35 4>;
	interrupt-parent = <&saplic>;
	spa,wdt-disabled;
	spa,wdt-enable-restart-handler;
	status = "okay";
};
```

这个节点里最关键的属性包括：

| 属性 | 作用 |
| :--- | :--- |
| `compatible = "spacemit-k1,wdt"` | 绑定 SpacemiT watchdog 驱动 |
| `clocks` | 提供 watchdog 核心时钟和 bus 时钟 |
| `clock-names = "clk", "clk-bus"` | 名字必须与驱动 `devm_clk_get()` 一致 |
| `resets = <&syscon_apbc RESET_APBC_TIMERS0>` | 提供 reset |
| `reg` | 第一段是 WDT 本体寄存器，第二段是 MPMU 相关寄存器 |
| `spa,wdt-disabled` | 平台私有属性，控制默认行为 |
| `spa,wdt-enable-restart-handler` | 平台私有属性，允许注册 restart handler |
| `status = "okay"` | 启用节点 |

### 两段 `reg` 的含义

这点比较关键，K3 当前 watchdog 节点不是单段寄存器：

```dts
reg = <0x0 0xd4014000 0x0 0xff>,
      <0x0 0xd4050000 0x0 0x1024>;
```

结合驱动 probe 可以看出：

- 第 0 段资源映射到 `wdt_base`
- 第 1 段资源映射到 `mpmu_base`

后者用于处理：

- `MPMU_APRR`
- `MPMU_ARSR`
- reboot command / restart path

所以这不是“多写了一段无关地址”，而是驱动真的会同时访问 watchdog block 和 MPMU block。

## 平台私有行为说明

### 1. `spa,wdt-disabled`

驱动里通过：

```c
if (of_get_property(np, "spa,wdt-disabled", NULL))
	info->ctrl = 0;
else
	info->ctrl = 1;
```

来决定默认控制策略。

也就是说：

- 有 `spa,wdt-disabled` 时，`info->ctrl = 0`
- 没有这个属性时，`info->ctrl = 1`

从后续逻辑看，`ctrl` 会影响：

- 自动喂狗 hrtimer 是否持续运行
- suspend/shutdown 时 watchdog 的处理方式

### 2. `spa,wdt-enable-restart-handler`

驱动里通过：

```c
if (of_get_property(np, "spa,wdt-enable-restart-handler", NULL))
	info->enable_restart_handler = 1;
```

控制是否注册：

```c
register_restart_handler(&info->restart_handler)
```

启用后，系统执行 restart 时会走：

- `spa_wdt_restart_handler()`
- 设置 reboot reason
- 打开 WDT
- 将 `WDT_WMR` 设成极小值
- 触发硬件复位

所以这个属性不是“可有可无的功能开关”，而是决定 K3 是否把 watchdog 纳入系统重启通路。

## 驱动实现细节

### 1. WDT 计数与超时关系

驱动里注释说明：

- watchdog timer 是 16 bit
- 频率是 `256Hz`

`spa_wdt_set_timeout()` 的核心逻辑是：

```c
tick = timeout << 8;
```

如果 tick 超过 16 bit 可表示范围，就回退到最大超时：

- `255s`

### 2. 启动时默认先设置 100 秒超时

`spa_wdt_start()` 里不是直接沿用外部传入 timeout，而是先调用：

```c
spa_wdt_set_timeout(&info->wdt_dev, SPACEMIT_WATCHDOG_EXPIRE_TIME);
```

也就是：

- 默认先设为 `100s`

然后：

```c
spa_wdt_write(info, WDT_WMER, 0x3);
```

使能 counter 和 reset/interrupt。

### 3. 驱动内部会自动喂狗

驱动用 hrtimer 周期执行：

```c
spa_wdt_ping(&info->wdt_dev);
```

喂狗周期为：

- `30s`

这说明如果平台默认策略启用，watchdog 并不是“开了就等用户空间来喂”，而是内核驱动自己会续命。

### 4. 启动时会判断是否因 WDT 复位而重启

probe 里会读：

```c
is_wdt_reset = spa_wdt_read(info, WDT_WSR);
```

并打印：

- `System boots up because of SoC watchdog reset.`
- 或 `System boots up not because of SoC watchdog reset.`

所以用户排查异常重启时，第一手信息其实就来自这个启动日志。

### 5. restart handler 会保存 reboot 命令信息

驱动里还维护了：

- `reboot_cmd_mem`
- `reboot_cmd_size`
- `MPMU_ARSR_REBOOT_CMD()`

并在 restart handler 中把 reboot reason / cmd 写入保留区和 MPMU 状态位。

这意味着 K3 的 watchdog 重启链路不只是“暴力复位”，还试图给下次启动留下原因信息。

## 接口介绍

### 标准 watchdog 接口

启用成功后，系统通常会暴露：

```text
/dev/watchdog
/dev/watchdog0
```

用户空间常见操作包括：

- 打开 watchdog 设备
- 发送 keepalive
- 设置 timeout
- 关闭设备（若 nowayout 允许）

### 驱动私有 sysfs 节点

驱动里还创建了：

- `dev_attr_wdt_ctrl`

如果打开测试配置，还会额外创建：

- `dev_attr_wdt_debug`

这说明除了标准 `/dev/watchdog*`，平台还留了一条驱动私有调试入口。

## 使用与测试介绍

### 1. 先确认驱动是否起来

优先看启动日志里是否出现 watchdog 相关 probe 信息，以及是否打印：

- 是否由 SoC watchdog reset 启动

### 2. 查看 watchdog 设备节点

```shell
ls /dev/watchdog*
```

### 3. 查看 watchdog class 信息

通常还可以看：

```shell
ls /sys/class/watchdog/
```

例如：

```shell
cat /sys/class/watchdog/watchdog0/status
cat /sys/class/watchdog/watchdog0/timeout
cat /sys/class/watchdog/watchdog0/timeleft
```

### 4. 基础喂狗测试

如果用户空间直接测试标准 watchdog 接口，可以使用常见工具或程序周期性 keepalive，验证：

- timeout 能否设置；
- keepalive 后 `timeleft` 是否恢复；
- 停止喂狗后系统是否按预期复位。

### 5. restart 路径测试

如果 DTS 带了：

```dts
spa,wdt-enable-restart-handler;
```

可以重点测试：

- `reboot`
- `reboot <reason>`

观察系统是否通过 watchdog 路径完成复位，并在下次启动时留下对应痕迹。

## Debug 介绍

### 1. 驱动没起来，先查三件套

优先检查：

- `CONFIG_SPACEMIT_K1_WATCHDOG`
- DTS `compatible = "spacemit-k1,wdt"`
- `clock-names = "clk", "clk-bus"`

因为驱动 probe 时明确会去拿：

- `devm_clk_get(..., "clk")`
- `devm_clk_get(..., "clk-bus")`

名字不匹配就起不来。

### 2. 别漏看第二段 `reg`

如果只给了 watchdog 本体寄存器，没给 MPMU 那段：

```dts
<0x0 0xd4050000 0x0 0x1024>
```

probe 也会失败，因为驱动确实会映射第二段资源。

### 3. 明明节点是 `okay`，为什么 watchdog 没真跑起来？

要看 DTS 是否带了：

```dts
spa,wdt-disabled;
```

这个属性会改变驱动默认控制逻辑，不能简单理解为“节点 okay 就一定持续运行”。

### 4. 为什么重启没有走 watchdog 路径？

先看：

```dts
spa,wdt-enable-restart-handler;
```

如果没开这个属性，驱动就不会注册 restart handler。

### 5. 异常重启怎么先判断是不是 WDT 导致？

最先看启动日志里 probe 打出来的：

- `System boots up because of SoC watchdog reset.`

这通常比先猜原因更直接。

## FAQ

### 1. K3 当前 watchdog 驱动到底是哪一个？

是：

```text
drivers/watchdog/spacemit-k1-wdt.c
```

虽然文件名是 K1，但 K3 当前 DTS 就是在用这条驱动链。

### 2. 为什么 K3 还在用 `spacemit-k1,wdt`？

因为当前 SDK 复用了这套通用 watchdog IP。和 I2C、DMA 一样，**K3 平台并不意味着所有 IP 都改成了 `k3-*` 命名**。

### 3. K3 watchdog 是不是必须依赖用户空间喂狗？

不一定。当前驱动内部本身就带 hrtimer 自动喂狗逻辑，不是纯用户空间 daemon 模式。

### 4. 文档里最该记住哪条经验？

**先看 DTS 里的两个私有属性：`spa,wdt-disabled` 和 `spa,wdt-enable-restart-handler`。**

K3 这条 watchdog 链路里，很多行为差异恰恰就是这两个属性决定的。
