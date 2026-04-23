# WDT

本文介绍 WDT（看门狗）的基本原理、配置方法以及调试方式。

## 模块介绍  

**WDT（watchdog，看门狗）控制器**是一种监控系统正常运行的电器元件。

在配置超时时间并启用看门狗后，系统需要在规定时间内定期"喂狗"。
如果在超时时间内未进行喂狗（例如系统死机或异常卡死），看门狗将触发复位信号，从而对 K3 芯片进行系统复位。

### 功能说明 

内核通过 **WDT 框架接口** 将看门狗驱动注册到 **WDT 框架**和**应用层**，并生成设备节点 `/dev/watchdog0`。  

用户层程序可通过对该设备节点执行 `open()` 和 `ioctl()` 操作，实现以下功能：

- 启用 / 关闭看门狗
- 设置超时时间
- 执行喂狗操作

### 源码结构介绍

WDT 控制器驱动代码位于内核目录 `drivers/watchdog` 下：

```
drivers/watchdog
|--watchdog_core.c        # 内核 WDT 框架接口代码
|--watchdog_dev.c         # 内核 WDT 框架注册字符设备到用户层代码
|--spacemit-k1-wdt.c      # K3 WDT 驱动  
```

### 硬件特性

K3 看门狗使用 24bit 硬件计时器：
- 计数器位宽：24bit
- 最大计数值：16777215 (0xFFFFFF)
- 时钟分频：256 (DEFAULT_SHIFT = 8)
- 最大超时时间：65535 秒（约 18.2 小时）

## 关键特性  

- 超时可产生复位信号
- 使用 24bit 计时器
- 最高可设置 **65535s**（约 18.2 小时）超时时间
- 支持作为系统重启处理器（restart handler）

## 配置介绍

WDT 的使用主要涉及两部分配置：

- **内核 CONFIG 配置**
- **设备树（DTS）配置**

### 内核 CONFIG 配置

#### 启用 Watchdog 框架

`CONFIG_WATCHDOG` 为内核平台 WDT 框架提供支持，用于启用内核的 WDT 框架。
在使用 K3 WDT 驱动时，该选项必须设置为 `y`。

```
Symbol: WATCHDOG [=y]
Device Drivers
      -> Watchdog Timer Support (WATCHDOG [=y])
```

#### 启用 SpacemiT K3 WDT 驱动

在启用 WDT 框架后，需要将 `CONFIG_SPACEMIT_K1_WATCHDOG` 设置为 `y`，以支持 K3 的 WDT 驱动。

```
Symbol: SPACEMIT_K1_WATCHDOG [=y]
      -> Spacemit K1/K3 SoC Watchdog (SPACEMIT_K1_WATCHDOG [=y])
```

### DTS 配置

#### DTSI 配置示例

在 `dtsi` 文件中定义 WDT 控制器的寄存器地址、时钟和复位资源。
通常情况下，该部分**无需修改**。

**主域 Watchdog（k3.dtsi）**

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

**属性说明：**

- `spa,wdt-disabled`：禁用看门狗自动启动。若未配置此属性，WDT 驱动在加载时将自动启用看门狗，并使用 `hrtimer` 定时进行喂狗
- `spa,wdt-enable-restart-handler`：启用看门狗作为系统重启处理器

#### DTS 配置示例

DTS 完整配置，如下所示：

```dts
&watchdog {
    status = "okay";
};
```

如需在驱动加载时自动启动看门狗，删除 `spa,wdt-disabled` 属性：

```dts
&watchdog {
    /delete-property/ spa,wdt-disabled;
    status = "okay";
};
```

## 接口说明

### API 介绍

Linux 内核主要实现了将 watchdog 注册为字符设备，将设备文件节点提供给应用层使用。
常用的 `file_operations` 接口包括：

- 此接口实现了支持用户层打开 watchdog 节点

   ```c
   static int watchdog_open(struct inode *inode, struct file *file)
   ```

- 该接口通过不同的 `cmd` 实现以下功能：
  - 启用 / 关闭看门狗
  - 设置 / 获取超时时间
  - 执行喂狗操作

   ```c
   static long watchdog_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
   ```

### 用户空间接口

用户空间通过 `/dev/watchdog0` 设备节点与看门狗交互。

**基本操作：**

```c
#include <linux/watchdog.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

int fd;
int flags;
int timeout;

// 打开 WDT 节点
fd = open("/dev/watchdog0", O_WRONLY);

// 使能看门狗设备
flags = WDIOS_ENABLECARD;
ioctl(fd, WDIOC_SETOPTIONS, &flags);

// 设置看门狗超时时间（秒）
timeout = 30;
ioctl(fd, WDIOC_SETTIMEOUT, &timeout);

// 获取看门狗当前超时时间
ioctl(fd, WDIOC_GETTIMEOUT, &timeout);

// 喂狗操作
ioctl(fd, WDIOC_KEEPALIVE, 0);

// 关闭看门狗设备
flags = WDIOS_DISABLECARD;
ioctl(fd, WDIOC_SETOPTIONS, &flags);

close(fd);
```

## Debug 说明

由于内核 watchdog 框架将看门狗驱动注册成字符设备提供给应用层，测试需用户自行实现一个 watchdog 应用。

### 测试程序示例

基于 **Buildroot 系统** 验证：

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/watchdog.h>

int main(int argc, char *argv[])
{
    int fd;
    int flags;
    int timeout = 10;  // 10 秒超时
    int ret;

    // 打开看门狗设备
    fd = open("/dev/watchdog0", O_WRONLY);
    if (fd < 0) {
        perror("open /dev/watchdog0");
        return -1;
    }

    // 设置超时时间
    ret = ioctl(fd, WDIOC_SETTIMEOUT, &timeout);
    if (ret) {
        perror("WDIOC_SETTIMEOUT");
        close(fd);
        return -1;
    }

    printf("Watchdog timeout set to %d seconds\n", timeout);

    // 启用看门狗
    flags = WDIOS_ENABLECARD;
    ret = ioctl(fd, WDIOC_SETOPTIONS, &flags);
    if (ret) {
        perror("WDIOC_SETOPTIONS enable");
        close(fd);
        return -1;
    }

    printf("Watchdog enabled, feeding every 5 seconds...\n");

    // 定期喂狗
    while (1) {
        sleep(5);
        ret = ioctl(fd, WDIOC_KEEPALIVE, 0);
        if (ret) {
            perror("WDIOC_KEEPALIVE");
            break;
        }
        printf("Watchdog fed\n");
    }

    close(fd);
    return 0;
}
```

### 命令行测试

也可以使用简单的命令行方式测试：

```bash
# 启动看门狗（打开设备节点即自动启动）
cat /dev/watchdog0 &

# 查看看门狗状态
cat /sys/class/watchdog/watchdog0/status
cat /sys/class/watchdog/watchdog0/timeout

# 停止喂狗测试（系统将在超时后复位）
killall cat
```

## 测试说明

可通过以下方式验证 WDT 功能：

### 正常喂狗测试

1. 编译并运行上述测试程序
2. 观察程序定期输出 "Watchdog fed"
3. 系统持续运行，不发生复位

### 停止喂狗测试

1. 运行测试程序
2. 强制终止程序（Ctrl+C 或 kill）
3. 等待超时时间后，系统应触发复位
4. 系统重启后可通过 `dmesg` 查看重启原因

### 超时时间测试

```bash
# 查看当前超时时间
cat /sys/class/watchdog/watchdog0/timeout

# 查看支持的超时范围
cat /sys/class/watchdog/watchdog0/max_timeout
cat /sys/class/watchdog/watchdog0/min_timeout
```

## FAQ

### 看门狗无法启动

检查以下几点：

1. 确认内核配置已启用 `CONFIG_WATCHDOG` 和 `CONFIG_SPACEMIT_K1_WATCHDOG`
2. 确认 DTS 中 watchdog 节点 status 为 "okay"
3. 检查设备节点是否存在：`ls -l /dev/watchdog*`
4. 查看内核日志：`dmesg | grep -i watchdog`

### 看门狗自动启动问题

如果不希望看门狗在驱动加载时自动启动，确保 DTS 中配置了 `spa,wdt-disabled` 属性：

```dts
&watchdog {
    spa,wdt-disabled;
    status = "okay";
};
```

### 系统意外重启

如果系统频繁意外重启，可能是看门狗超时导致：

1. 检查是否有程序打开了 `/dev/watchdog0` 但未正常喂狗
2. 增加超时时间
3. 检查系统负载，确保喂狗程序能及时执行
4. 查看 `/var/log/messages` 或 `dmesg` 确认重启原因

### 如何禁用看门狗

**临时禁用（重启后恢复）：**

```bash
# 方法 1：通过 ioctl 禁用
echo V > /dev/watchdog0

# 方法 2：卸载驱动模块（如果编译为模块）
rmmod spacemit_k1_watchdog
```

**永久禁用：**

在 DTS 中设置 watchdog 节点 status 为 "disabled"：

```dts
&watchdog {
    status = "disabled";
};
```

### 看门狗作为重启处理器

K3 看门狗支持作为系统重启处理器。当配置了 `spa,wdt-enable-restart-handler` 属性后，执行 `reboot` 命令时会通过看门狗触发系统复位。

如需禁用此功能，删除该属性：

```dts
&watchdog {
    /delete-property/ spa,wdt-enable-restart-handler;
    status = "okay";
};
```
