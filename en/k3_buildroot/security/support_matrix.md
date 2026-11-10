---
sidebar_position: 1
---

# Security Feature Support Matrix

| Item | Details |
|---|---|
| Applicable SoC | SpacemiT K3 |
| Applicable solution | Buildroot (secure image) |
| TEE version | `k3-v1.0` (consistent with the OP-TEE build configuration) |
| Matrix version | k3-v1.0 (2026-10-12) |

The status assessments and minimum software versions in this matrix apply to the version above. A feature is unavailable below its stated version; after an upgrade, reconfirm the prerequisites in the notes.

This matrix covers the k3-v1.0 secure system, not production-test configurations. Status values are defined in the legend below. Items whose status has not yet been decided are treated as not supported. The matrix will be updated with subsequent releases.

## Overview

### Purpose

This matrix summarizes security feature support for SpacemiT K3 with Buildroot by feature, SoC model, and software version. It provides status and minimum version requirements for solution selection, requirements review, and production evaluation.

### Scope

This document applies to Buildroot solutions for SpacemiT K3 series SoCs. For the secure boot chain of trust, see the [Secure Boot Development Guide](../device/secureboot.md). For TEE structure and integration, see the [TEE Architecture and Integration Guide](tee_architecture.md).

> [!IMPORTANT]
> **Boot media capacity requirements**
>
> The current secure image has the following boot media requirements:
>
> - **Capacity:** At least 8 MB. A 4 MB SPI-NOR device cannot accommodate the secure `u-boot.itb`: at approximately 2383 KB, it exceeds the 1536 KB partition allocation. Devices of 8 MB or more are sufficient; 8 MB SPI-NOR devices are available.
> - **Supported media:** SPI-NOR of at least 8 MB, SPI-NAND, or eMMC, SD, or UFS.
> - **Details:** See [Storage Media Differences](#storage-media-differences) and the [TEE Image Build and Flashing Guide](tee_image.md).

### Status Legend

| Status | Meaning |
|---|---|
| ✓ Supported | Implemented and verified by testing or build validation; ready for use in products |
| ✗ Not supported | Not implemented, or implemented but disabled by default with no defined enablement path |
| Partially supported | Available with restrictions; use only under the conditions in the notes |
| Disabled | Code is ready but disabled in the current build; it can be enabled through configuration and rebuilding |
| Planned | Planned but not currently available |

The minimum software version identifies the first software baseline on which a feature is available. The feature is unavailable below that version; after upgrading, reconfirm the prerequisites in the notes.

> [!NOTE]
> There is currently one SoC model (K3) and one software baseline, so the matrix has a single status column. Future SoCs or versions can be added as columns in the same format without changing the structure.

## SoC Model

| SoC | Core | Storage support | Notes |
|---|---|---|---|
| K3 | RISC-V `spacemit,x100` (`rv64imafdcvh`) | eMMC/SD, UFS, SPI-NOR, NAND | Current baseline |

## Security Features by SoC and Software Version

### Root of Trust and Secure Boot

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| Root of trust (RoT) | `k3-v1.0` | ✓ Supported | On-chip read-only, immutable BootROM and eFuse key material; the RoT public-key hash is in bank 4. This is a hart firmware root of trust in the specification's classification; see [white paper section 5](white_paper.md#5-root-of-trust). |
| Secure Boot (staged FIT signature verification) | `k3-v1.0` | Partially supported | Requires ROTPKH in eFuse bank 4. BootROM verifies FSBL according to the secure boot bit; verification of subsequent images is enabled in the SPL configuration. When verification is disabled, boot proceeds without signature verification. |
| ROTPKH storage | `k3-v1.0` | ✓ Supported | eFuse bank 4, 256 bits, containing the SHA-256 hash of the root public key. |
| FSBL certificate chain | `k3-v1.0` | ✓ Supported | `root_key` + `spl_key`; `oem_key` carries the SPL and U-Boot public keys. |
| FIT public-key injection | `k3-v1.0` | ✓ Supported | SPL DTB and each board-level U-Boot DTB. |
| Anti-rollback counters | `k3-v1.0` | Disabled | Counter fields are defined (ARCN-NS / ARCN-Sec), but are not enabled or checked. |
| Image partition replacement (updatability) | `k3-v1.0` | ✓ Supported | Partition images can be replaced independently; measurement and signature verification are not performed before boot. See [white paper section 12](white_paper.md#12-system-updates). |
| TEE image signing | `k3-v1.0` | Partially supported | The default FIT description includes CRC32 only, without RSA signature verification. `KEY_DIR` can be supplied to package a signed FIT, but the current build flow does not supply it. |

### Key Hierarchy and eFuse

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| Read-only eFuse access (secure world) | `k3-v1.0` | ✓ Supported | CPU reads the GEU register block directly; 12 banks of 256 bits each. |
| eFuse programming | `k3-v1.0` | ✓ Supported | Performed on the production line using a dedicated programming tool. By design, the secure world provides no write path. |
| eFuse readback verification | `k3-v1.0` | ✓ Supported | Per-bank lock bits can be read. |
| TE200 access to eFuse | `k3-v1.0` | ✓ Supported | The security engine's key ladder retrieves the root key from eFuse internally, without CPU involvement. Other eFuse contents are read by the CPU-side read-only driver. |
| HUK origin and derivation | `k3-v1.0` | ✓ Supported | The SoC manufacturer programs the root key into eFuse. The security engine's key ladder derives working and subordinate keys internally. The CPU cannot read the root key, and software never accesses it. |
| Boot policy without programmed fuses | `k3-v1.0` | Partially supported | Boot is possible in CM/DM stages (DM requires a programmed chip ID), but derived keys are not device-unique. SP requires chip ID, HUK, and secure boot fuses. |
| Security engine HUK self-test | `k3-v1.0` | Disabled | Controlled by `CFG_TE200_HUK_SELFTEST`, disabled by default. When enabled, it logs CMAC/KDF and root-key status at boot; a failure produces a warning but does not block boot. |
| Subkey derivation | `k3-v1.0` | ✓ Supported | RPMB / SSK / DIE_ID / UNIQUE_TA / TA_ENC / SE050; derived from a working key inside the security engine using CMAC and KDF. |
| Storage key hierarchy (SSK/TSK/FEK) | `k3-v1.0` | ✓ Supported | Derived in stages from the HUK. |
| Core hardening (ASLR, stack protection, etc.) | `k3-v1.0` | Partially supported | The platform explicitly disables address randomization and stack protection. |

### Secure Storage

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| REE FS storage | `k3-v1.0` | Partially supported | Default backend; **no rollback protection**. |
| REE FS integrity with RPMB | `k3-v1.0` | Disabled | Depends on the RPMB file system (`CFG_REE_FS_INTEGRITY_RPMB`); currently disabled. |
| RPMB storage (eMMC) | `k3-v1.0` | Disabled | `CFG_RPMB_FS` is disabled by default; the device must be provisioned with an RPMB key before RPMB storage is enabled. |
| RPMB storage (UFS) | `k3-v1.0` | Disabled | As with eMMC, controlled by `CFG_RPMB_FS`. The probe interface supports UFS, but block-count and CRC handling differ from eMMC. |
| RPMB key derivation | `k3-v1.0` | ✓ Supported | Derived from HUK and the device identifier. eMMC uses its native CID with variable fields (PRV and CRC) masked; UFS uses a hashed device_id. |
| RPMB provisioning target selection | `k3-v1.0` | Disabled | Enabled in production-test builds with `CFG_RPMB_WRITE_KEY_CID`; the target can be selected by device identifier to avoid programming the wrong device. |
| RPMB probe capability announcement | `k3-v1.0` | Disabled | `CFG_RPMB_ANNOUNCE_PROBE_CAP` is enabled by default and announces probe capability to the kernel, but `CFG_RPMB_FS` is disabled, so probing returns an empty device list. |
| tee-supplicant RPMB emulation | `k3-v1.0` | Disabled | Explicitly disabled to avoid file-based ioctl emulation. |

### Cryptography and Random Numbers

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| Hardware PRNG | `k3-v1.0` | ✓ Supported | Uses the platform hardware random number generator; software backend disabled. |
| Asymmetric cryptographic acceleration | `k3-v1.0` | ✓ Supported | RSA, ECDSA, SM2DSA; curves P192-P521. |
| Symmetric cryptographic acceleration | `k3-v1.0` | ✓ Supported | AES, SM4; ECB/CBC/CTR. |
| Security engine hash | `k3-v1.0` | Disabled | Controlled by `CFG_TE200_HASH_DRV`; software implementation is used when disabled. |
| Security engine TRNG | `k3-v1.0` | ✗ Not supported | Not supported. |

### TEE Runtime and Integration

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| OP-TEE OS (S-mode / RISC-V) | `k3-v1.0` | ✓ Supported | Platform `plat-k3`. |
| OpenSBI OPTEED dispatcher | `k3-v1.0` | ✓ Supported | Enables OP-TEE dispatch and MPXY transport. |
| SPL preload of TEE image | `k3-v1.0` | ✓ Supported | Boot is aborted if the image is missing on MMC/MTD paths. |
| Dedicated TEE partition | `k3-v1.0` | ✓ Supported | eMMC/SD: 4 MB; SPI-NOR: 512 KB. |
| Memory layout (reserved shared memory) | `k3-v1.0` | ✓ Supported | TEE RAM and shared memory have fixed addresses. |
| Dynamic shared memory | `k3-v1.0` | Disabled | Reserved shared memory is used instead. |
| Kernel TEE/OPTEE driver | `k3-v1.0` | ✓ Supported | Includes MPXY mailbox and protocol drivers. |
| Automatic tee-supplicant startup | `k3-v1.0` | ✓ Supported | Started by an initialization script. |
| Multicore and multithreading | `k3-v1.0` | ✓ Supported | 16 cores and 16 threads. |
| Secure device tree (domain gating) | `k3-v1.0` | ✓ Supported | Provides trusted and untrusted domain nodes. |

### TA, CA, and User Space

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| User TA | `k3-v1.0` | ✓ Supported | Targets 64-bit RISC-V. |
| TA floating point | `k3-v1.0` | ✗ Not supported | Disabled to reduce the attack surface. |
| TA cache maintenance API | `k3-v1.0` | ✓ Supported | For TAs declaring the cache maintenance attribute. |
| Built-in early TA | `k3-v1.0` | ✓ Supported | Trusted Keys. |
| TEE Client API (libteec) | `k3-v1.0` | ✓ Supported | CA-side programming interface. |
| Test suite (xtest) | `k3-v1.0` | ✓ Supported | Eight test suite groups. |
| Official examples | `k3-v1.0` | ✓ Supported | Seven examples. |
| PKCS#11 | `k3-v1.0` | ✓ Supported | User-space components (opensc / pcsc / p11-kit) can be manually included in Buildroot. |
| RTC PTA | `k3-v1.0` | ✓ Supported | Exposes the platform RTC to the normal world. |
| PTA device enumeration | `k3-v1.0` | ✓ Supported | Enables the normal world to probe TEE storage devices. |
| TA loading encryption | `k3-v1.0` | Disabled | Build option `CFG_ENCRYPT_TA` is disabled by default. **The default encryption key is a fixed value publicly available in the source code** and must be replaced before enabling this option. |
| fTPM | `k3-v1.0` | ✗ Not supported | Not currently provided. |

### Access Control and Other Features

| Feature | Minimum software version | K3 (current baseline) | Notes |
|---|---|---|---|
| Address translation and protection (MMU) | `k3-v1.0` | ✓ Supported | Isolates peer workloads: TAs run in U-mode, with access limited by TEE OS page tables, and do not share memory by default. **SPMP is not used.** |
| PMP | `k3-v1.0` | ✓ Supported | Hart-side domain isolation configured by OpenSBI per domain. The number of rule entries is fixed, so domains should not be excessively fragmented. |
| IOPMP | `k3-v1.0` | ✓ Supported | Nine checker instances (IOPMP4 is always disabled), all using static SID (AxUSER). For instance-to-master and address-space mappings, see [white paper section 9.1](white_paper.md#91-k3-iopmp-deployment). |
| Debug access control | `k3-v1.0` | Partially supported | Debug masters (DBG_AW/DBG_AR) are constrained by IOPMP6 policy; debug access can be disabled for secure regions. JTAG can be permanently disabled by fuse. See [white paper section 10](white_paper.md#10-secure-debugging). |
| Security lifecycle | `k3-v1.0` | Partially supported | eFuse fields select CM / DM / SP / RMA states, which determine access to secure assets and boot decisions. See [white paper section 11](white_paper.md#11-security-lifecycle). |
| Secure time source | `k3-v1.0` | ✓ Supported | SoC CLINT/ACLINT machine timer at 24 MHz, matching device-tree `timebase-frequency`; read by the secure world in S-mode using `rdtime`. |
| TEE core log level | `k3-v1.0` | Partially supported | Currently at the most verbose level; reducing it is recommended for production. |

## Storage Media Differences

The following table highlights differences in feature behavior across storage media.

| Aspect | eMMC | UFS | SPI-NOR | NAND |
|---|---|---|---|---|
| Can run secure image | Yes | Yes | Yes (8 MB or more; 4 MB cannot accommodate secure U-Boot) | Yes (8 MB or more) |
| Secure image TEE partition | 4 MB | Same as eMMC | 512 KB | 512 KB |
| RPMB availability | Yes | Yes | No | No |
| Device identifier handling for RPMB key derivation | Native CID (16 bytes), with variable PRV and CRC fields masked | No native CID; device_id hashed to 16 bytes with BLAKE2s | Not applicable | Not applicable |
| RPMB read block-count semantics | Block-count field in the frame must be 0 | Device obtains the length from the frame field | Not applicable | Not applicable |
| SPL behavior when TEE image is missing | Abort boot | Remains permissive | Abort boot | Abort boot |
| Known restrictions | — | — | 4 MB cannot accommodate secure U-Boot within the partition allowance | Same as SPI-NOR |

## Revision History

| Version | Date | Change |
|---|---|---|
| k3-v1.0 | 2026-10-12 | Initial release: support status and minimum version requirements by feature, SoC model, and software version. Statuses are supported, partially supported, disabled, not supported, and planned; undecided items are treated as not supported. Subsequent changes are incorporated into this entry. |