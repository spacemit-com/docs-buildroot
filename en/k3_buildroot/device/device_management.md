---
sidebar_position: 1
---

# Device Management

This document describes 
- how the SDK manages devices, including the configuration files associated with a device,
- how to add a new device,
- and how EEPROM can be used to support automatic adaptation across different devices.

## Device Configuration Files

A device, also referred to as a board or board variant, is defined by a set of configuration files. 

Using DEB1 as an example, which corresponds to [K3 Pico-ITX](https://spacemit.com/community/development-kit/k3-pico-itx), the following files are typically involved:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico-itx.dts
bsp-src/uboot-2022.10/arch/riscv/dts/Makefile
bsp-src/uboot-2022.10/board/spacemit/k3/configs/uboot_fdt.its
bsp-src/uboot-2022.10/include/configs/k3.h
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-pico-itx.dts
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/Makefile
buildroot-ext/configs/spacemit_k3_defconfig
buildroot-ext/board/spacemit/k3/env_k3.txt
```

**`bsp-src/uboot-2022.10/arch/riscv/dts/<board>.dts`**

Device DTS file used by U-Boot.

**`bsp-src/uboot-2022.10/arch/riscv/dts/Makefile`**

Makefile for device DTS files in U-Boot.

**`bsp-src/uboot-2022.10/board/spacemit/k3/configs/uboot_fdt.its`**

Configuration file for the U-Boot FIT image.

**`bsp-src/uboot-2022.10/include/configs/k3.h`**

K3 chip configuration file in U-Boot.

**`bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/<board>.dts`**

Device DTS file used by the kernel.

**`bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/Makefile`**

Makefile for device DTS files in the kernel.

**`buildroot-ext/configs/spacemit_k3_defconfig`**

Buildroot configuration file.

**`buildroot-ext/board/spacemit/k3/env_k3.txt`**

U-Boot environment file.

## Adding a New Device

If the new device is derived from DEB1, the quickest way to bring it up and verify basic functionality is to start from the DEB1 configuration. In most cases, only the following files need to be modified in the initial stage:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico-itx.dts
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-pico-itx.dts
```

After bring-up and feature validation are complete, it is recommended to add the board to the SDK as a dedicated device entry. The following example uses `k3_newdev`.

1. Use the U-Boot `k3-pico-itx.dts` file as a template, and add `bsp-src/uboot-2022.10/arch/riscv/dts/k3_newdev.dts`.

2. Update `bsp-src/uboot-2022.10/arch/riscv/dts/Makefile` and add the new DTB.

   ```diff
   @@ -18,7 +18,7 @@ dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1
                                        k1-x_uav.dtb k1-x_MUSE-Paper2.dtb k1-x_MUSE-Pi2.dtb
    dtb-$(CONFIG_TARGET_SPACEMIT_K3) += k3_fpga_1x1.dtb k3_spl.dtb k3_evb.dtb k3_deb1.dtb k3_com260.dtb \
                                       k3_com260_kit_v02.dtb k3_dc_board.dtb k3_gemini_c0.dtb \
   -                                   k3_gemini_c1.dtb k3_evb2-1.dtb k3_evb2-2.dtb k3-pico-itx.dtb
   +                                   k3_gemini_c1.dtb k3_evb2-1.dtb k3_evb2-2.dtb k3-pico-itx.dtb k3_newdev.dtb
    include $(srctree)/scripts/Makefile.dts

    targets += $(dtb-y)

   ```

3. Update `bsp-src/uboot-2022.10/board/spacemit/k3/configs/uboot_fdt.its` and add a new node.

   ```diff
   @@ -97,6 +97,15 @@
                                   algo = "crc32";
                           };
                   };
   +               fdt_9 {
   +                       description = "k3_newdev.dtb";
   +                       type = "flat_dt";
   +                       compression = "none";
   +                       data = /incbin/("../dtb/k3_newdev.dtb");
   +                       hash-1 {
   +                               algo = "crc32";
   +                       };
   +               };
           };

           configurations {
   @@ -148,5 +157,10 @@
                           loadables = "uboot";
                           fdt = "fdt_8";
                   };
   +               conf_9 {
   +                       description = "k3_newdev";
   +                       loadables = "uboot";
   +                       fdt = "fdt_9";
   +               };
           };
    };
   ```

4. Update `bsp-src/uboot-2022.10/include/configs/k3.h` and change `product_name`.

   FSBL and U-Boot use `product_name` to determine which DTB to load. If the device supports EEPROM programming and the EEPROM already stores `product_name` and related board information, this change is optional. In that case, FSBL and U-Boot can read the EEPROM data and adapt automatically.

   ```diff
   @@ -25,7 +25,7 @@
    #define CONFIG_GATEWAYIP	10.0.92.1
    #define CONFIG_NETMASK		255.255.255.0
   
   -#define DEFAULT_PRODUCT_NAME	"k3-pico-itx"
   +#define DEFAULT_PRODUCT_NAME	"k3_newdev"
   
    #define k3X_SPL_BOOT_LOAD_ADDR	(0x20200000)
    #define DDR_TRAINING_DATA_BASE	(0xc0829000)
   ```

5. Use the kernel `k3-pico-itx.dts` file as a template, and add `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3_newdev.dts`.

6. Update `bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/Makefile` and add the new DTB.

   ```diff
   @@ -5,7 +5,7 @@ dtb-$(CONFIG_SOC_SPACEMIT_K1) += k1-orangepi-rv2.dtb

   dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_fpga_1x1.dtb
   dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_evb.dtb k3_evb2-1.dtb k3_evb2-2.dtb
   -dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_deb1.dtb k3-pico-itx.dtb
   +dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_deb1.dtb k3-pico-itx.dtb k3_newdev.dtb
   dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_com260.dtb
   dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_dc_board.dtb
   dtb-$(CONFIG_SOC_SPACEMIT_K3) += k3_com260_kit_v02.dtb
   ```

7. Update `buildroot-ext/configs/spacemit_k3_defconfig` and add the new DTB.

   ```diff
   @@ -37,7 +37,7 @@ BR2_LINUX_KERNEL_IMAGE_TARGET_CUSTOM=y
    BR2_LINUX_KERNEL_IMAGE_TARGET_NAME="Image"
    BR2_LINUX_KERNEL_IMAGE_NAME="Image"
    BR2_LINUX_KERNEL_DTS_SUPPORT=y
   -BR2_LINUX_KERNEL_INTREE_DTS_NAME="spacemit/k3_fpga_1x1 spacemit/k3_evb spacemit/k3_deb1 spacemit/k3_com260 spacemit/k3_com260_kit_v02 spacemit/k3_dc_board spacemit/k3_gemini_c0 spacemit/k3_gemini_c1 spacemit/k3_evb2-1 spacemit/k3_evb2-2 spacemit/k3-pico-itx"
   +BR2_LINUX_KERNEL_INTREE_DTS_NAME="spacemit/k3_fpga_1x1 spacemit/k3_evb spacemit/k3_deb1 spacemit/k3_com260 spacemit/k3_com260_kit_v02 spacemit/k3_dc_board spacemit/k3_gemini_c0 spacemit/k3_gemini_c1 spacemit/k3_evb2-1 spacemit/k3_evb2-2 spacemit/k3-pico-itx spacemit/spacemit/k3_newdev"
    BR2_PACKAGE_LINUX_TOOLS_GPIO=y
    BR2_PACKAGE_BUSYBOX_SHOW_OTHERS=y
    BR2_PACKAGE_ALSA_UTILS=y
   ```

8. After the changes are complete, rebuild U-Boot, the kernel, and the SDK.

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

## EEPROM-Based Device Adaptation

Firmware built by the SDK can support multiple device variants through EEPROM-based adaptation.

Related files:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k3_spl.dts
bsp-src/uboot-2022.10/arch/riscv/dts/<device>.dts
```

### Supported EEPROM Devices

- `atmel,24c02`

### Adding a New EEPROM

1. Update `bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico-itx.dts` and add the new EEPROM configuration. 
   The following example adds `atmel,24c04` with I2C address `0xA0`.

   ```diff
   @@ -119,9 +119,9 @@
           pinctrl-0 = <&pinctrl_i2c2_1>;
           status = "okay";

   -       eeprom@50{
   -               compatible = "atmel,24c02";
   -               reg = <0x50>;
   +       eeprom@A0{
   +               compatible = "atmel,24c04";
   +               reg = <0xA0>;
                   status = "okay";
           };
    };
   ```

2. Rebuild U-Boot and the SDK.

   ```shell
   make uboot-rebuild
   make
   ```

### Programming EEPROM Data

If the EEPROM is blank, the programming utility in [Titantools](https://spacemit.com/community/document/info?lang=en&nodepath=tools/user_guide/flasher_user_guide.md) can be used to write the data.

### Updating EEPROM Data

EEPROM data is stored in TLV format and can be modified with the U-Boot `tlv_eeprom` command.

The following examples show how to update `Product Name`, `Base MAC Address`, and `MAC Address`.

1. Connect the PC to the device debug UART. During boot, press `s` in the serial terminal until the U-Boot shell appears.

   ```shell
   Autoboot in 0 seconds
   => 
   ```

2. Check which fields are supported by `tlv_eeprom` and review their stored values.
   ```shell
   => tlv_eeprom read
   EEPROM data loaded from device to memory.
   => tlv_eeprom list
   TLV Code    TLV Name
   [  13.851] ========    =================
   [  13.854] 0x21        Product Name
   [  13.857] 0x22        Part Number
   [  13.861] 0x23        Serial Number
   [  13.864] 0x24        Base MAC Address
   [  13.867] 0x60        Wifi MAC Address
   [  13.871] 0x61        Bluetooth Address
   [  13.875] 0x25        Manufacture Date
   [  13.878] 0x26        Device Version
   [  13.882] 0x27        Label Revision
   [  13.885] 0x28        Platform Name
   [  13.888] 0x29        ONIE Version
   [  13.891] 0x2A        MAC Addresses
   [  13.895] 0x2B        Manufacturer
   [  13.898] 0x2C        Country Code
   [  13.901] 0x2D        Vendor Name
   [  13.904] 0x2E        Diag Version
   [  13.908] 0x2F        Service Tag
   [  13.911] 0xFD        Vendor Extension
   [  13.914] 0x44        DDR tx odt
   [  13.917] 0x45        DDR Part Number
   [  13.921] 0x80        PMIC Type
   [  13.924] 0xFE        CRC-32
   ```

3. Update `Product Name`, for example to `k3-pico-itx`.

   ```shell
   => tlv_eeprom set 0x21 k3-pico-itx
   => tlv_eeprom write
   Programming passed.
   ```

   `Programming passed.` indicates that the write operation succeeded.


4. Program `Base MAC Address`, for example `FE:FE:FE:21:47:AC`.

   ```shell
   => tlv_eeprom set 0x24 FE:FE:FE:21:47:AC
   => tlv_eeprom write
   Programming passed.
   ```

   If the device has two Ethernet ports, `MAC Address` must also be programmed. The address for the second port is then incremented automatically.

   ```shell
   => tlv_eeprom set 0x2A 2
   => tlv_eeprom write
   Programming passed.
   ```

5. After the update is complete, run `tlv_eeprom` again to verify the final result.
   ```shell
   => [ 349.723] tlv_eeprom
   TLV: 0
   [ 352.148] TlvInfo Header:
   [ 352.148]    Id String:    TlvInfo
   [ 352.151]    Version:      1
   [ 352.154]    Total Length: 78
   [ 352.157] TLV Name             Code Len Value
   [ 352.161] -------------------- ---- --- -----
   [ 352.165] Serial Number        0x23  16 HW3MPK3161280011
   [ 352.170] Base MAC Address     0x24   6 50:0A:52:0B:CA:0B
   [ 352.175] MAC Addresses        0x2A   2 1
   [ 352.179] DDR Part Number      0x45  13 MT62F2G32D4DS
   [ 352.184] Part Number          0x22   4 MPK3
   [ 352.188] Product Name         0x21  11 k3-pico-itx
   [ 352.193] PMIC Type            0x80   6 au4562
   [ 352.197] CRC-32               0xFE   4 0xE405755E
   [ 352.202] Checksum is valid.
   ```