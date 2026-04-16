---
sidebar_position: 1
---

# 设备管理

本文档介绍SDK如何管理设备，包括设备的配置文件、如何添加新设备和如何通过EEPROM实现自适应不同设备等。

## 设备的配置文件

设备（Device），或叫板子、板型。以DEB1为例，对应[K3 Pico-ITX](https://spacemit.com/community/development-kit/k3-pico-itx)，通常包含以下配置文件：

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

**bsp-src/uboot-2022.10/arch/riscv/dts/`<board>`.dts**

u-boot中设备的dts。

**bsp-src/uboot-2022.10/arch/riscv/dts/Makefile**

u-boot中设备的dts的Makefile。

**bsp-src/uboot-2022.10/board/spacemit/k3/configs/uboot_fdt.its**

u-boot FIT Image的配置文件。

**bsp-src/uboot-2022.10/include/configs/k3.h**

u-boot中k3芯片的配置。

**bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/`<board>`.dts**

kernel中设备的dts。

**bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/Makefile**

kernel中设备的dts的Makefile。

**buildroot-ext/configs/spacemit_k3_defconfig**

buildroot的配置。

**buildroot-ext/board/spacemit/k3/env_k3.txt**

u-boot的env。

## 添加新设备

如果新设备是基于DEB1修改的，为了快速bringup，验证功能，可以基于DEB1的配置修改，通常只需修改以下配置文件：

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico-itx.dts
bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3-pico-itx.dts
```

bringup后，功能验证完，推荐在SDK添加一个新设备，假设为`k3_newdev`。

1. 以u-boot的`k3-pico-itx.dts`为模版，添加`bsp-src/uboot-2022.10/arch/riscv/dts/k3_newdev.dts`。

2. 修改`bsp-src/uboot-2022.10/arch/riscv/dts/Makefile`，添加新dtb。

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

3. 修改`bsp-src/uboot-2022.10/board/spacemit/k3/configs/uboot_fdt.its`，添加新节点。

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

4. 修改`bsp-src/uboot-2022.10/include/configs/k3.h`，更新`product_name`，FSBL和u-boot将根据`product_name`加载dtb。如果设备支持EEPROM烧号功能，并且EEPROM记录有`product_name`等信息，可以不修改，FSBL和u-boot可读取EEPROM的板号等信息实现自适应。

   ```diff
   @@ -25,7 +25,7 @@
    #define CONFIG_GATEWAYIP	10.0.92.1
    #define CONFIG_NETMASK		255.255.255.0
   
   -#define DEFAULT_PRODUCT_NAME	"k3-pico-itx"
   +#define DEFAULT_PRODUCT_NAME	"k3_newdev"
   
    #define k3X_SPL_BOOT_LOAD_ADDR	(0x20200000)
    #define DDR_TRAINING_DATA_BASE	(0xc0829000)
   ```

5. 以kernel的`k3-pico-itx.dts`为模版，添加`bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/k3_newdev.dts`。

6. 修改`bsp-src/linux-6.18/arch/riscv/boot/dts/spacemit/Makefile`，添加新dtb。

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

7. 修改`buildroot-ext/configs/spacemit_k3_defconfig`，添加新dtb。

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

8. 修改完，重新编译u-boot、内核和SDK即可。

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

## 通过EEPROM实现自适应

SDK构建的固件支持通过EEPROM实现自适应多设备。

相关文件：

```shell
bsp-src/uboot-2022.10/arch/riscv/dts/k3_spl.dts
bsp-src/uboot-2022.10/arch/riscv/dts/<device>.dts
```

### EEPROM支持列表

- `atmel,24c02`

### 添加新的EEPROM

1. 修改`bsp-src/uboot-2022.10/arch/riscv/dts/k3-pico-itx.dts`，添加新EEPROM的配置。例如添加`atmel,24c04`，I2C地址为`0xA0`。

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

2. 重新编译u-boot和SDK即可。

   ```shell
   make uboot-rebuild
   make
   ```

### 烧写EEPROM信息

如果 EEPROM 是空白的，可以使用 [Titantools](https://spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md) 的写号工具烧写。

### 修改EEPROM信息

在 EEPROM 中的信息是按照TLV编码的，可以使用 u-boot 的 `tlv_eeprom` 命令。下面介绍修改如何修改 `Product Name`、 `Base MAC Address` 和 `MAC Address`。

1. PC接上设备的调试串口，设备启动时，在PC串口终端按下键盘的`s`键，直到进入u-boot shell。

   ```shell
   Autoboot in 0 seconds
   => 
   ```

2. 查看当前设备 `tlv_eeprom` 支持保存哪些信息，以及这些信息保存的地址。
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

3. 修改`Product Name`，例如`k3-pico-itx`。

   ```shell
   => tlv_eeprom set 0x21 k3-pico-itx
   => tlv_eeprom write
   Programming passed.
   ```

   `Programming passed.`表示写入成功。


4. 烧写`Base MAC Address`，例如`FE:FE:FE:21:47:AC`。

   ```shell
   => tlv_eeprom set 0x24 FE:FE:FE:21:47:AC
   => tlv_eeprom write
   Programming passed.
   ```

   如果有两个网口，需要烧写`MAC Address`，这样第二个网口的地址会自动加1。

   ```shell
   => tlv_eeprom set 0x2A 2
   => tlv_eeprom write
   Programming passed.
   ```

5. 修改完后，可以直接执行 `tlv_eeprom` 指令查看最终修改结果。
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