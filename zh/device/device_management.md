---
sidebar_position: 1
---

# 设备管理

本文档介绍SDK如何管理设备，包括设备的配置文件、如何添加新设备和如何通过EEPROM实现自适应不同设备等。

## 设备的配置文件

设备（Device），或叫板子、板型。以DEB1为例，对应[BPI-F3](https://docs.banana-pi.org/en/BPI-F3/BananaPi_BPI-F3)，通常包含以下配置文件：

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

u-boot中设备的dts。

**bsp-src/uboot-2022.10/arch/riscv/dts/Makefile**

u-boot中设备的dts的Makefile。

**bsp-src/uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its**

u-boot FIT Image的配置文件。

**bsp-src/uboot-2022.10/include/configs/k1-x.h**

u-boot中K1芯片的配置。

**bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/`<board>`.dts**

kernel中设备的dts。

**bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/Makefile**

kernel中设备的dts的Makefile。

**buildroot-ext/configs/spacemit_k1_defconfig**

buildroot的配置。

**buildroot-ext/board/spacemit/k1/env_k1-x.txt**

u-boot的env。

## 添加新设备

如果新设备是基于DEB1修改的，为了快速bringup，验证功能，可以基于DEB1的配置修改，通常只需修改以下配置文件：

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
```

bringup后，功能验证完，推荐在SDK添加一个新设备。例如，添加`hs450`：

1. 以u-boot的`k1-x_deb1.dts`为模版，添加`bsp-src/uboot-2022.10/arch/riscv/dts/k1_hs450.dts`。

2. 修改`bsp-src/uboot-2022.10/arch/riscv/dts/Makefile`，添加新dtb。

   ```diff
   @@ -8,7 +8,7 @@ dtb-$(CONFIG_TARGET_SIFIVE_UNLEASHED) += hifive-unleashed-a00.dtb
    dtb-$(CONFIG_TARGET_SIFIVE_UNMATCHED) += hifive-unmatched-a00.dtb
    dtb-$(CONFIG_TARGET_SIPEED_MAIX) += k210-maix-bit.dtb
    dtb-$(CONFIG_TARGET_SPACEMIT_K1PRO) += k1-pro_qemu.dtb k1-pro_sim.dtb k1-pro_fpga.dtb
   - dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_spl.dtb
   + dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_hs450.dtb k1-x_spl.dtb
    
    include $(srctree)/scripts/Makefile.dts
   ```

3. 修改`bsp-src/uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its`，添加新节点。

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

4. 修改`bsp-src/uboot-2022.10/include/configs/k1-x.h`，更新`product_name`，FSBL和u-boot将根据`product_name`加载dtb。如果设备有EEPROM记录`product_name`等信息，可以不修改，FSBL和u-boot通过EEPROM的信息实现自适应。

   ```diff
   @@ -25,7 +25,7 @@
    #define CONFIG_GATEWAYIP	10.0.92.1
    #define CONFIG_NETMASK		255.255.255.0
   
   -#define DEFAULT_PRODUCT_NAME	"k1_deb1"
   +#define DEFAULT_PRODUCT_NAME	"k1_hs450"
   
    #define K1X_SPL_BOOT_LOAD_ADDR	(0x20200000)
    #define DDR_TRAINING_DATA_BASE	(0xc0829000)
   ```

5. 以kernel的`k1-x_deb1.dts`为模版，添加`bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_hs450.dts`。

6. 修改`bsp-src/linux-6.1/arch/riscv/boot/dts/spacemit/Makefile`，添加新dtb。

   ```diff
   @@ -1,4 +1,4 @@
    # SPDX-License-Identifier: GPL-2.0
    dtb-$(CONFIG_SOC_SPACEMIT_K1PRO) += k1-pro_sim.dtb k1-pro_fpga.dtb k1-pro_fpga_1x4.dtb k1-pro_fpga_
   2x2.dtb k1-pro_qemu.dtb k1-pro_verify.dtb
   -dtb-$(CONFIG_SOC_SPACEMIT_K1X) += k1-x_fpga.dtb k1-x_fpga_1x4.dtb k1-x_fpga_2x2.dtb k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb
   +dtb-$(CONFIG_SOC_SPACEMIT_K1X) += k1-x_fpga.dtb k1-x_fpga_1x4.dtb k1-x_fpga_2x2.dtb k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_hs450.dtb
    obj-$(CONFIG_BUILTIN_DTB) += $(addsuffix .o, $(dtb-y))
   ```

7. 修改`buildroot-ext/configs/spacemit_k1_defconfig`，添加新dtb。

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

   修改完，重新编译u-boot、内核和SDK即可。

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

## 单CS DDR

FSBL默认支持双CS DDR，修改`bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts`可以支持单CS DDR。

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

如何设备有EEPROM，支持通过EEPROM实现自适应。

## 通过EEPROM实现自适应

SDK构建的固件支持通过EEPROM实现自适应多设备。

相关文件：

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts
bsp-src/uboot-2022.10/arch/riscv/dts/<device>.dts
```

### EEPROM支持列表

- `atmel,24c02`

### 添加新的EEPROM

1. 修改`bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_spl.dts`，更新EEPROM的I2C地址，例如新地址为`0xA0`。

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

2. 修改`bsp-src/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts`，添加新EEPROM的配置。

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

3. 重新编译u-boot和SDK即可。

   ```shell
   make uboot-rebuild
   make
   ```

### 烧写EEPROM信息

如果 EEPROM 是空白的，可以使用 Titantools 的写号工具烧写。

### 修改EEPROM信息

在 EEPROM 中的信息是按照TLV编码的，可以使用 u-boot 的 tlv_eeprom 命令。下面介绍修改如何修改 `Product Name`、 `Base MAC Address` 和 `MAC Address`。

1. PC接上设备的调试串口，设备启动时，在PC串口终端按下键盘的`s`键，直到进入u-boot shell。

   ```shell
   Autoboot in 0 seconds
   => 
   ```

2. 烧写`Product Name`，例如`k1-x_MUSE-Book`。

   ```shell
   => tlv_eeprom set 0x21 k1-x_MUSE-Book
   => tlv_eeprom write
   Programming passed.
   ```

   `Programming passed.`表示写入成功。

   v1.0beta3.1和之后的版本，请以设备dts的文件名（不带后缀）命名，方便u-boot自动加载dtb。

3. `reset`检查是否可以加载`k1-x_MUSE-Book`的dtb，正常的话u-boot有如下打印。

   ```shell
   product_name: k1-x_MUSE-Book
   detect dtb_name: k1-x_MUSE-Book.dtb
   ```

4. 烧写`Base MAC Address`，例如`FE:FE:FE:96:A5:47`。

   ```shell
   => tlv_eeprom set 0x24 FE:FE:FE:96:A5:47
   => tlv_eeprom write
   Programming passed.
   ```

   如果有两个网口，需要烧写`MAC Address`，这样第二个网口的地址会自动加1。

   ```shell
   => tlv_eeprom set 0x2A 2
   => tlv_eeprom write
   Programming passed.
   ```
