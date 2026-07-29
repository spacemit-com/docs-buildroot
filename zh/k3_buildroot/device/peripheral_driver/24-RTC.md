# RTC

本文介绍 RTC（实时时钟）的基本原理、配置方法以及调试方式。

## 模块介绍  

**RTC（Real-Time Clock，实时时钟）**是一种用于记录和维护系统时间的硬件模块。

RTC 可以在系统断电或休眠状态下继续工作（通常由备用电池供电），保持时间的连续性。系统启动后可以从 RTC 读取当前时间，确保系统时间的准确性。

### 功能说明 

K3 平台使用 **RPMI（RISC-V Platform Management Interface）** 协议实现 RTC 功能。RPMI RTC 通过 mailbox 机制与固件通信，实现时间的读取、设置和闹钟功能。

内核通过 **RTC 框架接口** 将 RTC 驱动注册到系统，并生成设备节点 `/dev/rtc0`。  

用户层程序可通过以下方式操作 RTC：

- 读取当前时间
- 设置系统时间
- 设置和管理闹钟
- 通过 sysfs 接口查看 RTC 状态

### 源码结构介绍

RTC 驱动代码位于内核目录 `drivers/rtc` 下：

```
drivers/rtc
|--rtc-core.c           # RTC 框架核心代码
|--rtc-dev.c            # RTC 字符设备接口
|--rtc-sysfs.c          # RTC sysfs 接口
|--rtc-rpmi.c           # K3 RPMI RTC 驱动  
```

### RPMI RTC 架构

RPMI RTC 基于 mailbox 通信机制：

```
用户空间
    ↓
RTC 框架
    ↓
RPMI RTC 驱动
    ↓
Mailbox 子系统
    ↓
MPXY Mailbox
    ↓
硬件 RTC
```

## 关键特性  

- 支持读取和设置系统时间
- 支持闹钟功能（Alarm）
- 支持中断通知
- 基于 RPMI 协议通信
- 通过 mailbox 与固件交互
- 支持系统休眠唤醒

## 配置介绍

RTC 的使用主要涉及两部分配置：

- **内核 CONFIG 配置**
- **设备树（DTS）配置**

### 内核 CONFIG 配置

#### 启用 RTC 框架

`CONFIG_RTC_CLASS` 为内核 RTC 框架提供支持，用于启用内核的 RTC 子系统。
在使用 K3 RTC 驱动时，该选项必须设置为 `y`。

```
Symbol: RTC_CLASS [=y]
Device Drivers
      -> Real Time Clock (RTC_CLASS [=y])
```

#### 启用 RPMI RTC 驱动

在启用 RTC 框架后，需要将 `CONFIG_RTC_DRV_RPMI` 设置为 `y`，以支持 K3 的 RPMI RTC 驱动。

```
Symbol: RTC_DRV_RPMI [=y]
Device Drivers
      -> Real Time Clock
            -> RISC-V RPMI RTC (RTC_DRV_RPMI [=y])
```

### DTS 配置

#### DTSI 配置示例

在 `dtsi` 文件中定义 RPMI RTC 的 mailbox 通道和中断资源。
通常情况下，该部分**无需修改**。

**RPMI RTC 节点（k3.dtsi）**

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

**属性说明：**

- `compatible`：兼容字符串，标识为 RPMI RTC 设备
- `mboxes`：mailbox 通道配置，用于与固件通信
  - `&mpxy_mbox`：mailbox 控制器
  - `0xe`：RTC 服务 ID
  - `0x0`：通道参数
- `interrupts`：中断号配置（用于闹钟中断）
- `interrupt-names`：中断名称

#### DTS 配置示例

通常 RPMI RTC 在 dtsi 中已经默认启用（`status = "okay"`），板级 DTS 无需额外配置。

如需禁用 RTC，可在板级 DTS 中设置：

```dts
&rpmi_rtc {
    status = "disabled";
};
```

## 接口说明

### 内核 API

RTC 驱动实现了标准的 `rtc_class_ops` 接口：

```c
static const struct rtc_class_ops rpmi_rtc_ops = {
    .read_time = rpmi_read_time,      // 读取时间
    .set_time = rpmi_set_time,        // 设置时间
    .read_alarm = rpmi_read_alarm,    // 读取闹钟
    .set_alarm = rpmi_set_alarm,      // 设置闹钟
    .alarm_irq_enable = rpmi_alarm_irq_enable,  // 使能闹钟中断
};
```

### RPMI 服务 ID

RPMI RTC 支持以下服务：

| 服务 ID | 服务名称 | 功能说明 |
|---------|---------|---------|
| 0x01 | ENABLE_NOTIFICATION | 使能通知 |
| 0x02 | SET_TIME | 设置时间 |
| 0x03 | GET_TIME | 获取时间 |
| 0x04 | SET_ALARM | 设置闹钟 |
| 0x05 | GET_ALARM | 获取闹钟 |
| 0x06 | ALARM_GET_EN | 获取闹钟使能状态 |
| 0x07 | ALARM_SET_EN | 设置闹钟使能 |
| 0x08 | QUERY_PENDING | 查询待处理事件 |
| 0x09 | CLR_PENDING | 清除待处理事件 |

### 用户空间接口

#### 命令行工具

使用 `hwclock` 命令操作 RTC：

```bash
# 读取 RTC 时间
hwclock -r

# 将系统时间写入 RTC
hwclock -w

# 将 RTC 时间同步到系统时间
hwclock -s

# 设置 RTC 时间
hwclock --set --date="2025-04-22 18:30:00"
```

使用 `date` 命令操作系统时间：

```bash
# 查看系统时间
date

# 设置系统时间
date -s "2025-04-22 18:30:00"

# 设置系统时间后同步到 RTC
date -s "2025-04-22 18:30:00" && hwclock -w
```

#### sysfs 接口

通过 sysfs 查看和操作 RTC：

```bash
# 查看 RTC 时间
cat /sys/class/rtc/rtc0/time
cat /sys/class/rtc/rtc0/date

# 查看 RTC 设备信息
cat /sys/class/rtc/rtc0/name

# 查看闹钟设置
cat /sys/class/rtc/rtc0/wakealarm

# 设置闹钟（10 秒后触发）
echo +10 > /sys/class/rtc/rtc0/wakealarm
```

#### 编程接口

使用 ioctl 操作 RTC：

```c
#include <linux/rtc.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>

int fd;
struct rtc_time rtc_tm;

// 打开 RTC 设备
fd = open("/dev/rtc0", O_RDONLY);

// 读取 RTC 时间
ioctl(fd, RTC_RD_TIME, &rtc_tm);

// 设置 RTC 时间
ioctl(fd, RTC_SET_TIME, &rtc_tm);

// 读取闹钟
struct rtc_wkalrm alarm;
ioctl(fd, RTC_WKALM_RD, &alarm);

// 设置闹钟
ioctl(fd, RTC_WKALM_SET, &alarm);

close(fd);
```

## Debug 说明

### 基本测试

1. 检查 RTC 设备是否存在

   ```bash
   ls -l /dev/rtc*
   ```

2. 查看 RTC 驱动加载情况

   ```bash
   dmesg | grep -i rtc
   ```

3. 查看 RTC 设备信息

   ```bash
   cat /sys/class/rtc/rtc0/name
   cat /sys/class/rtc/rtc0/date
   cat /sys/class/rtc/rtc0/time
   ```

### 时间读写测试

```bash
# 读取当前 RTC 时间
hwclock -r

# 设置系统时间
date -s "2025-04-22 18:30:00"

# 将系统时间写入 RTC
hwclock -w

# 验证 RTC 时间
hwclock -r
```

### 闹钟测试

```bash
# 设置 10 秒后触发闹钟
echo +10 > /sys/class/rtc/rtc0/wakealarm

# 查看闹钟设置
cat /sys/class/rtc/rtc0/wakealarm

# 等待闹钟触发，查看内核日志
dmesg | tail
```

### 测试程序示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/rtc.h>
#include <time.h>

int main(void)
{
    int fd, ret;
    struct rtc_time rtc_tm;

    // 打开 RTC 设备
    fd = open("/dev/rtc0", O_RDONLY);
    if (fd < 0) {
        perror("open /dev/rtc0");
        return -1;
    }

    // 读取 RTC 时间
    ret = ioctl(fd, RTC_RD_TIME, &rtc_tm);
    if (ret < 0) {
        perror("RTC_RD_TIME");
        close(fd);
        return -1;
    }

    printf("RTC Time: %04d-%02d-%02d %02d:%02d:%02d\n",
           rtc_tm.tm_year + 1900,
           rtc_tm.tm_mon + 1,
           rtc_tm.tm_mday,
           rtc_tm.tm_hour,
           rtc_tm.tm_min,
           rtc_tm.tm_sec);

    close(fd);
    return 0;
}
```

## 测试说明

### 时间保持测试

1. 设置 RTC 时间
2. 系统重启
3. 启动后检查 RTC 时间是否保持

```bash
# 设置时间
date -s "2025-04-22 18:30:00"
hwclock -w

# 重启系统
reboot

# 启动后检查
hwclock -r
```

### 闹钟唤醒测试

1. 设置闹钟
2. 系统进入休眠
3. 闹钟触发后系统唤醒

```bash
# 设置 60 秒后触发闹钟
echo +60 > /sys/class/rtc/rtc0/wakealarm

# 系统休眠（如果支持）
echo mem > /sys/power/state

# 等待闹钟唤醒系统
```

## FAQ

### RTC 设备不存在

检查以下几点：

1. 确认内核配置已启用 `CONFIG_RTC_CLASS` 和 `CONFIG_RTC_DRV_RPMI`
2. 确认 DTS 中 rpmi_rtc 节点 status 为 "okay"
3. 检查 mailbox 驱动是否正常加载：`dmesg | grep -i mbox`
4. 查看内核日志：`dmesg | grep -i rtc`

### RTC 时间不准确

可能的原因：

1. RTC 未正确初始化，需要手动设置时间
2. 系统时间与 RTC 时间不同步
3. 固件 RTC 实现问题

解决方法：

```bash
# 设置正确的系统时间
date -s "2025-04-22 18:30:00"

# 同步到 RTC
hwclock -w

# 验证
hwclock -r
```

### 闹钟不工作

检查以下几点：

1. 确认中断配置正确：`cat /proc/interrupts | grep rtc`
2. 检查闹钟是否已使能：`cat /sys/class/rtc/rtc0/wakealarm`
3. 查看内核日志是否有错误信息

### 系统时间与 RTC 时间不同步

系统启动时，通常会自动从 RTC 读取时间。如果不同步：

```bash
# 手动从 RTC 同步到系统时间
hwclock -s

# 或者从系统时间同步到 RTC
hwclock -w
```

### RPMI 通信失败

如果出现 RPMI 通信错误：

1. 检查 mailbox 驱动是否正常：`dmesg | grep mpxy`
2. 确认固件版本是否支持 RPMI RTC
3. 查看详细错误信息：`dmesg | grep -i "rpmi\|rtc"`

### 如何在启动时自动同步 RTC 时间

在系统启动脚本中添加：

```bash
# /etc/init.d/S01rtc
#!/bin/sh

case "$1" in
    start)
        echo "Syncing system time from RTC..."
        hwclock -s
        ;;
    stop)
        echo "Syncing RTC from system time..."
        hwclock -w
        ;;
esac
```

## 注意事项

1. **RPMI 依赖**：RPMI RTC 依赖于固件支持，确保固件版本正确
2. **Mailbox 通道**：RTC 使用 mailbox 通道 0xe，不要与其他设备冲突
3. **中断共享**：RPMI RTC 与 RPMI PWRKEY 共享中断 64，这是正常的
4. **时间格式**：RTC 时间使用 UTC 时间，需要考虑时区转换
5. **备用电源**：RTC 在断电后需要备用电池供电才能保持时间
