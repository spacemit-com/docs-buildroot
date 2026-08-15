---
sidebar_position: 4
---

# Secure Boot Development Guide

## Overview

### Purpose

This guide describes the Secure Boot implementation, key hierarchy, enablement procedure, and upgrade path for the SpacemiT K3 platform. It covers the complete signature-verification chain from the chip root of trust to the Linux kernel.

### Scope

This document applies to SpacemiT K3-series SoCs.

For foundational information such as the flashing procedure, partition layout, and standard boot configuration, see the [Boot Development Guide](boot.md). This guide focuses exclusively on Secure Boot.

### Intended Audience

- Secure Boot development engineers
- Flashing and boot engineers
- Production engineers

### Document Structure

1. **Secure Boot Mechanism**: Trust-chain architecture, key hierarchy, FIT signing containers, and public-key injection.
2. **Key Preparation**: Key and certificate generation and the files required in a custom key directory.
3. **Enabling Secure Boot**: Build configuration and signing workflows for each component.
4. **Flashing and eFuse**: ROTPK hash programming and irreversible constraints.
5. **Signing the deb Upgrade Path**: Building and installing signed kernel deb packages.
6. **Behavioral Differences in Signed Mode**: Differences from standard boot.
7. **Verification and Troubleshooting**: Signed-artifact inspection and diagnosis of common failures.

### Important Constraint

:::danger

Secure Boot becomes **irreversible** once enabled through eFuse. After enablement, the device can run signed firmware only and cannot return to unsigned boot. Validate the complete signing workflow before programming the eFuse.

:::

## Secure Boot Mechanism

### Trust Chain Overview

K3 Secure Boot uses the on-chip BootROM as its root of trust and verifies signatures at each stage. Before handing over control, each stage verifies the integrity and provenance of the image at the next stage:

```text
BootROM (immutable root of trust)
  │  Verifies the FSBL certificate chain against the ROTPK hash in eFuse
  ▼
FSBL (u-boot-spl + FSBL header)
  │  Uses the uboot_key public key injected into the SPL DTB at build time
  │  Verifies the FIT signatures of u-boot.itb / fw_dynamic.itb
  ▼
OpenSBI (fw_dynamic.itb) + U-Boot (u-boot.itb)
  │  U-Boot uses the kernel_key public key injected into its control FDT
  │  at build time to verify the FIT signature of Image.itb
  ▼
Linux Kernel + DTB (Image.itb)
```

Signature-verification responsibilities at each stage:

| Verified object | Verifier | Verification public key (corresponding signing private key) | Public-key location |
|---|---|---|---|
| FSBL | BootROM | ROTPK (`root_key_prv`) | SHA256 stored in eFuse bank 4 |
| `fw_dynamic.itb` | FSBL (SPL) | `uboot_key_pub` (`uboot_key_prv`) | SPL DTB |
| `u-boot.itb` | FSBL (SPL) | `uboot_key_pub` (`uboot_key_prv`) | SPL DTB |
| `Image.itb` | U-Boot | `kernel_key_pub` (`kernel_key_prv`) | U-Boot control FDT |

- Hash algorithm: `SHA256`
- Signature algorithm: `SHA256 + RSA2048`

### Key Hierarchy

K3 uses a hierarchical key architecture; each key serves a distinct purpose:

| Key | Purpose | Public-key destination |
|---|---|---|
| `root_key_prv` | ROTPK; signs cert0, the first FSBL certificate | Public-key hash programmed to eFuse |
| `spl_key_prv` | Signs the FSBL (SPL) binary itself | Carried by cert0 |
| `uboot_key_prv` | Signs `esos.itb`, `u-boot.itb`, and `fw_dynamic.itb` | Injected into the SPL DTB at build time |
| `kernel_key_prv` | Signs `Image.itb` (kernel + DTB) | Injected into **every board-specific DTB** in U-Boot at build time |
| `rootfs_key_prv` | Reserved for rootfs verification | Pre-provisioned dtsi; unused by the current boot chain |

`root_key` and `spl_key` form the FSBL certificate chain verified by BootROM. `uboot_key` and `kernel_key` are used for FIT signatures verified by the preceding bootloader stage.

:::tip

`uboot_key` and `kernel_key` may be distinct keys or the same key. OpenSBI and U-Boot both use `uboot_key_prv`, so their key directories must remain consistent; otherwise, SPL cannot verify both components.

:::

### FIT Signing Container

Secure Boot uses the U-Boot **FIT (Flattened Image Tree)** format. An ITS (Image Tree Source) describes the FIT, and `mkimage` packages and signs the image based on that description.

The following example shows the key structures in a signing ITS, using `devices/k3/common/kernel_fdt_sign.its`:

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
        algo = "sha256";        /* Hash calculated independently for each image */
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
      description = "k3-pico-itx";    /* Must match product_name */
			kernel = "kernel";
			fdt = "fdt_11";
			signature-kernel {
				algo = "sha256,rsa2048";
        key-name-hint = "kernel_key_prv";  /* Corresponding key filename */
        sign-images = "kernel", "fdt";     /* Signature coverage */
			};
		};
	};
};
```

Key fields:

| Field | Description |
|---|---|
| `hash` | Digest node for each image; included in signature calculation |
| `key-name-hint` | Key filename without an extension. `mkimage` uses it to find `<hint>.key` and `<hint>.crt` in the key directory. |
| `sign-images` | List of images covered by the signature; images not listed are not protected by the signature. |
| `description` | Board identifier for the configuration, used for board matching at runtime |

Signing ITS files and signature coverage at each stage:

| ITS file | Signed object | `key-name-hint` | `sign-images` |
|---|---|---|---|
| `opensbi/platform/generic/spacemit/fw_dynamic_sign.its` | OpenSBI | `uboot_key_prv` | `firmware` |
| `uboot-2022.10/board/spacemit/k3/configs/uboot_fdt_sign.its` | U-Boot + DTB | `uboot_key_prv` | `loadables`, `fdt` |
| `devices/k3/common/kernel_fdt_sign.its` | Kernel + DTB | `kernel_key_prv` | `kernel`, `fdt` |
| `devices/k3/common/uboot-opensbi_sign.its` | OpenSBI + U-Boot + ESOS + DTB | `uboot_pubkey_prv` | `firmware`, `loadables`, `fdt` |

:::warning

The `key-name-hint` in `devices/k3/common/uboot-opensbi_sign.its` still uses the legacy key name `uboot_pubkey_prv`, whereas SDK key directories provide the unified `uboot_key_prv`. Before signing with this ITS, change its `key-name-hint` to `uboot_key_prv`; otherwise, `mkimage` fails because it cannot find the corresponding key file.

:::

### Public-Key Injection Mechanism

This is the most important and most frequently misunderstood aspect of K3 Secure Boot: **the public keys used for verification are not loaded from an external source at runtime; they are statically injected into the bootloader device trees at build time**.

Injection is performed by `uboot-2022.10/arch/riscv/dts/secure_boot.dtsi`:

```c
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_RSA_VERIFY)
  #include "key/uboot_key_pub.dtsi"    /* SPL stage: inject U-Boot verification public key */
#endif

#ifndef CONFIG_SPL_BUILD
#ifdef CONFIG_RSA_VERIFY
  #include "key/kernel_key_pub.dtsi"   /* U-Boot: inject kernel verification public key */
#endif
#endif
```

This dtsi is included by the board-specific DTS. Therefore:

```text
When building SPL
  CONFIG_SPL_RSA_VERIFY=y
  → Includes uboot_key_pub.dtsi
  → SPL DTB contains the uboot_key public key
  → SPL can verify u-boot.itb / fw_dynamic.itb

When building U-Boot
  CONFIG_RSA_VERIFY=y
  → Includes kernel_key_pub.dtsi
  → Every board-specific DTB contains the kernel_key public key
  → U-Boot can verify Image.itb
```

Public-key dtsi format (`arch/riscv/dts/key/kernel_key_pub.dtsi`):

```dts
/ {
	signature {
		key-kernel_key_prv {
      required = "conf";              /* Configuration signature verification is mandatory */
			algo = "sha256,rsa2048";
			rsa,r-squared = <...>;
      rsa,modulus = <...>;            /* RSA public-key modulus */
			rsa,exponent = <0x00000000 0x00010001>;
			rsa,n0-inverse = <...>;
			rsa,num-bits = <0x00000800>;
			key-name-hint = "kernel_key_prv";
		};
	};
};
```

At runtime, U-Boot reads the `/signature` node from its control FDT (`gd->fdt_blob`) and verifies the signatures using every public key marked `required = "conf"`. Boot is rejected if verification fails for any required public key.

:::danger

When replacing keys, **replacing the key files alone is insufficient**. The public-key modulus is hard-coded in the public-key `dtsi`. Regenerate the `dtsi` with the new public key and rebuild U-Boot; otherwise, the device retains the old public key and cannot verify images signed with the new private key.

:::

### Board-Specific FIT Configuration Selection

Signed FIT images typically contain configurations for multiple board types. U-Boot selects a configuration by exact string matching:

```text
Read product_name (from EEPROM TLV or a default value)
  ↓
Iterate through descriptions of FIT configurations
  ↓
strcmp(product_name, description) matches → Select that configuration
  ↓
No matches → Fall back to the configuration specified by default in the ITS
```

The `default` property in the `configurations` node defines the fallback target, for example, `default = "conf_1"`.

Therefore, a configuration `description` must be the **bare board name** without a `.dtb` suffix:

```dts
conf_11 {
  description = "k3-pico-itx";        /* Correct */
  /* description = "k3-pico-itx.dtb";    Incorrect; cannot match */
};
```

:::warning

If board matching fails, U-Boot reports no error and silently falls back to the configuration specified by `default`. If the fallback DTB does not match the actual hardware, for example because its storage controller is disabled, the kernel may fail to mount the rootfs, obscuring the root cause. When creating a signed FIT for multiple board types, ensure that each `description` exactly matches the corresponding board `product_name`.

:::


### ESOS Firmware Signing

ESOS (Embedded System OS) is real-time firmware that runs on the K3 coprocessors (`rcpu0` / `rcpu1`). It is independent of the Linux boot chain on the main processor and is delivered in a separate `esos.itb` partition. When Secure Boot is enabled, `esos.itb` is also FIT-signed and uses the same verification key as OpenSBI and U-Boot.

**Signing Key**

ESOS is signed with `uboot_key_prv`, using the same `key-name-hint` as OpenSBI and U-Boot. The corresponding `uboot_key_pub` public key is statically injected through the SPL DTB when U-Boot is built. The `-K` parameter is not required during signing.

**ITS Structure**

The signing path uses `esos_rt24_sign.its`:

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

`compression = "none"` means that ELF data packaged into the FIT is uncompressed. Before invoking `mkimage`, the build script runs `lzop` on `.elf` files in the output directory. However, `lzop` retains source files by default, so `.elf` and `.elf.lzo` coexist and `mkimage` can read the `.elf` file normally.

**KEY_DIR Master Switch**

`KEY_DIR` controls whether ESOS is signed, following the same convention used by the other three repositories:

| `KEY_DIR` | ITS template | Artifact |
|---|---|---|
| Unset | `esos_rt24.its`      | Standard `esos.itb` |
| Set | `esos_rt24_sign.its` | Signed `esos.itb` |

Both paths produce an artifact named `esos.itb`, which is programmed to the `esos` partition (NOR Flash example: 1 MiB @ 704 KiB).
## Key Preparation

### Key and Certificate Generation

Use `openssl` to generate RSA2048 private keys and self-signed certificates. Secure private keys appropriately; disclosure is equivalent to compromising Secure Boot.

Using `kernel_key_prv` as an example:

```sh
# Generate a private key (without a passphrase for automated builds)
openssl genrsa -out kernel_key_prv.key 2048

# Generate a certificate (adjust the validity period as required)
openssl req -batch -new -x509 -days 3650 \
    -key kernel_key_prv.key -out kernel_key_prv.crt

# Export the public key from the private key
openssl rsa -in kernel_key_prv.key -pubout -out kernel_key_pub.key
```

Inspect the contents:

```sh
openssl rsa  -in kernel_key_prv.key -text -noout      # Private key
openssl x509 -in kernel_key_prv.crt -text -noout      # Certificate
openssl rsa  -in kernel_key_pub.key -pubin -text -noout   # Public key
```

Generate the following complete key set as required (`rootfs_key` is unused by the current boot chain):

```sh
for name in root_key_prv spl_key_prv uboot_key_prv kernel_key_prv; do
    openssl genrsa -out "$name.key" 2048
    openssl req -batch -new -x509 -days 3650 -key "$name.key" -out "$name.crt"
    openssl rsa -in "$name.key" -pubout -out "${name%_prv}_pub.key"
done
```

### Files Required for a Custom Key Directory

The key directory specified by `KEY_DIR` / `key_dir` must contain the following files with the required names. The SDK-provided `uboot-2022.10/board/spacemit/k3/configs/key/` directory provides an example layout.

**Required files**: Missing files cause the build or subsequent signature verification to fail.

| File | Consumer | Purpose |
|---|---|---|
| `root_key_prv.key` | FSBL packaging tool | Signs cert0; the SHA256 of its public key is ROTPKH. |
| `spl_key_prv.key` | FSBL packaging tool | Signs the FSBL binary (cert1) |
| `uboot_key_pub.key` | FSBL packaging tool | Embedded in cert0 `oem_key` for SPL signature verification |
| `uboot_key_prv.key` | `mkimage` | Signs `u-boot.itb` and `fw_dynamic.itb` |
| `uboot_key_prv.crt` | `mkimage` | Extracts public-key parameters for the SPL DTB |
| `kernel_key_prv.key` | `mkimage` | Signs `Image.itb` |
| `kernel_key_prv.crt` | `mkimage` | Extracts public-key parameters for the U-Boot DTB |

**Optional files**:

| File | Description |
|---|---|
| `root_key_pub.key`, `spl_key_pub.key`, `kernel_key_pub.key` | Useful for verification and archiving; not read directly by the build workflow |
| `rootfs_key_prv.crt`, `rootfs_key_pub.key` | Reserved for rootfs verification; unused by the current boot chain |

Naming conventions:

- **`.key` and `.crt` pair**: Based on the ITS `key-name-hint`, `mkimage -k <dir>` locates `<hint>.key` (the private key used for signing) and `<hint>.crt` (the certificate used to extract public-key parameters). Both are required.
- **Standalone `_pub.key`**: The `uboot_pubkey` entry in `fsbl.json` directly references `key/uboot_key_pub.key` because this public key is embedded in the certificate only and does not participate in signing.
- **`root_key` and `spl_key` require private keys only**: They are used by the FSBL packaging tool, which derives public keys from private keys and does not require `.crt` files.

A minimal functional key directory:

```text
keys/
├── root_key_prv.key          # ROTPK private key
├── spl_key_prv.key           # Signs FSBL
├── uboot_key_prv.key         # Signs OpenSBI / U-Boot
├── uboot_key_prv.crt
├── uboot_key_pub.key         # Embedded in the FSBL certificate
├── kernel_key_prv.key        # Signs the kernel
└── kernel_key_prv.crt
```

Example generation script:

```sh
mkdir -p keys && cd keys

# root_key and spl_key: private keys only
for name in root_key_prv spl_key_prv; do
    openssl genrsa -out "$name.key" 2048
done

# uboot_key and kernel_key: private keys + certificates
for name in uboot_key_prv kernel_key_prv; do
    openssl genrsa -out "$name.key" 2048
    openssl req -batch -new -x509 -days 3650 \
        -key "$name.key" -out "$name.crt"
done

# uboot_key public key (required by the FSBL certificate)
openssl rsa -in uboot_key_prv.key -pubout -out uboot_key_pub.key
```

:::warning

`uboot_key_pub.key` must correspond exactly to `uboot_key_prv.key`. If they do not match, the public key embedded in FSBL cannot verify `u-boot.itb` signed with that private key, resulting in signature-verification failure at the SPL stage.

:::

### Other Key Directories

The SDK also contains several key-related directories with different purposes:

| Path | Purpose |
|---|---|
| `uboot-2022.10/board/spacemit/k3/configs/key/` | Default U-Boot key directory used when `KEY_DIR` is unspecified; use as a naming reference. |
| `opensbi/platform/generic/spacemit/key/` | Default OpenSBI key directory; overridden by keys in `KEY_DIR` during `deb` builds. |
| `uboot-2022.10/arch/riscv/dts/key/` | Public-key `dtsi` files generated from keys, not standard key files |

Recommended practice: Store proprietary keys in a dedicated directory outside the source tree and provide the directory through `KEY_DIR` / `key_dir`, preventing private keys from entering version control.

:::warning

SDK-provided keys are for development validation only. **All customers receive the same set** and they must never be used for production. Before mass production, replace them with proprietary keys and regenerate the public-key dtsi files.

:::

## Enabling Secure Boot

K3 provides two build paths with different signing enablement methods:

| Path | Purpose | Signing enablement |
|---|---|---|
| **deb package build** (each component uses `scripts/build.sh -d`) | Produces upgradable `deb` packages | Pass `KEY_DIR`; scripts automatically configure signing and inject public keys. |
| **Image build** (top-level `make`) | Produces flashable images | Manually modify the defconfig, then run `make fit_sign`. |

The `deb` path is recommended because it provides a higher degree of automation and reduces the risk of manual errors.

### Method 1: `deb` Build Path (Recommended)

`KEY_DIR` is the signing master switch. Setting it enables signing; leaving it unset produces standard packages.

```sh
# OpenSBI
cd opensbi && KEY_DIR=/path/to/keys ./scripts/build.sh -d

# U-Boot
cd uboot-2022.10 && KEY_DIR=/path/to/keys ./scripts/build.sh -d

# Kernel
cd linux-6.18 && KEY_DIR=/path/to/keys ./scripts/build_kernel.sh -d
```

The scripts perform the following tasks automatically:

| Component | `KEY_DIR` set | `KEY_DIR` unset |
|---|---|---|
| OpenSBI | Appends `CONFIG_FIT_SIGNATURE=y` to the defconfig; overrides the provided key directory with keys from `KEY_DIR` | Removes this line from the defconfig |
| U-Boot | Appends `CONFIG_SPL_FIT_SIGNATURE=y` and `CONFIG_RSA_VERIFY=y` to the defconfig; checks key pairs in `KEY_DIR` and **regenerates corresponding public-key dtsi files when present** | Removes these two lines from the defconfig |
| Kernel | Post-processes the `deb` and injects signed `Image.itb` | Produces a standard `deb` |

Automatic regeneration of the U-Boot public-key dtsi files is critical: it ensures that the public keys compiled into the device correspond exactly to the signing private keys without requiring manual maintenance:

```text
KEY_DIR/kernel_key_prv.{key,crt} present
  → Regenerate arch/riscv/dts/key/kernel_key_pub.dtsi

KEY_DIR/uboot_key_prv.{key,crt} present
  → Regenerate arch/riscv/dts/key/uboot_key_pub.dtsi
```

:::warning

If the corresponding `.key` / `.crt` is missing from `KEY_DIR`, the script prints a warning and **retains the existing `dtsi`** rather than terminating with an error. The resulting firmware still embeds the old public key and cannot verify images signed with a new key. After building, check the log for `[sign][WARN] ... left as-is`.

:::

:::tip

Changes that `KEY_DIR` makes to the defconfig and public-key `dtsi` files are **retained in the working tree** and visible in `git diff`. This is intentional, allowing customers to manage proprietary public keys in version control. When switching back to a standard build, scripts automatically remove the signing switches.

:::

### Method 2: Image Build Path

This path produces complete flashable images and requires manual enablement of signing configuration.

#### U-Boot Configuration

By default, `uboot-2022.10/configs/k3_defconfig` contains only `CONFIG_FIT=y`. Add:

```sh
# FIT signature verification (one setting each for U-Boot and SPL)
CONFIG_FIT_SIGNATURE=y
CONFIG_SPL_FIT_SIGNATURE=y

# Hash algorithm
CONFIG_SHA256=y
CONFIG_SPL_SHA256=y

# RSA signature verification
CONFIG_RSA=y
CONFIG_RSA_VERIFY=y
CONFIG_SPL_RSA=y
CONFIG_SPL_RSA_VERIFY=y
```

:::warning

`k3_defconfig` contains the explicit disable entry `# CONFIG_SPL_SHA256 is not set`, which conflicts with `CONFIG_SPL_SHA256=y` above. Delete this line or change it to `=y`; otherwise, SPL lacks a SHA256 implementation and signature verification cannot work.

:::

The two switch groups apply to different stages, and both are required:

| Configuration | Active stage | Purpose |
|---|---|---|
| `CONFIG_SPL_FIT_SIGNATURE`, `CONFIG_SPL_RSA_VERIFY` | SPL (FSBL) | Verifies `u-boot.itb` / `fw_dynamic.itb`; triggers `uboot_key_pub.dtsi` injection. |
| `CONFIG_FIT_SIGNATURE`, `CONFIG_RSA_VERIFY` | U-Boot | Verifies `Image.itb`; triggers `kernel_key_pub.dtsi` injection. |

Alternatively, select these options under `Boot images` and `Library routines → Security support` through `make uboot_menuconfig`.

#### OpenSBI Configuration

Add the following to `opensbi/platform/generic/configs/k3_defconfig`:

```sh
CONFIG_FIT_SIGNATURE=y
```

This switch determines whether the artifact is `fw_dynamic_sign.itb` (signed) or `fw_dynamic.itb` (unsigned). The build script copies the appropriate artifact accordingly.

:::tip

If a signed package was previously built through the `deb` path, `scripts/build.sh` has already appended this line to the defconfig and retained it in the working tree, as shown by `git diff`. Do not add it again.

:::

#### Kernel Configuration

No dedicated kernel-side configuration is required. The kernel is the verified component, and signing occurs when `Image.itb` is packaged.

#### Regenerate Public-Key dtsi Files

For the manual path, update the public-key dtsi files so that the public keys injected at build time match the signing private keys.

U-Boot provides a helper script:

```sh
cd uboot-2022.10
sh scripts/regen_pubkey_dtsi.sh kernel_key_prv /path/to/keys \
    arch/riscv/dts/key/kernel_key_pub.dtsi
sh scripts/regen_pubkey_dtsi.sh uboot_key_prv /path/to/keys \
    arch/riscv/dts/key/uboot_key_pub.dtsi
```

It uses `mkimage -K` to write public-key parameters to an empty DTB, then decompiles it into a `dtsi`. Equivalent manual steps:

```sh
# 1. Create an empty DTB as the public-key output target
printf "/dts-v1/;\n/ {\n};" > pubkey.dts
dtc -I dts -O dtb -o pubkey.dtb pubkey.dts

# 2. Sign any FIT and write the public key to pubkey.dtb
mkimage -f <sign.its> -K pubkey.dtb -k /path/to/keys -r out.itb

# 3. Decompile to inspect or extract the signature node
dtc -I dtb -O dts -o pubkey.dts pubkey.dtb
fdtdump -s pubkey.dtb
```

:::tip

`mkimage -K` determines only the external DTB to which the public key is written; it does not affect runtime verification. Public keys used at runtime come from the control FDT compiled into the bootloader. Therefore, **U-Boot must be rebuilt after modifying the dtsi**.

:::

#### Perform Signing

```sh
cd <SDK root directory>
make fit_sign key_dir=$(pwd)/keys
```

This target performs the following steps:

| Step | Artifact | Key used |
|---|---|---|
| Sign kernel | `Image_sign.itb` | `kernel_key_prv` |
| Sign U-Boot | `u-boot_sign.itb` | `uboot_key_prv` |
| Repackage FSBL | `FSBL.bin` | `root_key_prv` + `spl_key_prv` |
| Sign combined image | `uboot-opensbi_sign.itb` | See the note below. |

After signing, package and flash the image through the standard procedure. See the [Boot Development Guide](boot.md).

## Flashing and eFuse

### Boot-Related eFuse Fields

K3 eFuse contains 12 banks, each 256 bit (32 bytes), at base address `0xf0702800`. The following two banks are relevant to Secure Boot and are defined in `uboot-2022.10/drivers/misc/spacemit_k1x_efuse.c`:

| Bank | Register offset | Field | Length | Description |
|---|---|---|---|---|
| 4 | `0x2A0` | **ROTPKH** | 256 bit (32 bytes) | Root of Trust Public Key Hash: SHA256 of the `root_key` public key and the Secure Boot root of trust |
| 5 | `0x2C0` | ARCN-NS | 224 bit (28 bytes) | Anti-rollback counter for non-secure images |
| 5 | `0x2C0 + 28` | ARCN-Sec | 32 bit (4 bytes) | Anti-rollback counter for secure images |

The remaining banks store the chip unique identifier, vendor keys, hardware locks, lifecycle state, and other data with no direct relationship to the signature-verification flow.

The tool automatically calculates `ROTPKH` when packaging FSBL:

```text
SHA256(root_key public key) → 32 bytes → Write to eFuse bank 4
                             → Also embed in FSBL cert0 for BootROM comparison
```

The corresponding configuration is in `uboot-2022.10/board/spacemit/k3/configs/fsbl.json`:

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

U-Boot currently has **no** code to read eFuse and determine whether Secure Boot is enabled. All Secure Boot code paths are controlled by the build-time macros `CONFIG_RSA_VERIFY` / `CONFIG_SPL_RSA_VERIFY`. Confirm whether ROTPKH has been programmed from production records.

:::

### FSBL Certificate Chain Structure

Because eFuse capacity is limited, BootROM cannot store a complete public key directly. Instead, the eFuse stores the hash while the image carries the public key. FSBL constructs a two-level certificate chain:

```text
eFuse: ROTPK hash (SHA256)
  │  BootROM verifies that the ROTPK public-key hash carried in cert0 matches
  ▼
cert0 (signed by the root_key private key)
  │  Contains: rotpk_hash, keydata, oem_key
  │  oem_key carries the spl_key and uboot_key public keys
  ▼
cert1 (signed by the spl_key private key)
  │  Covers the FSBL (SPL) binary itself
  ▼
FSBL code execution
```

Verification sequence:

1. BootROM obtains the ROTPK public key from cert0, calculates SHA256, and compares it with the hash in eFuse.
2. After the hashes match, it uses this public key to verify the cert0 signature (`signature0`).
3. It obtains the `spl_key` public key from cert0 `oem_key` and verifies the cert1 signature (`signature1`).
4. After verification succeeds, it transfers control to FSBL.

The tool automatically calculates `rotpk_hash` from the `root_key` public key when packaging FSBL. See `uboot-2022.10/board/spacemit/k3/configs/fsbl.json` for the configuration.

### Key Slot Definitions

The `keydata` section of `fsbl.json` defines four key slots:

| Slot | `key_name` | Purpose |
|---|---|---|
| 0 | `spl` | Verify FSBL |
| 1 | `uboot` | Verify OpenSBI / U-Boot |
| 2 | `kernel` | Verify the kernel |
| 3 | `rootfs` | Reserved |

The public keys for `spl_key` and `uboot_key` are embedded in cert0 `oem_key` for use by BootROM and SPL. The `kernel_key` public key is injected through the U-Boot DTB and is not passed through FSBL.

### Program eFuse

:::danger

**eFuse programming is irreversible.** Once the ROTPK hash is written, it takes permanent effect and the device can boot only FSBL images signed with the corresponding `root_key`. A device programmed with an incorrect key or a test key can no longer run production firmware and must be discarded.

:::

Required checks before programming:

| Check item | Confirmation method |
|---|---|
| Production keys are used, not SDK-provided test keys | Check key fingerprints |
| Keys are securely backed up | A lost private key prevents signing any future firmware. |
| End-to-end signed boot is validated on a device without programmed eFuse | Boot the physical device to login |
| ROTPK hash corresponds to the actual `root_key` | Compare with packaging-tool output |

Recommended production process:

```text
1. Generate production keys and back up private keys offline.
2. Complete the end-to-end signed build with the production keys.
3. Flash the firmware to a sample device without programmed eFuse and verify boot.
4. After confirmation, program eFuse on the production line.
5. Verify boot again after programming eFuse.
```

:::tip

The eFuse programming method depends on the production-line tool. See [Production Programming Guide](tlv.md) and [Product Line Tool](plt.md).

:::

## Signing the deb Upgrade Path

When upgrading the kernel through a `deb` package after a device ships, a Secure Boot-enabled device requires the `deb` to contain signed `Image.itb`; otherwise, the device cannot boot after the upgrade.

The kernel repository provides this capability through `scripts/build_kernel.sh` and `scripts/make_signed_kernel_deb.sh`.

### KEY_DIR as the Signing Master Switch

Whether `KEY_DIR` is set determines whether the output is a standard package or a signed package:

```sh
# Without KEY_DIR: produce a standard deb (without Image.itb)
./scripts/build_kernel.sh -d

# With KEY_DIR: produce a signed deb
KEY_DIR=/path/to/keys ./scripts/build_kernel.sh -d
```

Differences between the two modes:

| Item | `KEY_DIR` unset | `KEY_DIR` set |
|---|---|---|
| Debian source package name | `linux-riscv-spacemit-generic` | `linux-riscv-spacemit-signed-generic` |
| Binary package name | `linux-image-<ABI>` | `linux-image-<ABI>` (same name) |
| `/boot/Image.itb` | Absent | Present, signed FIT |
| `/boot/vmlinuz-<ABI>` | Complete kernel image | Placeholder stub |
| `Depends` | `spacemit-flash-dtbs` | Adds `linux-base` |

The binary package name remains unchanged, so the same upgrade command applies:

```sh
dpkg -i linux-image-<ABI>_<version>_riscv64.deb
```

### Build Process

A signed `deb` is not an additional package. It is a post-processed standard `bindeb-pkg` artifact and remains **a single deb with the same name**:

```text
make bindeb-pkg
    → Generate a complete linux-image-<ABI>.deb
      (modules / vmlinuz / DTB / config / System.map / maintainer scripts)
  ↓
make_signed_kernel_deb.sh
  ├── Unpack the deb
  ├── Collect the ITS inputs and sign them with mkimage to generate Image.itb
  ├── Inject /boot/Image.itb
  ├── Replace /boot/vmlinuz-<ABI> with a stub
  ├── Add the linux-base dependency and signed-package identifier to control
  ├── Add initrd symbolic-link updates to postinst
  ├── Update md5sums
  └── Repackage and atomically replace the deb with the same name
```

The signed package therefore retains standard behavior for modules, kernel hooks, initramfs generation, and more. Customers install only one package.

### Configuration Options

`make_signed_kernel_deb.sh` supports the following environment variables:

| Variable | Default | Description |
|---|---|---|
| `KEY_DIR` | Required | Directory containing `<hint>.key` and `<hint>.crt` |
| `KEY_HINT` | `kernel_key_prv` | Key name; must match `key-name-hint` in the ITS. |
| `KERNEL_ITS` | Automatically generated | Customer-defined ITS |
| `KERNEL_IMAGE` | `arch/riscv/boot` | **Directory** containing kernel images, which can include `Image` and `Image.gz` |
| `DTB_DIR` | `arch/riscv/boot/dts/spacemit` | Board-specific DTB directory |
| `DTB_LIST` | All `*.dtb` files in the directory | DTBs to include in the signature |
| `MKIMAGE` | Located from `PATH` | Path to `mkimage` |

When `KERNEL_ITS` is not specified, the script generates an ITS automatically: the kernel is gzip-compressed, one configuration is generated for each DTB, and `description` is the DTB filename without its `.dtb` suffix.

For a custom ITS, the script parses `/incbin/` references and automatically prepares input files. Three path types are supported:

| Reference in ITS | Actual source |
|---|---|
| `./Image.gz` | `$KERNEL_IMAGE/Image.gz` |
| `./Image` | `$KERNEL_IMAGE/Image` |
| `./kernel/<name>.dtb` | `$DTB_DIR/<name>.dtb` |

Custom ITS files can therefore use either compressed or uncompressed kernels. References to other paths result in an error, preventing generation of an incomplete FIT.

### vmlinuz Placeholder

Under Secure Boot, U-Boot loads only `/boot/Image.itb` and never reads `/boot/vmlinuz-<ABI>`. However, this path is still used as an argument by kernel hooks and `linux-update-symlinks`, so it cannot be removed.

The signed package replaces it with a very small stub file to retain only path existence, significantly reducing `deb` size.

:::warning

After replacement with a stub, unsigned boot paths such as `booti /boot/vmlinuz-*` are unavailable. Because Secure Boot is irreversible once enabled, the device can subsequently boot only through `Image.itb`; this restriction does not affect normal operation.

:::

### Installation Verification

After installation, check the following key items:

```sh
# 1. Package status and version
dpkg-query -W -f='${Package} ${Status} ${Version}\n' linux-image-<ABI>

# 2. Image.itb is provided by this package
dpkg -S /boot/Image.itb

# 3. Package integrity (no output indicates success)
dpkg -V linux-image-<ABI>

# 4. initrd symbolic link points to the current version and must be relative
readlink /boot/initrd.img

# 5. Signature information
mkimage -l /boot/Image.itb
```

`/boot/initrd.img` must be a **relative symbolic link**, such as `initrd.img-6.18.3-generic`. The U-Boot ext4 driver resolves paths relative to the bootfs partition itself. An absolute symbolic link points to a nonexistent location and causes initrd loading to fail. `linux-update-symlinks` (provided by the `linux-base` package) generates relative symbolic links by default, which is why the signed package adds a `linux-base` dependency.

## Behavioral Differences in Signed Mode

After Secure Boot is enabled, the boot process has several behaviors that differ from standard mode. Consider them during troubleshooting.

### No External Environment Variable File

In standard mode, U-Boot reads `env_k3.txt` from bootfs to override boot variables. In signed mode, this logic is skipped:

```text
CONFIG_RSA_VERIFY=y
  → U-Boot does not import env_k3.txt from bootfs
CONFIG_SPL_RSA_VERIFY=y
  → SPL also does not load env from external flash
```

This is an intentional security design: if an unsigned environment-variable file could override boot parameters, an attacker could bypass signature-verification logic.

Resulting behavior:

| Variable | Value source in signed mode |
|---|---|
| `knl_name` | Built-in default `Image.itb` |
| `ramdisk_name` | Built-in default `initrd.img` |
| `dtb_dir` | Built-in default |

Therefore, the initrd filename in bootfs must match the built-in default or be bridged with a symbolic link. This is why signed-kernel deb postinst invokes `linux-update-symlinks` to create the `/boot/initrd.img` symbolic link.

:::tip

Changes to `env_k3.txt` do not take effect in signed mode. To adjust boot variables, modify `k3.env` and rebuild U-Boot so it becomes part of the build-time image.

:::

### Kernel Image Name and Boot Command

```text
CONFIG_FIT_SIGNATURE=y   → knl_name=Image.itb  → Use bootm (with signature verification)
Not enabled               → knl_name=Image.gz   → Use booti (without signature verification)
```

After loading the image, U-Boot determines whether it is in FIT format. FIT images use `bootm` to trigger signature verification; otherwise, `booti` starts the image directly.

## Verification and Troubleshooting

### Inspect Signed Artifacts

After building, confirm that images at every stage contain signatures. `mkimage -l` lists the FIT structure and signature information:

```sh
mkimage -l Image.itb
```

Normal output includes `Sign algo` and `Sign value`:

```text
FIT description: Signed image with single Linux kernel and FDT blob
 Configuration 0 (conf_11)
  Description:  k3-pico-itx
  Kernel:       kernel
  FDT:          fdt_11
  Sign algo:    sha256,rsa2048:kernel_key_prv
  Sign value:   6dfa743bfce286560160bd94435ed877...
```

Check each stage:

```sh
mkimage -l fw_dynamic.itb    # Expected: uboot_key_prv
mkimage -l u-boot.itb        # Expected: uboot_key_prv
mkimage -l Image.itb         # Expected: kernel_key_prv
```

:::warning

`mkimage -l` proves only that a signature node **exists** in the FIT; it does not prove that the target device can verify the signature. It cannot check whether the signing private key matches the public key embedded in the device. Final confirmation still requires booting on physical hardware.

:::

### Verify Correct Public-Key Injection

Confirm that the public key in the build artifact matches the signing key:

```sh
# Inspect the public-key node in the U-Boot DTB
fdtdump -s u-boot.dtb | grep -A 10 signature

# Or decompile for inspection
dtc -I dtb -O dts u-boot.dtb | grep -A 12 "signature"
```

The output must show a key name in `key-name-hint` that matches the ITS, along with parameters such as `rsa,modulus`.

Compare whether the public-key modulus matches the private key:

```sh
# Export the public-key modulus from the private key
openssl rsa -in kernel_key_prv.key -noout -modulus
```

Compare the output with `rsa,modulus` in the dtsi / DTB. Note that dtsi represents it as a hexadecimal array grouped into 32-bit values.

### Physical Device Boot Log

When Secure Boot succeeds, the serial log contains signature-verification success information, for example:

```text
## Loading kernel from FIT Image at ... 
   Using 'conf_11' configuration
   Verifying Hash Integrity ... sha256,rsa2048:kernel_key_prv+ OK
   Trying 'kernel' kernel subimage
   ...
```

In `kernel_key_prv+ OK`, `+` indicates successful signature verification.

### Common Failures

#### Signature Verification Fails and Boot Stops

Log output:

```text
Verifying Hash Integrity ... error!
Bad Data Hash
ERROR: can't get kernel image!
```

Troubleshooting order:

| Possible cause | Check method |
|---|---|
| Private key does not match the public key in the device | Compare `openssl rsa -modulus` with `rsa,modulus` in the DTB. |
| U-Boot was not rebuilt after changing keys | Confirm that `dtsi` files were updated and U-Boot was rebuilt. |
| `key-name-hint` does not match the key filename | Check the ITS hint against `<hint>.key` / `<hint>.crt`. |
| Image modified after signing | Sign it again |

#### Board Matching Fails and Boots the Wrong Configuration

Log behavior: Boot appears normal but hardware is abnormal, for example, rootfs cannot be mounted or peripherals are missing.

```text
  Using 'conf_1' configuration      ← The actual board type is not conf_1
```

Cause: `product_name` does not match the `description` of any configuration, so the system silently falls back to `default`.

Check:

```sh
# Display the current board type in the U-Boot command line
printenv product_name

# Compare descriptions of FIT configurations
mkimage -l Image.itb | grep Description
```

Correction: Ensure that `description` is the bare board name without a `.dtb` suffix and exactly matches `product_name`.

#### initrd Loading Fails

Log behavior: The kernel starts but cannot mount rootfs, or reports that the ramdisk cannot be found.

Cause: Signed mode does not read `env_k3.txt`, so `ramdisk_name` uses the built-in default `initrd.img`, while bootfs contains an actual versioned filename.

Check:

```sh
# Must exist and must be a relative symbolic link
readlink /boot/initrd.img
```

Correct form:

```text
initrd.img-6.18.3-generic        ← Relative path, correct
/boot/initrd.img-6.18.3-generic  ← Absolute path, incorrect
```

The U-Boot ext4 driver resolves paths relative to the bootfs partition itself, so an absolute symbolic link points to a location that does not exist in the partition.

#### Failure at the SPL Stage

Log behavior: No U-Boot output is present on the serial console, or SPL reports an error and stops.

| Possible cause | Description |
|---|---|
| `CONFIG_SPL_SHA256` is not enabled | SPL lacks a hash-algorithm implementation. |
| `uboot_key_pub.dtsi` is not updated | SPL DTB embeds the old public key. |
| OpenSBI and U-Boot use different keys | SPL verifies both with `uboot_key_prv`, so they must be identical. |

#### Changes to `env_k3.txt` Have No Effect

This is expected behavior in signed mode, not a fault. U-Boot does not import `env_k3.txt` from bootfs in signed mode; modify `k3.env` and rebuild U-Boot instead.

## FAQ

### How Do I Use a Custom Key Name?

Default key names are `root_key_prv` / `spl_key_prv` / `uboot_key_prv` / `kernel_key_prv`. The locations that must be changed to use proprietary names depend on the build path.

#### Signed Kernel deb: No Code Changes Required

`make_signed_kernel_deb.sh` is already parameterized; pass environment variables directly:

```sh
KEY_DIR=/path/to/keys KEY_HINT=myvendor_kernel_key \
    ./scripts/build_kernel.sh -d
```

The script dynamically generates the ITS using this hint and searches the key directory for `myvendor_kernel_key.key` and `.crt`.

#### Other Components: Files Must Be Modified

Key names for U-Boot and OpenSBI are hard-coded in ITS files, `dtsi` files, and build scripts, and must be changed in each location.

**1. ITS `key-name-hint`**

| File | Current value |
|---|---|
| `uboot-2022.10/board/spacemit/k3/configs/uboot_fdt_sign.its` | `uboot_key_prv` |
| `opensbi/platform/generic/spacemit/fw_dynamic_sign.its` | `uboot_key_prv` |
| `devices/k3/common/kernel_fdt_sign.its` | `kernel_key_prv` |
| `devices/k3/common/uboot-opensbi_sign.its` | `uboot_pubkey_prv` (legacy name; see below) |

**2. Public-key dtsi node names and hints**

Under `uboot-2022.10/arch/riscv/dts/key/`:

| File | Content to change |
|---|---|
| `uboot_key_pub.dtsi` | Node name `key-uboot_key_prv` and field `key-name-hint` |
| `kernel_key_pub.dtsi` | Node name `key-kernel_key_prv` and field `key-name-hint` |
| `rootfs_key_pub.dtsi` | Same as above (unused by the current boot chain) |

For the deb build path, these two `dtsi` files are regenerated automatically by `scripts/build.sh`, but the **trigger condition is the presence of the fixed filenames `kernel_key_prv.{key,crt}` and `uboot_key_prv.{key,crt}` in `KEY_DIR`**. After renaming, modify this conditional logic accordingly:

```sh
# Hard-coded check in uboot-2022.10/scripts/build.sh
if [ -f "$KEY_DIR/kernel_key_prv.key" ] && [ -f "$KEY_DIR/kernel_key_prv.crt" ]; then
    sh "$REGEN" kernel_key_prv "$KEY_DIR" "$DTSI_DIR/kernel_key_pub.dtsi"
```

**3. FSBL configuration `fsbl.json`**

Key file paths in `uboot-2022.10/board/spacemit/k3/configs/fsbl.json`:

| Location | Current value | Description |
|---|---|---|
| `source` for root_key | `key/root_key_prv.key` | ROTPK private key |
| `source` for spl_key | `key/spl_key_prv.key` | Private key for signing SPL |
| `source` for uboot_pubkey | `key/uboot_key_pub.key` | Note the `_pub.key` suffix. |

The `key_name` field (`spl` / `uboot` / `kernel` / `rootfs`) in `keytable` is a key-slot identifier and is unrelated to the filename.

:::warning

Whether `key_name` in `keytable` can be changed freely has not been verified. These identifiers may be used by BROM for slot matching. Keep their default values and modify only the filenames referenced by `source`.

:::

#### Minimal Change Recommendation

Retain the `<purpose>_key_prv` naming suffix convention and change only the prefix:

```text
myvendor_uboot_key_prv.key / .crt
myvendor_kernel_key_prv.key / .crt
```

This requires only bulk replacement of key names in ITS and `dtsi` files and the two checks in `build.sh`; the filename suffix convention in `fsbl.json` remains unchanged.

### What Is the Difference Between `uboot_pubkey_prv` and `uboot_key_prv`?

`uboot_pubkey_prv` is the **legacy key name**, while `uboot_key_prv` is the unified new name.

In the current SDK, `devices/k3/common/uboot-opensbi_sign.its` still uses the legacy name, while all key directories provide files with the new name. Before signing with this ITS, change its `key-name-hint` to `uboot_key_prv`; otherwise, `mkimage` fails because it cannot find `uboot_pubkey_prv.key`.

Use `uboot_key_prv` consistently for new development.

### Can OpenSBI and U-Boot Use Different Keys?

No. SPL verifies both FIT images with the same public key:

```text
FSBL (SPL)
  ├── Verify fw_dynamic.itb  ← uboot_key_prv
  └── Verify u-boot.itb      ← uboot_key_prv
```

Only one `uboot_key` public key is injected into the SPL DTB, so both ITS files must use the same key. Distinguishing them requires injecting two public keys into SPL and specifying each separately in the ITS, which exceeds the default design.

`kernel_key` and `uboot_key` can be different keys: U-Boot verifies the former, SPL verifies the latter, and their public keys are injected into DTBs at different stages.

### Why Does Signature Verification Still Fail After Replacing Keys?

The most common cause is **replacing only key files without regenerating public-key `dtsi` files and rebuilding U-Boot**.

The public key used by the device for signature verification comes from the `/signature` node in the bootloader DTB built into the image; it is not read externally at runtime. Complete key replacement requires:

```text
1. Replace the key files.
2. Regenerate uboot_key_pub.dtsi / kernel_key_pub.dtsi.
3. Rebuild U-Boot, including both SPL and U-Boot components.
4. Sign all images again with the new private keys.
5. Flash the new FSBL / U-Boot / kernel.
```

For the deb path, steps 2 and 3 are automated by `scripts/build.sh`, but confirm that the log does not contain `[sign][WARN] ... left as-is`.

### Can Secure Boot Be Disabled?

**It cannot be disabled after eFuse enablement.** Once the ROTPK hash is programmed to eFuse, it takes permanent effect and the device accepts signed firmware only.

**Before** programming eFuse, return to unsigned boot as follows:

```sh
# deb path: omit KEY_DIR
./scripts/build.sh -d

# Image path: remove signing switches from the defconfig
```

The recommended verification sequence is therefore to complete end-to-end signed verification on a device without programmed eFuse, confirm normal boot, and then program the eFuse.

### How Much Boot Time Does Signing Add?

Each verification stage calculates a SHA256 digest and performs one RSA2048 public-key operation. Actual time depends on image size and CPU frequency. This document provides no specific value; measure and compare on target hardware.

Note that the signed kernel uses gzip compression (`Image.gz`), and decompression also adds processing time.

### How Can I Confirm That a Device Is Running in Secure Boot Mode?

Check the following indicators:

```sh
# 1. Whether the boot log contains signature-verification output
#    "Verifying Hash Integrity ... sha256,rsa2048:kernel_key_prv+ OK"

# 2. Kernel image name in use
#    Signed mode uses Image.itb; standard mode uses Image.gz

# 3. Whether env_k3.txt takes effect
#    Changes to this file do not affect boot in signed mode
```

:::tip

U-Boot currently provides no command to read eFuse and determine Secure Boot status. Confirm whether the ROTPK hash has been programmed through production-line programming records.

:::

### How Can Multiple Board Types Share One Signed Image?

Create a configuration for each board type in the same FIT, and set `description` to each board `product_name`:

```dts
configurations {
	default = "conf_1";
	conf_1  { description = "k3_deb1";     kernel = "kernel"; fdt = "fdt_1";  signature { ... }; };
	conf_2  { description = "k3-pico-itx"; kernel = "kernel"; fdt = "fdt_2";  signature { ... }; };
};
```

By default, the signed-kernel deb scans all `*.dtb` files in the DTB directory and generates one configuration for each. Limit this through `DTB_LIST`:

```sh
KEY_DIR=/path/to/keys DTB_LIST="k3-pico-itx.dtb k3_deb1.dtb" \
    ./scripts/build_kernel.sh -d
```

The configuration referenced by `default` is the fallback when board matching fails; select the board type with the best compatibility.
