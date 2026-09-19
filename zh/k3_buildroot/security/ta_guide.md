---
sidebar_position: 5
---

# TA 开发指南

## 概述

### 编写目的

本文是 K3 平台上开发 TA（Trusted Application，可信应用）的操作手册：从准备环境、建立目录骨架、实现入口点，到构建、部署与验证，按步骤给出可照做的流程。TA 的分类与运行模型、可用 API、参数对应等作为背景与参考放在流程之后。

### 适用范围

本文适用于 SpacemiT K3 系列 SoC 的 Buildroot 方案。

- **构建环境**：交叉编译——在开发主机上编译，产物在 RISC-V 目标设备上运行。
- **相关文档**：CA 侧见 [CA 开发指南 / TEE Client API 参考](ca_guide.md)；可直接运行的例子见 [TEE 示例集](examples.md)；结构基础见 [TEE 架构与集成指南](tee_architecture.md)。

### 文档结构

1. **TA 分类与运行模型**：三类 TA、目标架构与实例/会话模型。
2. **可用 API 概览**：TA 能调用的能力范围。
3. **与 CA 的参数对应**。
4. **安全存储的使用**。
5. **快速上手**：五步流程总览。
6. **步骤 1–5**：准备开发环境 → 建立目录骨架 → 实现入口点 → 设置属性 → 构建、部署与验证。
7. **完整示例**、**调试与常见问题**。

### 重要约束

> [!CAUTION]
> TA 运行在 RISC-V 64 位目标上，且**浮点支持已关闭**。TA 代码中不应使用浮点运算；确需浮点时应重新评估安全与性能取舍，而不是直接打开开关。
>
> 另外，**TA 的签名密钥必须与 CA 侧无关、且需在量产前替换**。默认签名密钥随 SDK 提供，任何拿到该密钥的人都能签出被本平台接受的 TA。

## TA 分类与运行模型

### 三类 TA

| 类型 | 说明 | 加载方式 | 典型用途 |
|---|---|---|---|
| User TA | 常规可信应用，以文件形式存在，运行时由 TEE 加载 | 由普通世界文件系统提供，TEE 装载 | 业务逻辑、密钥运算、安全存储 |
| Early TA | 编译进 TEE 内核镜像，随 TEE 一同启动 | 内置于 `tee.bin`，启动即可用 | 需要在 TEE 初始化阶段就可用的基础服务 |
| PTA（Pseudo TA） | 不属于普通 TA，是 TEE 内核内部的伪 TA | 与 TEE 内核同体 | 向普通世界暴露平台能力（如设备枚举、RTC） |

K3 当前内置的 early TA 为 **Trusted Keys**（UUID `f04a0fe7-1f5d-4b9b-abf7-619b85b4ce8c`）。平台向普通世界暴露的 PTA 接口头文件随 dev kit 提供，包括设备枚举、RTC 等。**本指南的步骤针对 User TA**。

### 目标架构

| 项 | 值 |
|---|---|
| TA 目标 | RISC-V 64 位（`ta_rv64`） |
| 浮点 | 关闭 |
| 并发 | 支持多核，TA 需自行处理并发访问 |

### 实例与会话模型

- TA 由普通世界通过会话（session）访问。
- 默认情况下，**同一个 TA 的多个会话共享同一个实例**，TA 需自行保证实例内数据的并发安全。
- 若需要每个会话独立实例，需使用对应的属性；若需要实例在最后一个会话关闭后仍保留，另有对应标志。二者见「步骤 4」的属性表。
- TA 之间默认不共享内存，跨 TA 协作需经 TEE 提供的通道。

## 可用 API 概览

TA 通过 `libutee` 提供的 Internal Core API 访问 TEE 能力。

- **接口来源**：**GlobalPlatform（GP）** 定义的 TEE Internal Core API 规范，K3 上由 libutee 实现。
- **CA 侧对应**：GP 的 TEE Client API，见 [CA 开发指南 / TEE Client API 参考](ca_guide.md)。
- **两套规范的对应关系**：见 [TEE 架构与集成指南](tee_architecture.md) 的 GP API 小节。
- **本质**：调用即对 TEE 的系统调用。

| 能力类别 | 主要内容 |
|---|---|
| 密码学 | 摘要、对称加解密、非对称运算、随机数、密钥派生与生成 |
| 安全存储 | 持久对象与临时对象的创建、读写、查找、删除、重命名 |
| 时间 | 获取系统时间与等待 |
| 大整数运算 | 大整数的初始化、比较与算术 |
| 内存 | 各类分配与释放、共享内存的注册与访问 |
| 属性与会话 | 读取自身属性、会话上下文管理 |

K3 平台的密码学能力由安全引擎承载：非对称运算支持 RSA、ECDSA 与 SM2DSA，对称运算支持 AES 与 SM4。**摘要运算使用软件实现**（默认不使用安全引擎摘要），因此 TA 侧无需为选择软件/硬件而做特殊处理。

平台的 PTA 接口头文件（如设备枚举、RTC）也在 dev kit 的 `include/` 中提供，供需要访问平台能力的 TA 使用。

## 与 CA 的参数对应

TA 与 CA 之间的参数传递有四种形式，必须两侧一致：

| CA 侧类型 | TA 侧读取方式 | 说明 |
|---|---|---|
| `TEEC_VALUE_*` | `TEE_Param` 的 `value.a` / `value.b` | 两个 32 位整数，适合小数据 |
| `TEEC_MEMREF_TEMP_*` | `TEE_Param` 的 `memref.buffer` / `memref.size` | 临时内存引用，仅本次调用有效 |
| `TEEC_MEMREF_WHOLE` | 同上，指向整个共享内存块 | 已注册的共享内存 |
| `TEEC_MEMREF_PARTIAL_*` | 同上，按偏移与长度访问 | 已注册共享内存的一部分 |

对应关系由 `paramTypes` 描述，**类型不匹配不会在编译期报错**，而是在运行期返回参数错误。TA 侧应在入口处校验参数类型与尺寸，并在不匹配时返回明确错误，而不是继续执行。

## 安全存储的使用

TA 通过安全存储 API 读写持久对象。数据实际落地位置由 TEE 的存储后端决定：

| 后端 | 落地位置 | 抗回滚 |
|---|---|---|
| 普通世界文件系统（默认） | Linux 根文件系统，由 `tee-supplicant` 代管 | 无 |
| 存储器件 RPMB 分区 | 器件的 RPMB 分区 | 有（需器件完成密钥灌注） |

对 TA 而言两者接口一致。需要注意：

- 默认后端下，**`tee-supplicant` 未运行会导致存储操作失败**。
- 每个 TA 拥有独立密钥（由 TEE 的密钥层级派生），不同 TA 的存储数据互不可见。
- 默认后端不提供抗回滚保护，TA 不应假设对象不会被回滚或删除。

更完整的存储分层与密钥层级见 [TEE 架构与集成指南](tee_architecture.md) 的安全存储章节。

## 快速上手

TA 的完整开发流程是五步，每步的细节见后文对应章节：

| 步骤 | 做什么 | 产出 |
|---|---|---|
| 1 | 拿到 TA dev kit 与交叉工具链 | 可用的 `TA_DEV_KIT_DIR` 与 `riscv64-unknown-linux-gnu-` 工具链 |
| 2 | 建立目录骨架：`Makefile`、`sub.mk`、`user_ta_header_defines.h`、源文件、`include/` | 可构建的 TA 目录 |
| 3 | 实现五个入口点，在 `TA_InvokeCommandEntryPoint` 中按命令 ID 分发 | 入口点实现 |
| 4 | 在属性头文件中填 UUID、标志位与栈 / 堆大小 | 确定的 TA 属性 |
| 5 | 构建、安装到设备、验证调用 | `<uuid>.ta` 与验证结果 |

先做对第 2、4 步（目录骨架与属性），再改第 3 步的代码，可以避免大多数「装不上 / 打不开会话」类问题。

## 步骤 1：准备开发环境

### 1.1 TA dev kit

TA 的构建依赖 TEE 侧导出的开发包（dev kit），其中包含头文件、静态库、链接脚本与构建规则：

| 目录/文件 | 内容 |
|---|---|
| `include/` | TA 侧头文件：`tee_api.h`、`tee_internal_api.h`、`user_ta_header.h`、`utee_defines.h`、`utee_types.h`、`pta_*.h` 等 |
| `lib/` | `libutee.a`（Internal Core API 实现）、`libutils.a`、`libmbedtls.a`、`libdl.a` |
| `mk/ta_dev_kit.mk` | TA 的构建规则入口 |
| `src/` | `ta.ld.S`（TA 链接脚本）、`user_ta_header.c` |
| `keys/default_ta.pem` | 默认 TA 签名密钥 |
| `scripts/` | 签名与辅助脚本 |
| `host_include/`、`ta/` | 宿主侧与 TA 侧补充头文件 |

在 Buildroot 构建树中，dev kit 由 TEE 的构建输出导出，位置为：

```text
output/k3/build/optee-os-custom/out/export-ta_rv64/
```

目录名中的 `optee-os-custom` 来自 Buildroot 为自定义版本号的 optee-os 包分配的构建目录。上表列出的 `include/`、`lib/`、`mk/`、`keys/default_ta.pem` 均在该目录下；构建时把 `TA_DEV_KIT_DIR` 指向它。

### 1.2 交叉工具链

TA 在开发主机上编译、在目标设备上运行，属于交叉编译：

| 项 | 值 |
|---|---|
| 编译主机 | 开发所用的 x86_64 主机 |
| 目标 | RISC-V 64 位（`ta_rv64`），与设备一致 |
| 工具链前缀 | `riscv64-unknown-linux-gnu-` |
| 工具链位置（宿主） | `output/k3/host/opt/ext-toolchain/bin/`：工具链本体，可脱离构建容器运行 |
| 工具链位置（容器内） | `output/k3/host/bin/`（容器内为 `/k3/host/bin/`）：Buildroot 生成的 wrapper，需要容器环境的 glibc |

`CROSS_COMPILE` 要写成**完整路径前缀**，或先把工具链的 `bin` 目录加入 `PATH`。只写 `riscv64-unknown-linux-gnu-` 而 `PATH` 中没有该工具链时，会报 `riscv64-unknown-linux-gnu-gcc: not found`（make 报 Error 127）。TA 侧的构建只用 dev kit 里的库，不需要额外的 `--sysroot`。

## 步骤 2：建立 TA 目录骨架

一个可构建的 TA 至少要提供四类文件：构建入口（`Makefile`）、源文件清单（`sub.mk`）、TA 属性（`user_ta_header_defines.h`）与入口点实现（C 源文件）。推荐目录结构如下（与 OP-TEE 官方约定一致）：

```text
my_ta/
├── Makefile                    # 声明 BINARY=<uuid>，包含 dev kit 的构建规则
├── sub.mk                      # 列出参与构建的源文件与头文件目录
├── user_ta_header_defines.h    # TA_UUID、TA_FLAGS、TA_STACK_SIZE、TA_DATA_SIZE 等
├── include/
│   └── my_ta.h                 # 对普通世界公开的命令 ID 与 UUID 定义（CA 侧共用）
└── my_ta.c                     # 五个入口点的实现
```

各文件职责：

| 文件 | 是否必需 | 职责 |
|---|---|---|
| `Makefile` | 必需 | 设置配置变量并包含 `$(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk` |
| `sub.mk` | 必需 | `srcs-y` 列出源文件，`global-incdirs-y` 列出头文件目录 |
| `user_ta_header_defines.h` | 必需 | 定义 TA 的标志位、栈与堆大小等属性 |
| 入口点实现（`.c`） | 必需 | 实现五个强制入口点 |
| `include/` | 建议 | 放 CA 与 TA 共用的 UUID、命令 ID 定义，避免两侧漂移 |
| `Android.mk` | 不需要 | Android 构建入口；Buildroot 方案不使用 |

### 2.1 Makefile

```make
# TA 的 UUID，决定产物文件名 <uuid>.ta
BINARY = <uuid>

# TA 侧日志等级（调试期可提高，量产建议降低）
CFG_TEE_TA_LOG_LEVEL ?= 4
CFG_TA_OPTEE_CORE_API_COMPAT_1_1 = y

include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk
```

`BINARY` 必须填 TA 的 UUID（不能换成别的名字）；若构建的是供 TA 链接的静态库，则改用 `LIBNAME`——`BINARY` 与 `LIBNAME` 互斥，不能同时使用。

### 2.2 sub.mk

```make
# 参与构建的源文件（相对当前目录）
srcs-y += my_ta.c

# 头文件搜索路径
global-incdirs-y += include/
```

### 2.3 user_ta_header_defines.h

```c
#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

/* UUID 与命令 ID 定义放在共用头文件中，随 sub.mk 的 global-incdirs-y 引入 */
#include <my_ta.h>

/* 属性标志：按需组合，含义见「步骤 4」 */
#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)

/* 栈与堆（TEE_Malloc 的内存池）大小 */
#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)

#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "K3 示例 TA"

#endif
```

### 2.4 生成 UUID

```bash
python -c 'import uuid; print(uuid.uuid4())'
```

UUID 一旦发布就不可更改：已安装的 CA 以 UUID 定位 TA，改动会使 CA 找不到目标 TA。

## 步骤 3：实现入口点

TA 的五个强制入口点（声明见 dev kit 的 `include/tee_internal_api.h`）：

| 入口点 | 时机 | 说明 |
|---|---|---|
| `TA_CreateEntryPoint` | TA 实例被创建时 | 分配实例级资源；失败则实例不可用 |
| `TA_DestroyEntryPoint` | TA 实例被销毁时 | 释放实例级资源 |
| `TA_OpenSessionEntryPoint` | 每次打开会话 | 校验参数、准备会话上下文 |
| `TA_CloseSessionEntryPoint` | 每次关闭会话 | 释放会话上下文 |
| `TA_InvokeCommandEntryPoint` | 每次命令调用 | **核心逻辑**：按命令 ID 分发处理 |

实现文件先包含属性头文件，再逐个实现入口点：

```c
#include <tee_internal_api.h>
#include <my_ta.h>                     /* UUID 与命令 ID */

TEE_Result TA_CreateEntryPoint(void)
{
	return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void)
{
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
				    TEE_Param params[4],
				    void **sess_ctx)
{
	/* 校验参数类型，准备会话上下文 */
	*sess_ctx = NULL;
	return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *sess_ctx)
{
}

TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
				      uint32_t cmd_id,
				      uint32_t param_types,
				      TEE_Param params[4])
{
	/* 按 cmd_id 分发；参数类型必须与 CA 侧一致 */
	return TEE_ERROR_NOT_IMPLEMENTED;
}
```

其中 `TA_InvokeCommandEntryPoint` 接收会话上下文、命令 ID 与参数，参数的读写方式由 `paramTypes` 决定。**命令 ID 与参数类型必须在 CA 与 TA 两侧完全一致**，对应关系见「与 CA 的参数对应」。

## 步骤 4：设置属性与标志

TA 的属性通过标志位与属性字符串声明，写在 `user_ta_header_defines.h` 与共用头文件中。

### 标志位

| 标志 | 含义 |
|---|---|
| `TA_FLAG_SINGLE_INSTANCE` | 同一 TA 仅一个实例，多个会话共享 |
| `TA_FLAG_MULTI_SESSION` | 支持多会话；与单实例组合使用 |
| `TA_FLAG_INSTANCE_KEEP_ALIVE` | 最后一个会话关闭后实例仍保留 |
| `TA_FLAG_CACHE_MAINTENANCE` | 允许使用缓存维护系统调用（本平台已使能该能力） |
| `TA_FLAG_CONCURRENT` | 允许多个命令并发执行，TA 需自行保证线程安全 |
| `TA_FLAG_DEVICE_ENUM` | 允许在无后台服务的情况下枚举设备 |
| `TA_FLAG_DEVICE_ENUM_SUPP` | 允许在有后台服务的情况下枚举设备 |
| `TA_FLAG_SECURE_DATA_PATH` | 使用安全数据路径内存 |

未使用并发标志时，TEE 会串行化该 TA 的命令调用，TA 内部无需额外加锁；使用后必须自行处理并发。

上表列出的是 `TA_FLAGS` 中的各个位：

- 在 `user_ta_header_defines.h` 中把它们按位或起来赋值给 `TA_FLAGS`，未列出的位保持为 0。
- 另有若干已废弃的标志（`TA_FLAG_USER_MODE`、`TA_FLAG_EXEC_DDR`、`TA_FLAG_REMAP_SUPPORT`）恒为 0，不要使用。

### 属性字符串与对应宏

属性字符串（`gpd.ta.*`）由 dev kit 在编译期根据 `user_ta_header_defines.h` 中的宏自动生成，**TA 代码里不需要手写这些字符串**。两者的对应关系：

| 属性字符串 | 含义 | 在 `user_ta_header_defines.h` 中的写法 |
|---|---|---|
| `gpd.ta.singleInstance` | 单实例 | `TA_FLAGS` 中置 `TA_FLAG_SINGLE_INSTANCE` |
| `gpd.ta.multiSession` | 多会话 | `TA_FLAGS` 中置 `TA_FLAG_MULTI_SESSION` |
| `gpd.ta.instanceKeepAlive` | 实例保活 | `TA_FLAGS` 中置 `TA_FLAG_INSTANCE_KEEP_ALIVE` |
| `gpd.ta.dataSize` | 数据段（堆）大小 | `#define TA_DATA_SIZE (32 * 1024)` |
| `gpd.ta.stackSize` | 栈大小 | `#define TA_STACK_SIZE (2 * 1024)` |
| `gpd.ta.version` | 版本标识 | `#define TA_VERSION "1.0"` |
| `gpd.ta.description` | 描述信息 | `#define TA_DESCRIPTION "…"` |
| `gpd.ta.endian` | 字节序 | 由 dev kit 生成（固定小端），无需设置 |

写法示例（`user_ta_header_defines.h` 片段）：

```c
/* 前三条属性来自 TA_FLAGS：按位或组合，未列出的位即不启用 */
#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)

#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)
#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "K3 示例 TA"
```

两点说明：

- `TA_VERSION` 与 `TA_DESCRIPTION` 未定义时，dev kit 会填入默认值（`"Undefined version"`、`"Undefined description"`）；这两个属性在排查「设备上跑的是哪个 TA」时很有用，建议填写。
- 需要额外属性时用 `TA_CURRENT_TA_EXT_PROPERTIES` 追加；dev kit 已提供的标志也能带出对应属性，例如 `gpd.ta.doesNotCloseHandleOnCorruptObject` 对应 `TA_FLAG_DONT_CLOSE_HANDLE_ON_CORRUPT_OBJECT`。

## 步骤 5：构建、部署与验证

### 5.1 构建

对 TA 目录执行 `make`，指定交叉工具链与 dev kit 路径：

```bash
BR=<Buildroot 输出目录>                              # 容器内为 /k3
KIT=$BR/build/optee-os-custom/out/export-ta_rv64

# 宿主用工具链本体；容器内改用 $BR/host/bin/riscv64-unknown-linux-gnu-
CROSS=$BR/host/opt/ext-toolchain/bin/riscv64-unknown-linux-gnu-

rm -rf out    # 执行前先清掉上次的中间文件，见下方说明
make CROSS_COMPILE=$CROSS TA_DEV_KIT_DIR=$KIT O=out -C <TA 目录> all
```

产物为 `<TA 目录>/out/<uuid>.ta`，已由构建过程签名。

**每次编译前先执行 `rm -rf out`。** 上一次若是在 Buildroot 构建目录或容器里编的，`out/` 下的依赖文件 `*.d` 会记录容器内绝对路径（容器构建用的是 staging 里的 dev kit，因此形如 `/k3/host/.../sysroot/lib/optee/export-ta_rv64/src/ta.ld.S`）；在宿主上重编时 `make` 会读入这些 `.d`，报 `No rule to make target '/k3/...'`。清掉 `out/` 即可正常编译。

### 5.2 部署

TA 的安装位置与权限（目录的来源见 [TEE 镜像构建与烧录说明](tee_image.md) 的「编译产物与在系统中的位置」）：

```text
/lib/optee_armtz/<uuid>.ta      # 权限 444（只读）
```

**推荐做法：把 TA 加进 Buildroot 的 optee-examples 包，随镜像一起安装** —— 目录要求、改造要点与接入步骤见 [官方示例与测试](examples.md) 的「新增一个测试用例」。

该包在构建时通配编译源码树中每个子目录下的 `ta/`，并把产物 `*/ta/out/*.ta` 安装到镜像的 `/lib/optee_armtz/`（权限 444），因此不需要自己写安装规则。

其余方式：

| 方式 | 做法 | 适用场景 |
|---|---|---|
| 随镜像构建（推荐） | 加进 optee-examples 包（如上），或在自己的包里加安装步骤 | 产品集成 |
| 运行期替换 | 在设备上直接覆盖 `/lib/optee_armtz/` 下的文件 | 开发调试 |

> [!CAUTION]
> 若 TA 需要出现在 initramfs 中，**必须显式加入 initramfs 的白名单**。TA 文件不是标准可执行文件，依赖分析无法自动识别，只放进根文件系统会导致早期阶段找不到 TA。

### 5.3 验证

按顺序确认三项：

1. **文件就位**：`ls -l /lib/optee_armtz/<uuid>.ta`，确认存在且权限为 `444`。
2. **后台服务在运行**：涉及安全存储的调用依赖 `tee-supplicant`。
3. **调用成功**：用同一 UUID 的 CA（见 [CA 开发指南](ca_guide.md) 的最小示例）或 `xtest` 发起调用，确认返回预期结果。

### 5.4 签名密钥

TA 在构建时被签名，TEE 侧使用编译期嵌入的公钥校验 TA。默认使用 SDK 提供的签名密钥：

```text
keys/default_ta.pem
```

量产前**必须替换为自有密钥**。更换密钥后，所有 TA 都需要用新密钥重新签名，并重新构建 TEE，使新的公钥被嵌入内核。

## 完整示例

把「步骤 2」的骨架补齐，即得到最小可运行 TA。以「对传入数值加一」为例。

目录：

```text
hello_ta/
├── Makefile
├── sub.mk
├── user_ta_header_defines.h
├── include/hello_ta.h
└── hello_ta.c
```

`include/hello_ta.h`（CA 与 TA 共用）：

```c
#ifndef HELLO_TA_H
#define HELLO_TA_H

/* TA 的 UUID：同时决定产物文件名与 CA 的定位依据 */
#define TA_HELLO_UUID { 0x8aaaf200, 0x2450, 0x11e4, \
	{ 0xab, 0xe2, 0x00, 0x02, 0xa5, 0xd5, 0xc5, 0x1b } }

/* 命令 ID：CA 与 TA 必须一致 */
#define CMD_INC_VALUE 0

#endif
```

`Makefile`：

```make
BINARY = 8aaaf200-2450-11e4-abe2-0002a5d5c51b
CFG_TEE_TA_LOG_LEVEL ?= 4
CFG_TA_OPTEE_CORE_API_COMPAT_1_1 = y

include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk
```

`sub.mk`：

```make
srcs-y += hello_ta.c
global-incdirs-y += include/
```

`user_ta_header_defines.h`：

```c
#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

#include <hello_ta.h>

#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)
#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)

#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "Hello TA（对传入数值加一）"

#endif
```

`hello_ta.c`：

```c
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>
#include <hello_ta.h>

TEE_Result TA_CreateEntryPoint(void)
{
	return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void)
{
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
				    TEE_Param params[4],
				    void **sess_ctx)
{
	uint32_t exp = TEE_PARAM_TYPES(TEE_PARAM_TYPE_NONE,
				       TEE_PARAM_TYPE_NONE,
				       TEE_PARAM_TYPE_NONE,
				       TEE_PARAM_TYPE_NONE);

	if (param_types != exp)
		return TEE_ERROR_BAD_PARAMETERS;

	*sess_ctx = NULL;
	return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *sess_ctx)
{
}

TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
				      uint32_t cmd_id,
				      uint32_t param_types,
				      TEE_Param params[4])
{
	uint32_t exp = TEE_PARAM_TYPES(TEE_PARAM_TYPE_VALUE_INOUT,
				       TEE_PARAM_TYPE_NONE,
				       TEE_PARAM_TYPE_NONE,
				       TEE_PARAM_TYPE_NONE);

	switch (cmd_id) {
	case CMD_INC_VALUE:
		if (param_types != exp)
			return TEE_ERROR_BAD_PARAMETERS;
		params[0].value.a++;
		return TEE_SUCCESS;
	default:
		return TEE_ERROR_NOT_IMPLEMENTED;
	}
}
```

构建与部署按「步骤 5」执行。对应的 CA 侧代码见 [CA 开发指南](ca_guide.md) 的最小示例，两者的 UUID 与命令 ID 必须一致。

## 调试与常见问题

### 日志

TA 侧通过 `IMSG`、`DMSG`、`EMSG` 等宏输出日志，日志经 TEE 内核打印到串口。TA 的日志等级由构建时的日志等级配置控制；等级偏低时 TA 的日志不会出现，容易被误判为「TA 没有执行」。

### 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 打开会话返回找不到对象 | UUID 不匹配，或 TA 未安装 | 核对两侧 UUID；确认 `/lib/optee_armtz/` 下存在对应 `.ta` |
| 运行期返回参数错误 | 两侧参数类型或尺寸不一致 | 逐项核对 `paramTypes` 与 TA 侧校验逻辑 |
| 返回未实现 | 命令 ID 两侧不一致 | 核对命令 ID 定义 |
| 安全存储操作失败 | `tee-supplicant` 未运行，或后端不可用 | 确认后台服务在运行 |
| TA 日志完全看不到 | TA 日志等级偏低 | 提高日志等级后重新构建 |
| 改动 TA 后行为不变 | 设备上仍是旧 TA | 重新安装 `.ta`；确认安装路径与 UUID 对应 |
| 早期阶段找不到 TA | TA 未进 initramfs 白名单 | 将 TA 显式加入白名单 |
| 构建报找不到 `ta_dev_kit.mk` | `TA_DEV_KIT_DIR` 未指定或路径不对 | 按「步骤 1」确认 dev kit 路径 |
| 构建报 `riscv64-unknown-linux-gnu-gcc: not found`（Error 127） | `CROSS_COMPILE` 只写了短前缀，而工具链不在 `PATH` 中 | 用完整路径前缀（见「步骤 1.2」），或先把工具链 `bin` 目录加入 `PATH` |
| 构建报 `No rule to make target '/k3/.../export-ta_rv64/src/ta.ld.S'` | 在 Buildroot 的构建目录里手工编 TA，`ta/out/` 下残留了容器内生成的 `.d`，其中记录了 staging dev kit 的 `/k3` 路径 | 删除 `ta/out/` 后重新编译；或在自己的目录里编、或交给 `make optee-examples-rebuild` |


## 与其它文档的关系

| 主题 | 文档 |
|---|---|
| CA 开发与 TEE Client API | [CA 开发指南 / TEE Client API 参考](ca_guide.md) |
| 示例与测试套件 | [TEE 示例集](examples.md) |
| TEE 结构与内存约束 | [TEE 架构与集成指南](tee_architecture.md) |
| 特性支持状态 | [安全特性支持矩阵](support_matrix.md) |
