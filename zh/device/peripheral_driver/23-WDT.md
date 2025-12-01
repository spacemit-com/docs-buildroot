# WDT

介绍WDT的配置和调试方式

## 模块介绍  
**WDT（watchdog，看门狗）控制器**是一种监控系统正常运行的电器元件。设定超时时间并开启看门狗后需定时喂狗，若超过超时时间没有喂狗入系统挂死时，则看门狗元件触发复位信号，复位K1芯片

### 功能介绍  

![wdt](static/WDT.png) 

内核通过 **WDT框架层接口** 将看门狗驱动注册到到WDT框架及应用层，生成/dev/watchdog0设备节点。
用户层进程可以通过对设备节点的open ioctl操作来实现先看门狗的使能、设置超时时间及喂狗操作。  

### 源码结构介绍

WDT控制器驱动代码在 `drivers/watchdog` 目录下：  

```
drivers/watchdog
|--watchdog_core.c        #内核WDT框架接口代码
|--watchdog_dev.c         #内核WDT框架注册字符设备到用户层代码
|--k1x_wdt.c              #k1 WDT驱动  
```  

## 关键特性  

- 超时可产生复位信号
- 最高可设置 **255s** 超时时间

## 配置介绍

主要包括 **驱动使能配置** 和 **dts配置**

### CONFIG配置

**CONFIG_WATCHDOG**
此为内核平台WDT框架提供支持，支持K1 WDT驱动情况下，应为 **Y**

```
Symbol: WATCHDOG [=y]
Device Drivers
      -> Watchdog Timer Support (WATCHDOG [=y])
```

在支持平台层WDT框架后，配置 **CONFIG_SPACEMIT_WATCHDOG** 为 **Y**，支持 K1 PWM驱动

```
Symbol: SPACEMIT_WATCHDOG [=y]
      -> Spacemit-k1x SoC Watchdog (SPACEMIT_WATCHDOG [=y])
```

### dts配置

#### dtsi配置示例

dtsi中配置WDT控制器基地址和时钟复位资源，正常情况无需改动
其中若不配置spa,wdt-disabled，watchdog驱动加载时会使能watchdog，并自己申请hrtimer定时喂狗

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

#### dts配置示例

dts完整配置，如下所示

```dts
 &watchdog {
     status = "okay";
 };
```

## 接口介绍

### API介绍

linux内核主要实现了将watchdog注册为字符设备，将设备文件节点提供给应用层使用。
常用的file_operations中接口有：

```
static int watchdog_open(struct inode *inode, struct file *file)
# 该接口实现了支持用户层打开watchdog节点
static long watchdog_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
# 该接口通过不同cmd实现了设置、获取超时时间，使能、关闭看门狗，喂狗等功能
```

## Debug介绍

由于内核watchdog框架将看门狗驱动注册成字符设备提供给应用层，测试要用户自行实现一个watchdog应用，示例如下：
以下基于 **Buildroot 系统** 验证

```sh
fd = open("/dev/watchdog", O_WRONLY);
#打开WDT节点
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


## 测试介绍

可以测试正常喂狗情况下，系统不会复位和超时未喂狗会出现系统复位

## FAQ
