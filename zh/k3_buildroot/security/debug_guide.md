---
sidebar_position: 8
---

# TEE 调试指南

## 概述

### 编写目的

本文提供 K3 平台 TEE 的调试方法，覆盖启动阶段的问题定位、运行期故障排查与 JTAG/GDB 在线调试，帮助开发者快速区分「构建配置问题」「环境问题」与「功能缺陷」。

本文明确区分两类能力，二者不可混淆：

- **调试能力**：是否允许使用 JTAG/GDB、断点、单步、寄存器/内存读取、panic/恢复和源码符号定位。
- **日志能力**：TEE Core、ldelf 和用户 TA 是否输出 ERROR、INFO、DEBUG、FLOW 日志。

> [!CAUTION]
> 日志等级高不等于 JTAG 已开放；JTAG 可连接也不等于设备允许调试。两类能力由不同的机制控制（日志由编译配置 + 生命周期状态控制，JTAG 由 eFuse 硬件门控控制）。

### 适用范围

本文适用于 SpacemiT K3 系列 SoC 的 Buildroot 方案。平台结构与启动时序参见 [TEE 架构与集成指南](tee_architecture.md)；示例与测试套件的运行方法参见 [TEE 示例集](examples.md)。

### 文档结构

1. **调试能力与生命周期**：能力边界、LCS 生命周期与 JTAG 管控现状。
2. **启动阶段定位**：按启动链路分段检查。
3. **运行期定位**：OP-TEE 日志定位、常见故障与判据。
4. **JTAG/GDB 在线调试**：硬件门控现状、官方连接指南与符号文件位置。
5. **调试辅助手段**。
6. **常见问题（FAQ）**。

## 调试能力与生命周期

### LCS 生命周期

K3 通过 eFuse 中的 **LCS（Lifecycle State）** 区分设备生命周期，并据此决定日志与调试能力。

| 生命周期 | 代码枚举 | 设备状态 | 日志能力 |
|---|---|---|---|
| CM | `K3_LCS_CM` | 工程/开发板 bring-up | 允许 ERROR/INFO/DEBUG/FLOW |
| DM | `K3_LCS_DM` | 产测/开发验证 | 允许 ERROR/INFO/DEBUG/FLOW |
| SP | `K3_LCS_SP` | 量产设备 | 允许 ERROR/INFO |
| RMA | `K3_LCS_RMA` | 售后返修 | 允许 ERROR/INFO/DEBUG/FLOW |
| UNKNOWN | `K3_LCS_UNKNOWN` | LCS 异常/读取失败 | Core 运行时阈值回退为 ERROR |

**非量产阶段（CM/DM/RMA）默认全部 debug 日志都已打开**（ERROR/INFO/DEBUG/FLOW）；量产（SP）阶段只保留 ERROR/INFO。默认日志级别由编译配置决定，运行时由 LCS 状态控制：仅 OP-TEE Core 层日志随 LCS 切换，**ldelf 与用户 TA 的日志不受 LCS 影响，始终按编译配置输出**。

### JTAG 管控现状

K3 暴露 JTAG Debug Module，但当前 JTAG 通路的管控能力如下，请按**现状**判断，不要套用「量产默认关 JTAG」的假设：

- **eFuse 关闭位**：eFuse bank0 bit14 的 `JTAG_DISABLE` 字段，是当前**唯一**已实现的 JTAG 硬件关闭开关。烧录该位后调试器无法枚举目标，JTAG 完全不可用。
- **量产设备没有要求一定关闭 `JTAG_DISABLE`**：是否关闭由产品自行决定，当前默认不强制关闭。
- **认证调试（authenticated debug）尚未实现**。
- **IOPMP 对 debug 通道已实施访问管控，但当前配置为全部放开**：IOPMP INT0 的 SID0 已管控 debug 通道，但当前未拦截任何访问（全部放开），故实际不构成调试限制。

因此，调试时「能否连上 JTAG」只取决于 `JTAG_DISABLE` 熔丝位是否被烧录；认证调试尚未实现，IOPMP 虽管控 debug 通道但当前全部放开，均不构成额外限制。

## 启动阶段定位

启动失败时按启动链路顺序排查，每一段有独立的判据：

| 阶段 | 期望现象 | 未出现时的检查点 |
|---|---|---|
| SPL 加载 TEE 镜像 | 默认无「加载成功」提示；**仅当 SPL 开启安全启动**时才会打印 optee 加载结果 | 安全分区表中 TEE 分区是否存在；镜像是否写入该分区；是否误用了普通镜像的分区表 |
| OpenSBI 拉起 OP-TEE | 出现 OP-TEE 版本横幅 | OpenSBI 固件是否含分发器符号；设备树是否包含域门控节点 |
| OP-TEE 初始化 | 横幅之后无致命错误 | 平台时钟、随机数、eFuse 与安全引擎初始化是否成功 |
| 内核探测 TEE | 出现 TEE 驱动启动成功 | 内核是否含 TEE 与 OP-TEE 驱动；设备树是否描述了 TEE 节点 |
| `tee-supplicant` | 进程存在 | 初始化脚本是否执行；TA 存放目录是否挂载 |

三个高频误判：

1. **串口完全没有新输出**。通常不是 TEE 问题，而是烧录阶段就失败了。先核对镜像包的文件集是否完整，尤其是打包清单中被依赖的公共文件。
2. **没有 OP-TEE 横幅但没有报错**。多数是设备树缺少域门控节点，OpenSBI 因此没有配置可信域，直接跳过 TEE 进入普通启动，链路本身并不报错。
3. **SPL 日志有、后续全无**。检查 TEE 镜像是否超出分区容量。超限会导致写入不完整，表现为加载成功但执行异常。

## 运行期定位

### OP-TEE 日志架构

TEE 日志统一从 **TEE 串口** 输出；普通系统侧只能通过内核 `dmesg` 看到驱动层少量日志。**两条观察通道互不替代**。

安全世界内部按模块分三层，各自带前缀，便于区分日志来源：

| 模块 | 前缀 | 说明 |
|---|---|---|
| Core（核心） | `I/TC`、`E/TC` | OP-TEE 内核与平台初始化日志 |
| ldelf（加载器） | `I/LD`、`E/LD` | TA 加载器日志 |
| 用户 TA | `I/TA`、`E/TA` | TA 应用自身日志 |

### 日志等级配置

日志等级由构建配置控制，等级越高输出越详细，同时镜像体积与运行开销也越大：

| 配置项 | 作用 | 取值 |
|---|---|---|
| `CFG_TEE_CORE_LOG_LEVEL` | OP-TEE 内核与平台日志等级 | 0 关闭 / 1 仅 ERROR / 2 +INFO / 3 +DEBUG / 4 +FLOW（最详细） |
| `CFG_TEE_TA_LOG_LEVEL` | 用户 TA 日志等级 | 同上 |

当前平台默认 `CFG_TEE_CORE_LOG_LEVEL ?= 4`、`CFG_TEE_TA_LOG_LEVEL=4`，便于开发定位。**量产构建应将其降级**，原因有二：一是详细日志会向串口暴露内部状态；二是会增加镜像体积，而 TEE 分区容量有限。

### 先确认层次

```bash
# 设备节点
ls /dev/tee*

# 后台服务
ps | grep tee-supplicant

# 内核侧 TEE 相关日志
dmesg | grep -i tee
```

### 常见故障

| 现象 | 最可能原因 | 处理 |
|---|---|---|
| CA 打开会话返回找不到对象 | TA 未安装或 UUID 不一致 | 核对 TA 存放目录与 UUID |
| CA 返回参数错误 | 参数类型与 TA 不匹配 | 逐个核对 `paramTypes`，并确认操作结构体已清零 |
| 安全存储调用返回存储不可用 | `tee-supplicant` 未运行，或存储后端不可用 | 启动后台服务；确认存储后端配置 |
| `xtest` 大量用例跳过 | 对应功能未在构建中使能 | 属构建配置问题，按需使能后重编 |
| 涉及 RPMB 的用例失败 | 器件未完成 RPMB key 灌注，或 RPMB 未使能 | 确认构建形态与器件灌注状态 |
| 示例运行但无 TEE 日志 | 日志等级偏低 | 按需提高日志等级后重编 |
| 随机数结果固定 | 随机数源未生效 | 检查平台随机数驱动是否被使用 |

### 错误来源定位

CA 返回的错误码同时带有「来源」字段，是区分故障层次的最快手段：

| 来源 | 指向 |
|---|---|
| 客户端库 | 调用方式错误，如参数为 NULL、上下文未初始化 |
| 通信层 | 环境问题，如后台服务未运行、设备节点缺失或权限不足 |
| TEE 内核 | 平台或内核侧问题 |
| TA | TA 自身的逻辑、参数校验或存储操作 |

优先看通信层与 TA 两类：绝大多数接入期问题落在其中。

## JTAG/GDB 在线调试

K3 暴露 JTAG Debug Module，调试链路为 **J-Link 硬件探针 → OpenOCD（PC）→ TCP :1024 → RISC-V GDB**。JTAG 通路硬件门控现状见上文「JTAG 管控现状」。

JTAG 连接、J-Link/OpenOCD/GDB 的驱动安装、连接步骤与调试操作（halt/resume、断点、单步、调用栈等）均参见官方工具使用指南：

> [JTAG 调试工具使用指南](https://www.spacemit.com/community/document/info?lang=en&nodepath=tools/user_guide/jtag_debug_user_guide.md)

源码级调试需要加载 OP-TEE 带符号的 ELF 文件 `tee.elf`，位于 OP-TEE 构建输出目录：

```text
optee/optee_os/out/riscv/core/tee.elf
```

（即使用 `O=out/riscv` 构建目录时，产物在 `core/tee.elf`；该文件必须与板端实际烧录的镜像一致。）

## 调试辅助手段

### 构建侧核对

```bash
# OP-TEE 构建产物（带符号 ELF 与裸二进制）
ls -l optee/optee_os/out/riscv/core/tee.elf optee/optee_os/out/riscv/core/tee.bin

# TEE 镜像体积（optee 分区约 512K，超限会导致写入不完整）
ls -l optee/optee_os/out/riscv/core/tee.bin
```

### JTAG DISABLE 验证

烧录 `JTAG_DISABLE` 熔丝位（eFuse bank0 bit14）后，可通过 OpenOCD 连接行为确认关闭是否生效。烧录后的典型输出：

```text
Info : VTarget = 1.786 V
Error: JTAG scan chain interrogation failed: all zeroes
Error: post.unknown: IR capture error; saw 0x0 not 0x1
Warn : Bypassing JTAG setup events due to errors
```

判读要点：

- `VTarget` 有稳定电压读数，说明目标已上电、参考电压已连接，排除线缆断开或目标无电。
- 扫链结果为**全零**（而非全一）：TDO 悬空/未连接时通常读为全一；全零表示 TDO 被目标侧主动拉低，与 TAP 被禁用后的表现一致。
- OpenOCD 随后仍会以配置好的扫描链继续并启动 GDB 端口（`Trying to use configured scan chain anyway`），这不代表连接成功，后续任何目标操作都会失败。

为排除其他可能（引脚被固件复用、TDO 短路、模式不匹配），建议做对照：同一套连接与脚本在未烧录的同型板子上应能正常通过扫链并读出 IDCODE；同时确认被测板串口有正常启动日志。

> [!CAUTION]
> JTAG 的关闭不是在所有状态下都成立：生命周期进入 RMA 后，JTAG 会无视禁用位重新使能、启动链不再验签（软件侧的启动门对 RMA 放行并打印告警）。如果在已烧录 JTAG DISABLE 的板子上仍能连上 JTAG，先确认 eFuse 的生命周期字段是否为 RMA，再排查连接问题。参看[安全方案架构白皮书](white_paper.md)的安全生命周期章节。
>
> 该禁用位在 eFuse bank0 bit14。固件在启动早期读的 `SYS_BOOT_CNTRL` 寄存器不独立存储状态，只是把若干 eFuse 位汇总映射出来，因此排查时应以熔丝位为准，不要把该寄存器里的位号当作熔丝位号。

## 常见问题（FAQ）

1. **量产设备一定连不上 JTAG 吗？** 不一定。量产设备没有要求一定关闭 `JTAG_DISABLE`，当前默认不强制关闭；是否关闭由产品决定。认证调试尚未实现，IOPMP 虽管控 debug 通道但当前全部放开。
2. **非量产阶段默认输出哪些日志？** CM/DM/RMA 阶段默认 ERROR/INFO/DEBUG/FLOW 全部打开；量产（SP）阶段只保留 ERROR/INFO。
3. **日志等级很高，是否说明 JTAG 已经开放？** 否。日志能力由编译配置 + LCS 状态控制，JTAG 由 eFuse 硬件门控控制，二者相互独立。

## 上报问题时建议附带的信息

- 设备 LCS 生命周期状态 / 是否已烧录 eFuse（影响日志等级与 JTAG 行为）
- 使用的是安全镜像还是普通镜像，以及镜像文件名中的时间戳
- 串口完整日志（从 SPL 开始）
- `dmesg | grep -i tee` 的输出
- 复现所用的具体命令与返回码、错误来源
