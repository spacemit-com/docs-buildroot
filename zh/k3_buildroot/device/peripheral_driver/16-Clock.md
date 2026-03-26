# Clock

介绍 K3 平台时钟系统的组成、驱动实现、设备树配置方式以及常见调试方法。

## 模块介绍

Clock 子系统负责给 SoC 内部各个模块提供工作时钟，并根据模块类型完成以下几类工作：

- 从固定时钟源或 PLL 派生出不同频率；
- 为外设选择时钟父源；
- 通过 divider 调整输出频率；
- 通过 gate 控制时钟开关；
- 与 reset 控制一起，为外设提供完整的“时钟 + 复位”资源。

对用户来说，K3 的时钟文档最重要的不是“时钟理论”，而是下面这些问题：

- K3 的 clock provider 在哪里；
- DTS 里的 `clocks = <&syscon_xxx ID>` 到底引用的是什么；
- 哪些模块一般要同时配 `clocks`、`clock-names`、`resets`；
- 遇到驱动 probe 失败、频率不对、模块没工作时，应该先查哪一层。

### 功能介绍

![](static/clock.png)

Linux Common Clock Framework（CCF）通常包括：

1. **Clock Provider**  
   SoC 平台侧提供时钟树、门控、分频、MUX、PLL 等实现；
2. **Clock Consumer**  
   各外设驱动通过 `clk_get()` / `devm_clk_get()` 获取并使能时钟；
3. **Device Tree 描述**  
   DTS 用 phandle + clock ID 说明“这个设备用哪个 provider 的哪个时钟”；
4. **调试接口**  
   用户通过 `clk_summary`、内核日志、consumer 节点等确认时钟是否真正启用。

K3 平台的时钟体系并不是单一一个 provider，而是按域拆成多组 system-controller / CCU：

- `syscon_mpmu`
- `syscon_apmu`
- `syscon_apbc`
- `syscon_apbc2`
- `syscon_dciu`
- `syscon_rcpu_sysctrl`
- `syscon_rcpu_uartctrl`
- `syscon_rcpu_i2sctrl`
- `syscon_rcpu_spictrl`
- `syscon_rcpu_i2cctrl`
- `syscon_rpmu`
- `syscon_rcpu_pwmctrl`
- `pll`

所以 K3 DTS 里常见的时钟引用形式是：

```dts
clocks = <&syscon_apbc CLK_APBC_SPI0>, <&syscon_apbc CLK_APBC_SPI0_BUS>;
```

这表示：

- provider 是 `syscon_apbc`
- 第一个 clock ID 是 `CLK_APBC_SPI0`
- 第二个 clock ID 是 `CLK_APBC_SPI0_BUS`

## 源码结构介绍

K3 时钟相关代码主要位于：

```text
linux-6.18/
|-- drivers/clk/spacemit/
|   |-- ccu-k1x.c
|   |-- ccu-k3.c                    # K3 时钟控制单元驱动
|   |-- clk-spacemit-k1x.c
|   |-- reset-spacemit-k1x.c
|   `-- ...
|-- include/dt-bindings/clock/
|   |-- spacemit,k1-syscon.h
|   `-- spacemit,k3-syscon.h        # K3 clock/reset ID 定义
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3-rdomain.dtsi
    `-- k3*.dts
```

其中本轮最关键的两个文件是：

- `drivers/clk/spacemit/ccu-k3.c`
- `include/dt-bindings/clock/spacemit,k3-syscon.h`

### `ccu-k3.c` 做了什么

从当前 SDK 来看，`ccu-k3.c` 里定义了大量 K3 时钟节点，类型包括：

- `CCU_GATE_DEFINE`
- `CCU_MUX_GATE_DEFINE`
- `CCU_MUX_DIV_GATE_FC_DEFINE`

这意味着 K3 时钟树里常见的是：

- gate clock
- mux + gate clock
- mux + divider + gate clock

实际在代码里可以直接看到这些典型时钟：

- `sdh_axi_aclk`
- `sdh0_clk`
- `sdh1_clk`
- `sdh2_clk`
- `usb2_bus_clk`
- `usb3_porta_bus_clk`
- `usb3_portb_bus_clk`
- `usb3_portc_bus_clk`
- `usb3_portd_bus_clk`
- `qspi_clk`
- `twsi0_clk ~ twsi8_clk`
- `twsi0_bus_clk ~ twsi8_bus_clk`
- `spi0_clk / spi1_clk / spi3_clk`
- `spi0_bus_clk / spi1_bus_clk / spi3_bus_clk`
- `rtc_clk / rtc_bus_clk`

这也解释了为什么很多设备节点会同时拿到两路时钟：

- 一路功能时钟（func/core）
- 一路总线时钟（bus/axi/apb）

## 关键特性

### 特性

| 特性 | 特性说明 |
| :--- | :--- |
| 基于 CCF | K3 时钟系统基于 Linux Common Clock Framework |
| 多 provider 拆分 | A-domain、R-domain、PLL、PMU 等由不同 syscon/provider 提供 |
| clock/reset 紧密配对 | 多数外设 DTS 同时配置 `clocks` 与 `resets` |
| 时钟 ID 集中定义 | K3 clock/reset ID 统一放在 `spacemit,k3-syscon.h` |
| 大量使用 gate / mux / divider | 通过 `ccu-k3.c` 组合实现模块时钟树 |
| DTS 消费者写法统一 | 绝大多数设备节点按 `clocks` + `clock-names` + `resets` 模式接入 |

### K3 常见 provider 节点

从 `k3.dtsi` 当前内容看，K3 主要的时钟/复位 provider 包括：

| 节点 | 地址 | 典型用途 |
| :--- | :--- | :--- |
| `syscon_mpmu` | `0xd4050000` | MPMU 相关基础时钟/低速时钟 |
| `syscon_apmu` | `0xd4282800` | APMU 域时钟，如 QSPI、SDH、USB、CPU 等 |
| `syscon_apbc` | `0xd4015000` | APB 外设时钟，如 UART/PWM/I2C/SPI/RTC |
| `syscon_apbc2` | `0xf0610000` | 第二组 APBC / 安全域部分时钟 |
| `syscon_dciu` | `0xd8440000` | DCIU 相关控制 |
| `syscon_rcpu_sysctrl` | `0xc0880000` | R-domain 系统控制相关时钟/复位 |
| `syscon_rcpu_uartctrl` | `0xc0881f00` | R-domain UART |
| `syscon_rcpu_i2sctrl` | `0xc0882000` | R-domain I2S |
| `syscon_rcpu_spictrl` | `0xc0885f00` | R-domain SPI |
| `syscon_rcpu_i2cctrl` | `0xc0886f00` | R-domain I2C |
| `syscon_rpmu` | `0xc088c000` | R-domain PMU |
| `syscon_rcpu_pwmctrl` | `0xc088d000` | R-domain PWM |

### K3 clock ID 的组织方式

`include/dt-bindings/clock/spacemit,k3-syscon.h` 里不只是一个小枚举，而是把多个域的 ID 都集中定义了，例如：

- PLL clocks：
  - `CLK_PLL1`
  - `CLK_PLL2`
  - `CLK_PLL3`
  - `CLK_PLL4`
  - `CLK_PLL1_D2`
  - `CLK_PLL2_D8`
  - `CLK_PLL6_80`
  - ...
- MPMU clocks：
  - `CLK_MPMU_APB`
  - `CLK_MPMU_SLOW_UART`
  - `CLK_MPMU_I2S_SYSCLK`
  - ...
- APBC clocks：
  - `CLK_APBC_UART0`
  - `CLK_APBC_UART0_BUS`
  - `CLK_APBC_PWM0`
  - `CLK_APBC_PWM0_BUS`
  - `CLK_APBC_TWSI0`
  - `CLK_APBC_TWSI0_BUS`
  - `CLK_APBC_SPI0`
  - `CLK_APBC_SPI0_BUS`
  - `CLK_APBC_RTC`
  - `CLK_APBC_RTC_BUS`
  - ...
- reset IDs：
  - `RESET_MPMU_WDT`
  - 以及 APBC / APMU / RCPU 各域 reset ID

这意味着对于用户来说，**DTS 里的 clock/reset ID 不是拍脑袋写的数字**，而应该明确来自这个 dt-binding 头文件。

## 配置介绍

主要包括 **clock provider** 和 **clock consumer** 两部分。

### 1. provider 侧

K3 的 provider 通常在 `k3.dtsi` 里已经定义好，板级 DTS 一般不需要重新创建 provider，只需要作为 consumer 去引用。

也就是说，大多数开发工作不是“新增一个 clock controller 节点”，而是：

- 找对 provider；
- 找对 ID；
- 填对 `clock-names`；
- 必要时再配 `assigned-clocks` / `assigned-clock-parents` / `assigned-clock-rates`。

### 2. consumer 侧通用写法

大多数 K3 外设节点写法都类似：

```dts
xxx: device@addr {
	clocks = <&provider CLK_ID0>, <&provider CLK_ID1>;
	clock-names = "func", "bus";
	resets = <&provider RESET_ID>;
	status = "disabled";
};
```

不同设备的 `clock-names` 会有区别，常见取值包括：

- `func`
- `bus`
- `core`
- `gate`
- `qspi_clk`
- `qspi_bus_clk`
- `dbi`
- `mstr`
- `slv`

因此写文档或配 DTS 时，**不要只看 clock 数量，也要看驱动里期望的 `clock-names`**。

## DTS 配置示例

### 1. SPI 控制器

```dts
spi0: spi@d4040000 {
	compatible = "spacemit,k3-spi";
	clocks = <&syscon_apbc CLK_APBC_SPI0>,
		 <&syscon_apbc CLK_APBC_SPI0_BUS>;
	clock-names = "func", "bus";
	resets = <&syscon_apbc RESET_APBC_SPI0>;
	status = "disabled";
};
```

说明：

- `func` 是 SPI 控制器工作时钟；
- `bus` 是 APB 总线访问时钟；
- reset 也由同一个 provider 提供。

### 2. QSPI 控制器

```dts
qspi: spi@d420c000 {
	compatible = "spacemit,k3-qspi";
	clocks = <&syscon_apmu CLK_APMU_QSPI>,
		 <&syscon_apmu CLK_APMU_QSPI_BUS>;
	clock-names = "qspi_clk", "qspi_bus_clk";
	resets = <&syscon_apmu RESET_APMU_QSPI>,
		 <&syscon_apmu RESET_APMU_QSPI_BUS>;
	reset-names = "qspi_reset", "qspi_bus_reset";
	status = "disabled";
};
```

说明：

- 这类高速模块通常挂在 `syscon_apmu`；
- 不仅要配 clocks，还要把 `reset-names` 对齐驱动预期。

### 3. R-domain UART

```dts
r_uart0: serial@c0881000 {
	compatible = "spacemit,k1-uart", "intel,xscale-uart";
	clocks = <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0>,
		 <&syscon_rcpu_uartctrl CLK_RCPU_UARTCTRL_RUART0_BUS>,
		 <&syscon_mpmu CLK_MPMU_SLOW_UART>;
	clock-names = "core", "bus", "gate";
	resets = <&syscon_rcpu_uartctrl RESET_RCPU_UARTCTRL_RUART0>;
	status = "disabled";
};
```

说明：

- R-domain 外设经常不是两路时钟，而是三路；
- 除了 `core` 和 `bus`，还可能多一个 `gate` / slow clock；
- 所以不能把 A-domain 外设的 clock 模式简单套给 R-domain。

### 4. I2C 控制器

```dts
i2c0: i2c@d4010800 {
	compatible = "spacemit,k1-i2c";
	clocks = <&syscon_apbc CLK_APBC_TWSI0>,
		 <&syscon_apbc CLK_APBC_TWSI0_BUS>;
	clock-names = "func", "bus";
	resets = <&syscon_apbc RESET_APBC_TWSI0>;
	status = "disabled";
};
```

这也是 K3 最典型的外设时钟写法之一。

## 使用与调试介绍

### 1. 优先确认 DTS 是否把时钟写对了

重点检查：

- provider 对不对；
- ID 对不对；
- `clock-names` 顺序对不对；
- reset 是否也同步写了；
- 驱动里实际 `devm_clk_get()` 请求的名字是什么。

很多 probe 失败并不是“驱动坏了”，而是：

- `clock-names` 和驱动期望不匹配；
- 少写一组 bus clock；
- reset 没有释放；
- provider/ID 用错域了。

### 2. 看内核日志里的 clock/reset 报错

常见失败信息包括：

- `failed to get clk`
- `failed to enable clk`
- `failed to deassert reset`
- `probe deferred`

如果出现 `-EPROBE_DEFER`，通常要反查：

- provider 是否已经先注册；
- 依赖的 clock/reset 控制器驱动是否正常起来。

### 3. 用 debugfs 看时钟树

如果内核打开了 CCF 调试接口，可以查看：

```shell
cat /sys/kernel/debug/clk/clk_summary
```

或者：

```shell
grep -i spi /sys/kernel/debug/clk/clk_summary
grep -i qspi /sys/kernel/debug/clk/clk_summary
grep -i twsi /sys/kernel/debug/clk/clk_summary
grep -i uart /sys/kernel/debug/clk/clk_summary
```

这对确认下面这些问题很有帮助：

- 时钟是否真的 enable；
- 父时钟是谁；
- enable count / prepare count 是否正常；
- 频率是不是你预期的值。

### 4. 从 consumer 视角定位问题最有效

调 K3 clock 时，最实用的方法通常不是先盯 provider 驱动，而是先选一个具体模块往回追：

例如调 SPI：

1. 看 SPI 节点的 `clocks` / `clock-names` / `resets`；
2. 看 SPI 驱动如何获取这些 clock；
3. 看 `clk_summary` 里 SPI 相关 clock 是否真正打开；
4. 再回头看 `ccu-k3.c` 里这个时钟属于哪种 gate/mux/div 结构。

这样比一上来就看整棵时钟树更快。

## FAQ

### 1. 为什么同一个设备要配两路甚至三路时钟？

因为设备往往不只有“功能时钟”一种，还可能需要：

- bus / APB / AXI 访问时钟；
- 低速 gate / slow clock；
- 某个参考时钟或 parent source。

DTS 必须按驱动预期完整提供，不能只给一路看起来“最像主时钟”的时钟。

### 2. 为什么写了 clocks 设备还是不工作？

常见原因：

- `clock-names` 不匹配；
- reset 没释放；
- pinctrl / power-domain / regulator 还有其他依赖没满足；
- 对应 provider 还没 probe 完成。

所以 K3 外设 bring-up 时，clock 只是关键一环，不是唯一一环。

### 3. 为什么有些模块挂 `syscon_apbc`，有些却挂 `syscon_apmu`？

因为它们处于不同时钟/电源/总线域。通常来说：

- 低速 APB 外设更多挂在 `syscon_apbc`；
- 高速或更复杂模块更多挂在 `syscon_apmu`；
- R-domain 模块则更多挂在 `syscon_rcpu_*` 系列 provider。

判断时不要凭名字猜，直接看 `k3.dtsi` 和对应驱动最稳。

### 4. 文档里最该记住哪条经验？

**先从 consumer 节点倒推 provider，而不是先试图背整棵时钟树。**

对用户来说，这种方式最省时间，也最符合日常 bring-up / 调试习惯。
