---
sidebar_position: 1
---

# Device Management

This document introduces how the SDK manages devices, including device configuration files, how to add new devices, and how to achieve adaptability to different devices through EEPROM.

## Device Configuration Files

A device (or board) example, DEB1, corresponds to [BPI-F3](https://docs.banana-pi.org/en/BPI-F3/BananaPi_BPI-F3), and usually includes the following configuration files:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
bsp-src/uboot-2022.10/arch/riscv/dts/Makefile
bsp-src/uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its
bsp-src/uboot-2022.10/include/configs/k1-x.h
bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/Makefile
buildroot-ext/configs/spacemit_k1_defconfig
buildroot-ext/board/spacemit/k1/env_k1-x.txt
```

**bsp-src/uboot-2022.10/arch/riscv/dts/`<board>`.dts**

The dts of board in u-boot.

**bsp-src/uboot-2022.10/arch/riscv/dts/Makefile**

The Makefile of dts in u-boot.

**bsp-src/uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its**

The FIT configuration of u-boot image.

**bsp-src/uboot-2022.10/include/configs/k1-x.h**

Configuration for K1 chip in u-boot.

**bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/`<board>`.dts**

The dts of board in kernel.

**bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/Makefile**

The Makefile of dts in kernel.

**buildroot-ext/configs/spacemit_k1_defconfig**

Configuration for buildroot.

**buildroot-ext/board/spacemit/k1/env_k1-x.txt**

The env of u-boot.

## Adding New Devices

If the new device is based on modifications to DEB1, to quickly bring up and verify functionality, you can modify the configuration based on DEB1. Usually, only the following configuration files need to be modified:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
```

After bringing up and verifying functionality, it is recommended to add a new device to the SDK. For example, to add `hs450`：

1. Use `k1-x_deb1.dts` in u-boot as a template to add `bsp-src/uboot-2022.10/arch/riscv/dts/k1_hs450.dts`.

2. Modify `bsp-src/uboot-2022.10/arch/riscv/dts/Makefile` to add the new dtb.

   ```diff
   @@ -8,7 +8,7 @@ dtb-$(CONFIG_TARGET_SIFIVE_UNLEASHED) += hifive-unleashed-a00.dtb
    dtb-$(CONFIG_TARGET_SIFIVE_UNMATCHED) += hifive-unmatched-a00.dtb
    dtb-$(CONFIG_TARGET_SIPEED_MAIX) += k210-maix-bit.dtb
    dtb-$(CONFIG_TARGET_SPACEMIT_K1PRO) += k1-pro_qemu.dtb k1-pro_sim.dtb k1-pro_fpga.dtb
   - dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_spl.dtb
   + dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_hs450.dtb k1-x_spl.dtb
    
    include $(srctree)/scripts/Makefile.dts
   ```

3. Modify `bsp-src/uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its` to add a new node.

   ```diff
   @@ -46,15 +46,6 @@
    				algo = "crc32";
    			};
    		};
   +		fdt_4 {
   +                        description = "k1_hs450";
   +                        type = "flat_dt";
   +                        compression = "none";
   +                        data = /incbin/("../dtb/k1-x_hs450.dtb");
   +                        hash-1 {
   +                                algo = "crc32";
   +                        };
   +                };
    	};
    
    	configurations {
   @@ -74,10 +65,5 @@
    			loadables = "uboot";
    			fdt = "fdt_3";
    		};
   +		conf_4 {
   +                        description = "k1_hs450";
   +                        loadables = "uboot";
   +                        fdt = "fdt_4";
   +                };
    	};
    };
   ```

4. Modify `bsp-src/uboot-2022.10/include/configs/k1-x.h` to update `product_name`. FSBL and u-boot will load the dtb based on `product_name`. If the device has EEPROM recording `product_name` and other information, you can skip this step. FSBL and u-boot will adapt based on the information from EEPROM.

   ```diff
   @@ -25,7 +25,7 @@
    #define CONFIG_GATEWAYIP	10.0.92.1
    #define CONFIG_NETMASK		255.255.255.0
   
   -#define DEFAULT_PRODUCT_NAME	"k1_deb1"
   +#define DEFAULT_PRODUCT_NAME	"k1_hs450"
   
    #define K1X_SPL_BOOT_LOAD_ADDR	(0x20200000)
    #define DDR_TRAINING_DATA_BASE	(0xc0829000)
   ```

5. Use `k1-x_deb1.dts` in the kernel as a template to add `bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_hs450.dts`.

6. Modify `bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/Makefile` to add the new dtb.

   ```diff
   @@ -1,4 +1,4 @@
    # SPDX-License-Identifier: GPL-2.0
    dtb-$(CONFIG_SOC_SPACEMIT_K1PRO) += k1-pro_sim.dtb k1-pro_fpga.dtb k1-pro_fpga_1x4.dtb k1-pro_fpga_
   2x2.dtb k1-pro_qemu.dtb k1-pro_verify.dtb
   -dtb-$(CONFIG_SOC_SPACEMIT_K1X) += k1-x_fpga.dtb k1-x_fpga_1x4.dtb k1-x_fpga_2x2.dtb k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb
   +dtb-$(CONFIG_SOC_SPACEMIT_K1X) += k1-x_fpga.dtb k1-x_fpga_1x4.dtb k1-x_fpga_2x2.dtb k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_hs450.dtb
    obj-$(CONFIG_BUILTIN_DTB) += $(addsuffix .o, $(dtb-y))
   ```

7. Modify `buildroot-ext/configs/spacemit_k1_defconfig` to add the new dtb.

   ```diff
   @@ -33,7 +33,7 @@ BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG=y
    BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="$(LINUX_OVERRIDE_SRCDIR)/arch/riscv/configs/k1_defconfig"
    BR2_LINUX_KERNEL_DTS_SUPPORT=y
   -BR2_LINUX_KERNEL_INTREE_DTS_NAME="spacemit/k1-x_deb1 spacemit/k1-x_deb2 spacemit/k1-x_evb"
   +BR2_LINUX_KERNEL_INTREE_DTS_NAME="spacemit/k1-x_deb1 spacemit/k1-x_deb2 spacemit/k1-x_evb spacemit/k1-x_hs450"
    BR2_PACKAGE_LINUX_TOOLS_GPIO=y
    BR2_PACKAGE_LINUX_TOOLS_PERF=y
    BR2_PACKAGE_LINUX_TOOLS_PERF_SCRIPTS=y
   ```

   After modification, recompile u-boot, the kernel, and the SDK.

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

## Single CS DDR

FSBL supports dual CS DDR by default. Modify `bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts` to support single CS DDR.

```diff
@@ -79,7 +79,7 @@
 	ddr@c0000000 {
 		/* dram data rate, should be 1200, 1600, or 2400 */
 		datarate = <2400>;
-		cs-num = <2>;
+		cs-num = <1>;
 		u-boot,dm-spl;
 	};
```

If the device has EEPROM, it can adapt through EEPROM.

## Adaptability through EEPROM

The firmware built by the SDK supports adaptability to multiple devices through EEPROM.

Related files:

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts
bsp-src/uboot-2022.10/arch/riscv/dts/<device>.dts
```

### Supported EEPROM List

- `atmel,24c02`

### Adding New EEPROM

1. Modify `bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts` to update the I2C address of the EEPROM. For example, the new address is `0xA0`.

   ```c
   @@ -121,7 +121,7 @@
    		eeprom@50{
    			compatible = "atmel,24c02";
    			u-boot,dm-spl;
   -			reg = <0x50>;
   +			reg = <0xA0>;
    			bus = <6>;
    			#address-cells = <1>;
    			#size-cells = <1>;
   ```

2. Modify `bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts` to add the new EEPROM configuration.

   ```c
   @@ -60,9 +60,9 @@
    	pinctrl-0 = <&pinctrl_i2c2_0>;
    	status = "okay";
    
   -	eeprom@50{
   -		compatible = "atmel,24c02";
   -		reg = <0x50>;
   +	eeprom@A0{
   +		compatible = "atmel,24c04";
   +		reg = <0xA0>;
    		vin-supply-names = "eeprom_1v8";
    		status = "okay";
    	};
   ```

3. Recompile u-boot and the SDK.

   ```shell
   make uboot-rebuild
   make
   ```

### Flash EEPROM

If the EEPROM is empty, you can use the Titantools Key Programing tools to program it.

### Using tlv_eeprom

The information in the EEPROM is encoded in TLV format, and you can use the `tlv_eeprom` command in u-boot. Below is an introduction on how to modify `Product Name`, `Base MAC Address`, and `MAC Address`.

1. Connect the PC to the device's debug serial port. When the device starts, press the `s` key on the PC serial terminal until entering the u-boot shell.

   ```shell
   Autoboot in 0 seconds
   => 
   ```

2. Write `Product Name`, for example, `k1-x_MUSE-Book`.

   ```shell
   => tlv_eeprom set 0x21 k1-x_MUSE-Book
   => tlv_eeprom write
   Programming passed.
   ```

   `Programming passed.` indicates successful writing.

   For versions v1.0beta3.1 and later, name it according to the device dts file name (without suffix) to facilitate u-boot automatic loading of dtb.

3. `reset` to check if the dtb for `k1-x_MUSE-Book` can be loaded. If successful, u-boot will print the following:

   ```shell
   product_name: k1_deb1
   detect dtb_name: k1-x_deb1.dtb
   ```

4. Program the `Base MAC Address`, for example, `FE:FE:FE:96:A5:47`.

   ```shell
   => tlv_eeprom set 0x24 FE:FE:FE:96:A5:47
   => tlv_eeprom write
   Programming passed.
   ```

   If there are two network interfaces, you need to program the `MAC Address`, so the address of the second network interface will automatically increment by 1.

   ```shell
   => tlv_eeprom set 0x2A 2
   => tlv_eeprom write
   Programming passed.
   ```
