# UFS

介绍 SpacemiT K3 平台 UFS 主机控制器驱动的结构、配置方法和调试要点。本文以 `linux-6.18/drivers/ufs/host/ufs-spacemit.c` 为主线，面向 UFS 驱动开发、板级适配和问题定位。

## 模块介绍

UFS（Universal Flash Storage）是面向高速嵌入式存储的串行接口。K3 当前实现采用 Linux 通用 UFS Core（`ufshcd`）+ 平台层（`ufshcd-pltfrm`）+ SpacemiT K3 variant 驱动（`ufs-spacemit`）的分层方式。UFS 设备在系统中最终通过 SCSI 子系统向上暴露，常见为 `/dev/sdX` 及其分区节点。

### 功能介绍

K3 UFS 软件栈可以分成以下几层：

- **UFS Core**：负责 UTP 传输、UIC 命令、SCSI 透传、错误恢复、sysfs/debugfs 接口等通用逻辑。
- **Platform Glue**：负责解析通用 DT 属性、时钟列表、lane 数等。
- **K3 Variant Driver**：负责 K3 控制器私有寄存器、M-PHY 初始化、UniPro 参数调优、Hibern8 进出序列和 K3 特有 quirk。
- **SCSI / Block 层**：负责把 UFS 设备导出为块设备和分区，供文件系统或原始块读写使用。

### 源码结构介绍

K3 UFS 相关代码和描述文件主要位于以下位置：

```text
linux-6.18/
|-- drivers/ufs/Kconfig
|-- drivers/ufs/core/
|   |-- ufshcd.c
|   |-- ufs-sysfs.c
|   |-- ufs-debugfs.c
|-- drivers/ufs/host/
|   |-- ufshcd-pltfrm.c
|   |-- ufs-spacemit.c
|   `-- ufs-spacemit.h
|-- arch/riscv/configs/k3_defconfig
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3_deb1.dts
    |-- k3_evb.dts
    `-- ...
```

其中：

- `drivers/ufs/host/ufs-spacemit.c` 是 K3 UFS variant 驱动主体。
- `drivers/ufs/host/ufs-spacemit.h` 定义了 K3 私有寄存器、链路能力和 host 私有数据结构。
- `drivers/ufs/host/ufshcd-pltfrm.c` 负责解析 `freq-table-hz`、`lanes-per-direction` 等通用属性。
- `drivers/ufs/core/ufshcd.c` 是 UFS Core，负责 probe、SCSI 枚举、功耗管理、错误恢复等。
- `drivers/ufs/core/ufs-sysfs.c`、`ufs-debugfs.c` 分别提供 sysfs 和 debugfs 调试接口。
- `arch/riscv/boot/dts/spacemit/k3.dtsi` 定义了 K3 UFS 控制器基础节点。
- 各板级 dts（如 `k3_deb1.dts`、`k3_evb.dts`）通常只负责按需打开 `&ufshc` 节点。

## 关键特性

| 项目 | 当前实现 |
| :----- | :----- |
| 驱动模型 | `ufshcd` Core + `ufshcd-pltfrm` + `ufs-spacemit` |
| 设备树 compatible | `spacemit,k3-ufshcd` |
| Lane 能力 | 每个方向最多 2 lane |
| HS 能力上限 | HS Gear 3 |
| PWM 能力上限 | PWM Gear 4 |
| HS Rate | `PA_HS_MODE_B` |
| UFS ACLK | 支持配置为 `409600000`、`491520000`、`500000000`、`600000000` Hz；参考 dts 当前配置为 `491520000` Hz |
| Ref Clock | 参考 dts 当前配置为 `19200000` Hz |
| Hibern8 策略 | 关闭 Auto-Hibern8；仅在 UFS Core 显式发起 Hibern8 进出时使用 variant 自定义时序，默认 PM policy 不走 Hibern8 |
| 默认 PM Level | `rpm_lvl = UFS_PM_LVL_2`，`spm_lvl = UFS_PM_LVL_2` |
| Standby 特殊处理 | `standby` 场景下临时提升到 `UFS_PM_LVL_5` |
| 错误观测能力 | 支持 vendor register dump、PA/DL/ABORT 事件日志 |
| 当前并发限制 | 每个 LU 的 SCSI queue depth 当前固定为 `1` |

## 配置介绍

主要包括驱动 **Kconfig 使能配置** 和 **DTS 配置**。

### Kconfig 配置

K3 UFS 基础配置至少需要：

- `CONFIG_SCSI_UFSHCD`
- `CONFIG_SCSI_UFSHCD_PLATFORM`
- `CONFIG_SCSI_UFS_SPACEMIT_K3`

`arch/riscv/configs/k3_defconfig` 当前配置如下：

```text
CONFIG_SCSI_UFSHCD=y
CONFIG_SCSI_UFSHCD_PLATFORM=y
CONFIG_SCSI_UFS_SPACEMIT_K3=y
```

相关 Kconfig 路径：

```text
Device Drivers
        SCSI device support
        Universal Flash Storage Controller (SCSI_UFSHCD)
                Platform bus based UFS Controller support (SCSI_UFSHCD_PLATFORM)
                Spacemit K3 specific hooks to UFS controller platform driver (SCSI_UFS_SPACEMIT_K3)
```

开发调试阶段常用的可选配置：

- `CONFIG_DEBUG_FS`
  用于打开 `drivers/ufs/core/ufs-debugfs.c` 提供的 debugfs 节点。
- `CONFIG_SCSI_UFS_BSG`
  如需通过 BSG 设备节点发送 UPIU / Query 请求，可打开该选项。

**注意：**

- 如果根文件系统位于 UFS 设备上，`CONFIG_SCSI_UFSHCD` 不建议编译成模块。

### DTS 配置

#### 1. SoC 基础节点

K3 UFS 控制器基础节点定义在 `arch/riscv/boot/dts/spacemit/k3.dtsi`：

```dts
ufshc: ufshc@0xc0e00000 {
	#address-cells = <2>;
	#size-cells = <2>;
	compatible = "spacemit,k3-ufshcd";
	reg = <0x0 0xc0e00000 0x0 0x40000>;
	interrupt-parent = <&saplic>;
	interrupts = <135 0x4>;
	clocks = <&syscon_apmu CLK_APMU_UFS_ACLK>;
	clock-names = "ufs-aclk";
	resets = <&syscon_apmu RESET_APMU_UFS_ACLK>;
	reset-names = "ufs-aclk-rst";
	clock-freq = <491520000>;
	freq-table-hz = <491520000 491520000>;
	lanes-per-direction = <2>;
	ref-clk-freq = <19200000>;
	status = "disabled";
};
```

其中几个关键属性的含义如下：

- `clock-names = "ufs-aclk"`：K3 variant 通过该名字获取主时钟。
- `reset-names = "ufs-aclk-rst"`：probe 早期会先 assert/deassert 该 reset。
- `clock-freq`：K3 variant 私有属性，`ufs_spacemit_platform_init()` 会优先用它设置 `ufs-aclk` 频率。当前项目支持的 `ufs-aclk` 配置档位为 `409600000`、`491520000`、`500000000`、`600000000` Hz。
- `freq-table-hz`：UFS 平台层通用频率表属性。
- `lanes-per-direction`：平台层解析该属性并保存到 `hba->lanes_per_direction`，默认值为 2。
- `ref-clk-freq`：UFS Core 用于解析参考时钟频率枚举，当前配置为 19.2 MHz。

#### 2. 板级使能

K3 当前大多数板级 dts 都已经复用 SoC 基础节点，因此板级上通常只需要打开节点：

```dts
&ufshc {
	status = "okay";
};
```

仓库中已经出现该写法的板级文件包括：

- `k3_deb1.dts`
- `k3_evb.dts`
- `k3_evb2-1.dts`
- `k3_evb2-2.dts`
- `k3_com260.dts`
- `k3_com260_kit_v02.dts`
- `k3_gemini_c0.dts`
- `k3_gemini_c1.dts`

#### 3. 供电说明

当前 K3 平台的 UFS 供电由**能效核（ESOS）**处理，`ufshc` 节点**不应**配置以下 supply 属性：

- `vdd-hba-supply`
- `vcc-supply`
- `vccq-supply`
- `vccq2-supply`

关于供电控制的整体机制，详情请参考 [ESOS 电源管理](../../esos/esos_power.md) 章节。

需要区分两层含义：

- **通用 UFS Core 的能力**：`ufshcd-pltfrm.c` / `ufshcd.c` 确实支持解析 `*-supply` 属性，并通过 regulator 框架获取、使能和关闭这些电源。
- **当前 K3 平台的约束**：当前项目由能效核统一处理 UFS 供电，因此 `ufshc` 节点不应提供这些 supply phandle，也不应由 UFS 驱动接管供电控制。

#### 4. 新板卡开发建议

新板卡导入 UFS 时，建议按下面顺序检查：

1. 先继承 `k3.dtsi` 中已有的 `ufshc` 基础节点，只在板级 dts 中把 `status` 改成 `"okay"`。
2. 确认 `ufs-aclk` 的真实输入频率与 `clock-freq` 保持一致，否则 `UFS_SYS1CLK_1US` 和链路启动定时参数可能被错误编程。
   `clock-freq` 建议从 `409600000`、`491520000`、`500000000`、`600000000` Hz 中选择。
5. 当前 K3 平台不要给 `ufshc` 增补 `*-supply` 属性；UFS 供电仍应由能效核（ESOS）处理，详情参考 [ESOS 电源管理](../../esos/esos_power.md) 章节。

## 接口介绍

### Variant Ops 关键回调

K3 UFS variant 通过 `struct ufs_hba_variant_ops` 接入 UFS Core，当前主要回调如下：

| 回调 | 作用 | 开发关注点 |
| :----- | :----- | :----- |
| `init` | 初始化 host 私有数据、设置默认 PM level、注册 quirk | 当前会把 `rpm_lvl` / `spm_lvl` 都设为 `UFS_PM_LVL_2` |
| `link_startup_notify` | 链路启动前后回调 | PRE 阶段做 M-PHY 初始化和 UniPro 参数写入，POST 阶段做版本兼容处理 |
| `pwr_change_notify` | 功耗模式协商 | 当前上限固定为 2-lane、HS-G3、Rate-B |
| `setup_clocks` | 时钟打开后恢复 `ufs-aclk` 频率 | 当前仅在 `POST_CHANGE && on` 路径生效，依赖 `clock-freq` / `freq-table-hz` |
| `setup_xfer_req` | 请求下发前的 barrier | 在 ring doorbell 前补 `wmb()`，缓解高并发随机 IO 下的 command loss |
| `config_scsi_dev` | 配置每个 SCSI 设备 | 当前强制把 queue depth 限到 1，规避稳定性问题 |
| `device_reset` | 设备 reset 时序 | 首次初始化跳过，后续 reset 会清空 M-PHY 相关寄存器 |
| `hibern8_notify` | Hibern8 进出序列 | 当前仅处理 UFS Core 显式 Hibern8 enter/exit，不依赖 Auto-Hibern8，也不等同于所有 suspend/resume 路径 |
| `event_notify` | 错误事件观测 | 重点关注 PA / DL / ABORT 三类事件 |
| `apply_dev_quirks` | 针对 device quirk 调整链路参数 | 包含 LCC disable、`TX_MIN_ACTIVATETIME` 调整、WDC 特殊处理 |
| `hce_enable_notify` | Host Controller Enable 时序控制 | 首次和后续 HCE 处理路径不同 |
| `dbg_register_dump` | vendor register dump | 出错时用于辅助定位私有寄存器状态 |

### 关键时序说明

#### 1. Probe 与平台初始化

`probe()` 的关键路径如下：

1. 按 `compatible = "spacemit,k3-ufshcd"` 匹配设备树节点。
2. 进入 `ufs_spacemit_platform_init()`：
   - 获取 `ufs-aclk`
   - 获取 `ufs-aclk-rst`
   - 执行一次 reset assert/deassert
   - 根据 `clock-freq` 或 `freq-table-hz` 设置 `ufs-aclk` 频率
3. 调用 `ufshcd_pltfrm_init()` 进入通用 UFS Core 初始化流程。

如果 probe 早期就失败，优先检查：

- `ufs-aclk` 是否能获取
- `ufs-aclk-rst` 是否存在
- `clock-freq` 是否合理
- `status = "okay"` 是否在最终生效的板级 dts 中打开

#### 2. 链路启动前处理

`link_startup_notify(PRE_CHANGE)` 中，K3 variant 重点做了三件事：

- 执行 `ufs_spacemit_mphy_init()`，完成 M-PHY reset、power up、device reset deassert、PLL lock 等基础时序。
- 执行 `ufs_spacemit_uniprov1p6_init()`，写入一组 UniPro / M-PHY 调优寄存器。
- 按当前 `ufs-aclk` 频率编程 `UFS_SYS1CLK_1US`、`UFS_TX_SYMBOL_CLK_NS_US`、`UFS_PA_LINK_STARTUP_TIMER`。

这一步是 K3 UFS bring-up 的核心阶段，任何时钟配置错误、PLL 不锁定、M-PHY 时序异常，通常都会在这里暴露。

#### 3. 链路启动后处理

`link_startup_notify(POST_CHANGE)` 中，当前实现会：

- 读取本地 / 对端 UniPro 版本
- 在部分旧版本对端场景下发送 dummy frame 兼容处理
- 把 `DL_AFC0REQTIMEOUTVAL` 设置为最大值

如果出现 “链路偶尔能起、偶尔起不来” 的现象，需要重点回看这一阶段的兼容处理是否真正生效。

#### 4. 功耗模式协商

`pwr_change_notify(PRE_CHANGE)` 会调用 `ufshcd_negotiate_pwr_params()`，并把 K3 默认能力限定为：

- RX / TX lane：2
- HS Gear：G3
- PWM Gear：G4
- PWM power mode：`SLOW_MODE`
- HS power mode：`FAST_MODE`
- HS rate：`PA_HS_MODE_B`

`POST_CHANGE` 阶段则会缓存协商结果，并再次检查 M-PHY PLL lock。

#### 5. Hibern8 与系统休眠

K3 上需要把“链路 Hibern8”和“系统休眠”分开看。当前实现并不是所有 suspend / resume 都会走 Hibern8：

- `hibern8_notify()` 只会在 UFS Core 发起 `UIC_CMD_DME_HIBER_ENTER` / `UIC_CMD_DME_HIBER_EXIT` 时执行。
- K3 variant 显式设置了 `UFSHCD_QUIRK_BROKEN_AUTO_HIBERN8`。当前没有开启 HCI Auto-Hibern8，主要是出于稳定性考虑；`$UFS_DEV/auto_hibern8` 在当前平台上读写通常返回 `Operation not supported`，这是预期行为。
- 默认 `rpm_lvl = UFS_PM_LVL_2`、`spm_lvl = UFS_PM_LVL_2`，对应 `UFS_SLEEP_PWR_MODE + UIC_LINK_ACTIVE_STATE`。也就是说，默认 runtime suspend 和普通 system suspend 路径都不会主动让 link 进入 Hibern8。
- 只有当 PM level 被设置成 link target 为 `HIBERN8` 的档位时，UFS Core 才会触发 K3 的自定义 Hibern8 时序。可通过 `$UFS_DEV/rpm_target_link_state` 和 `$UFS_DEV/spm_target_link_state` 直接确认当前目标状态。
- K3 的 Hibern8 退出路径会重新拉起参考时钟、恢复 M-PHY 内部 power state，并等待 PLL lock；进入路径则会调整若干 RX/TX MIB，并在 POST 阶段下电 M-PHY。
- `standby` 走的是另一条路径。`ufs_spacemit_suspend_prepare()` 在 `pm_suspend_target_state == PM_SUSPEND_STANDBY` 时，会临时把 `spm_lvl` 提升到 `UFS_PM_LVL_5`，`resume_complete()` 再恢复原值。`UFS_PM_LVL_5` 对应 `UFS_POWERDOWN_PWR_MODE + UIC_LINK_OFF_STATE`，验证的是掉电 / link-off 恢复，不是 Hibern8 进出。

因此，若 `standby` 后 UFS 掉盘、重新枚举或卡死，优先排查的是 powerdown / link-off 恢复；若是显式使用 Hibern8 档位后失败，再重点回看 `hibern8_notify()` 中的 PLL lock 和 MIB 恢复时序。

#### 6. 当前实现限制

当前实现里有几个必须明确写进开发中的限制：

- `config_scsi_dev()` 会把每个 LU 的 queue depth 限制为 `1`，这是为了规避高并发随机 IO 下的稳定性问题。
- 当前没有开启 Auto-Hibern8，主要是为了规避 Hibern8 相关稳定性问题，因此不能把 `auto_hibern8` 当作可用的调优入口。
- 当前默认 PM policy 刻意绕开 Hibern8：`rpm_lvl` / `spm_lvl` 默认都为 `2`，而 `standby` 还会临时强制到 `5`。

## Debug介绍

### 内核日志

最直接的入口仍然是查看内核日志：

```bash
dmesg | grep -iE 'ufs|ufshcd|scsi'
```

建议重点关注以下关键信息：

- `M-PHY PLL lock timeout`
- `Runtime suspend failed`
- `Runtime resume failed`
- `queue depth limited to 1`
- PA / DL / ABORT 相关错误打印

如果 UFS Core 触发了 register dump，K3 variant 的 `dbg_register_dump` 还会额外把 vendor specific register、M-PHY 和 ATOP 相关寄存器打印到日志中。

### sysfs

UFS Core 会自动创建一组 sysfs 节点。可以先定位 UFS platform device：

```bash
UFS_DEV=/sys/bus/platform/devices/c0e00000.ufshc
echo "$UFS_DEV"
```

然后重点查看：

```bash
cat $UFS_DEV/rpm_lvl
cat $UFS_DEV/rpm_target_dev_state
cat $UFS_DEV/rpm_target_link_state

cat $UFS_DEV/spm_lvl
cat $UFS_DEV/spm_target_dev_state
cat $UFS_DEV/spm_target_link_state

cat $UFS_DEV/auto_hibern8 2>/dev/null || echo "auto_hibern8: Operation not supported (expected on K3)"

cat $UFS_DEV/power_info/lane
cat $UFS_DEV/power_info/mode
cat $UFS_DEV/power_info/rate
cat $UFS_DEV/power_info/gear
cat $UFS_DEV/power_info/dev_pm
cat $UFS_DEV/power_info/link_state
```

这些节点的含义如下：

| 节点 | 含义 |
| :----- | :----- |
| `$UFS_DEV/rpm_lvl` | Runtime PM level，表示运行时挂起使用的目标功耗等级。写入后会影响 runtime suspend 时希望切到的 device state 和 link state。 |
| `$UFS_DEV/rpm_target_dev_state` | 当前 `rpm_lvl` 对应的目标 device state。排查“为什么没进 Hibern8 / 为什么没掉电”时，先看这里，不要只看 level 数字。 |
| `$UFS_DEV/rpm_target_link_state` | 当前 `rpm_lvl` 对应的目标 link state。若这里不是 `HIBERN8`，runtime suspend 路径就不会进入 Hibern8。 |
| `$UFS_DEV/spm_lvl` | System PM level，表示系统休眠使用的目标功耗等级。写入后会影响 system suspend 时希望切到的 device state 和 link state。 |
| `$UFS_DEV/spm_target_dev_state` | 当前 `spm_lvl` 对应的目标 device state。适合区分普通 suspend、显式 Hibern8 suspend 和 `standby` 的 powerdown 路径。 |
| `$UFS_DEV/spm_target_link_state` | 当前 `spm_lvl` 对应的目标 link state。默认 `spm_lvl = 2` 时这里会显示 `ACTIVE`；若在 suspend 前改成 Hibern8 档位，这里会显示 `HIBERN8`。`standby` 内部临时切到 `OFF` 的动作只发生在 suspend prepare 阶段，恢复后该节点会回到原配置。 |
| `$UFS_DEV/auto_hibern8` | 通用 UFS Core Auto-Hibern8 空闲定时器接口。若平台支持，会显示链路空闲多久后自动进入 Hibern8；K3 当前预期返回 `Operation not supported`。 |
| `$UFS_DEV/power_info/lane` | 当前链路协商后的 lane 数，代码中实际显示的是 `lane_rx`。可用于确认最终是否跑在 1-lane 或 2-lane。 |
| `$UFS_DEV/power_info/mode` | 当前链路功耗模式，常见为 `FAST_MODE`、`SLOW_MODE`、`FASTAUTO_MODE`、`SLOWAUTO_MODE`。 |
| `$UFS_DEV/power_info/rate` | 当前 HS 速率档位，常见为 `HS_RATE_A` 或 `HS_RATE_B`。 |
| `$UFS_DEV/power_info/gear` | 当前实际 gear。若链路在 HS 模式，显示 `HS_GEARx`；若在 PWM 模式，显示 `PWM_GEARx`。 |
| `$UFS_DEV/power_info/dev_pm` | 当前 UFS device 电源状态，常见为 `ACTIVE`、`SLEEP`、`POWERDOWN`、`DEEPSLEEP`。 |
| `$UFS_DEV/power_info/link_state` | 当前 UIC 链路状态，常见为 `ACTIVE`、`HIBERN8`、`OFF`、`BROKEN`。 |

如果要观察描述符与属性，也可以查看：

```bash
ls $UFS_DEV/device_descriptor
ls $UFS_DEV/interconnect_descriptor
ls $UFS_DEV/geometry_descriptor
ls $UFS_DEV/health_descriptor
ls $UFS_DEV/flags
ls $UFS_DEV/attributes
```

这些目录本身就是一组内核节点的分组，含义如下：

| 目录 | 含义 |
| :----- | :----- |
| `$UFS_DEV/device_descriptor` | UFS Device Descriptor 镜像，包含设备类型、LUN 数量、厂商 ID、队列深度、FFU 超时等静态能力信息。 |
| `$UFS_DEV/interconnect_descriptor` | UFS Interconnect Descriptor 镜像，重点看 UniPro 版本和 M-PHY 版本。 |
| `$UFS_DEV/geometry_descriptor` | UFS Geometry Descriptor 镜像，重点看原始容量、块大小、分配单元、Write Booster 相关几何参数。 |
| `$UFS_DEV/health_descriptor` | UFS Health Descriptor 镜像，重点看寿命估算和 EOL 信息。 |
| `$UFS_DEV/flags` | UFS Query Flag 节点集合，适合查看 `device_init`、`wb_enable`、`busy_rtc` 等布尔状态。 |
| `$UFS_DEV/attributes` | UFS Query Attribute 节点集合，适合查看 `current_power_mode`、`reference_clock_frequency`、`exception_event_status` 等运行时属性。 |

如果要做基础性能观测，可使用 monitor 组：

```bash
echo 1 > $UFS_DEV/monitor/monitor_enable
cat $UFS_DEV/monitor/read_total_sectors
cat $UFS_DEV/monitor/write_total_sectors
cat $UFS_DEV/monitor/read_req_latency_avg
cat $UFS_DEV/monitor/write_req_latency_avg
```

这里各节点的含义如下：

| 节点 | 含义 |
| :----- | :----- |
| `$UFS_DEV/monitor/monitor_enable` | 监控开关。写 `1` 开始统计，写 `0` 关闭并清空当前统计。 |
| `$UFS_DEV/monitor/read_total_sectors` | 自开启监控以来累计读出的 sector 数。 |
| `$UFS_DEV/monitor/write_total_sectors` | 自开启监控以来累计写入的 sector 数。 |
| `$UFS_DEV/monitor/read_req_latency_avg` | 读请求平均时延，单位为微秒。 |
| `$UFS_DEV/monitor/write_req_latency_avg` | 写请求平均时延，单位为微秒。 |

`monitor` 目录下还存在 `read_total_busy`、`read_nr_requests`、`read_req_latency_max`、`read_req_latency_min`、`write_total_busy`、`write_nr_requests` 等节点，分别用于查看累计 busy 时间、请求数以及最大/最小时延。

**注意：**

- `auto_hibern8` 在 K3 上返回 `Operation not supported` 属于预期现象，不表示驱动 probe 失败。
- 想确认某个 PM level 是否真的会走 Hibern8，优先看 `rpm_target_link_state` / `spm_target_link_state`，不要只根据 `rpm_lvl` / `spm_lvl` 的数字推断。
- sysfs 中看到的 `clock_scaling`、`write_booster` 等能力是否为 1，取决于当前 capability 是否真的在 variant 中启用。

### debugfs

如果内核打开了 `CONFIG_DEBUG_FS`，UFS Core 会在 debugfs 下创建目录：

```bash
if grep -wq debugfs /proc/mounts; then
    :
elif [ -d /sys/kernel/debug ]; then
    mount -t debugfs debugfs /sys/kernel/debug
fi

if [ -d /sys/kernel/debug/ufshcd ]; then
    ls /sys/kernel/debug/ufshcd
else
    echo "ufshcd debugfs unavailable"
fi
```

如果 `mount` 提示 `Device or resource busy`，通常只是 debugfs 已经挂载，可直接继续。

进入具体控制器目录后，可查看：

```bash
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/stats
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/saved_err
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/saved_uic_err
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/exception_event_mask
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/exception_event_rate_limit_ms
```

这些节点的含义如下：

| 节点 | 含义 |
| :----- | :----- |
| `/sys/kernel/debug/ufshcd/<dev_name>/stats` | UFS 错误事件统计汇总。可查看 PA/DL/NL/TL/DME 错误次数、Auto-Hibern8 错误、Link Startup 失败、PM suspend/resume 失败、LU reset、host reset、abort 次数。 |
| `/sys/kernel/debug/ufshcd/<dev_name>/saved_err` | UFS Core 保存的 controller error 位图，错误恢复路径会参考该值。适合看最近一次或累计保留的 HCI/中断级错误。 |
| `/sys/kernel/debug/ufshcd/<dev_name>/saved_uic_err` | UFS Core 保存的 UIC error 位图，适合定位 PA/DL/NL/TL 相关链路层异常。 |
| `/sys/kernel/debug/ufshcd/<dev_name>/exception_event_mask` | 用户侧 exception event mask，控制哪些 exception event 需要临时屏蔽或重新放开。写入的是位掩码。 |
| `/sys/kernel/debug/ufshcd/<dev_name>/exception_event_rate_limit_ms` | exception event 的速率限制时间，单位毫秒。发生异常后，Core 会按这个时间窗口延后恢复 EE control，避免异常风暴。 |

其中 `saved_err` 和 `saved_uic_err` 都是“错误位图”，不是字符串化结论，使用时通常需要结合 `dmesg`、`stats` 和 UFS Core 错误处理代码一起看。

## 测试介绍

### 枚举与识别测试

完成节点使能后，先验证控制器是否正常枚举：

```bash
dmesg | grep -iE 'ufs|ufshcd|scsi'
cat /proc/scsi/scsi
cat /proc/partitions
ls /dev/sd* 2>/dev/null
```

如果系统已经识别到 UFS 盘，通常会出现：

- UFS Host Controller 初始化成功日志
- 对应的 SCSI disk 设备
- `/dev/sdX` 或 `/dev/sdXn` 节点

如果系统提供 `lsblk`，也可以额外执行 `lsblk -o NAME,SIZE,MODEL`。

### 基础读写测试

如果系统本身就是从 UFS 启动，建议直接按 rootfs 场景做文件系统层测试，不要在线对 `/dev/sda` 做原始块写入。

以当前系统为例，UFS 盘枚举为 `/dev/sda`，其中：

- `/dev/sda` 是整块 UFS 盘
- `/dev/sda1`、`/dev/sda2` 是前面的 256 MiB 分区
- `/dev/sda3` 是约 `118.8G` 的主数据分区，通常也是 rootfs 所在分区

可先确认当前根文件系统确实落在 UFS 上：

```bash
mount | grep ' / '
fdisk -l /dev/sda
df -h /
```

确认 `/` 挂载在 UFS 分区后，可直接在 rootfs 上做非破坏性测试。下面示例把测试文件放在 `/root`：

```bash
dd if=/dev/zero of=/root/ufs_write_test.bin bs=1M count=256 conv=fsync
md5sum /root/ufs_write_test.bin > /root/ufs_write_test.bin.md5
sync
md5sum -c /root/ufs_write_test.bin.md5
dd if=/root/ufs_write_test.bin of=/dev/null bs=1M
rm -f /root/ufs_write_test.bin /root/ufs_write_test.bin.md5
```

像 `/tmp` 这类目录在某些配置下可能是 tmpfs，不能代表真实的 UFS 路径；如果要验证 rootfs 所在的 UFS 分区，测试文件应放在 `/root`、`/var` 或其他确认位于 `/dev/sda3` 上的目录。

如果需要做随机 IO 压测，建议把 `iodepth` 先控制在 `1`，因为当前驱动会把 queue depth 限到 1：

```bash
if command -v fio >/dev/null; then
    fio --name=ufs_randrw \
        --filename=/root/fio.bin \
        --rw=randrw \
        --bs=128k \
        --iodepth=1 \
        --size=1G \
        --direct=1 \
        --runtime=60 \
        --time_based
    rm -f /root/fio.bin
else
    echo "fio is not installed in current rootfs"
fi
```

如果 rootfs 可用空间不足，可把 `count` 或 `--size` 适当调小。由于这是在线 rootfs 压测，执行前应先确认当前系统没有关键后台写入任务。

## FAQ

### 1. 为什么当前驱动并发性能看起来不高？

因为 `config_scsi_dev()` 会把每个 LU 的 queue depth 限制为 `1`。这是当前 K3 UFS 驱动为了规避高并发随机 IO 下 command loss 风险而采取的保守策略。在问题彻底定位前，不建议只为了性能把这个限制直接放开。

### 2. 为什么文档里没有单独写 pinctrl 配置？

因为当前 K3 UFS 的 pin 是固定的，没有像 UART、SPI 那样需要板级单独切换或调优的 pinctrl 组，所以文档里不单独展开。
