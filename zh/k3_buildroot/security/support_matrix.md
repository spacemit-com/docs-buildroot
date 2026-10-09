---
sidebar_position: 1
---

# 安全特性支持矩阵

| 项目 | 内容 |
|---|---|
| 适用 SoC | SpacemiT K3 |
| 适用方案 | Buildroot（安全镜像） |
| 对应 TEE 版本 | `spacemit.k3.v1.0`（与 OP-TEE 构建配置一致） |
| 矩阵版本 | v1.0（2026-09-18） |

本矩阵中的状态判定与「最低软件版本」列均以上述版本为准。低于标注版本时不具备该特性；版本升级后需按备注中的前提重新确认。

> [!IMPORTANT]
> **版本号为占位符**
>
> 安全方案尚未正式发布，本文出现的版本号（含「最低软件版本」列一律填写的 `spacemit.k3.v1.0`）均为占位符，用于标识特性归属的软件基线，**不代表已发布的版本号**。安全版本正式发布后统一替换为实际版本。

本矩阵面向安全系统 v1.0，不覆盖产测形态；状态取值见下方图例，尚未定论的事项一律按「暂不支持」处理。后续版本迭代时同步更新。

## 概述

### 编写目的

本表汇总 SpacemiT K3 平台在 Buildroot 方案下的安全特性支持情况，按「特性 × SoC 型号 × 软件版本」给出状态与最低版本要求，供方案选型、需求评审与量产评估使用。

### 适用范围

本文档适用于 SpacemiT K3 系列 SoC 的 Buildroot 方案。安全启动的信任链细节参见[安全启动开发指南](../device/secureboot.md)，TEE 的结构与集成参见 [TEE 架构与集成指南](tee_architecture.md)。

> [!IMPORTANT]
> **启动介质容量要求**
>
> 当前安全镜像对启动介质的要求：
>
> - **容量**：要求**大于等于 8M**。4M SPI-NOR 放不下——安全版 `u-boot.itb`（约 2383K）超出其分区余量（1536K）；8M 及以上均可，存在 8M 的 SPI-NOR。
> - **可用介质**：SPI-NOR 8M 及以上、SPI-NAND，或 eMMC / SD / UFS。
> - **详见**：本文「存储介质差异」与 [TEE 镜像构建与烧录说明](tee_image.md)。

### 状态图例

| 标记 | 含义 |
|---|---|
| ✓ 支持 | 已实现并通过实测或构建验证，可直接用于产品 |
| ✗ 不支持 | 当前未实现，或已实现但默认未使能且无明确启用路径 |
| 部分支持 | 可用但受条件限制，需按备注中的前提使用 |
| 规划中 | 已列入计划但当前不可用 |
| 不在范围 | 规范涉及但本方案未采用，不作支持状态判定 |

「最低软件版本」指出该特性首次可用的软件基线。低于该版本时不具备该特性，升级后需按备注中的前提重新确认。

> [!NOTE]
> 当前仅有一个 SoC 型号（K3）与一个软件基线，因此矩阵暂为单列。后续新增 SoC 或版本时按同一格式扩展列，无需改动结构。

## SoC 型号

| SoC | 核心 | 存储支持 | 说明 |
|---|---|---|---|
| K3 | RISC-V `spacemit,x100`（`rv64imafdcvh`） | eMMC/SD、UFS、SPI-NOR、NAND | 当前基线 |

## 安全特性 × SoC × 软件版本

### 信任根与安全启动

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| 信任根（RoT） | `spacemit.k3.v1.0` | ✓ | 芯片内只读、不可篡改的 BootROM 与 eFuse 密钥材料（RoT 公钥哈希在 bank4）；规范中 hart 固件类信任根，见白皮书 §5 |
| Secure Boot（FIT 逐级验签） | `spacemit.k3.v1.0` | 部分支持 | 需烧入 ROTPKH（eFuse bank4）；BootROM 依 eFuse 的 secure boot 位决定是否启用，启用后对 FSBL 验签；U-Boot 等后续镜像的验签由 SPL 配置开启；未使能时为非签名启动 |
| ROTPKH 存储 | `spacemit.k3.v1.0` | ✓ | eFuse bank4，256 bit，值为 root 公钥的 SHA256 |
| FSBL 证书链 | `spacemit.k3.v1.0` | ✓ | `root_key` + `spl_key`，`oem_key` 携带 spl / uboot 公钥 |
| FIT 公钥注入 | `spacemit.k3.v1.0` | ✓ | SPL DTB 与各板级 U-Boot DTB |
| 防回滚计数器 | `spacemit.k3.v1.0` | ✗ | 字段已定义（ARCN-NS / ARCN-Sec），当前未启用、未参与校验 |
| 镜像分区替换（可更新性） | `spacemit.k3.v1.0` | ✓ | 各分区镜像可独立替换；启动前不做度量与验签，见白皮书 §12 |
| TEE 镜像签名 | `spacemit.k3.v1.0` | 部分支持 | 签名脚本已随方案提供，但未接入构建流程，需手动执行签名；默认 FIT 描述仅含 crc32，无 RSA 验签，依赖 SPL 加载路径可信 |

### 密钥体系与 eFuse

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| eFuse 只读访问（安全世界） | `spacemit.k3.v1.0` | ✓ | CPU 直读 GEU 寄存器块；12 bank × 256 bit |
| eFuse 烧写 | `spacemit.k3.v1.0` | ✓ | 由 U-Boot 明文烧写；安全世界按设计不提供写路径 |
| eFuse 读回校验 | `spacemit.k3.v1.0` | ✓ | 可读 per-bank 锁定位 |
| TE200 访问 eFuse | `spacemit.k3.v1.0` | ✗ | BROM 阶段可用，U-Boot 阶段不可用；当前不提供经安全引擎读取 eFuse 的路径 |
| HUK 来源与派生 | `spacemit.k3.v1.0` | ✓ | 由两个 fuse bank 混合派生；任一方单独无法复现 |
| 未烧 fuse 兜底 | `spacemit.k3.v1.0` | 部分支持 | 默认使用公开开发 HUK，仅供 bring-up，**不可出货** |
| HUK 指纹校验 | `spacemit.k3.v1.0` | 部分支持 | 默认关闭；仅用于追溯，出货前须移除 |
| 子密钥派生 | `spacemit.k3.v1.0` | ✓ | RPMB / SSK / DIE_ID / UNIQUE_TA / TA_ENC 等 |
| 存储密钥层级（SSK/TSK/FEK） | `spacemit.k3.v1.0` | ✓ | 由 HUK 逐级派生 |
| 核心加固（ASLR / 栈保护等） | `spacemit.k3.v1.0` | 部分支持 | 平台强制关闭地址随机化与栈保护 |

### 安全存储

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| REE FS 存储 | `spacemit.k3.v1.0` | 部分支持 | 默认后端；**无抗回滚保护** |
| REE FS 完整性 + RPMB | `spacemit.k3.v1.0` | ✗ | 依赖 RPMB 文件系统，当前未使能 |
| RPMB 存储（eMMC） | `spacemit.k3.v1.0` | 部分支持 | 代码就绪，默认未使能 |
| RPMB 存储（UFS） | `spacemit.k3.v1.0` | 部分支持 | 探测接口已支持 UFS，块数与 CRC 处理与 eMMC 不同 |
| RPMB key 派生 | `spacemit.k3.v1.0` | ✓ | 由 HUK 结合器件标识派生；eMMC 用其原生 CID 并掩掉可变字段（PRV、CRC），UFS 用哈希后的 device_id |
| RPMB 灌注目标定向 | `spacemit.k3.v1.0` | 部分支持 | 可按 CID 钉住目标器件，避免多器件写错 |
| RPMB 探测能力声明 | `spacemit.k3.v1.0` | ✗ | 默认声明支持，但 RPMB 文件系统未使能，属已知不一致 |
| tee-supplicant RPMB 模拟 | `spacemit.k3.v1.0` | ✗ | 已显式关闭，避免 ioctl 被文件模拟 |

### 密码与随机数

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| 硬件 PRNG | `spacemit.k3.v1.0` | ✓ | 使用平台硬件随机数，软件后端关闭 |
| 加密加速（非对称） | `spacemit.k3.v1.0` | ✓ | RSA、ECDSA、SM2DSA；曲线 P192–P521 |
| 加密加速（对称） | `spacemit.k3.v1.0` | ✓ | AES、SM4；ECB/CBC/CTR |
| 安全引擎摘要 | `spacemit.k3.v1.0` | ✗ | 未启用，使用软件实现 |
| 安全引擎 TRNG | `spacemit.k3.v1.0` | ✗ | 不支持 |

### TEE 运行环境与集成

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| OP-TEE OS（S 态 / RISC-V） | `spacemit.k3.v1.0` | ✓ | 平台 `plat-k3` |
| OpenSBI OPTEED 分发器 | `spacemit.k3.v1.0` | ✓ | 使能 OP-TEE 分发与 MPXY 传递 |
| SPL 预加载 TEE 镜像 | `spacemit.k3.v1.0` | ✓ | MMC/MTD 路径上镜像缺失即中止启动 |
| 独立 TEE 分区 | `spacemit.k3.v1.0` | ✓ | eMMC/SD 4 MB；SPI-NOR 512 KB |
| 内存布局（保留式共享内存） | `spacemit.k3.v1.0` | ✓ | TEE RAM 与共享内存地址固定 |
| 动态共享内存 | `spacemit.k3.v1.0` | ✗ | 未启用，使用保留式共享内存 |
| 内核 TEE/OPTEE 驱动 | `spacemit.k3.v1.0` | ✓ | 含 MPXY mailbox 与协议驱动 |
| tee-supplicant 自启 | `spacemit.k3.v1.0` | ✓ | 由初始化脚本拉起 |
| 多核与多线程 | `spacemit.k3.v1.0` | ✓ | 16 核 16 线程 |
| 安全设备树（域门控） | `spacemit.k3.v1.0` | ✓ | 提供 trusted / untrusted 域节点 |

### TA / CA 与用户态

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| User TA | `spacemit.k3.v1.0` | ✓ | 目标 RISC-V 64 位 |
| TA 浮点 | `spacemit.k3.v1.0` | ✗ | 已关闭，用于减小攻击面 |
| TA 缓存维护 API | `spacemit.k3.v1.0` | ✓ | 供声明缓存维护属性的 TA 使用 |
| 内置早期 TA | `spacemit.k3.v1.0` | ✓ | Trusted Keys |
| TEE Client API（libteec） | `spacemit.k3.v1.0` | ✓ | CA 侧编程接口 |
| 测试套件（xtest） | `spacemit.k3.v1.0` | ✓ | 八个套件分组 |
| 官方示例 | `spacemit.k3.v1.0` | ✓ | 七个示例 |
| PKCS#11 | `spacemit.k3.v1.0` | ✓ | opensc / pcsc / p11-kit 已进 initramfs |
| RTC PTA | `spacemit.k3.v1.0` | ✓ | 向普通世界暴露平台 RTC |
| PTA 设备枚举 | `spacemit.k3.v1.0` | ✓ | 供普通世界探测 TEE 存储设备 |
| TA 加载加密 | `spacemit.k3.v1.0` | ✗ | 构建期开关为 `CFG_ENCRYPT_TA`，默认未启用。**默认加密密钥是源码中公开的固定值**，启用前必须替换 |
| fTPM | `spacemit.k3.v1.0` | ✗ | 当前未提供 |

### 访问控制与其它

| 特性 | 最低软件版本 | K3（当前基线） | 备注 |
|---|---|---|---|
| PMP | `spacemit.k3.v1.0` | ✓ | hart 侧域隔离，由 OpenSBI 按域配置。规则表项数量固定，域划分不宜过度碎片化 |
| IOPMP | `spacemit.k3.v1.0` | ✓ | 9 个检查器实例（IOPMP4 恒为关闭，功能冗余）；全部采用静态 SID（AxUSER）；实例与主设备、管控地址空间的对应见[安全方案架构白皮书](white_paper.md) §9.1 |
| 调试访问控制 | `spacemit.k3.v1.0` | 部分支持 | 调试主设备（DBG_AW/DBG_AR）经 IOPMP6 按策略约束，TEE RAM 等安全区域可对调试关闭；eFuse bank0 bit14 可永久关闭 JTAG；不支持按特权级/按域分离使能，详见[安全方案架构白皮书](white_paper.md) §10 |
| 安全生命周期 | `spacemit.k3.v1.0` | 部分支持 | 状态由 eFuse bank6 byte4–byte7 的 32 位 LCS 字段指定，CM / DM / SP / RMA 四组各占 3 bit（另有 DFT 8 bit 与保留 12 bit），每组只要有 1 bit 置 1 即生效，优先级 RMA > SP > DM > CM；SP 下提供 AES 密钥的 HUK（bank1）与 SSK（bank2）不可访问、仅安全引擎可用，RMA 下这两个 bank 对 CPU 与安全引擎均不可访问，且 JTAG 无视禁用位重新使能、启动只校验镜像头（不做解密与验签、跳过防回滚）、镜像头校验失败进下载模式而非 panic；无状态查询与状态转换接口，详见[安全方案架构白皮书](white_paper.md) §11 |
| 安全时间源 | `spacemit.k3.v1.0` | ✓ | 取自 SoC 的 CLINT/ACLINT 机器定时器（MTIMER），基址 `0xe081_c000`（`mtime` 在 `+0xbff8`）；频率 24 MHz，与设备树 `timebase-frequency` 一致。安全世界在 S 态经 `rdtime` CSR 读取 |
| TEE 核心日志等级 | `spacemit.k3.v1.0` | 部分支持 | 当前为最详细等级；量产建议降级 |

### 其它安全构建块（不在范围）

规范中的下列主题 K3 当前未采用，登记为不在范围，供方案评审时对照：

| 主题 | K3（当前基线） | 备注 |
|---|---|---|
| 控制流完整性（CFI） | 不在范围 | 规范要求以影子栈与标记跳转目标防护；本方案未采用 |
| 软件内存标记（MTE） | 不在范围 | 同上 |
| 架构元数据存储 | 不在范围 | 同上 |
| 能力式架构（CHERI） | 不在范围 | 同上 |
| 机密计算（CoVE） | 不在范围 | K3 采用静态分区 TEE 模型，不是 CoVE 模型 |
| 后量子密码就绪（PQC） | 不在范围 | 规范的建议性要求；本方案未评估 |

## 存储介质差异

同一特性在不同存储介质上行为不同，单独列表以避免正文混写。

| 维度 | eMMC | UFS | SPI-NOR | NAND |
|---|---|---|---|---|
| 可运行安全镜像 | ✓ | ✓ | ✓（8M 及以上；4M 放不下安全 U-Boot） | ✓（8M 及以上） |
| 安全镜像 TEE 分区 | 4 MB | 同 eMMC | 512 KB | 512 KB |
| RPMB 可用性 | ✓ | ✓ | ✗ | ✗ |
| RPMB key 派生的器件标识处理 | 用原生 CID（16 字节），掩掉可变字段 PRV 与 CRC | 无原生 CID：device_id 经 BLAKE2s 哈希为 16 字节 | — | — |
| RPMB 读块数语义 | 帧内块数字段须为 0 | 由设备从帧内字段取长度 | — | — |
| TEE 镜像缺失时 SPL 行为 | 中止启动 | 保持宽松 | 中止启动 | 中止启动 |
| 已知限制 | — | — | 4M 容量放不下安全版 U-Boot（超出分区余量） | 同 SPI-NOR |

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-09-18 | 首版，按「特性 × SoC 型号 × 软件版本」给出支持状态与最低版本要求。状态取「支持 / 部分支持 / 暂不支持 / 规划中 / 不在范围」五种，未定论项一律按「暂不支持」处理；「最低软件版本」列统一填 `spacemit.k3.v1.0`，该版本号及本文其他版本号均为占位符，安全版本正式发布后统一替换；版本说明与变更记录均为单条，后续改动直接并入本行。 |
