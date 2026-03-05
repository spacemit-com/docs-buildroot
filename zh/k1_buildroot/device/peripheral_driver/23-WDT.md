# WDT

本文介绍 WDT（看门狗）的基本原理、配置方法以及调试方式。

## 模块介绍  

**WDT（watchdog，看门狗）控制器**是一种监控系统正常运行的电器元件。

在配置超时时间并启用看门狗后，系统需要在规定时间内定期“喂狗”。
如果在超时时间内未进行喂狗（例如系统死机或异常卡死），看门狗将触发复位信号，从而对 K1 芯片进行系统复位。

### 功能说明 

![wdt](static/WDT.png)

内核通过 **WDT 框架接口** 将看门狗驱动注册到 **WDT 框架**和**应用层**，并生成设备节点 `/dev/watchdog0`。  

用户层程序可通过对该设备节点执行 `open()` 和 `ioctl()` 操作，实现以下功能：

- 启用 / 关闭看门狗
- 设置超时时间
- 执行喂狗操作

### 源码结构介绍

WDT 控制器驱动代码位于内核目录 `drivers/watchdog` 下：

```bash
drivers/watchdog
|--watchdog_core.c        #内核 WDT 框架接口代码
|--watchdog_dev.c         #内核 WDT 框架注册字符设备到用户层代码
|--k1x_wdt.c              #K1 WDT 驱动  
```  

## 关键特性  

- 超时可产生复位信号
- 最高可设置 **255s** 超时时间

## 配置介绍

WDT 的使用主要涉及两部分配置：

- **内核 CONFIG 配置**
- **设备树（DTS）配置**

### 内核 CONFIG 配置

#### 启用 Watchdog 框架

`CONFIG_WATCHDOG` 为内核平台 WDT 框架提供支持, 用于启用 内核的 WDT 框架。
在使用 K1 WDT 驱动时，该选项必须设置为 `y`。

```css
Symbol: WATCHDOG [=y]
Device Drivers
      -> Watchdog Timer Support (WATCHDOG [=y])
```

#### 启用 SpacemiT K1 WDT 驱动

在启用 WDT 框架后，需要将 `CONFIG_SPACEMIT_WATCHDOG` 设置为 `y`，以支持 K1 的 WDT 驱动。

```css
Symbol: SPACEMIT_WATCHDOG [=y]
      -> Spacemit-k1x SoC Watchdog (SPACEMIT_WATCHDOG [=y])
```

### DTS 配置

#### DTSI 配置示例

在 `dtsi` 文件中定义 WDT 控制器的寄存器地址、时钟和复位资源。
通常情况下，该部分**无需修改**。

> 注意：
> 若未配置 `spa,wdt-disabled` 属性，WDT 驱动在加载时将自动启用看门狗，并使用 `hrtimer` 定时进行喂狗。

```dts
watchdog: watchdog@d4080000 {
    compatible = "spacemit,soc-wdt";
    clocks = <&ccu CLK_WDT>;
    resets = <&reset RESET_WDT>;
    reg = <0x0 0xd4080000 0x0 0xff>,
    <0x0 0xd4050000 0x0 0x1024>;
    interrupts = <35>;
    interrupt-parent = <&intc>;
    spa,wdt-disabled;
    status = "disabled";
};
```

#### DTS 配置示例

DTS 完整配置，如下所示

```dts
 &watchdog {
     status = "okay";
 };
```

## 接口说明

### API 介绍

Linux 内核主要实现了将 watchdog 注册为字符设备，将设备文件节点提供给应用层使用。
常用的 `file_operations` 中接包括：

- 此接口实现了支持用户层打开 watchdog 节点

   ```
   static int watchdog_open(struct inode *inode, struct file *file)
   ```

- 该接口通过不同的 `cmd` 实现以下功能：
  - 启用 / 关闭看门狗
  - 设置 / 获取超时时间
  - 执行喂狗操作

   ```
   static long watchdog_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
   ```

## Debug 说明

由于内核 watchdog 框架将看门狗驱动注册成字符设备提供给应用层，测试要用户自行实现一个 watchdog 应用，示例如下：

基于 **Buildroot 系统** 验证

```sh
fd = open("/dev/watchdog", O_WRONLY);
#打开 WD T节点

flags = WDIOS_ENABLECARD;
ret = ioctl(fd, WDIOC_SETOPTIONS, &flags);
#使能看门狗设备

flags = WDIOS_DISABLECARD;
ret = ioctl(fd, WDIOC_SETOPTIONS, &flags);
#关闭看门狗设备

ret = ioctl(fd, WDIOC_GETTIMEOUT, &flags);
#获取看门狗当前超时时间

flag=超市时间
ret = ioctl(fd, WDIOC_SETTIMEOUT, &flags);
#设置看门狗超时时间
```

## 测试说明

可通过以下方式验证 WDT 功能：
- 正常喂狗：系统持续运行，不发生复位
- 停止喂狗：超过超时时间后，系统触发复位

## FAQ

（待补充）