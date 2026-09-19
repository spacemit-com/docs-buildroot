---
sidebar_position: 7
---

# TEE 示例集

## 概述

### 编写目的

本文汇总 K3 平台随镜像提供的 TEE 示例与测试套件的运行方法、预期输出与结果判读方式，作为功能验证与二次开发的起点。

### 适用范围

本文适用于 SpacemiT K3 系列 SoC 的 Buildroot 方案，运行环境为已启动的安全镜像。镜像的构建与烧录参见 [TEE 镜像构建与烧录说明](tee_image.md)；CA 的编写方法参见 [CA 开发指南 / TEE Client API 参考](ca_guide.md)。

### 文档结构

1. **前置条件**。
2. **官方示例**：七个随镜像提供的示例。
3. **新增一个测试用例**：把自有用例加进 optee-examples 并随镜像安装。
4. **测试套件 xtest**：套件分组与结果判读。
5. **PKCS#11 示例**。
6. **结果判读与常见现象**。

## 前置条件

示例运行在已启动的安全镜像上，需先确认 TEE 环境就绪：

```bash
# TEE 设备节点存在
ls /dev/tee*

# 后台服务已运行（由初始化脚本自动拉起）
ps | grep tee-supplicant

# 查看已安装的 TA
ls /lib/optee_armtz/
```

若 `tee-supplicant` 未运行，涉及安全存储的示例会失败。

## 官方示例

镜像中提供以下七个示例：

| 示例 | 演示内容 |
|---|---|
| `optee_example_hello_world` | 最基本的 CA 与 TA 交互 |
| `optee_example_aes` | 在 TA 中做 AES 加解密 |
| `optee_example_random` | 由 TA 生成随机数 |
| `optee_example_secure_storage` | TA 侧安全存储对象的创建、读取与删除 |
| `optee_example_acipher` | TA 中的非对称加密 |
| `optee_example_hotp` | 基于共享密钥的 HOTP 计算 |
| `optee_example_plugins` | 插件机制 |

### hello world

```bash
optee_example_hello_world
```

CA 侧输出：

```text
Invoking TA to increment 0
TA incremented value to 1
```

同时可在串口上看到来自 TA 的日志：

```text
I/TA: Hello World!
```

两种输出的来源不同：`Invoking` 与 `TA incremented` 是 CA 打印到标准输出，`Hello World!` 是 TA 通过 TEE 日志打印。若只看到前者，说明交互正常但 TEE 日志等级较低。

### AES 加解密

```bash
optee_example_aes
```

输出依次为会话建立、编码、加载密钥、复位向量、缓冲区编码，随后是解码流程，最后给出明文与解码结果的比对：

```text
Clear text and decoded text match
```

出现 `Clear text and decoded text differ => ERROR` 表示加解密结果不一致，属异常。

### 随机数

```bash
optee_example_random
```

```text
Invoking TA to generate random UUID... 
TA generated UUID value = 0x...
```

每次运行输出的 UUID 应不同。若多次运行结果一致，需检查随机数源配置。

### 安全存储

```bash
optee_example_secure_storage
```

输出为两次对象测试：第一次创建、读回并删除对象；第二次先检查对象是否存在，随后按存在与否分别处理。

```text
Test on object "myobj.1"
- Create and load object in the TA secure storage
- Read back the object
- Delete the object
```

判读要点：

- `- Object not found in TA secure storage, create it.` 与 `- Object found in TA secure storage, delete it.` 是第二次测试的正常分支，取决于上一次运行是否留下了对象。
- 若出现 `Command WRITE_RAW failed` 或 `Command READ_RAW failed`，属异常，通常指向存储后端不可用。

### 非对称加密

```bash
optee_example_acipher <key_size> <string to encrypt>
```

未带参数时会打印用法。参数为密钥位数与待加密字符串，成功时输出加密后的字节序列：

```text
Encrypted buffer: xx xx xx ...
```

### HOTP

```bash
optee_example_hotp
```

输出为注册的共享密钥与计算得到的 HOTP 值：

```text
Register the shared key: xx xx ...
HOTP: <数值>
```

若出现 `Got unexpected HOTP from TEE!`，表示计算结果与预期不符，属异常。

### 插件

```bash
optee_example_plugins
```

用于演示插件机制，需要普通世界侧的日志与插件支持。若环境中缺少相关组件，示例可能无法完成，属预期行为，不作为 TEE 环境故障判据。

## 新增一个测试用例

在自己的 optee-examples 源码树里加一个用例（TA、CA，或两者），**不需要改动 Buildroot 的包定义**：该包已把「编译 TA → 签名 → 安装到 `/lib/optee_armtz/`（权限 444）」整条链路接好，并随镜像一起打包。

### 目录与文件

以 `hello_world` 为模板新建 `<my_example>/`：

| 文件 | 必需 | 作用 |
|---|---|---|
| `ta/Makefile` | 是 | TA 的构建入口。TA 由 Buildroot 的钩子按 `*/ta/Makefile` 通配查找，缺少它，TA 既不会编译也不会安装 |
| `ta/sub.mk` | 是 | TA 的源码与头文件清单（`global-incdirs-y`、`srcs-y`） |
| `ta/<my_example>_ta.c` | 是 | TA 实现 |
| `ta/include/<my_example>_ta.h` | 是 | UUID 与命令 ID，CA 与 TA 共用 |
| `ta/user_ta_header_defines.h` | 是 | 属性字符串与 TA 的 UUID（文件名不可改） |
| `CMakeLists.txt` | 有 CA 时 | 定义 CA 可执行文件：`add_executable`、`target_include_directories(... ta/include)`、`target_link_libraries(... teec)`、`install(TARGETS ... DESTINATION ${CMAKE_INSTALL_BINDIR})` |
| `host/main.c` | 有 CA 时 | CA 源码 |
| 根 `Makefile` | 否 | 脱离 Buildroot 单独编译时使用（照着 `hello_world/Makefile` 抄：递归调用 `host/` 与 `ta/`） |

### 从 hello_world 改造：要改哪些地方

最直接的入门方式是把 `hello_world/` 整个目录复制成自己的用例再改。**必改三处**：目录与文件名、UUID、命令 ID。

| 文件 | 要改的内容 |
|---|---|
| 目录名 `hello_world/` | 改成自己的用例名（如 `my_demo/`）。仅作标识，不影响构建 |
| `ta/Makefile` | `BINARY=8aaaf200-2450-11e4-abe2-0002a5d5c51b` 换成新 TA 的 UUID（生成方式见 [TA 开发指南](ta_guide.md) 的「生成 UUID」） |
| `ta/include/hello_world_ta.h` | 改名为 `<my_demo>_ta.h`；`TA_HELLO_WORLD_UUID` 换成新 UUID；`TA_HELLO_WORLD_CMD_*` 换成自己的命令 ID（编号自定，CA 与 TA 两边一致即可） |
| `ta/user_ta_header_defines.h` | 文件名**不能改**；`#include <hello_world_ta.h>` 换成自己的头；`TA_UUID` 保持为对共享头里 UUID 宏的引用；`TA_VERSION`、`TA_DESCRIPTION`、`TA_CURRENT_TA_EXT_PROPERTIES` 里的示例串与属性名换成自己的；`TA_FLAGS`、`TA_STACK_SIZE`、`TA_DATA_SIZE` 按需调整（见 [TA 开发指南](ta_guide.md) 的「设置属性与标志」） |
| `ta/sub.mk` | `srcs-y += hello_world_ta.c` 换成自己的源文件名（多个源文件写多行）；头文件不在 `include/` 下时同步 `global-incdirs-y` |
| `ta/hello_world_ta.c` | 改名为 `<my_demo>_ta.c`；`#include <hello_world_ta.h>` 换成自己的头；五个入口点函数（`TA_CreateEntryPoint`、`TA_DestroyEntryPoint`、`TA_OpenSessionEntryPoint`、`TA_CloseSessionEntryPoint`、`TA_InvokeCommandEntryPoint`）保留；`TA_InvokeCommandEntryPoint()` 里的 `switch (cmd_id)` 按自己的命令 ID 重写（示例中是两个静态函数 `inc_value`、`dec_value`） |
| `CMakeLists.txt` | 仅当同时提供 CA：`project (optee_example_hello_world C)` 换成自己的名字，该名字就是安装到 `/usr/bin/` 的可执行文件名；`set (SRC host/main.c)` 按需增删源文件 |
| `host/main.c` | 仅当同时提供 CA：`#include <hello_world_ta.h>` 换成自己的头；`TEEC_UUID uuid = TA_HELLO_WORLD_UUID;` 换成同一 UUID；`TEEC_InvokeCommand()` 的命令 ID 换成自己的；`paramTypes` 与 `params` 与 TA 侧逐项对应 |
| 根 `Makefile` | 无需改动（只递归调用 `host/` 与 `ta/`） |

UUID 只写在两处，且必须相同：

```make
# ta/Makefile —— 构建时用它命名并签名产物
BINARY=<新 UUID>
```

```c
/* ta/include/<my_demo>_ta.h —— CA 与 TA 共用 */
#define TA_MY_DEMO_UUID { 0x00112233, 0x4455, 0x6677, { 0x88, 0x99, 0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff } }
```

`ta/user_ta_header_defines.h` 里的 `#define TA_UUID <新 UUID 宏>` 与 CA 里的 `TEEC_UUID uuid = <新 UUID 宏>;` 都引用同一个宏，因此只需改头文件一处。命名习惯上，UUID 宏用 `TA_<用例名大写>_UUID`、命令 ID 用 `TA_<用例名大写>_CMD_<动作>`，宏名与文件名保持一致便于对照。

### 为什么不用改 Buildroot

- **CA 侧**：optee-examples 顶层 `CMakeLists.txt` 用 `file(GLOB dirs *)` 遍历子目录，凡是含 `CMakeLists.txt` 的子目录都会被执行 `add_subdirectory`，新目录自动纳入构建。
- **TA 侧**：`optee-examples.mk` 的构建钩子用 `$(wildcard $(@D)/*/ta/Makefile)` 收集所有 TA 的 Makefile，安装钩子把 `*/ta/out/*.ta` 安装到 `/lib/optee_armtz`（权限 444）。两者都是通配，无需登记新用例。

因此只要文件齐备，`make optee-examples-rebuild` 就会把新用例一起编出来。

### 让镜像包含新用例

optee-examples 默认从上游拉取源码，要带上自己的用例，把源码树放到本地并覆盖包的源码目录：

1. 把 optee-examples 的源码复制到本地（例如 `package-src/optee-examples`），在其中加入新用例目录；
2. 在 `buildroot-ext/local.mk` 里加一行（该文件对 uboot、opensbi、optee_os 等已有同样写法）：

```make
OPTEE_EXAMPLES_OVERRIDE_SRCDIR = $(TOPDIR)/../package-src/optee-examples
```

3. `make optee-examples-rebuild` 编译该包，随后 `make` 让镜像重新打包。

产物落点与确认：CA → `/usr/bin/<PROJECT_NAME>`（`PROJECT_NAME` 取自用例的 `CMakeLists.txt`），TA → `/lib/optee_armtz/<uuid>.ta`（权限 444）。

```bash
ls output/k3/target/lib/optee_armtz/*.ta        # TA 已安装
ls output/k3/target/usr/bin/optee_example_*     # CA（若同时提供）
```

设备侧再确认三项：文件就位且权限 444、`tee-supplicant` 在运行、用同 UUID 的 CA 调用得到预期结果（见 [TA 开发指南](ta_guide.md) 的验证小节）。需要出现在 initramfs 中时，**必须把 TA 显式加入 initramfs 白名单**：TA 不是标准可执行文件，依赖分析无法自动识别，只放进根文件系统会导致早期阶段找不到 TA。

## 测试套件 xtest

```bash
xtest
```

不带参数时运行全部套件。按套件分组运行：

```bash
xtest -t regression_6000
```

套件分组：

| 套件 | 内容 |
|---|---|
| `regression_1000` | 核心自检与 PTA 参数 |
| `regression_2000` | TCP socket API 测试 |
| `regression_4000` | TEE Internal API 摘要运算 |
| `regression_4100` | TEE Internal API 大整数运算 |
| `regression_5000` | GlobalPlatform TEEC 接口 |
| `regression_6000` | 安全存储（对象创建） |
| `regression_8000` | TEE Internal API 密钥派生扩展 |
| `regression_8100` | TA 侧 mbedTLS 自检与证书链 |

结果判读：

- `xtest` 正常结束时，未通过的用例数应为 0。
- 单个用例的执行结果分为 OK、SKIP 与 FAIL 三类。**SKIP 不视为失败**，通常表示该用例依赖的功能在本次构建中未使能。
- 涉及 socket 的套件在缺少网络配置时可能整体跳过，属预期。

查看某个套件内的用例清单：

```bash
xtest -l 2>&1 | head
```

## PKCS#11 示例

TEE 侧还提供了 PKCS#11 接口，可用标准工具枚举：

```bash
pkcs11-tool --module /usr/lib/pkcs11/opensc-pkcs11.so -L
```

`-L` 列出可用的槽位与令牌。若需要注册到系统，可使用随镜像提供的注册工具。

## 结果判读与常见现象

| 现象 | 判读 |
|---|---|
| 示例提示找不到 TA | TA 未随本次构建打包，或 TA 存放目录内容缺失 |
| 安全存储示例报存储不可用 | `tee-supplicant` 未运行，或存储后端不可用 |
| `xtest` 中大量用例 SKIP | 相关功能未在本次构建使能，属构建配置问题而非故障 |
| 示例可运行但串口无 TEE 日志 | TEE 日志等级较低，不影响功能 |
| 随机数示例每次结果相同 | 随机数源配置异常，需检查平台随机数驱动 |

