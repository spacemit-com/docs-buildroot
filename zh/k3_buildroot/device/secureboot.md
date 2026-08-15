---
sidebar_position: 4
---

# 安全启动开发指南

## 概述

### 编写目的

本指南介绍 SpacemiT K3 平台安全启动（Secure Boot）的实现机制、密钥体系、启用方法与升级路径，帮助开发者在产品中建立从芯片信任根到 Linux 内核的完整验签链。

### 适用范围

本文档适用于 SpacemiT 的 K3 系列 SOC。

刷机流程、分区布局、普通启动配置等基础内容参见[启动开发指南](boot.md)，本文档只关注安全启动相关部分。

### 相关人员

- 安全启动开发工程师
- 刷机/启动开发工程师
- 产线量产工程师

### 文档结构

1. **安全启动机制**：信任链结构、密钥体系、FIT 签名容器与公钥注入原理。
2. **密钥准备**：密钥与证书生成、自定义密钥目录需准备的文件。
3. **启用安全启动**：各组件编译配置与签名构建流程。
4. **烧录与 eFuse**：ROTPK hash 烧录及不可逆约束。
5. **deb 升级路径签名**：签名内核 deb 的构建与安装。
6. **签名模式的行为差异**：与普通启动的差异点。
7. **验证与排查**：签名产物检查与常见故障定位。

### 重要约束

:::danger

安全启动一旦通过 eFuse 使能即**不可逆**。使能后设备只能运行签名固件，无法回退到非签名启动。请在充分验证签名流程后再烧录 eFuse。

:::

## 安全启动机制

### 信任链总览

K3 安全启动以芯片内固化的 BootROM 为信任根，逐级验签，每一级在跳转前校验下一级镜像的完整性与来源：

```text
BootROM (固化，信任根)
  │  用 eFuse 中的 ROTPK hash 校验 FSBL 证书链
  ▼
FSBL (u-boot-spl + FSBL 头)
  │  用编译期注入 SPL DTB 的 uboot_key 公钥
  │  验证 u-boot.itb / fw_dynamic.itb 的 FIT 签名
  ▼
OpenSBI (fw_dynamic.itb) + U-Boot (u-boot.itb)
  │  U-Boot 用编译期注入自身 control fdt 的
  │  kernel_key 公钥验证 Image.itb 的 FIT 签名
  ▼
Linux Kernel + DTB (Image.itb)
```

各级验签的对应关系：

| 被验对象 | 验签者 | 验签公钥（签名私钥） | 公钥所在位置 |
|---|---|---|---|
| FSBL | BootROM | ROTPK（`root_key_prv`） | eFuse bank 4 存放其 SHA256 |
| `fw_dynamic.itb` | FSBL (SPL) | `uboot_key_pub`（`uboot_key_prv`） | SPL 的 DTB |
| `u-boot.itb` | FSBL (SPL) | `uboot_key_pub`（`uboot_key_prv`） | SPL 的 DTB |
| `Image.itb` | U-Boot | `kernel_key_pub`（`kernel_key_prv`） | U-Boot 的 control fdt |

- 哈希算法：`SHA256`
- 签名算法：`SHA256 + RSA2048`

### 密钥体系

K3 使用分级密钥，各密钥职责独立：

| 密钥 | 作用 | 公钥去向 |
|---|---|---|
| `root_key_prv` | ROTPK，签名 FSBL 的一级证书 cert0 | 公钥 hash 烧入 eFuse |
| `spl_key_prv` | 签名 FSBL（SPL）二进制本身 | 由 cert0 携带 |
| `uboot_key_prv` | 签名 `esos.itb`, `u-boot.itb` 与 `fw_dynamic.itb` | 编译期注入 SPL 的 DTB |
| `kernel_key_prv` | 签名 `Image.itb`（内核 + DTB） | 编译期注入 U-Boot 的**每个板级 DTB** |
| `rootfs_key_prv` | 预留给 rootfs 校验 | 预置 dtsi，当前启动链未使用 |

`root_key` 与 `spl_key` 构成 FSBL 的证书链，由 BootROM 校验；`uboot_key` 与 `kernel_key` 用于 FIT 签名，由上一级引导程序校验。

:::tip

`uboot_key` 与 `kernel_key` 可以是不同密钥，也可以复用同一把。OpenSBI 与 U-Boot 均使用 `uboot_key_prv`，因此两者的密钥目录必须保持一致，否则 SPL 无法同时验证两者。

:::

### FIT 签名容器

安全启动基于 U-Boot 的 **FIT (Flattened Image Tree)** 格式实现。FIT 由 ITS（Image Tree Source）描述，`mkimage` 依据 ITS 打包并签名。

签名 ITS 的关键结构（以内核为例，源自 `devices/k3/common/kernel_fdt_sign.its`）：

```dts
/dts-v1/;

/ {
	description = "Signed image with single Linux kernel and FDT blob";
	#address-cells = <2>;

	images {
		kernel {
			description = "Spacemit Linux kernel";
			data = /incbin/("./Image.gz");
			type = "kernel";
			arch = "riscv";
			os = "linux";
			compression = "gzip";
			load = <0x1 0x02000000>;
			entry = <0x1 0x02000000>;
			hash {
				algo = "sha256";        /* 每个 image 单独计算 hash */
			};
		};

		fdt_11 {
			description = "Flattened Device Tree blob for k3-pico-itx";
			data = /incbin/("./kernel/k3-pico-itx.dtb");
			type = "flat_dt";
			arch = "riscv";
			compression = "none";
			load = <0x1 0x28000000>;
			hash { algo = "sha256"; };
		};
	};

	configurations {
		default = "conf_1";
		conf_11 {
			description = "k3-pico-itx";    /* 必须与 product_name 一致 */
			kernel = "kernel";
			fdt = "fdt_11";
			signature-kernel {
				algo = "sha256,rsa2048";
				key-name-hint = "kernel_key_prv";  /* 对应密钥文件名 */
				sign-images = "kernel", "fdt";     /* 签名覆盖范围 */
			};
		};
	};
};
```

关键字段说明：

| 字段 | 含义 |
|---|---|
| `hash` | 每个 image 的摘要节点，签名时参与计算 |
| `key-name-hint` | 密钥文件名（不含后缀），`mkimage` 据此在密钥目录中查找 `<hint>.key` 与 `<hint>.crt` |
| `sign-images` | 签名覆盖的 image 列表，未列入的 image 不受签名保护 |
| `description` | configuration 的板型标识，运行时用于板型匹配 |

各阶段签名 ITS 及其签名覆盖范围：

| ITS 文件 | 签名对象 | `key-name-hint` | `sign-images` |
|---|---|---|---|
| `opensbi/platform/generic/spacemit/fw_dynamic_sign.its` | OpenSBI | `uboot_key_prv` | `firmware` |
| `uboot-2022.10/board/spacemit/k3/configs/uboot_fdt_sign.its` | U-Boot + DTB | `uboot_key_prv` | `loadables`, `fdt` |
| `devices/k3/common/kernel_fdt_sign.its` | 内核 + DTB | `kernel_key_prv` | `kernel`, `fdt` |
| `devices/k3/common/uboot-opensbi_sign.its` | OpenSBI + U-Boot + ESOS + DTB | `uboot_pubkey_prv` | `firmware`, `loadables`, `fdt` |

:::warning

`devices/k3/common/uboot-opensbi_sign.its` 中的 `key-name-hint` 仍是旧密钥名 `uboot_pubkey_prv`，而 SDK 各密钥目录提供的是统一后的 `uboot_key_prv`。若使用该 ITS 签名，需先将其 `key-name-hint` 改为 `uboot_key_prv`，否则 `mkimage` 会因找不到对应密钥文件而失败。

:::

### 公钥注入机制

这是 K3 安全启动最关键、也最容易误解的一环：**验签所用公钥不是运行期从外部载入的，而是在编译期静态注入到引导程序自身的设备树中**。

注入由 `uboot-2022.10/arch/riscv/dts/secure_boot.dtsi` 完成：

```c
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_RSA_VERIFY)
	#include "key/uboot_key_pub.dtsi"    /* SPL 阶段：注入验 U-Boot 的公钥 */
#endif

#ifndef CONFIG_SPL_BUILD
#ifdef CONFIG_RSA_VERIFY
	#include "key/kernel_key_pub.dtsi"   /* U-Boot：注入验内核的公钥 */
#endif
#endif
```

该 dtsi 被板级 DTS 包含，因此：

```text
编译 SPL 时
  CONFIG_SPL_RSA_VERIFY=y
  → 包含 uboot_key_pub.dtsi
  → SPL DTB 内含 uboot_key 公钥
  → SPL 可验证 u-boot.itb / fw_dynamic.itb

编译 U-Boot 时
  CONFIG_RSA_VERIFY=y
  → 包含 kernel_key_pub.dtsi
  → 每个板级 DTB 内含 kernel_key 公钥
  → U-Boot 可验证 Image.itb
```

公钥 dtsi 的内容形式（`arch/riscv/dts/key/kernel_key_pub.dtsi`）：

```dts
/ {
	signature {
		key-kernel_key_prv {
			required = "conf";              /* 必须验证 configuration 签名 */
			algo = "sha256,rsa2048";
			rsa,r-squared = <...>;
			rsa,modulus = <...>;            /* RSA 公钥模数 */
			rsa,exponent = <0x00000000 0x00010001>;
			rsa,n0-inverse = <...>;
			rsa,num-bits = <0x00000800>;
			key-name-hint = "kernel_key_prv";
		};
	};
};
```

运行时，U-Boot 从自身 control fdt（`gd->fdt_blob`）读取 `/signature` 节点，对所有 `required = "conf"` 的公钥执行验签，任一必需公钥验证失败即拒绝启动。

:::danger

更换密钥时，**只替换密钥文件是不够的**。公钥 dtsi 中硬编码了公钥模数，必须同步用新公钥重新生成 dtsi 并重新编译 U-Boot，否则设备内固化的仍是旧公钥，无法验证用新私钥签名的镜像。

:::

### 板型 FIT config 选择

签名 FIT 通常包含多个板型的 configuration。U-Boot 通过字符串精确匹配选择：

```text
读取 product_name（来自 EEPROM TLV，或默认值）
  ↓
遍历 FIT 各 configuration 的 description
  ↓
strcmp(product_name, description) 相等 → 选中该 configuration
  ↓
全部不匹配 → 回退到 ITS 中 default 指定的 configuration
```

回退目标由 `configurations` 节点的 `default` 字段决定，例如 `default = "conf_1"`。

因此 configuration 的 `description` 必须是**裸板型名**，不能带 `.dtb` 后缀：

```dts
conf_11 {
	description = "k3-pico-itx";        /* 正确 */
	/* description = "k3-pico-itx.dtb";    错误，无法匹配 */
};
```

:::warning

板型匹配失败不会报错，而是静默回退到 `default` 指定的 configuration。若该默认板型的 DTB 与实际硬件不符（例如存储控制器被 disable），会表现为内核启动后无法挂载 rootfs，排查方向容易被误导。制作多板型签名 FIT 时务必确认 `description` 与各板 `product_name` 严格一致。

:::


### ESOS 固件签名

ESOS（Embedded System OS）是运行在 K3 协处理器（rcpu0 / rcpu1）上的实时固件，独立于主处理器的 Linux 启动链，以独立的 `esos.itb` 分区交付。启用安全启动后，`esos.itb` 同样经过 FIT 签名，验签密钥与 OpenSBI / U-Boot 共用同一套。

**签名密钥**

ESOS 使用 `uboot_key_prv` 签名，`key-name-hint` 与 OpenSBI / U-Boot 一致。对应公钥 `uboot_key_pub` 已在 U-Boot 编译期通过 SPL DTB 静态注入，签名时无需 `-K` 参数。

**ITS 结构**

签名路径使用 `esos_rt24_sign.its`：

```dts
images {
    rcpu0-fw {
        data = /incbin/("../output/esos/rt24_os0_rcpu.elf");
        compression = "none";
        hash { algo = "sha256"; };
    };
    rcpu1-fw {
        data = /incbin/("../output/esos/rt24_os1_rcpu.elf");
        compression = "none";
        hash { algo = "sha256"; };
    };
};
configurations {
    default = "conf_dual";
    conf_dual {
        loadables = "rcpu0-fw", "rcpu1-fw";
        signature-1 {
            algo = "sha256,rsa2048";
            key-name-hint = "uboot_key_prv";
            sign-images = "loadables";
        };
    };
};
```

`compression = "none"` 表示打包进 FIT 的 elf 数据不压缩。构建脚本在调用 `mkimage` 前会对 output 目录内的 `.elf` 运行 `lzop`，但 `lzop` 默认保留源文件，因此 `.elf` 与 `.elf.lzo` 并存，`mkimage` 可正常读取 `.elf`。

**KEY_DIR 总开关**

`KEY_DIR` 控制 ESOS 是否签名，与其他三个仓库逻辑一致：

| `KEY_DIR` | ITS 模板 | 产物 |
|---|---|---|
| 未设置 | `esos_rt24.its`      | 普通 `esos.itb` |
| 已设置 | `esos_rt24_sign.its` | 签名 `esos.itb` |

两种路径产物均命名为 `esos.itb`，烧录到 `esos` 分区（NOR Flash 示例：1 MiB @ 704 KiB）。
## 密钥准备

### 密钥与证书生成

使用 `openssl` 生成 RSA2048 私钥与自签名证书。私钥须妥善保管，泄露即等同于安全启动失效。

以 `kernel_key_prv` 为例：

```sh
# 生成私钥（无口令，便于自动化构建）
openssl genrsa -out kernel_key_prv.key 2048

# 生成证书（有效期按需调整）
openssl req -batch -new -x509 -days 3650 \
    -key kernel_key_prv.key -out kernel_key_prv.crt

# 从私钥导出公钥
openssl rsa -in kernel_key_prv.key -pubout -out kernel_key_pub.key
```

查看内容：

```sh
openssl rsa  -in kernel_key_prv.key -text -noout      # 私钥
openssl x509 -in kernel_key_prv.crt -text -noout      # 证书
openssl rsa  -in kernel_key_pub.key -pubin -text -noout   # 公钥
```

完整密钥集需生成以下几把（按需，`rootfs_key` 当前启动链未使用）：

```sh
for name in root_key_prv spl_key_prv uboot_key_prv kernel_key_prv; do
    openssl genrsa -out "$name.key" 2048
    openssl req -batch -new -x509 -days 3650 -key "$name.key" -out "$name.crt"
    openssl rsa -in "$name.key" -pubout -out "${name%_prv}_pub.key"
done
```

### 自定义密钥目录需准备哪些文件

`KEY_DIR` / `key_dir` 指向的密钥目录需按固定命名提供以下文件。可参考 SDK 自带目录 `uboot-2022.10/board/spacemit/k3/configs/key/` 的组织方式。

**必需文件**（缺失将导致构建失败或验签不通过）：

| 文件 | 消费者 | 用途 |
|---|---|---|
| `root_key_prv.key` | FSBL 打包工具 | 签名 cert0；其公钥 SHA256 即 ROTPKH |
| `spl_key_prv.key` | FSBL 打包工具 | 签名 FSBL 二进制（cert1） |
| `uboot_key_pub.key` | FSBL 打包工具 | 嵌入 cert0 的 `oem_key`，供 SPL 验签使用 |
| `uboot_key_prv.key` | `mkimage` | 签名 `u-boot.itb` 与 `fw_dynamic.itb` |
| `uboot_key_prv.crt` | `mkimage` | 提取公钥参数写入 SPL DTB |
| `kernel_key_prv.key` | `mkimage` | 签名 `Image.itb` |
| `kernel_key_prv.crt` | `mkimage` | 提取公钥参数写入 U-Boot DTB |

**可选文件**：

| 文件 | 说明 |
|---|---|
| `root_key_pub.key`、`spl_key_pub.key`、`kernel_key_pub.key` | 便于核对与归档，构建流程不直接读取 |
| `rootfs_key_prv.crt`、`rootfs_key_pub.key` | 预留给 rootfs 校验，当前启动链未使用 |

命名规则的由来：

- **`.key` + `.crt` 成对**：`mkimage -k <dir>` 依据 ITS 中的 `key-name-hint` 查找 `<hint>.key`（私钥，用于签名）与 `<hint>.crt`（证书，用于提取公钥参数）。两者缺一不可。
- **`_pub.key` 单独存在**：`fsbl.json` 中 `uboot_pubkey` 项直接引用 `key/uboot_key_pub.key`，因为该公钥只需嵌入证书，不参与签名。
- **`root_key` 与 `spl_key` 只需私钥**：它们由 FSBL 打包工具使用，工具从私钥自行派生公钥，无需 `.crt`。

一份最小可用的密钥目录：

```text
keys/
├── root_key_prv.key          # ROTPK 私钥
├── spl_key_prv.key           # 签 FSBL
├── uboot_key_prv.key         # 签 OpenSBI / U-Boot
├── uboot_key_prv.crt
├── uboot_key_pub.key         # 嵌入 FSBL 证书
├── kernel_key_prv.key        # 签内核
└── kernel_key_prv.crt
```

生成脚本示例：

```sh
mkdir -p keys && cd keys

# root_key 与 spl_key：仅需私钥
for name in root_key_prv spl_key_prv; do
    openssl genrsa -out "$name.key" 2048
done

# uboot_key 与 kernel_key：需私钥 + 证书
for name in uboot_key_prv kernel_key_prv; do
    openssl genrsa -out "$name.key" 2048
    openssl req -batch -new -x509 -days 3650 \
        -key "$name.key" -out "$name.crt"
done

# uboot_key 的公钥（FSBL 证书需要）
openssl rsa -in uboot_key_prv.key -pubout -out uboot_key_pub.key
```

:::warning

`uboot_key_pub.key` 必须与 `uboot_key_prv.key` 严格对应。若两者不匹配，FSBL 中嵌入的公钥无法验证用该私钥签名的 `u-boot.itb`，表现为 SPL 阶段验签失败。

:::

### 其他密钥目录

SDK 中还存在几个密钥相关目录，作用不同：

| 路径 | 作用 |
|---|---|
| `uboot-2022.10/board/spacemit/k3/configs/key/` | U-Boot 默认密钥目录（未指定 `KEY_DIR` 时使用），可作为命名参考 |
| `opensbi/platform/generic/spacemit/key/` | OpenSBI 默认密钥目录；deb 构建时会被 `KEY_DIR` 中的密钥覆盖 |
| `uboot-2022.10/arch/riscv/dts/key/` | 公钥 dtsi，由密钥派生生成，非标准密钥文件 |

推荐做法：自有密钥集中放在源码树之外的独立目录，通过 `KEY_DIR` / `key_dir` 注入，避免私钥进入版本库。

:::warning

SDK 自带密钥仅供开发验证，**所有客户拿到的是同一套**，绝不能用于量产。量产前必须替换为自有密钥，并同步重新生成公钥 dtsi。

:::

## 启用安全启动

K3 提供两条构建路径，签名的启用方式不同：

| 路径 | 用途 | 签名启用方式 |
|---|---|---|
| **deb 包构建**（各组件 `scripts/build.sh -d`） | 产出可升级的 deb 包 | 传入 `KEY_DIR`，脚本自动完成配置与公钥注入 |
| **镜像构建**（顶层 `make`） | 产出烧录镜像 | 手工改 defconfig，再执行 `make fit_sign` |

推荐优先使用 deb 路径，自动化程度更高、更少人工出错。

### 方式一：deb 构建路径（推荐）

`KEY_DIR` 是签名总开关。设置即启用签名，不设置即产出普通包。

```sh
# OpenSBI
cd opensbi && KEY_DIR=/path/to/keys ./scripts/build.sh -d

# U-Boot
cd uboot-2022.10 && KEY_DIR=/path/to/keys ./scripts/build.sh -d

# Kernel
cd linux-6.18 && KEY_DIR=/path/to/keys ./scripts/build_kernel.sh -d
```

脚本自动完成的工作：

| 组件 | `KEY_DIR` 已设置 | `KEY_DIR` 未设置 |
|---|---|---|
| OpenSBI | 向 defconfig 追加 `CONFIG_FIT_SIGNATURE=y`；用 `KEY_DIR` 中密钥覆盖自带密钥目录 | 从 defconfig 删除该行 |
| U-Boot | 向 defconfig 追加 `CONFIG_SPL_FIT_SIGNATURE=y`、`CONFIG_RSA_VERIFY=y`；检查 `KEY_DIR` 中的密钥对，**存在则重新生成对应公钥 dtsi** | 从 defconfig 删除这两行 |
| Kernel | 后处理 deb，注入签名 `Image.itb` | 产出普通 deb |

其中 U-Boot 的公钥 dtsi 自动重生成最为关键——它保证编译进设备的公钥与签名私钥严格对应，无需手工维护：

```text
KEY_DIR/kernel_key_prv.{key,crt} 存在
  → 重新生成 arch/riscv/dts/key/kernel_key_pub.dtsi

KEY_DIR/uboot_key_prv.{key,crt} 存在
  → 重新生成 arch/riscv/dts/key/uboot_key_pub.dtsi
```

:::warning

若 `KEY_DIR` 中缺少对应的 `.key` / `.crt`，脚本只打印告警并**沿用原有 dtsi**，不会报错中止。此时编译出的固件仍内嵌旧公钥，将无法验证用新密钥签名的镜像。构建后请检查日志中是否出现 `[sign][WARN] ... left as-is`。

:::

:::tip

`KEY_DIR` 对 defconfig 与公钥 dtsi 的修改会**保留在工作树中**，`git diff` 可见。这是有意设计，便于客户将自有公钥纳入版本管理。切换回普通构建时脚本会自动清除签名开关。

:::

### 方式二：镜像构建路径

用于产出完整烧录镜像，需手工启用签名配置。

#### U-Boot 配置

`uboot-2022.10/configs/k3_defconfig` 默认仅有 `CONFIG_FIT=y`，需补充：

```sh
# FIT 签名验证（U-Boot 与 SPL 各一份）
CONFIG_FIT_SIGNATURE=y
CONFIG_SPL_FIT_SIGNATURE=y

# 摘要算法
CONFIG_SHA256=y
CONFIG_SPL_SHA256=y

# RSA 验签
CONFIG_RSA=y
CONFIG_RSA_VERIFY=y
CONFIG_SPL_RSA=y
CONFIG_SPL_RSA_VERIFY=y
```

:::warning

`k3_defconfig` 中存在 `# CONFIG_SPL_SHA256 is not set` 显式关闭项，与上述清单中的 `CONFIG_SPL_SHA256=y` 冲突。必须删除该行或改为 `=y`，否则 SPL 缺少 SHA256 实现，验签无法工作。

:::

两组开关作用范围不同，缺一不可：

| 配置 | 生效阶段 | 作用 |
|---|---|---|
| `CONFIG_SPL_FIT_SIGNATURE`、`CONFIG_SPL_RSA_VERIFY` | SPL (FSBL) | 验证 `u-boot.itb` / `fw_dynamic.itb`；触发 `uboot_key_pub.dtsi` 注入 |
| `CONFIG_FIT_SIGNATURE`、`CONFIG_RSA_VERIFY` | U-Boot | 验证 `Image.itb`；触发 `kernel_key_pub.dtsi` 注入 |

也可通过 `make uboot_menuconfig` 在 `Boot images` 与 `Library routines → Security support` 下勾选。

#### OpenSBI 配置

`opensbi/platform/generic/configs/k3_defconfig` 需添加：

```sh
CONFIG_FIT_SIGNATURE=y
```

该开关决定产物是 `fw_dynamic_sign.itb`（签名）还是 `fw_dynamic.itb`（无签名），构建脚本会据此拷贝正确产物。

:::tip

若此前用 deb 路径构建过签名包，`scripts/build.sh` 已自动向该 defconfig 追加了这一行并保留在工作树中，`git diff` 可见。此时无需重复添加。

:::

#### Kernel 配置

内核侧无需专门配置——内核是被验证方，签名由打包 `Image.itb` 时完成。

#### 重新生成公钥 dtsi

手工路径下需自行更新公钥 dtsi，使编译期注入的公钥与签名私钥匹配。

U-Boot 提供了辅助脚本：

```sh
cd uboot-2022.10
sh scripts/regen_pubkey_dtsi.sh kernel_key_prv /path/to/keys \
    arch/riscv/dts/key/kernel_key_pub.dtsi
sh scripts/regen_pubkey_dtsi.sh uboot_key_prv /path/to/keys \
    arch/riscv/dts/key/uboot_key_pub.dtsi
```

其原理是用 `mkimage -K` 把公钥参数写入空 DTB，再反编译为 dtsi。手工等价操作：

```sh
# 1. 构造空 DTB 作为公钥输出目标
printf "/dts-v1/;\n/ {\n};" > pubkey.dts
dtc -I dts -O dtb -o pubkey.dtb pubkey.dts

# 2. 签名任意 FIT，同时把公钥写入 pubkey.dtb
mkimage -f <sign.its> -K pubkey.dtb -k /path/to/keys -r out.itb

# 3. 反编译查看/提取 signature 节点
dtc -I dtb -O dts -o pubkey.dts pubkey.dtb
fdtdump -s pubkey.dtb
```

:::tip

`mkimage -K` 只决定公钥写入哪个外部 DTB，与运行期验签无关。运行期使用的公钥来自编译进引导程序的 control fdt，因此**修改 dtsi 后必须重新编译 U-Boot 才会生效**。

:::

#### 执行签名

```sh
cd <SDK 根目录>
make fit_sign key_dir=$(pwd)/keys
```

该目标依次完成：

| 步骤 | 产物 | 使用密钥 |
|---|---|---|
| 签名内核 | `Image_sign.itb` | `kernel_key_prv` |
| 签名 U-Boot | `u-boot_sign.itb` | `uboot_key_prv` |
| 重打 FSBL | `FSBL.bin` | `root_key_prv` + `spl_key_prv` |
| 签名组合镜像 | `uboot-opensbi_sign.itb` | 见下方提示 |

签名完成后按常规流程打包与烧录，参见[启动开发指南](boot.md)。

## 烧录与 eFuse

### eFuse 中的启动相关字段

K3 的 eFuse 共 12 个 bank，每 bank 256 bit（32 字节），基址 `0xf0702800`。与安全启动直接相关的是以下两个 bank（定义见 `uboot-2022.10/drivers/misc/spacemit_k1x_efuse.c`）：

| Bank | 寄存器偏移 | 字段 | 长度 | 说明 |
|---|---|---|---|---|
| 4 | `0x2A0` | **ROTPKH** | 256 bit（32 字节） | Root of Trust Public Key Hash，即 `root_key` 公钥的 SHA256，安全启动信任根 |
| 5 | `0x2C0` | ARCN-NS | 224 bit（28 字节） | 非安全镜像防回滚计数器 |
| 5 | `0x2C0 + 28` | ARCN-Sec | 32 bit（4 字节） | 安全镜像防回滚计数器 |

其余 bank 存放芯片唯一标识、厂商密钥、硬件锁与生命周期状态等，与签名验证流程无直接关系。

`ROTPKH` 的值在打包 FSBL 时由工具自动计算：

```text
SHA256(root_key 公钥) → 32 字节 → 写入 eFuse bank 4
                      → 同时嵌入 FSBL 的 cert0 供 BootROM 比对
```

对应配置见 `uboot-2022.10/board/spacemit/k3/configs/fsbl.json`：

```json
{
  "hash": {
    "name": "rotpk_hash",
    "algorithm": "SHA256",
    "source": "root_key",
    "align": 32
  }
}
```

:::tip

U-Boot 当前**没有**读取 eFuse 判断安全启动是否已使能的代码。代码中的安全启动分支全部由编译期宏 `CONFIG_RSA_VERIFY` / `CONFIG_SPL_RSA_VERIFY` 控制。是否已烧录 ROTPKH 需通过产线记录确认。

:::

### FSBL 证书链结构

BootROM 无法直接存储完整公钥（eFuse 空间有限），因此采用"eFuse 存 hash、镜像带公钥"的方案。FSBL 内部构造了两级证书：

```text
eFuse: ROTPK hash (SHA256)
  │  BootROM 校验 cert0 中携带的 ROTPK 公钥 hash 是否匹配
  ▼
cert0（由 root_key 私钥签名）
  │  内含：rotpk_hash、keydata、oem_key
  │  oem_key 携带 spl_key 公钥与 uboot_key 公钥
  ▼
cert1（由 spl_key 私钥签名）
  │  覆盖 FSBL（SPL）二进制本身
  ▼
FSBL 代码执行
```

验证顺序：

1. BootROM 从 cert0 取出 ROTPK 公钥，计算 SHA256，与 eFuse 中的 hash 比对
2. hash 匹配后，用该公钥验证 cert0 的签名（`signature0`）
3. 从 cert0 的 `oem_key` 中取出 `spl_key` 公钥，验证 cert1 签名（`signature1`）
4. 验证通过后跳转执行 FSBL

`rotpk_hash` 在打包 FSBL 时由工具从 `root_key` 公钥自动计算，配置见 `uboot-2022.10/board/spacemit/k3/configs/fsbl.json`。

### 密钥槽位定义

`fsbl.json` 的 `keydata` 段定义了 4 个密钥槽位：

| 槽位 | `key_name` | 用途 |
|---|---|---|
| 0 | `spl` | 验证 FSBL |
| 1 | `uboot` | 验证 OpenSBI / U-Boot |
| 2 | `kernel` | 验证内核 |
| 3 | `rootfs` | 预留 |

其中 `spl_key` 与 `uboot_key` 的公钥实际嵌入 cert0 的 `oem_key`，供 BootROM 与 SPL 使用；`kernel_key` 的公钥通过 U-Boot DTB 注入，不经由 FSBL 传递。

### 烧录 eFuse

:::danger

**eFuse 烧录不可逆。** ROTPK hash 一旦写入即永久生效，此后设备只能启动由对应 `root_key` 签名的 FSBL。烧错密钥或烧入测试密钥的设备将无法再运行正式固件，只能报废。

:::

烧录前必须完成的检查：

| 检查项 | 确认方法 |
|---|---|
| 使用的是量产密钥，不是 SDK 自带测试密钥 | 核对密钥指纹 |
| 密钥已妥善备份 | 私钥丢失将无法再签任何固件 |
| 全链路签名启动已在未烧 eFuse 的设备上验证通过 | 实机启动到 login |
| ROTPK hash 与实际 `root_key` 对应 | 对比打包工具输出 |

推荐的量产流程：

```text
1. 生成量产密钥，离线备份私钥
2. 用量产密钥完成全链路签名构建
3. 在未烧 eFuse 的样机上烧录固件并验证启动
4. 确认无误后，在产线烧录 eFuse
5. 烧录 eFuse 后再次验证启动
```

:::tip

eFuse 的具体烧录方式与产线工具相关，参见[产线写号说明](tlv.md)与产线工具文档。

:::

## deb 升级路径签名

设备出厂后通过 deb 包升级内核时，若已启用安全启动，deb 中必须携带签名的 `Image.itb`，否则升级后设备无法启动。

内核仓库的 `scripts/build_kernel.sh` 与 `scripts/make_signed_kernel_deb.sh` 提供该能力。

### KEY_DIR 作为签名总开关

`KEY_DIR` 是否设置决定产出普通包还是签名包：

```sh
# 不传 KEY_DIR：产出普通 deb（不含 Image.itb）
./scripts/build_kernel.sh -d

# 传 KEY_DIR：产出签名 deb
KEY_DIR=/path/to/keys ./scripts/build_kernel.sh -d
```

两种模式的差异：

| 项目 | 未设置 `KEY_DIR` | 设置 `KEY_DIR` |
|---|---|---|
| Debian source 包名 | `linux-riscv-spacemit-generic` | `linux-riscv-spacemit-signed-generic` |
| binary 包名 | `linux-image-<ABI>` | `linux-image-<ABI>`（同名） |
| `/boot/Image.itb` | 无 | 有，签名 FIT |
| `/boot/vmlinuz-<ABI>` | 完整内核镜像 | 占位 stub |
| `Depends` | `spacemit-flash-dtbs` | 追加 `linux-base` |

binary 包名保持不变，客户升级命令无需区分：

```sh
dpkg -i linux-image-<ABI>_<version>_riscv64.deb
```

### 构建流程

签名 deb 不是额外的附加包，而是对标准 `bindeb-pkg` 产物做后处理，最终仍是**同名的单个 deb**：

```text
make bindeb-pkg
  → 生成完整 linux-image-<ABI>.deb
     （modules / vmlinuz / DTB / config / System.map / maintainer scripts）
  ↓
make_signed_kernel_deb.sh
  ├── 解包该 deb
  ├── 按 ITS 收集输入并用 mkimage 签名，生成 Image.itb
  ├── 注入 /boot/Image.itb
  ├── 将 /boot/vmlinuz-<ABI> 替换为 stub
  ├── control 追加 linux-base 依赖及签名包标识
  ├── postinst 追加 initrd 软链更新
  ├── 更新 md5sums
  └── 重新打包，原子替换同名 deb
```

因此签名包完整保留了 modules、内核 hooks、initramfs 生成等标准行为，客户只需安装一个包。

### 可配置项

`make_signed_kernel_deb.sh` 支持以下环境变量：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `KEY_DIR` | 必填 | 含 `<hint>.key` 与 `<hint>.crt` 的目录 |
| `KEY_HINT` | `kernel_key_prv` | 密钥名，需与 ITS 中 `key-name-hint` 一致 |
| `KERNEL_ITS` | 自动生成 | 客户自定义 ITS |
| `KERNEL_IMAGE` | `arch/riscv/boot` | 内核镜像所在**目录**，可含 `Image` 与 `Image.gz` |
| `DTB_DIR` | `arch/riscv/boot/dts/spacemit` | 板级 DTB 目录 |
| `DTB_LIST` | 目录下全部 `*.dtb` | 指定纳入签名的 DTB |
| `MKIMAGE` | 从 `PATH` 查找 | `mkimage` 路径 |

未指定 `KERNEL_ITS` 时脚本自动生成 ITS：内核为 gzip 压缩，每个 DTB 生成一个 configuration，`description` 取 DTB 文件名去掉 `.dtb` 后缀。

自定义 ITS 时，脚本会解析其中的 `/incbin/` 引用并自动准备输入文件，支持三类路径：

| ITS 中的引用 | 实际来源 |
|---|---|
| `./Image.gz` | `$KERNEL_IMAGE/Image.gz` |
| `./Image` | `$KERNEL_IMAGE/Image` |
| `./kernel/<name>.dtb` | `$DTB_DIR/<name>.dtb` |

因此自定义 ITS 可自由选择使用压缩或未压缩内核。引用其他路径会直接报错，避免产出内容不完整的 FIT。

### vmlinuz 占位说明

安全启动下 U-Boot 只加载 `/boot/Image.itb`，从不读取 `/boot/vmlinuz-<ABI>`。但该路径仍被内核 hooks 与 `linux-update-symlinks` 作为参数使用，因此不能删除。

签名包将其替换为极小的 stub 文件，仅保留路径存在性，deb 体积因此显著减小。

:::warning

替换为 stub 后，`booti /boot/vmlinuz-*` 这类非安全启动路径将不可用。由于安全启动一旦使能即不可逆，设备此后只能通过 `Image.itb` 启动，该限制不影响实际使用。

:::

### 安装验证

安装后检查关键项：

```sh
# 1. 包状态与版本
dpkg-query -W -f='${Package} ${Status} ${Version}\n' linux-image-<ABI>

# 2. Image.itb 由该包提供
dpkg -S /boot/Image.itb

# 3. 包内完整性（无输出即正常）
dpkg -V linux-image-<ABI>

# 4. initrd 软链指向当前版本，且必须是相对路径
readlink /boot/initrd.img

# 5. 签名信息
mkimage -l /boot/Image.itb
```

`/boot/initrd.img` 必须是**相对软链**（如 `initrd.img-6.18.3-generic`）。U-Boot 的 ext4 驱动以 bootfs 分区自身为根解析路径，绝对软链会指向不存在的位置导致 initrd 加载失败。`linux-update-symlinks`（由 `linux-base` 包提供）默认生成的正是相对软链，这也是签名包需要追加 `linux-base` 依赖的原因。

## 签名模式的行为差异

启用安全启动后，启动流程存在若干与普通模式不同的行为，排查问题时需注意。

### 不读取外部环境变量文件

普通模式下 U-Boot 会从 bootfs 读取 `env_k3.txt` 覆盖启动变量。签名模式下该逻辑被跳过：

```text
CONFIG_RSA_VERIFY=y
  → U-Boot 不导入 bootfs 中的 env_k3.txt
CONFIG_SPL_RSA_VERIFY=y
  → SPL 也不从外部 flash 加载 env
```

这是有意的安全设计：未签名的环境变量文件若能覆盖启动参数，攻击者可绕过验签逻辑。

由此带来的影响：

| 变量 | 签名模式取值来源 |
|---|---|
| `knl_name` | 内建默认值 `Image.itb` |
| `ramdisk_name` | 内建默认值 `initrd.img` |
| `dtb_dir` | 内建默认值 |

因此 bootfs 中的 initrd 文件名必须与内建默认值匹配，或通过软链桥接。签名内核 deb 的 postinst 调用 `linux-update-symlinks` 建立 `/boot/initrd.img` 软链正是为此。

:::tip

修改 `env_k3.txt` 在签名模式下不会生效。需要调整启动变量时，应修改 `k3.env` 并重新编译 U-Boot，使其成为编译期固化的一部分。

:::

### 内核镜像名与启动命令

```text
CONFIG_FIT_SIGNATURE=y   → knl_name=Image.itb  → 走 bootm（含验签）
未启用                    → knl_name=Image.gz   → 走 booti（无验签）
```

U-Boot 加载后会判断镜像是否为 FIT 格式：是则走 `bootm` 触发验签流程，否则走 `booti` 直接启动。

## 验证与排查

### 检查签名产物

构建后应确认各级镜像确实带有签名。`mkimage -l` 可列出 FIT 结构与签名信息：

```sh
mkimage -l Image.itb
```

正常输出应包含 `Sign algo` 与 `Sign value`：

```text
FIT description: Signed image with single Linux kernel and FDT blob
 Configuration 0 (conf_11)
  Description:  k3-pico-itx
  Kernel:       kernel
  FDT:          fdt_11
  Sign algo:    sha256,rsa2048:kernel_key_prv
  Sign value:   6dfa743bfce286560160bd94435ed877...
```

逐级检查：

```sh
mkimage -l fw_dynamic.itb    # 期望 uboot_key_prv
mkimage -l u-boot.itb        # 期望 uboot_key_prv
mkimage -l Image.itb         # 期望 kernel_key_prv
```

:::warning

`mkimage -l` 只证明 FIT 中**存在**签名节点，不证明该签名能被目标设备验过。它无法检查签名私钥与设备内固化公钥是否匹配。最终确认仍需实机启动。

:::

### 检查公钥是否正确注入

确认编译产物中的公钥与签名密钥一致：

```sh
# 查看 U-Boot DTB 中的公钥节点
fdtdump -s u-boot.dtb | grep -A 10 signature

# 或反编译查看
dtc -I dtb -O dts u-boot.dtb | grep -A 12 "signature"
```

应能看到 `key-name-hint` 与 ITS 中一致的密钥名，以及 `rsa,modulus` 等参数。

对比公钥模数是否与私钥匹配：

```sh
# 从私钥导出公钥模数
openssl rsa -in kernel_key_prv.key -noout -modulus
```

将输出与 dtsi / DTB 中的 `rsa,modulus` 对比（注意 dtsi 中是 32-bit 分组的十六进制数组）。

### 实机启动日志

安全启动成功时，串口日志会出现验签通过信息，例如：

```text
## Loading kernel from FIT Image at ... 
   Using 'conf_11' configuration
   Verifying Hash Integrity ... sha256,rsa2048:kernel_key_prv+ OK
   Trying 'kernel' kernel subimage
   ...
```

`kernel_key_prv+ OK` 中的 `+` 表示签名验证通过。

### 常见故障

#### 验签失败，启动中止

日志表现：

```text
Verifying Hash Integrity ... error!
Bad Data Hash
ERROR: can't get kernel image!
```

排查顺序：

| 可能原因 | 检查方法 |
|---|---|
| 私钥与设备内公钥不匹配 | 对比 `openssl rsa -modulus` 与 DTB 中 `rsa,modulus` |
| 更换密钥后未重新编译 U-Boot | 确认 dtsi 已更新且 U-Boot 已重新编译 |
| `key-name-hint` 与密钥文件名不一致 | 检查 ITS 中 hint 与 `<hint>.key` / `<hint>.crt` |
| 镜像在签名后被修改 | 重新签名 |

#### 板型匹配失败，启动到错误配置

日志表现：正常启动但硬件异常，例如无法挂载 rootfs、外设缺失。

```text
   Using 'conf_1' configuration      ← 实际板型不是 conf_1
```

原因：`product_name` 与所有 configuration 的 `description` 都不匹配，静默回退到 `default`。

检查：

```sh
# U-Boot 命令行查看当前板型
printenv product_name

# 对比 FIT 中各 configuration 的 description
mkimage -l Image.itb | grep Description
```

修正：确保 `description` 是裸板型名，不带 `.dtb` 后缀，且与 `product_name` 严格一致。

#### initrd 加载失败

日志表现：内核启动但无法挂载 rootfs，或提示找不到 ramdisk。

原因：签名模式不读 `env_k3.txt`，`ramdisk_name` 使用内建默认值 `initrd.img`，而 bootfs 中实际是版本化文件名。

检查：

```sh
# 必须存在，且必须是相对软链
readlink /boot/initrd.img
```

正确形式：

```text
initrd.img-6.18.3-generic        ← 相对路径，正确
/boot/initrd.img-6.18.3-generic  ← 绝对路径，错误
```

U-Boot 的 ext4 驱动以 bootfs 分区自身为根解析路径，绝对软链会指向分区内不存在的位置。

#### SPL 阶段即失败

日志表现：串口无 U-Boot 输出，或 SPL 报错后停止。

| 可能原因 | 说明 |
|---|---|
| `CONFIG_SPL_SHA256` 未启用 | SPL 缺少摘要算法实现 |
| `uboot_key_pub.dtsi` 未更新 | SPL DTB 内嵌旧公钥 |
| OpenSBI 与 U-Boot 使用了不同密钥 | 两者均由 SPL 用 `uboot_key_prv` 验证，必须一致 |

#### 修改 env_k3.txt 无效

这是签名模式的预期行为，不是故障。签名模式下 U-Boot 不导入 bootfs 中的 `env_k3.txt`，需修改 `k3.env` 并重新编译 U-Boot。

## FAQ

### 如何使用自定义密钥名？

默认密钥名为 `root_key_prv` / `spl_key_prv` / `uboot_key_prv` / `kernel_key_prv`。若要改用自有命名，需修改的位置取决于构建路径。

#### 内核签名 deb：无需改代码

`make_signed_kernel_deb.sh` 已参数化，直接传环境变量即可：

```sh
KEY_DIR=/path/to/keys KEY_HINT=myvendor_kernel_key \
    ./scripts/build_kernel.sh -d
```

脚本会用该 hint 动态生成 ITS，并在密钥目录中查找 `myvendor_kernel_key.key` 与 `.crt`。

#### 其他组件：需修改文件

U-Boot 与 OpenSBI 侧的密钥名硬编码在 ITS、dtsi 与构建脚本中，需逐处修改。

**1. ITS 的 `key-name-hint`**

| 文件 | 当前值 |
|---|---|
| `uboot-2022.10/board/spacemit/k3/configs/uboot_fdt_sign.its` | `uboot_key_prv` |
| `opensbi/platform/generic/spacemit/fw_dynamic_sign.its` | `uboot_key_prv` |
| `devices/k3/common/kernel_fdt_sign.its` | `kernel_key_prv` |
| `devices/k3/common/uboot-opensbi_sign.its` | `uboot_pubkey_prv`（旧名，见下文） |

**2. 公钥 dtsi 的节点名与 hint**

`uboot-2022.10/arch/riscv/dts/key/` 下：

| 文件 | 需改内容 |
|---|---|
| `uboot_key_pub.dtsi` | 节点名 `key-uboot_key_prv`、字段 `key-name-hint` |
| `kernel_key_pub.dtsi` | 节点名 `key-kernel_key_prv`、字段 `key-name-hint` |
| `rootfs_key_pub.dtsi` | 同上（当前启动链未使用） |

若走 deb 构建路径，这两个 dtsi 由 `scripts/build.sh` 自动重新生成，但**触发条件是 `KEY_DIR` 中存在 `kernel_key_prv.{key,crt}` 与 `uboot_key_prv.{key,crt}` 这两个固定文件名**。改名后需同步修改该判断逻辑：

```sh
# uboot-2022.10/scripts/build.sh 中的硬编码检查
if [ -f "$KEY_DIR/kernel_key_prv.key" ] && [ -f "$KEY_DIR/kernel_key_prv.crt" ]; then
    sh "$REGEN" kernel_key_prv "$KEY_DIR" "$DTSI_DIR/kernel_key_pub.dtsi"
```

**3. FSBL 配置 `fsbl.json`**

`uboot-2022.10/board/spacemit/k3/configs/fsbl.json` 中的密钥文件路径：

| 位置 | 当前值 | 说明 |
|---|---|---|
| root_key 的 `source` | `key/root_key_prv.key` | ROTPK 私钥 |
| spl_key 的 `source` | `key/spl_key_prv.key` | 签 SPL 的私钥 |
| uboot_pubkey 的 `source` | `key/uboot_key_pub.key` | 注意是 `_pub.key` 后缀 |

`keytable` 中的 `key_name` 字段（`spl` / `uboot` / `kernel` / `rootfs`）是密钥槽位标识，与文件名无关。

:::warning

`keytable` 的 `key_name` 是否可自由修改尚未验证。这些标识可能被 BROM 用于槽位匹配，建议保持默认值，仅修改 `source` 指向的文件名。

:::

#### 最小改动建议

保持 `<用途>_key_prv` 命名后缀约定，只改前缀：

```text
myvendor_uboot_key_prv.key / .crt
myvendor_kernel_key_prv.key / .crt
```

这样只需批量替换 ITS 与 dtsi 中的密钥名，以及 `build.sh` 中的两处检查，`fsbl.json` 的后缀约定不变。

### `uboot_pubkey_prv` 和 `uboot_key_prv` 有什么区别？

`uboot_pubkey_prv` 是**旧密钥名**，`uboot_key_prv` 是统一后的新名。

当前 SDK 中 `devices/k3/common/uboot-opensbi_sign.its` 仍使用旧名，而所有密钥目录提供的是新名文件。若使用该 ITS 签名，需先将其 `key-name-hint` 改为 `uboot_key_prv`，否则 `mkimage` 会因找不到 `uboot_pubkey_prv.key` 而失败。

新开发请统一使用 `uboot_key_prv`。

### 可以给 OpenSBI 和 U-Boot 使用不同密钥吗？

不可以。两者的 FIT 都由 SPL 使用同一把公钥验证：

```text
FSBL (SPL)
  ├── 验 fw_dynamic.itb  ← uboot_key_prv
  └── 验 u-boot.itb      ← uboot_key_prv
```

SPL 的 DTB 中只注入了一把 `uboot_key` 公钥，因此两个 ITS 必须使用相同密钥。若要区分，需为 SPL 注入两把公钥并分别在 ITS 中指定，属于超出默认设计的改动。

`kernel_key` 与 `uboot_key` 则可以是不同密钥——前者由 U-Boot 验证，后者由 SPL 验证，公钥注入到不同阶段的 DTB 中。

### 更换密钥后为什么验签还是失败？

最常见原因是**只换了密钥文件，没有重新生成公钥 dtsi 并重编 U-Boot**。

设备验签使用的公钥来自编译进引导程序 DTB 的 `/signature` 节点，不是运行期从外部读取的。完整更换密钥的步骤：

```text
1. 替换密钥文件
2. 重新生成 uboot_key_pub.dtsi / kernel_key_pub.dtsi
3. 重新编译 U-Boot（SPL 与 U-Boot 两部分都要）
4. 用新私钥重新签名所有镜像
5. 烧录新的 FSBL / U-Boot / 内核
```

走 deb 路径时第 2、3 步由 `scripts/build.sh` 自动完成，但需确认日志中没有 `[sign][WARN] ... left as-is`。

### 安全启动可以关闭吗？

**eFuse 使能后不可关闭。** ROTPK hash 一旦烧入 eFuse 即永久生效，设备此后只接受签名固件。

在烧录 eFuse **之前**，可以通过以下方式回到非签名启动：

```sh
# deb 路径：不传 KEY_DIR
./scripts/build.sh -d

# 镜像路径：从 defconfig 移除签名开关
```

因此建议的验证顺序是：先在未烧 eFuse 的设备上完成全链路签名验证，确认可正常启动后，再烧录 eFuse。

### 签名会增加多少启动时间？

每级验签需计算 SHA256 摘要并做一次 RSA2048 公钥运算。实际耗时取决于镜像大小与 CPU 频率，本文不提供具体数值，建议在目标硬件上实测对比。

需要注意的是签名内核使用 gzip 压缩（`Image.gz`），解压也会带来额外耗时。

### 如何确认设备当前是否运行在安全启动模式？

可从以下迹象判断：

```sh
# 1. 启动日志中是否有验签输出
#    "Verifying Hash Integrity ... sha256,rsa2048:kernel_key_prv+ OK"

# 2. 使用的内核镜像名
#    签名模式为 Image.itb，普通模式为 Image.gz

# 3. env_k3.txt 是否生效
#    签名模式下修改该文件不影响启动
```

:::tip

U-Boot 侧当前没有提供读取 eFuse 判断安全启动状态的命令。是否已烧录 ROTPK hash 需通过产线烧录记录确认。

:::

### 多个板型如何共用一份签名镜像？

在同一个 FIT 中为每个板型创建一个 configuration，`description` 填写各板的 `product_name`：

```dts
configurations {
	default = "conf_1";
	conf_1  { description = "k3_deb1";     kernel = "kernel"; fdt = "fdt_1";  signature { ... }; };
	conf_2  { description = "k3-pico-itx"; kernel = "kernel"; fdt = "fdt_2";  signature { ... }; };
};
```

内核签名 deb 默认会扫描 DTB 目录下所有 `*.dtb` 并逐个生成 configuration，也可通过 `DTB_LIST` 限定：

```sh
KEY_DIR=/path/to/keys DTB_LIST="k3-pico-itx.dtb k3_deb1.dtb" \
    ./scripts/build_kernel.sh -d
```

`default` 指向的 configuration 是板型匹配失败时的回退项，应选择兼容性最好的板型。
