---
sidebar_position: 7
---

# TA Development Guide

## Overview

### Purpose

This is an operational guide for developing TAs (Trusted Applications) on K3: preparing the environment, creating the directory skeleton, implementing entry points, and building, deploying, and verifying the TA step by step. TA categories, the runtime model, available APIs, and parameter correspondence follow the procedure as background reference.

### Scope

This guide applies to Buildroot solutions for SpacemiT K3 series SoCs.

- **Build environment:** Cross-compilation: build on a development host and run the binary on a RISC-V target.
- **Related documents:** See the [CA Development Guide / TEE Client API Reference](ca_guide.md) for the CA side, [TEE Examples](examples.md) for runnable examples, and the [TEE Architecture and Integration Guide](tee_architecture.md) for architecture.

### Document Structure

1. **Configuration ownership:** Which options belong to the core (TEE OS build) and which belong to the TA build.
2. **TA categories and runtime model:** The three TA types, storage locations, target architecture, and the instance/session model.
3. **Available API overview:** Capabilities a TA can call.
4. **Parameter correspondence with the CA**.
5. **Using secure storage**.
6. **TA versioning and rollback protection**, **TA encryption**, and **installing a Secure Storage TA**.
7. **Quick start:** A five-step overview.
8. **Steps 1-5:** Prepare the environment, create the directory skeleton, implement entry points, set attributes, and build, deploy, and verify.
9. **Complete example**, **debugging and troubleshooting**, and **references**.

### Important Constraints

> [!CAUTION]
> TAs run on a 64-bit RISC-V target with **floating-point support disabled**. TA code must not use floating-point operations; if floating point is genuinely required, reassess the security and performance trade-off rather than simply enabling it.
>
> In addition, **the TA signing key must be independent of the CA side and replaced before production**. The default signing key ships with the SDK; anyone possessing it can sign a TA accepted by this platform.

## Configuration Ownership: Core Options and TA Build Variables

TA-related options belong to two builds with different change costs:

- **Core options:** Applied when building the TEE OS (OP-TEE kernel); changing them requires rebuilding the TEE and updating firmware.
- **TA build variables:** Applied when building a TA; changing them only requires rebuilding and deploying the TA.

| Option / variable | Owner | Purpose | Value in this solution |
|---|---|---|---|
| `CFG_WITH_USER_TA` | Core | Whether TAs can run | Enabled |
| `CFG_REE_FS` | Core | Secure-storage backend (normal-world file system) | Enabled |
| `CFG_REE_FS_TA` | Core | Load TAs from the normal-world file system; also carries load-time TA version checking | Enabled |
| `CFG_SECSTOR_TA` | Core | Enable TAs in secure storage (the TA database). Depends on `CFG_REE_FS` and `CFG_WITH_USER_TA`, and is enabled by default when both are enabled | Enabled |
| `CFG_SECSTOR_TA_MGMT_PTA` | Core | Enable the management pseudo TA used to install TAs. Depends on `CFG_SECSTOR_TA` and follows it by default | Enabled |
| `CFG_TEE_TA_LOG_LEVEL` | TA build | TA-side log level | 4 |
| `CFG_TA_OPTEE_CORE_API_COMPAT_1_1` | TA build | TEE Core API 1.1 compatibility layer | y |
| `TA_SIGN_KEY` | TA build | TA signing private key | dev kit `keys/default_ta.pem` |
| `CFG_ENCRYPT_TA` | TA build | Whether to encrypt TAs when packaging | Disabled (unencrypted by default) |
| `TA_ENC_KEY` | TA build | TA packaging encryption key | Not set |
| `CFG_TA_VERSION` | TA build | Version written into the TA header (rollback protection) | Not set (0) |
| `TA_DEV_KIT_DIR` | TA build | TA dev kit path | See [Step 1](#step-1-prepare-the-development-environment) |

Runtime interfaces such as `tee_otp_get_ta_enc_key()` are implemented by the **core or platform** and are not build variables.

## TA Categories and Runtime Model

### Three TA Types

| Type | Description | Loading | Typical use |
|---|---|---|---|
| User TA | Standard trusted application stored as a file and loaded by the TEE at runtime | Provided by the normal-world file system and loaded by the TEE | Business logic, key operations, secure storage |
| Early TA | Compiled into the TEE kernel image and started with the TEE | Built into `tee.bin`, available at boot | Base services needed during TEE initialization |
| PTA (Pseudo TA) | Not a standard TA; a pseudo TA inside the TEE kernel | Part of the TEE kernel | Exposes platform capabilities such as device enumeration and RTC to the normal world |

The built-in early TA on K3 is **Trusted Keys** (UUID `f04a0fe7-1f5d-4b9b-abf7-619b85b4ce8c`). PTA interface headers such as device enumeration and RTC ship with the dev kit. **The steps below cover User TAs.**

### TA Storage Locations

The TEE tries three locations in fixed priority order. A **smaller number means higher priority**:

| Priority | Location | Form |
|---|---|---|
| 2 | Embedded TA (early TA) | Compiled into the TEE kernel image and updated with firmware |
| 4 | Secure Storage TA | Stored in the TEE secure-storage TA database and loaded at runtime; see [Installing a Secure Storage TA](#installing-a-secure-storage-ta) |
| 9 | Normal-world file system | Read by `tee-supplicant` from the TA directory |

The normal-world file system path is **read each time a session is opened**: the TA file is not copied into secure storage. Only a version record for rollback protection is persisted; see [TA Versioning and Rollback Protection](#ta-versioning-and-rollback-protection).

A given UUID loads only from the **highest-priority** location; lower-priority locations are not consulted when a TA exists at a higher priority. Installing a TA into secure storage (priority 4) therefore supersedes a same-named copy in the TA directory (priority 9), which remains but is unused — this is a deployment change.

### Target Architecture

| Item | Value |
|---|---|
| TA target | 64-bit RISC-V (`ta_rv64`) |
| Floating point | Disabled |
| Concurrency | Multicore supported; the TA must handle concurrent access itself |

### Instance and Session Model

- Normal-world software accesses a TA through sessions.
- By default, **multiple sessions of the same TA share one instance**, so the TA must protect instance data against concurrent access.
- Separate instances per session or instance persistence after the last session closes require the corresponding flags; see the attribute table in [Step 4](#step-4-set-attributes-and-flags).
- TAs do not share memory by default; cross-TA cooperation uses TEE-provided channels.

## Available API Overview

TAs access TEE capabilities through the Internal Core API provided by `libutee`, defined by the GlobalPlatform TEE Internal Core API specification (1.4). For the full specification, see [References](#references).

- **Interface source:** GlobalPlatform (GP) TEE Internal Core API, implemented by `libutee` on K3.
- **CA-side counterpart:** GP TEE Client API; see the [CA Development Guide / TEE Client API Reference](ca_guide.md).
- **Relationship between the specifications:** See [GP APIs and OP-TEE](tee_architecture.md#gp-apis-and-op-tee) in the TEE Architecture and Integration Guide.
- **Nature:** Calls are system calls into the TEE.

| Capability | Contents |
|---|---|
| Cryptography | Hash, symmetric and asymmetric operations, random numbers, key derivation and generation |
| Secure storage | Create, read, write, find, delete, and rename persistent and transient objects |
| Time | System time and waiting |
| Big integers | Initialization, comparison, and arithmetic on big integers |
| Memory | Allocation and release, shared-memory registration and access |
| Attributes and sessions | Read own attributes, manage session context |

K3 cryptography is provided by the security engine: RSA, ECDSA, and SM2DSA for asymmetric operations, and AES and SM4 for symmetric operations. **Hash operations use the software implementation** by default, so TAs need no special handling for software versus hardware selection.

Platform PTA headers such as device enumeration and RTC are also supplied in the dev kit `include/` directory.

## Parameter Correspondence with the CA

Parameters pass between CA and TA in four forms that must match on both sides:

| CA-side type | TA-side access | Description |
|---|---|---|
| `TEEC_VALUE_*` | `TEE_Param` `value.a` / `value.b` | Two 32-bit integers, suitable for small data |
| `TEEC_MEMREF_TEMP_*` | `TEE_Param` `memref.buffer` / `memref.size` | Temporary memory reference valid only for this call |
| `TEEC_MEMREF_WHOLE` | Same, referring to the whole shared-memory block | Registered shared memory |
| `TEEC_MEMREF_PARTIAL_*` | Same, accessed by offset and length | Part of registered shared memory |

`paramTypes` describes the correspondence. **Type mismatches are not compile-time errors**; they return a parameter error at runtime. The TA should validate parameter types and sizes at entry and return a clear error instead of continuing.

## Using Secure Storage

TAs read and write persistent objects through TEE Internal Core API storage interfaces. Both backends present the **same interface**; they differ only in rollback protection and enablement prerequisites:

- **The default backend** (normal-world file system managed by `tee-supplicant`) provides no rollback protection. TAs must not assume objects are immune to rollback or deletion. Storage operations fail if `tee-supplicant` is not running.
- **Each TA has a separate key** derived from the TEE key hierarchy, so TAs cannot read each other's data. Data is device-bound and cannot be unsealed elsewhere.
- **RPMB** can provide rollback protection, but it is **not enabled** in the current build. Enabling it requires `CFG_RPMB_FS` and prior RPMB key provisioning on the device.

Backend organization, locations, and capacity semantics are described in [Secure Storage](tee_architecture.md#secure-storage) in the TEE Architecture and Integration Guide.

## TA Versioning and Rollback Protection

A TA can carry a version number that prevents downgrade to an older TA.

| Item | Description |
|---|---|
| Version source | `CFG_TA_VERSION` at **TA build time**; defaults to `0` when unset |
| Transfer | Passed to the signing script as `--ta-version` and written into the TA header |
| Effect of the default | With `0`, version comparison never succeeds, so **rollback protection is effectively disabled**; set an explicit version to enable it |

Version checking occurs at two independent points:

| Check | Timing | Record |
|---|---|---|
| Load check | Every session open, for TAs in the normal-world file system; provided by the **core option** `CFG_REE_FS_TA` | Separate version record stores (one for TAs, one for subkeys) |
| Install check | When installing into secure storage | The TA entry's own version in secure storage |

Two key points:

- **Version records use TEE persistent storage** (`TEE_STORAGE_PRIVATE`), located according to the storage backend. Records are **created on demand** only when a comparison occurs, so their absence on a new device is not a configuration problem.
- **TAs installed in secure storage skip the load check**; rollback protection then depends on the install check and the secure-storage entry.

## TA Encryption

TA binaries are **unencrypted by default**. Encryption depends on whether an encryption key is supplied at packaging time and is independent of where the TA is stored.

The four-byte type field at offset 4 in the `.ta` header (`xxd -s 4 -l 4 <uuid>.ta`, little-endian) indicates the form:

| Type | Meaning |
|---|---|
| 0 | Signed plaintext |
| 1 | Signed plaintext plus bootstrap (**default when no encryption key is specified**) |
| 2 | Encrypted (AES-GCM) |
| 3 | Subkey delegation certificate (not a TA body; not loadable) |

Encryption is enabled **when building the TA**: set `CFG_ENCRYPT_TA=y` and replace the default `TA_ENC_KEY` with your own key. The default is a public fixed value in the source that real devices cannot decrypt; it is for demonstration only. Both settings are TA build variables and do not change the TEE kernel.

The encryption key must match on both sides:

| Side | Source |
|---|---|
| Packaging (build time) | `TA_ENC_KEY`, passed as `--enc-key` by the signing script |
| Decryption (runtime) | The `tee_otp_get_ta_enc_key()` interface implemented by the **core/platform**; the default derives from the **device root key**, bound to the TA's signing public-key modulus |

Three cautions:

- **Changing the TA signing key also changes the encryption key** because the signing public-key modulus is a derivation input. All previously encrypted TAs become invalid; rotate both keys together.
- **The key type is selected in the TA header** (device-unique or category-shared). The default implementation supports only device-unique; category-shared requires a platform implementation of the key-retrieval interface.
- **Encrypted TAs cannot be installed into secure storage**: the install path accepts only the unencrypted type-1 form with a bootstrap section, because secure storage encrypts the TA payload itself. The two encryption paths are mutually exclusive.

## Installing a Secure Storage TA

TAs in secure storage are **installed explicitly by a user-space program**: the caller opens the management pseudo TA, submits the signed TA binary, and the kernel verifies the signature and writes it to the secure-storage TA database. It is not flashed with firmware and does not happen automatically at boot. The test suite provides the official installation command:

```sh
# Install all TAs in a directory
xtest --install-ta /lib/optee_armtz

# Or list files explicitly
xtest --install-ta /lib/optee_armtz/<uuid>.ta
```

Prerequisites and side effects:

- **Prerequisites (core options set when building the TEE OS):** `CFG_SECSTOR_TA=y` and `CFG_SECSTOR_TA_MGMT_PTA=y`. The former depends on `CFG_REE_FS` and `CFG_WITH_USER_TA` and is enabled by default when both are enabled; all are enabled in this solution. The TA must be an unencrypted type-1 binary with a bootstrap section, and the background service must be running for the normal-world file-system backend.
- **Side effect:** After installation, that UUID loads from secure storage and a same-named copy in the TA directory is no longer used — a deployment change.
- **Removal:** There is no uninstall command; delete the secure-storage TA entry and its data files.

> [!NOTE]
> The install command attempts every `*.ta` in the directory. A subkey delegation certificate (type 3) is rejected and causes the command to exit early, appearing as a partial install. List the intended files explicitly to exclude certificates.

## Quick Start

The complete TA development flow has five steps, each described in detail later in this guide:

| Step | Action | Output |
|---|---|---|
| 1 | Obtain the TA dev kit and cross toolchain | Usable `TA_DEV_KIT_DIR` and `riscv64-unknown-linux-gnu-` toolchain |
| 2 | Create the directory skeleton: `Makefile`, `sub.mk`, `user_ta_header_defines.h`, sources, `include/` | A buildable TA directory |
| 3 | Implement the five entry points and dispatch by command ID in `TA_InvokeCommandEntryPoint` | Entry-point implementation |
| 4 | Set UUID, flags, and stack/heap sizes in the attribute header | Final TA attributes |
| 5 | Build, install on the device, and verify calls | `<uuid>.ta` and verification results |

Complete steps 2 and 4 first, then implement the code in step 3. Getting the directory skeleton and attributes right helps avoid most installation and session-opening problems.

## Step 1: Prepare the Development Environment

### 1.1 TA Dev Kit

TA builds require the development kit exported by the TEE, containing headers, static libraries, linker scripts, and build rules:

| Directory/file | Contents |
|---|---|
| `include/` | TA headers: `tee_api.h`, `tee_internal_api.h`, `user_ta_header.h`, `utee_defines.h`, `utee_types.h`, `pta_*.h` |
| `lib/` | `libutee.a` (Internal Core API), `libutils.a`, `libmbedtls.a`, `libdl.a` |
| `mk/ta_dev_kit.mk` | TA build rules |
| `src/` | `ta.ld.S` (linker script) and `user_ta_header.c` |
| `keys/default_ta.pem` | Default TA signing key |
| `scripts/` | Signing and helper scripts |
| `host_include/`, `ta/` | Additional host-side and TA-side headers |

In a Buildroot tree, the dev kit is exported at:

```text
output/k3/build/optee-os-custom/out/export-ta_rv64/
```

The `optee-os-custom` directory name comes from the build directory assigned by Buildroot to the customized `optee-os` package. The `include/`, `lib/`, and `mk/` directories and `keys/default_ta.pem` listed above are all under this directory. Set `TA_DEV_KIT_DIR` to this directory when building.

### 1.2 Cross Toolchain

TA builds are cross-compilations: build on the host, run on the target.

| Item | Value |
|---|---|
| Build host | The x86_64 development host |
| Target | 64-bit RISC-V (`ta_rv64`), matching the device |
| Toolchain prefix | `riscv64-unknown-linux-gnu-` |
| Toolchain location (host) | `output/k3/host/opt/ext-toolchain/bin/`: the toolchain itself, usable outside the build container |
| Toolchain location (container) | `output/k3/host/bin/` (`/k3/host/bin/` inside the container): Buildroot wrappers requiring the container glibc |

`CROSS_COMPILE` must be a **full path prefix**, or the toolchain `bin` directory must be on `PATH`. Using only `riscv64-unknown-linux-gnu-` when it is not on `PATH` produces `riscv64-unknown-linux-gnu-gcc: not found` (make error 127). TA builds use only dev kit libraries and need no extra `--sysroot`.

## Step 2: Create the TA Directory Skeleton

A buildable TA needs at least four kinds of file: a build entry (`Makefile`), a source list (`sub.mk`), TA attributes (`user_ta_header_defines.h`), and entry-point implementations. The recommended layout matches OP-TEE conventions:

```text
my_ta/
├── Makefile                    # Declares BINARY=<uuid> and includes the dev kit build rules
├── sub.mk                      # Lists sources and header directories
├── user_ta_header_defines.h    # TA_UUID, TA_FLAGS, TA_STACK_SIZE, TA_DATA_SIZE, etc.
├── include/
│   └── my_ta.h                 # Command IDs and UUID shared with the CA side
└── my_ta.c                     # Implementations of the five entry points
```

Responsibilities of each file:

| File | Required | Responsibility |
|---|---|---|
| `Makefile` | Yes | Sets configuration variables and includes `$(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk` |
| `sub.mk` | Yes | `srcs-y` lists sources; `global-incdirs-y` lists header directories |
| `user_ta_header_defines.h` | Yes | Defines flags, stack, and heap sizes |
| Entry-point source (`.c`) | Yes | Implements the five mandatory entry points |
| `include/` | Recommended | Shared UUID and command-ID definitions to prevent drift |
| `Android.mk` | Not needed | Android build entry; unused in Buildroot |

### 2.1 Makefile

```make
# TA UUID, which determines the output filename <uuid>.ta
BINARY = <uuid>

# TA-side log level (increase during debugging, lower for production)
CFG_TEE_TA_LOG_LEVEL ?= 4
CFG_TA_OPTEE_CORE_API_COMPAT_1_1 = y

include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk
```

`BINARY` must be the TA UUID. For a static library linked by a TA, use `LIBNAME` instead; `BINARY` and `LIBNAME` are mutually exclusive.

### 2.2 sub.mk

```make
# Sources to build (relative to this directory)
srcs-y += my_ta.c

# Header search path
global-incdirs-y += include/
```

### 2.3 user_ta_header_defines.h

```c
#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

/* UUID and command IDs live in the shared header, pulled in via global-incdirs-y */
#include <my_ta.h>

/* Attribute flags: combine as needed; see Step 4 */
#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)

/* Stack and heap (TEE_Malloc pool) sizes */
#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)

#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "K3 example TA"

#endif
```

### 2.4 Generate a UUID

```bash
python -c 'import uuid; print(uuid.uuid4())'
```

Once published, the UUID must not change: installed CAs locate the TA by UUID, so changing it prevents them from finding the TA.

## Step 3: Implement the Entry Points

The five mandatory TA entry points (declared in the dev kit `include/tee_internal_api.h`):

| Entry point | Timing | Description |
|---|---|---|
| `TA_CreateEntryPoint` | When a TA instance is created | Allocate instance-level resources; failure makes the instance unusable |
| `TA_DestroyEntryPoint` | When a TA instance is destroyed | Release instance-level resources |
| `TA_OpenSessionEntryPoint` | On each session open | Validate parameters and prepare session context |
| `TA_CloseSessionEntryPoint` | On each session close | Release session context |
| `TA_InvokeCommandEntryPoint` | On each command invocation | **Core logic:** dispatch by command ID |

Include the headers and implement each entry point in the source file:

```c
#include <tee_internal_api.h>
#include <my_ta.h>                     /* UUID and command IDs */

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
	/* Validate parameter types and prepare session context */
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
	/* Dispatch by cmd_id; parameter types must match the CA side */
	return TEE_ERROR_NOT_IMPLEMENTED;
}
```

`TA_InvokeCommandEntryPoint` receives the session context, command ID, and parameters; `paramTypes` determines how parameters are accessed. **Command IDs and parameter types must match exactly on both sides**; see [Parameter Correspondence with the CA](#parameter-correspondence-with-the-ca).

## Step 4: Set Attributes and Flags

TA attributes are declared through flags and attribute strings in `user_ta_header_defines.h` and the shared header.

### Flags

| Flag | Meaning |
|---|---|
| `TA_FLAG_SINGLE_INSTANCE` | One instance per TA, shared by multiple sessions |
| `TA_FLAG_MULTI_SESSION` | Support multiple sessions; used with single instance |
| `TA_FLAG_INSTANCE_KEEP_ALIVE` | Keep the instance after the last session closes |
| `TA_FLAG_CACHE_MAINTENANCE` | Allow cache-maintenance system calls; enabled on this platform |
| `TA_FLAG_CONCURRENT` | Allow concurrent command execution; the TA must ensure thread safety |
| `TA_FLAG_DEVICE_ENUM` | Allow device enumeration without a background service |
| `TA_FLAG_DEVICE_ENUM_SUPP` | Allow device enumeration with a background service |
| `TA_FLAG_SECURE_DATA_PATH` | Use secure data path memory |

Without the concurrency flag, the TEE serializes command calls to the TA, so no internal locking is needed. With it, the TA must handle concurrency.

These values are bits in `TA_FLAGS`:

- Combine them with bitwise OR in `user_ta_header_defines.h`; unlisted bits remain 0.
- Obsolete flags `TA_FLAG_USER_MODE`, `TA_FLAG_EXEC_DDR`, and `TA_FLAG_REMAP_SUPPORT` must remain 0 and should not be used.

### Attribute Strings and Corresponding Macros

The dev kit generates attribute strings (`gpd.ta.*`) at compile time from these macros; **do not write them by hand**:

| Attribute string | Meaning | Definition in `user_ta_header_defines.h` |
|---|---|---|
| `gpd.ta.singleInstance` | Single instance | Set `TA_FLAG_SINGLE_INSTANCE` in `TA_FLAGS` |
| `gpd.ta.multiSession` | Multiple sessions | Set `TA_FLAG_MULTI_SESSION` in `TA_FLAGS` |
| `gpd.ta.instanceKeepAlive` | Keep instance alive | Set `TA_FLAG_INSTANCE_KEEP_ALIVE` in `TA_FLAGS` |
| `gpd.ta.dataSize` | Data segment (heap) size | `#define TA_DATA_SIZE (32 * 1024)` |
| `gpd.ta.stackSize` | Stack size | `#define TA_STACK_SIZE (2 * 1024)` |
| `gpd.ta.version` | Version identifier | `#define TA_VERSION "1.0"` |
| `gpd.ta.description` | Description | `#define TA_DESCRIPTION "..."` |
| `gpd.ta.endian` | Byte order | Generated by the dev kit (fixed little-endian) |

Example from `user_ta_header_defines.h`:

```c
/* The first three attributes come from TA_FLAGS: OR-combined; unlisted bits are off */
#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)

#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)
#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "K3 example TA"
```

Two notes:

- Undefined `TA_VERSION` and `TA_DESCRIPTION` default to `"Undefined version"` and `"Undefined description"`. Set them: they help identify which TA runs on a device.
- Add extra attributes with `TA_CURRENT_TA_EXT_PROPERTIES`. Some dev kit flags also produce attributes, such as `gpd.ta.doesNotCloseHandleOnCorruptObject` for `TA_FLAG_DONT_CLOSE_HANDLE_ON_CORRUPT_OBJECT`.

## Step 5: Build, Deploy, and Verify

### 5.1 Build

Run `make` in the TA directory with the cross toolchain and dev kit paths:

```bash
BR=<Buildroot output directory>                      # /k3 inside the container
KIT=$BR/build/optee-os-custom/out/export-ta_rv64

# Host uses the toolchain itself; inside the container use $BR/host/bin/riscv64-unknown-linux-gnu-
CROSS=$BR/host/opt/ext-toolchain/bin/riscv64-unknown-linux-gnu-

rm -rf out    # Clear previous intermediates first; see below
make CROSS_COMPILE=$CROSS TA_DEV_KIT_DIR=$KIT O=out -C <TA directory> all
```

The output is `<TA directory>/out/<uuid>.ta`, already signed by the build.

**Run `rm -rf out` before each build.** If a previous build ran in the Buildroot tree or container, dependency files under `out/` record container absolute paths (the container uses the staging dev kit, e.g. `/k3/host/.../sysroot/lib/optee/export-ta_rv64/src/ta.ld.S`). Rebuilding on the host reads these `.d` files and reports `No rule to make target '/k3/...'`. Removing `out/` resolves it.

### 5.2 Deploy

TA install location and permissions (see [Build Artifacts and Their Locations](tee_image.md#build-artifacts-and-their-locations) in the TEE Image Build and Flashing Guide for the directory origin):

```text
/lib/optee_armtz/<uuid>.ta      # Mode 444 (read-only)
```

**Recommended: add the TA to the Buildroot optee-examples package so it installs with the image.** See [Add a Test Case](examples.md#add-a-test-case) in TEE Examples for directory requirements and integration steps.

The package globs each `ta/` subdirectory at build time and installs `*/ta/out/*.ta` to `/lib/optee_armtz/` with mode 444, so no custom install rule is needed.

Other approaches:

| Approach | Method | Use case |
|---|---|---|
| Build with the image (recommended) | Add to optee-examples as above, or add install steps to your own package | Product integration |
| Replace at runtime | Overwrite files under `/lib/optee_armtz/` on the device | Development and debugging |

> [!CAUTION]
> If the TA must be available in initramfs, **explicitly add it to the initramfs allowlist**. TA files are not standard executables and dependency analysis cannot detect them; installing only into the root file system leaves them unavailable in early boot.

### 5.3 Verify

Confirm three things in order:

1. **File present:** `ls -l /lib/optee_armtz/<uuid>.ta` shows the file with mode `444`.
2. **Background service running:** secure-storage calls depend on `tee-supplicant`.
3. **Successful call:** invoke through a CA using the same UUID (see the [complete example](ca_guide.md#complete-example) in the CA Development Guide) or `xtest`, and confirm the expected result.

### 5.4 Signing Key

TAs are signed at build time and the TEE verifies them with the public key embedded at compile time. The signing key is set by `TA_SIGN_KEY`, which defaults to the dev kit key directory:

```text
<TA_DEV_KIT_DIR>/keys/default_ta.pem
# In this solution: output/k3/build/optee-os-custom/out/export-ta_rv64/keys/default_ta.pem
```

**Replace it with your own key before production:** place your private key in that directory and point to it (or modify `TA_SIGN_KEY`), re-sign all TAs, and rebuild the TEE so the new public key is embedded.

The signing key affects the encryption key: the TA encryption key derivation includes the signing public-key modulus, so **changing the signing key also changes the encryption key**, invalidating all encrypted TAs. Rotate both together; see [TA Encryption](#ta-encryption).

## Complete Example

Completing the skeleton from [Step 2](#step-2-create-the-ta-directory-skeleton) yields a minimal runnable TA. This example increments a supplied value.

Directory:

```text
hello_ta/
├── Makefile
├── sub.mk
├── user_ta_header_defines.h
├── include/hello_ta.h
└── hello_ta.c
```

`include/hello_ta.h` (shared by CA and TA):

```c
#ifndef HELLO_TA_H
#define HELLO_TA_H

/* TA UUID: determines the output filename and how the CA locates the TA */
#define TA_HELLO_UUID { 0x8aaaf200, 0x2450, 0x11e4, \
	{ 0xab, 0xe2, 0x00, 0x02, 0xa5, 0xd5, 0xc5, 0x1b } }

/* Command ID: must match on CA and TA */
#define CMD_INC_VALUE 0

#endif
```

`Makefile`:

```make
BINARY = 8aaaf200-2450-11e4-abe2-0002a5d5c51b
CFG_TEE_TA_LOG_LEVEL ?= 4
CFG_TA_OPTEE_CORE_API_COMPAT_1_1 = y

include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk
```

`sub.mk`:

```make
srcs-y += hello_ta.c
global-incdirs-y += include/
```

`user_ta_header_defines.h`:

```c
#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

#include <hello_ta.h>

#define TA_FLAGS        (TA_FLAG_SINGLE_INSTANCE | TA_FLAG_MULTI_SESSION)
#define TA_STACK_SIZE   (2 * 1024)
#define TA_DATA_SIZE    (32 * 1024)

#define TA_VERSION      "1.0"
#define TA_DESCRIPTION  "Hello TA (increments the supplied value)"

#endif
```

`hello_ta.c`:

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

Build and deploy as in [Step 5](#step-5-build-deploy-and-verify). The corresponding CA code is the [complete example](ca_guide.md#complete-example) in the CA Development Guide; UUIDs and command IDs must match.

## Debugging and Troubleshooting

### Logging

TAs log through `IMSG`, `DMSG`, and `EMSG`, printed by the TEE kernel to the serial port. The TA log level is set at build time; with a low level, TA messages are absent and can be mistaken for the TA not running.

### Common Problems

| Symptom | Cause | Action |
|---|---|---|
| Opening a session returns object-not-found | UUID mismatch or TA not installed | Check UUIDs on both sides and confirm the `.ta` exists under `/lib/optee_armtz/` |
| Runtime returns bad parameters | Parameter types or sizes differ | Check `paramTypes` against TA validation |
| Returns not-implemented | Command IDs differ | Check command-ID definitions |
| Secure-storage operation fails | `tee-supplicant` not running or backend unavailable | Confirm the service is running |
| No TA logs at all | TA log level too low | Raise the level and rebuild |
| Behaviour unchanged after modifying the TA | Old TA still on the device | Reinstall the `.ta` and confirm path and UUID |
| TA not found during early boot | TA not in the initramfs allowlist | Add it explicitly |
| Build reports missing `ta_dev_kit.mk` | `TA_DEV_KIT_DIR` unset or wrong | Confirm the dev kit path as in [Step 1](#step-1-prepare-the-development-environment) |
| Build reports `riscv64-unknown-linux-gnu-gcc: not found` (error 127) | Short `CROSS_COMPILE` prefix but toolchain not on `PATH` | Use the full path prefix or add the toolchain `bin` to `PATH` |
| Build reports `No rule to make target '/k3/.../export-ta_rv64/src/ta.ld.S'` | When building a TA manually in the Buildroot build directory, container-generated `.d` dependency files remain in `ta/out/` and contain paths to the staging dev kit. | Remove `ta/out/` and rebuild, or build in your own directory or via `make optee-examples-rebuild` |

## Related Documents

| Topic | Document |
|---|---|
| CA development and TEE Client API | [CA Development Guide / TEE Client API Reference](ca_guide.md) |
| Examples and test suites | [TEE Examples](examples.md) |
| TEE structure and memory constraints | [TEE Architecture and Integration Guide](tee_architecture.md) |
| Feature support | [Security Feature Support Matrix](support_matrix.md) |

## References

The external specifications referenced in this guide are listed below; versions are left blank where unknown.

| Name | Version | Link |
|---|---|---|
| GlobalPlatform TEE Internal Core API Specification (TA-side API) | 1.4 | <https://globalplatform.org/specs-library/tee-internal-core-api-specification/> |
| GlobalPlatform TEE Client API Specification (CA-side API) | 1.0.1 | <https://globalplatform.org/specs-library/tee-client-api-specification/> |
| OP-TEE documentation (TA development, signing, and encryption) | latest | <https://optee.readthedocs.io/en/latest/> |
