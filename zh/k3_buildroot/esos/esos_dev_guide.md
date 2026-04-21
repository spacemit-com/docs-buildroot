---
sidebar_position: 1
---

# ESOS 开发指南
esos系统基于rt-thread开发，跑在rcpu上，其功能包含如下两部分
1. 配合大核完成能效管理，如调频调压、power-switch开关、系统reboot、系统休眠唤醒
2. 配合大核完成实时任务处理，如控制电机转动、控制继电器开关等

### ESOS目录结构
#### SDK根目录结构
```
|-- bsp-src
|   |-- linux-6.18
|   |-- opensbi
|   `-- uboot-2022.10
|-- buildroot
|-- buildroot-ext
|-- Makefile
|-- package-src
|   |-- esos
`-- scripts
    |-- build-muse-boot-plt.sh
    |-- build-plt-stability.sh
    |-- check-config.sh
    |-- check-dl-update.py
    |-- Dockerfile
    |-- envsetup.sh
    |-- gen_imgcfg.py
    |-- LICENSE
    |-- Makefile
    `-- ubuntu.mirror

```
#### ESOS内部目录结构
```.
|-- AUTHORS
|-- bsp
|   |-- spacemit
|-- build.sh
|-- build_top.sh
|-- ChangeLog.md
|-- components
|-- debian
|-- documentation
|-- esos_rt24.its
|-- esos_rt24_sign.its
|-- examples
|-- include
|-- Jenkinsfile
|-- Kconfig
|-- libcpu
|   |-- risc-v
|-- LICENSE
|-- null.spacemit
|-- README.md
|-- README_zh.md
|-- src
`-- tools

```
## ESOS编译方法
### 顶层SDK编译方法
```
# 编译，默认编译rt24 all core
make esos

# 清理
make esos-dirclean

# 重新编译，注意不是make esos-rebuild
make esos-reconfigure

```
### ESOS单独编译及配置
esos的修改或者menuconfig配置需要进入esos源码目录，并选择要修改的core：
```
./build.sh config
INFO: prepare to config esos sdk ...
All valid soc chips:
        0: n308
        1: rt24
Please select a chip:1
All valid boards:
        0: os0_rcpu
        1: os1_rcpu
Please select a board:0

INFO: target configuration is as follows:
INFO: -------------------------------------------------------------------------
export TARGET_CHIP=rt24
export TARGET_BOARD=os0_rcpu
export TARGET_DEFCONFIG=rt24_os0_rcpu_defconfig
export TARGET_ENTRY_POINT=0x100200000
INFO: -------------------------------------------------------------------------

```
选完core后就可以开始用menuconfig进行图形化配置，配置完成后修改保存在bsp/spacemit/platform/rt24/osX_rcpu/rt24_osX_rcpu_defconfig
```
./build.sh menuconfig

```
修改完成后，运行指令可以生成core的.elf文件
```
./build.sh

```
要生成esos.itb需要返回上层目录执行如下命令：
```
cd buildroot-k3
make esos-reconfigure

```
