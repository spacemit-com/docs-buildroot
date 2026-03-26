sidebar_position: 3

# USB 信号质量测试指南

本文介绍 K3 平台 USB 在 **信号质量（SQ）/ 电气测试 / 基础链路验证** 方面的建议方法。

先说结论：K3 这部分不能简单照 K1 原文搬。K1 里有单独的 USB2 OTG/EHCI 章节，而 K3 当前主线是 **DWC3 + USB2/USB3 PHY + Host/Device/OTG/Type-C 组合拓扑**。所以 K3 这篇更适合分成：

- **USB2 Device 高速测试模式**
- **USB Host 侧基础链路与功能验证**
- **USB3 / DRD / Type-C 板级联调注意点**
- **debugfs / role switch / link_state 调试**

如果后续实验室已经有更明确的 USB-IF 测试治具流程，还可以继续把本篇往“正式认证流程”补细。

## 测试分类建议

对 K3 来说，建议把 USB 测试分成三层：

### 1. 基础 bring-up / 功能测试

目标是确认：

- 控制器起来了没有
- PHY 起来了没有
- Host / Device 角色对不对
- 上位机能不能枚举
- Mass storage / ACM / NCM / UVC 等功能能不能跑通

这层通常最先做，也最容易定位问题。

### 2. Device 侧 USB2 测试模式（Test Mode）

目标是输出标准 USB2 测试波形，例如：

- `test_j`
- `test_k`
- `test_se0_nak`
- `test_packet`
- `test_force_enable`

K3 的 DWC3 驱动里明确支持这些测试模式，而且 debugfs 已经提供了 `testmode` 文件。

### 3. 实验室/认证类 SQ 测试

例如：

- USB2 eye diagram
- rise/fall time
- jitter / monotonicity
- USB3 link / compliance 相关测试

这部分最终还是要以 USB-IF 文档、示波器/分析仪、测试夹具和实验室流程为准。本文主要讲 K3 端如何配合输出测试状态。

## 参考资料

### USB 2.0 电气规范

- [USB 2.0 Electrical Compliance Specification | USB-IF](https://www.usb.org/document-library/usb-20-electrical-compliance-test-specification-version-107)

### xHCI 电气测试工具

- [xHCI Electrical Test Tool](https://www.usb.org/document-library/xhsett)

### Linux 内核中的现成测试/调试支持

K3 SDK 内核里能直接看到这些测试相关实现：

- `drivers/usb/dwc3/debugfs.c`
- `drivers/usb/dwc3/gadget.c`
- `drivers/usb/dwc3/ep0.c`
- `tools/usb/testusb.c`
- `tools/usb/hcd-tests.sh`
- `drivers/usb/gadget/function/f_loopback.c`
- `drivers/usb/gadget/function/f_sourcesink.c`

这意味着 K3 的测试能力并不是“纯靠外部工具”，内核本身已经提供了比较完整的调试抓手。

## K3 上和 SQ 测试最相关的 USB 端口

根据现有 K3 DTS 使用方式：

- `usb2_host`：固定 USB2 Host
- `usb3_porta`：最核心的 DRD / OTG / Device 端口
- `usb3_portb/c/d`：主要作为 Host 端口

如果你要做 **Device 侧 USB2 test mode**，当前最值得优先关注的是：

- `usb3_porta`

因为它在现有板级 DTS 中：

- 可能被配置成 `dr_mode = "peripheral"`
- 也可能被配置成 `dr_mode = "otg"`
- 还可能与 `usb-role-switch` / `FUSB301` / Type-C connector 一起工作

所以对 K3 来说，SQ 测试前最重要的一步不是直接发 test packet，而是先确认 **当前这个口真的处于 device 模式**。

## K3 DWC3 debugfs 能力

K3 SDK 中 `drivers/usb/dwc3/debugfs.c` 明确创建了这些 debugfs 节点：

- `regdump`
- `lsp_dump`
- `mode`（Dual Role 打开时）
- `testmode`
- `link_state`

对应代码里可以看到：

```c
if (IS_ENABLED(CONFIG_USB_DWC3_DUAL_ROLE))
        debugfs_create_file("mode", 0644, root, dwc,
                            &dwc3_mode_fops);

if (IS_ENABLED(CONFIG_USB_DWC3_DUAL_ROLE) ||
        IS_ENABLED(CONFIG_USB_DWC3_GADGET)) {
        debugfs_create_file("testmode", 0644, root, dwc,
                            &dwc3_testmode_fops);
        debugfs_create_file("link_state", 0644, root, dwc,
                            &dwc3_link_state_fops);
}
```

同时 debugfs 根目录名来自：

```c
root = debugfs_create_dir(dev_name(dwc->dev), usb_debug_root);
```

也就是说，实际路径不是写死的固定字符串，而是**按设备名动态生成**。因此 K3 文档里不应该直接硬编码成某一个 K1 风格的路径，而应该告诉用户：

1. 先挂载 debugfs
2. 再到 `/sys/kernel/debug/usb/` 下查看真实目录名

例如：

```bash
mount -t debugfs none /sys/kernel/debug
ls /sys/kernel/debug/usb/
```

## USB 2.0 Device 信号质量测试

### 测试波形类型

K3 的 DWC3 debugfs / gadget 逻辑支持这些 USB2 test mode：

- `test_j`
- `test_k`
- `test_se0_nak`
- `test_packet`
- `test_force_enable`
- `none`

这些字符串在 `drivers/usb/dwc3/debugfs.c` 中都能直接对应到：

- `USB_TEST_J`
- `USB_TEST_K`
- `USB_TEST_SE0_NAK`
- `USB_TEST_PACKET`
- `USB_TEST_FORCE_ENABLE`

### 适用前提

在 K3 上做 Device 侧 SQ 测试前，请先确保：

1. 目标端口真的是 `usb3_porta`
2. 当前口已经切到 **device/peripheral**
3. 如果是 Type-C 板级，`FUSB301` 已正常工作
4. `usb-role-switch` graph 正确
5. UDC 已经注册成功
6. debugfs 已挂载

如果当前 role 还在 host，向 `testmode` 写值通常没有意义。

### 典型 DTS 场景 1：固定 peripheral

例如 `k3_fpga_1x1.dts`：

```dts
&usb3_porta {
        status = "disabled";
        maximum-speed = "high-speed";
        dr_mode = "peripheral";
        /delete-property/ phy_type;
        phy_type = "utmi_wide";
};
```

这种板级最适合先做 Device 侧测试，因为路径比较单纯。

### 典型 DTS 场景 2：OTG + Type-C

例如 `k3_deb1.dts`、`k3_dc_board.dts`、`k3_com260_kit_v02.dts` 中都能看到：

```dts
&usb3_porta {
        dr_mode = "otg";
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        monitor-vbus;
        status = "okay";
};
```

这类板子做 SQ 测试前，要先解决：

- 插线后 role 有没有切到 device
- Type-C 控制器（如 `onsemi,fusb301`）有没有起来
- VBUS 监测是否正常

### K3 Device 侧 testmode 操作步骤

#### 1. 挂载 debugfs

```bash
mount -t debugfs none /sys/kernel/debug
```

#### 2. 找到真实 USB debugfs 节点

```bash
ls /sys/kernel/debug/usb/
```

然后进入对应的 DWC3 目录，例如：

```bash
cd /sys/kernel/debug/usb/<实际的dwc3目录名>/
ls
```

如果配置正确，通常应能看到：

- `mode`
- `testmode`
- `link_state`
- `regdump`
- `lsp_dump`

#### 3. 查看当前模式

```bash
cat mode
cat testmode
cat link_state
```

#### 4. 进入测试模式

常见顺序：

```bash
echo test_force_enable > testmode
echo test_packet > testmode
```

或者直接：

```bash
echo test_j > testmode
echo test_k > testmode
echo test_se0_nak > testmode
echo test_packet > testmode
```

#### 5. 查看当前状态

```bash
cat testmode
```

#### 6. 退出测试模式

```bash
echo none > testmode
```

### 上位机通过 xHCI Electrical Test Tool 触发

如果实验室流程是通过上位机工具控制设备进入测试模式，那么在 K3 上也可以沿用这条路线：

- 先让 `usb3_porta` 进入 device / peripheral
- 接到测试治具和上位机
- 使用 USB-IF 的 xHCI Electrical Test Tool 发相应测试命令

这条路径对 K3 仍然成立，因为底层仍是标准 USB Device / DWC3 gadget 体系。

## USB Host 侧测试建议

K3 当前 Host 端口更多，实际板级差异也更大，因此 Host 侧测试更适合分成两类：

### 1. Host 功能回归测试

重点确认：

- U 盘枚举是否正常
- HUB 枚举是否正常
- 键鼠/网卡/摄像头是否正常
- USB2 / USB3 速率是否符合预期
- Type-A / Type-C Host 口是否稳定

推荐最先执行：

```bash
dmesg | grep -Ei "usb|xhci|dwc3"
lsusb
lsusb -t
```

如果插入 U 盘：

```bash
lsblk
mount /dev/sdX1 /mnt
```

### 2. Host 侧协议/传输测试

K3 SDK 内核中已有：

- `tools/usb/testusb.c`
- `tools/usb/hcd-tests.sh`

这说明可以在 Host 场景下结合 `usbtest` 驱动做更系统的传输测试。

适合的场景包括：

- bulk/control/interrupt 传输稳定性
- loopback 类测试
- 基础主机控制器功能验证

如果后续要把 USB Host 测试这部分写得更深，可以专门补充：

- `usbtest` 驱动的配置方式
- `testusb` 工具的编译和运行方式
- 哪些测试项适合 K3 日常回归

## USB3 / DRD / Type-C 联调注意点

K3 上很多“像 SQ 问题”的故障，实际根因不一定是信号质量本身，而可能是：

- 角色没切对
- Type-C graph 没接好
- `monitor-vbus` 不工作
- `FUSB301` 中断脚不对
- `usb3_phy_switch` 没配好
- 板级把口限速成 `high-speed`

所以建议在正式怀疑 SQ 之前，先排这些。

### 1. 检查 role

```bash
ls /sys/class/usb_role/
cat /sys/class/usb_role/*/role
```

### 2. 检查 Type-C

```bash
ls /sys/class/typec/
dmesg | grep -i fusb
```

### 3. 检查 UDC

```bash
ls /sys/class/udc
```

### 4. 检查 link state

如果 DWC3 debugfs 节点存在：

```bash
cat /sys/kernel/debug/usb/<dwc3-dir>/link_state
```

### 5. 检查 mode

```bash
cat /sys/kernel/debug/usb/<dwc3-dir>/mode
```

Dual Role 模式下，这个节点对定位当前控制器到底在 Host 还是 Device 很有用。

## K3 板级测试建议

### 场景一：固定 peripheral 下载口

如果某板级和 `k3_fpga_1x1.dts` 类似，建议先做：

1. 枚举测试
2. gadget ACM / NCM 基础功能测试
3. testmode 输出 `test_packet`
4. 再做实验室眼图测试

### 场景二：Type-C OTG 口

如果某板级和 `k3_deb1.dts` / `k3_dc_board.dts` / `k3_com260_kit_v02.dts` 类似，建议先做：

1. Type-C 角色切换测试
2. Host / Device 双向枚举测试
3. gadget ACM / NCM 基础验证
4. 再做 testmode
5. 最后再做更严格的 SQ 测试

因为这种场景里，role switch 往往比纯电气问题更先出错。

### 场景三：固定 Host 口

对于 `usb2_host`、`usb3_portb/c/d`，建议优先做：

1. `lsusb` / `lsusb -t`
2. U 盘 / HUB / 网卡 / 摄像头枚举
3. `testusb` / `usbtest` 主机传输回归
4. 若需要实验室电气测试，再按具体控制器/夹具流程继续

## FAQ

### 1. K3 能不能像 K1 那样直接写死某个 debugfs 路径？

不建议。因为 K3 DWC3 debugfs 目录名来自 `dev_name(dwc->dev)`，更稳妥的方式是先：

```bash
ls /sys/kernel/debug/usb/
```

再进入真实目录。

### 2. K3 支不支持 `test_j` / `test_k` / `test_packet`？

支持。`drivers/usb/dwc3/debugfs.c` 明确实现了：

- `test_j`
- `test_k`
- `test_se0_nak`
- `test_packet`
- `test_force_enable`
- `none`

### 3. 为什么我写了 `testmode` 但没看到波形？

优先排查：

- 当前口是不是 `usb3_porta`
- 当前 role 是不是 `device`
- UDC 是否已注册
- debugfs 是否挂载
- 板级是不是 OTG/Type-C，且还没真正切到 device
- `maximum-speed` / `phy_type` 是否和板级硬件匹配

### 4. K3 的 SQ 文档为什么比 K1 更强调 role switch？

因为 K3 实际产品板大量采用：

- `dr_mode = "otg"`
- `usb-role-switch`
- `role-switch-default-mode = "peripheral"`
- `monitor-vbus`
- `onsemi,fusb301`

所以在 K3 上，很多测试前提都取决于角色链路是否先正确工作。
