---
sidebar_position: 9
---

# TEE Examples

## Overview

### Purpose

This guide describes how to run the TEE examples and test suites included in the K3 image, their expected output, and how to interpret results. It provides a starting point for functional verification and further development.

### Scope

This guide applies to SpacemiT K3-series SoCs running a Buildroot-based secure image. See the [TEE Image Build and Flashing Guide](tee_image.md) for image preparation and the [CA Development Guide / TEE Client API Reference](ca_guide.md) for CA development.

### Document Structure

1. **Prerequisites**.
2. **Official Examples:** Seven examples included in the image.
3. **Add a Test Case:** Add a custom example to optee-examples and install it in the image.
4. **xtest Test Suite:** Suite groups and result interpretation.
5. **PKCS#11 Example**.
6. **Interpreting Results and Common Issues**.

## Prerequisites

Boot a secure image and confirm that the TEE environment is ready:

```bash
# Check TEE device nodes
ls /dev/tee*

# Check the service started by the initialization script
ps | grep tee-supplicant

# List installed TAs
ls /lib/optee_armtz/
```

Examples using secure storage fail if `tee-supplicant` is not running.

## Official Examples

The image includes the following seven examples:

| Example | Demonstrates |
|---|---|
| `optee_example_hello_world` | Basic CA and TA interaction |
| `optee_example_aes` | AES encryption and decryption in a TA |
| `optee_example_random` | Random-number generation by a TA |
| `optee_example_secure_storage` | Creating, reading, and deleting secure-storage objects in a TA |
| `optee_example_acipher` | Asymmetric encryption in a TA |
| `optee_example_hotp` | HOTP calculation using a shared key |
| `optee_example_plugins` | Plugin mechanism |

### Hello World

```bash
optee_example_hello_world
```

CA output:

```text
Invoking TA to increment 0
TA incremented value to 1
```

The serial console also shows the TA log:

```text
I/TA: Hello World!
```

The outputs have different sources: the CA prints `Invoking` and `TA incremented` to standard output; the TA prints `Hello World!` through TEE logging. Seeing only the CA output indicates successful interaction but a low TEE log level.

### AES Encryption and Decryption

```bash
optee_example_aes
```

The output covers session setup, encoding, key loading, vector reset, and buffer encoding, followed by decoding and a comparison of the plaintext and decoded result:

```text
Clear text and decoded text match
```

`Clear text and decoded text differ => ERROR` indicates an encryption/decryption mismatch.

### Random Numbers

```bash
optee_example_random
```

```text
Invoking TA to generate random UUID...
TA generated UUID value = 0x...
```

The UUID should differ between runs. If repeated results are identical, check the random-number source configuration.

### Secure Storage

```bash
optee_example_secure_storage
```

The example runs two object tests. The first creates, reads back, and deletes an object; the second checks whether an object exists and handles each case accordingly.

```text
Test on object "myobj.1"
- Create and load object in the TA secure storage
- Read back the object
- Delete the object
```

Key points for interpreting the output:

- `- Object not found in TA secure storage, create it.` and `- Object found in TA secure storage, delete it.` are normal branches in the second test, depending on whether an earlier run left the object behind.
- `Command WRITE_RAW failed` or `Command READ_RAW failed` indicates an error, commonly an unavailable storage backend.

### Asymmetric Encryption

```bash
optee_example_acipher <key_size> <string to encrypt>
```

Without arguments, the command prints usage. Supply the key size in bits and the string to encrypt. On success, it outputs encrypted bytes:

```text
Encrypted buffer: xx xx xx ...
```

### HOTP

```bash
optee_example_hotp
```

The output contains the registered shared key and computed HOTP value:

```text
Register the shared key: xx xx ...
HOTP: <value>
```

`Got unexpected HOTP from TEE!` indicates that the computed value differs from the expected value.

### Plugins

```bash
optee_example_plugins
```

This example demonstrates the plugin mechanism and requires normal-world logging and plugin support. It may not complete if those components are missing; this is expected and does not by itself indicate a TEE environment fault.

## Add a Test Case

Add a case containing a TA, a CA, or both to your optee-examples source tree. **No Buildroot package-definition change is required:** the package already builds and signs TAs, installs them to `/lib/optee_armtz/` with mode 444, and packages them into the image.

### Directories and Files

Create `<my_example>/` using `hello_world` as a template:

| File | Required | Purpose |
|---|---|---|
| `ta/Makefile` | Yes | TA build entry point. The Buildroot hook discovers TAs using `*/ta/Makefile`; without it the TA is neither built nor installed. |
| `ta/sub.mk` | Yes | TA source and header list (`global-incdirs-y`, `srcs-y`) |
| `ta/<my_example>_ta.c` | Yes | TA implementation |
| `ta/include/<my_example>_ta.h` | Yes | UUID and command IDs shared by the CA and TA |
| `ta/user_ta_header_defines.h` | Yes | Attribute strings and TA UUID; the filename must not change |
| `CMakeLists.txt` | When a CA is provided | Defines the CA executable using `add_executable`, `target_include_directories(... ta/include)`, `target_link_libraries(... teec)`, and `install(TARGETS ... DESTINATION ${CMAKE_INSTALL_BINDIR})` |
| `host/main.c` | When a CA is provided | CA source |
| Root `Makefile` | No | For standalone builds outside Buildroot; follow `hello_world/Makefile` to recursively invoke `host/` and `ta/` |

### Adapting hello_world

The simplest approach is to copy the whole `hello_world/` directory. **Three things must change:** directory and filenames, UUID, and command IDs.

| File | Changes |
|---|---|
| `hello_world/` directory | Rename to your example name, such as `my_demo/`; it is an identifier and does not affect the build. |
| `ta/Makefile` | Replace `BINARY=8aaaf200-2450-11e4-abe2-0002a5d5c51b` with the new TA UUID; see UUID generation in the [TA Development Guide](ta_guide.md). |
| `ta/include/hello_world_ta.h` | Rename to `<my_demo>_ta.h`; replace `TA_HELLO_WORLD_UUID` and `TA_HELLO_WORLD_CMD_*` with your UUID and command IDs. CA and TA command IDs must agree. |
| `ta/user_ta_header_defines.h` | **Do not rename this file.** Replace the include with your header; keep `TA_UUID` referring to the shared UUID macro; replace example strings and property names in `TA_VERSION`, `TA_DESCRIPTION`, and `TA_CURRENT_TA_EXT_PROPERTIES`; adjust `TA_FLAGS`, `TA_STACK_SIZE`, and `TA_DATA_SIZE` as needed. See [Step 4: Set Attributes and Flags](ta_guide.md#step-4-set-attributes-and-flags). |
| `ta/sub.mk` | Replace `srcs-y += hello_world_ta.c` with your source files, one per line; adjust `global-incdirs-y` if headers are outside `include/`. |
| `ta/hello_world_ta.c` | Rename to `<my_demo>_ta.c` and include your header. Retain the five entry points (`TA_CreateEntryPoint`, `TA_DestroyEntryPoint`, `TA_OpenSessionEntryPoint`, `TA_CloseSessionEntryPoint`, `TA_InvokeCommandEntryPoint`); rewrite `switch (cmd_id)` for your commands. The example uses `inc_value` and `dec_value`. |
| `CMakeLists.txt` | For a CA, replace `project (optee_example_hello_world C)` with your project name, which becomes the executable installed under `/usr/bin/`; adjust `set (SRC host/main.c)` as needed. |
| `host/main.c` | For a CA, include your header, use the same UUID in `TEEC_UUID uuid = TA_HELLO_WORLD_UUID;`, replace `TEEC_InvokeCommand()` IDs, and match `paramTypes` and `params` to the TA. |
| Root `Makefile` | No change needed; it only invokes `host/` and `ta/` recursively. |

The UUID appears in two places and must match:

```make
# ta/Makefile: names and signs the build artifact
BINARY=<new UUID>
```

```c
/* ta/include/<my_demo>_ta.h: shared by CA and TA */
#define TA_MY_DEMO_UUID { 0x00112233, 0x4455, 0x6677, { 0x88, 0x99, 0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff } }
```

`#define TA_UUID <new UUID macro>` in `ta/user_ta_header_defines.h` and `TEEC_UUID uuid = <new UUID macro>;` in the CA reference the same macro, so change it once in the header. By convention, use `TA_<UPPERCASE_EXAMPLE_NAME>_UUID` and `TA_<UPPERCASE_EXAMPLE_NAME>_CMD_<ACTION>`; keep macro and file names consistent.

### Why Buildroot Does Not Need Changes

- **CA:** The top-level optee-examples `CMakeLists.txt` uses `file(GLOB dirs *)` to iterate subdirectories and calls `add_subdirectory` for each one containing a `CMakeLists.txt`.
- **TA:** The `optee-examples.mk` build hook collects TA Makefiles with `$(wildcard $(@D)/*/ta/Makefile)`; the install hook installs `*/ta/out/*.ta` to `/lib/optee_armtz` with mode 444. Both use wildcards, so no registration is needed.

With the files in place, `make optee-examples-rebuild` builds the new case.

### Include the New Case in the Image

optee-examples normally fetches upstream source. To include your case, place a source tree locally and override the package source directory:

1. Copy the optee-examples source locally, for example to `package-src/optee-examples`, and add the new example directory.
2. Add the following line to `buildroot-ext/local.mk`, which already uses this pattern for U-Boot, OpenSBI, and optee_os:

```make
OPTEE_EXAMPLES_OVERRIDE_SRCDIR = $(TOPDIR)/../package-src/optee-examples
```

3. Run `make optee-examples-rebuild`, then `make` to repackage the image.

The CA is installed at `/usr/bin/<PROJECT_NAME>`, where `PROJECT_NAME` comes from the example's `CMakeLists.txt`; the TA is installed at `/lib/optee_armtz/<uuid>.ta` with mode 444.

```bash
ls output/k3/target/lib/optee_armtz/*.ta        # Installed TAs
ls output/k3/target/usr/bin/optee_example_*     # CAs, if provided
```

On the device, confirm that the TA file is present with mode 444, `tee-supplicant` is running, and a CA using the same UUID gets the expected result. See [5.3 Verify](ta_guide.md#53-verify) in the TA Development Guide. If the TA must be available in initramfs, **explicitly add it to the initramfs allowlist**: TAs are not standard executables and dependency analysis cannot identify them automatically. Installing only in the root file system leaves them unavailable during early boot.

## xtest Test Suite

```bash
xtest
```

Without arguments, all suites run. Run a specific suite group with:

```bash
xtest -t regression_6000
```

Suite groups:

| Suite | Contents |
|---|---|
| `regression_1000` | Core self-tests and PTA parameters |
| `regression_2000` | TCP socket API tests |
| `regression_4000` | TEE Internal API hash operations |
| `regression_4100` | TEE Internal API big-integer operations |
| `regression_5000` | GlobalPlatform TEEC interfaces |
| `regression_6000` | Secure storage (object creation) |
| `regression_8000` | TEE Internal API key-derivation extensions |
| `regression_8100` | TA-side mbedTLS self-tests and certificate chains |

Interpreting results:

- On normal completion, the number of failed cases should be 0.
- Individual results are OK, SKIP, or FAIL. **SKIP is not a failure**; it usually means a required feature was not enabled in the build.
- Socket suites may be skipped entirely when network configuration is absent; this is expected.

List cases in a suite with:

```bash
xtest -l 2>&1 | head
```

## PKCS#11 Example

**PKCS#11 (Cryptoki)** is a cryptographic token interface standard that defines a platform-independent interface to cryptographic tokens, such as hardware security modules and smart cards that hold keys. It standardizes objects such as RSA keys, certificates, and symmetric keys and their creation, use, and deletion, so applications need not use each token's proprietary interface. Its main capabilities are token management (loading and configuration), key management (creation, storage, and retrieval), and encryption/decryption. The specification is called Cryptoki, short for cryptographic token interface; the names Cryptoki and PKCS#11 are often used interchangeably.

The TEE includes a PKCS#11 TA. The normal world accesses it using OpenSC `pkcs11-tool` through OP-TEE's PKCS#11 client library: `--module` must point to **`/usr/lib/libckteec.so.0`**, which proxies access to the TA, **not** OpenSC's smart-card module. This interface is generally used by middleware (OpenSSL + libp11 / p11-kit -> PKCS#11 -> libckteec -> tee-supplicant -> TA), rather than called directly by applications.

A token is a logical data partition within the PKCS#11 TA. Each token has a label, SO and User identities, and isolated persistent objects. Persistent state such as labels and PINs is stored in TEE persistent storage.

Check prerequisites:

```bash
ls -l /dev/tee0 /dev/teepriv0        # Device nodes
pidof tee-supplicant                 # Running service
ls -l /lib/optee_armtz/              # PKCS#11 TA present
```

Enumerate and initialize a token, generate keys, and sign. The PINs below are example values:

```bash
M="--module /usr/lib/libckteec.so.0"

# List available slots and tokens
pkcs11-tool $M -L

# Initialize the token (SO operation)
pkcs11-tool $M --init-token --label mytoken --so-pin 12345678

# Initialize the user PIN
pkcs11-tool $M --login --so-pin 12345678 --init-pin --pin 1234

# Generate a key pair (RSA / EC)
pkcs11-tool $M --login --pin 1234 --keypairgen --key-type RSA:2048 --label mykey --id 01
pkcs11-tool $M --login --pin 1234 --keypairgen --key-type EC:prime256v1 --label mykey --id 01

# Export the public key
pkcs11-tool $M --login --pin 1234 --read-object --type pubkey --id 01 --output-file mykey.pub

# Sign with the private key
pkcs11-tool $M --login --pin 1234 --sign --id 01 --mechanism ECDSA --input-file data.bin
```

**Authentication:** A token has two user roles: SO (administrator), which manages token initialization and the User PIN, and User, which uses keys. In addition to PIN authentication, the TA provides a **protected authentication path**: when the token's `CKF_PROTECTED_AUTHENTICATION_PATH` flag is enabled, omit `--pin` and the TA authorizes based on the current client identity. Use `pkcs11-tool --show-info` to check whether the token declares the flag.

Reference: [PKCS #11 Cryptographic Token Interface Base Specification v2.40](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html) (OASIS specification).

## Interpreting Results and Common Issues

| Symptom | Interpretation |
|---|---|
| Example reports that a TA cannot be found | The TA was not packaged in this build or is missing from its storage directory. |
| Secure-storage example reports storage unavailable | `tee-supplicant` is not running or the storage backend is unavailable. |
| Many `xtest` cases are skipped | Required features were not enabled in this build; this is a configuration issue, not a failure. |
| Example runs but no TEE logs appear on serial | TEE log level is low; functionality is unaffected. |
| Random example returns the same result each time | Random-source configuration is abnormal; check the platform random-number driver. |