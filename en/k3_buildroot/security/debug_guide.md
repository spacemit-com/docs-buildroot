---
sidebar_position: 10
---

# TEE Debugging Guide

## Overview

### Purpose

This guide describes how to debug the TEE on K3, including boot-stage diagnosis, runtime troubleshooting, and live debugging with JTAG/GDB. It helps developers distinguish build configuration problems, environmental problems, and functional defects.

Two distinct capabilities must not be confused:

- **Debugging capability:** Whether JTAG/GDB, breakpoints, single-stepping, register and memory access, panic/recovery, and source-level symbol lookup are permitted.
- **Logging capability:** Whether TEE Core, ldelf, and user TAs output ERROR, INFO, DEBUG, and FLOW logs.

> [!CAUTION]
> A high log level does not mean JTAG is enabled, and a JTAG connection does not mean the device permits debugging. The capabilities are controlled by different mechanisms: logging by build configuration and lifecycle state, and JTAG by eFuse hardware gating.

### Scope

This guide applies to Buildroot solutions for SpacemiT K3 series SoCs. For platform architecture and boot sequencing, see the [TEE Architecture and Integration Guide](tee_architecture.md). For examples and test suites, see [TEE Examples](examples.md).

### Document Structure

1. **Debugging Capability and Lifecycle:** Capability boundaries, the LCS lifecycle, and current JTAG controls.
2. **Boot-Stage Diagnosis:** Checks for each stage of the boot chain.
3. **Runtime Diagnosis:** OP-TEE logging, common faults, and diagnostic criteria.
4. **Live Debugging with JTAG/GDB:** Hardware gating, the official connection guide, and symbol-file location.
5. **Additional Debugging Tools.**
6. **Frequently Asked Questions (FAQ).**

## Debugging Capability and Lifecycle

### LCS Lifecycle

K3 uses **LCS (Lifecycle State)** in eFuse to distinguish device lifecycle stages and determine logging and debugging capabilities.

| Lifecycle | Code enum | Device state | Logging capability |
|---|---|---|---|
| CM | `K3_LCS_CM` | Engineering/development board bring-up | ERROR/INFO/DEBUG/FLOW allowed |
| DM | `K3_LCS_DM` | Production testing/development validation | ERROR/INFO/DEBUG/FLOW allowed |
| SP | `K3_LCS_SP` | Production device | ERROR/INFO allowed |
| RMA | `K3_LCS_RMA` | After-sales repair | ERROR/INFO/DEBUG/FLOW allowed |
| UNKNOWN | `K3_LCS_UNKNOWN` | LCS error/read failure | Core runtime threshold falls back to ERROR |

**All debug log levels are enabled by default in non-production stages (CM/DM/RMA)**; SP retains only ERROR/INFO. The default log level is set at build time and controlled by LCS at runtime. Only OP-TEE Core logging changes with LCS; **ldelf and user TA logs are unaffected by LCS and always follow their build configuration**.

### Current JTAG Controls

K3 exposes a JTAG Debug Module. Evaluate the current implementation below rather than assuming that JTAG is disabled by default in production:

- **eFuse disable bit:** `JTAG_DISABLE` at eFuse bank 0 bit 14 is the **only** currently implemented hardware JTAG disable switch. Once programmed, a debugger cannot enumerate the target and JTAG is unavailable, except in RMA as described under [Verify JTAG DISABLE](#verify-jtag-disable).
- **Disabling JTAG is not mandatory for production devices:** The product determines whether to program `JTAG_DISABLE`; disabling it is not currently enforced by default.
- **Authenticated debug is not yet implemented.**
- **IOPMP controls the debug channel, but the current configuration permits all access:** The SID0 entry in IOPMP INT0 controls access to the debug channel but currently blocks no accesses, so it does not restrict debugging.

Thus, JTAG connectivity depends on whether the `JTAG_DISABLE` fuse is programmed. Authenticated debug is not implemented, and the current IOPMP configuration imposes no additional restrictions.

## Boot-Stage Diagnosis

For boot failures, troubleshoot each stage of the boot chain in order:

| Stage | Expected behavior | Checks if absent |
|---|---|---|
| SPL loads the TEE image | No success message by default; the OP-TEE loading result is printed **only when Secure Boot is enabled in SPL** | Check whether the TEE partition exists in the secure partition table, whether the image was written there, and whether a normal-image partition table was used by mistake. |
| OpenSBI starts OP-TEE | OP-TEE version banner appears | Check that OpenSBI contains the dispatcher symbol and that the device tree contains domain-gating nodes. |
| OP-TEE initialization | No fatal error after the banner | Check platform clock, random-number generator, eFuse, and security-engine initialization. |
| Kernel probes TEE | TEE driver reports successful startup | Check kernel TEE and OP-TEE drivers and the TEE device-tree node. |
| `tee-supplicant` | Process is present | Check the initialization script and whether the TA directory is mounted. |

Three common misdiagnoses:

1. **No new serial output at all:** Flashing usually failed before the TEE stage. Check the image package file set, especially common files required by the packaging manifest.
2. **No OP-TEE banner and no error:** The device tree often lacks domain-gating nodes, so OpenSBI does not configure a trusted domain and silently proceeds to normal boot.
3. **SPL logs appear but nothing follows:** Check whether the TEE image exceeds the partition capacity. An oversized image may be written incompletely and appear to load successfully but fail at execution.

## Runtime Diagnosis

### OP-TEE Logging Architecture

All TEE logs are output on the **TEE serial port**. On the normal-system side, only a small number of driver logs are visible through kernel `dmesg`. **The two observation channels are not interchangeable.**

The secure world has three logging layers, distinguished by their prefixes:

| Module | Prefix | Description |
|---|---|---|
| Core | `I/TC`, `E/TC` | OP-TEE kernel and platform initialization logs |
| ldelf (loader) | `I/LD`, `E/LD` | TA loader logs |
| User TA | `I/TA`, `E/TA` | TA application logs |

### Log-Level Configuration

Log levels are controlled by build configuration. Higher levels provide more detail but increase image size and runtime overhead:

| Option | Purpose | Values |
|---|---|---|
| `CFG_TEE_CORE_LOG_LEVEL` | OP-TEE kernel and platform log level | 0 off / 1 ERROR only / 2 adds INFO / 3 adds DEBUG / 4 adds FLOW (most detailed) |
| `CFG_TEE_TA_LOG_LEVEL` | User TA log level | Same as above |

The platform currently defaults to `CFG_TEE_CORE_LOG_LEVEL ?= 4` and `CFG_TEE_TA_LOG_LEVEL=4` for development. **Reduce these levels in production builds:** detailed logs expose internal state over serial and increase image size when the TEE partition has limited capacity.

### Check the Layers First

```bash
# Device nodes
ls /dev/tee*

# Background service
ps | grep tee-supplicant

# Kernel-side TEE logs
dmesg | grep -i tee
```

### Common Faults

| Symptom | Most likely cause | Action |
|---|---|---|
| CA receives an object-not-found error when opening a session | TA is not installed or UUID does not match | Check the TA directory and UUID. |
| CA returns a bad-parameters error | Parameter types do not match the TA | Check each `paramTypes` value and ensure the operation structure is zero-initialized. |
| Secure storage reports storage unavailable | `tee-supplicant` is not running or the storage backend is unavailable | Start the service and check the backend configuration. |
| Many `xtest` cases are skipped | Corresponding features are not enabled in the build | This is a build-configuration issue; enable the required features and rebuild as needed. |
| RPMB-related tests fail | RPMB key has not been provisioned or RPMB is disabled | Check the build configuration and device provisioning state. |
| An example runs but no TEE logs appear | Log level is too low | Increase the log level as needed and rebuild. |
| Random-number output is constant | Random-number source is not active | Check whether the platform random-number driver is being used. |

### Identify the Error Origin

CA error returns include an origin field, providing a quick way to identify the failing layer:

| Origin | Indicates |
|---|---|
| Client library | Incorrect API usage, such as a NULL parameter or uninitialized context |
| Communication layer | Environment issues, such as a stopped service, missing device node, or insufficient permissions |
| TEE kernel | Platform or kernel-side problem |
| TA | TA logic, parameter validation, or storage operation |

Check the communication layer and TA first: most integration problems occur in these layers.

## Live Debugging with JTAG/GDB

K3 exposes a JTAG Debug Module. The debug path is **J-Link hardware probe -> OpenOCD (PC) -> TCP :1024 -> RISC-V GDB**. See Current JTAG Controls above for hardware gating.

For JTAG connection, J-Link/OpenOCD/GDB driver installation, connection steps, and operations such as halt/resume, breakpoints, single-stepping, and call stacks, see the official [JTAG Debugging Tool User Guide](https://www.spacemit.com/community/document/info?lang=en&nodepath=tools/user_guide/jtag_debug_user_guide.md).

Source-level debugging requires the OP-TEE ELF file with symbols, `tee.elf`, from the build output:

```text
optee/optee_os/out/riscv/core/tee.elf
```

When using `O=out/riscv`, the artifact is `core/tee.elf`. It must match the image flashed to the board.

## Additional Debugging Tools

### Build-Side Checks

```bash
# OP-TEE build artifacts (ELF with symbols and raw binary)
ls -l optee/optee_os/out/riscv/core/tee.elf optee/optee_os/out/riscv/core/tee.bin

# TEE image size (the optee partition is approximately 512 KB; an oversized image may be written incompletely)
ls -l optee/optee_os/out/riscv/core/tee.bin
```

### Verify JTAG DISABLE

After programming the `JTAG_DISABLE` fuse (eFuse bank 0 bit 14), use OpenOCD connection behavior to check that JTAG is disabled. Typical output is:

```text
Info : VTarget = 1.786 V
Error: JTAG scan chain interrogation failed: all zeroes
Error: post.unknown: IR capture error; saw 0x0 not 0x1
Warn : Bypassing JTAG setup events due to errors
```

Interpretation:

- A stable `VTarget` reading indicates that the target is powered and the reference voltage is connected, excluding a disconnected cable or unpowered target.
- A scan result of **all zeroes**, rather than all ones, is consistent with the target actively pulling TDO low after the TAP is disabled. A floating or disconnected TDO normally reads all ones.
- OpenOCD may continue using its configured scan chain and start the GDB port (`Trying to use configured scan chain anyway`). This does not indicate a successful connection; subsequent target operations fail.

To rule out pin multiplexing by firmware, a TDO short, or a mode mismatch, compare with an unprogrammed board of the same model using the same connections and scripts. Its scan should succeed and return an IDCODE. Also confirm that the test board produces normal serial boot logs.

> [!CAUTION]
> JTAG is not disabled in every lifecycle state. In RMA, JTAG is re-enabled regardless of the disable bit and the boot chain no longer verifies signatures; the software boot gate allows RMA with a warning. If JTAG remains accessible on a board with JTAG DISABLE programmed, first check whether the eFuse lifecycle state is RMA, then troubleshoot the connection. See the security lifecycle section of the [Security Architecture White Paper](white_paper.md).
>
> The disable bit is eFuse bank 0 bit 14. The `SYS_BOOT_CNTRL` register read early in firmware boot does not independently store this state; it maps several eFuse bits. Refer to the fuse bit during diagnosis rather than treating the register bit number as the fuse bit number.

## Frequently Asked Questions (FAQ)

1. **Can production devices never connect through JTAG?** Not necessarily. Programming `JTAG_DISABLE` is not mandatory or enforced by default; the product decides. Authenticated debug is not implemented, and the current IOPMP configuration permits all debug-channel access.
2. **Which logs are enabled by default in non-production stages?** CM/DM/RMA enable ERROR/INFO/DEBUG/FLOW; SP retains only ERROR/INFO.
3. **Does a high log level mean JTAG is enabled?** No. Logging is controlled by build configuration and LCS; JTAG is controlled by eFuse hardware gating. They are independent.

## Information to Include in a Problem Report

- Device LCS state and whether eFuse has been programmed, as these affect log levels and JTAG behavior
- Whether a secure or normal image is used, and the timestamp in its filename
- Complete serial logs starting from SPL
- Output of `dmesg | grep -i tee`
- Exact reproduction commands, return codes, and error origins