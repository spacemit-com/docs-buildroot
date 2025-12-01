# Crypto-Engine

介绍 crypto-engine 使用方法。

## 模块介绍  
  
crypto-engine 通过硬件实现加密算法，用于对明文数据进行加密处理。

### 功能介绍  

![](static/openssl.jpg)

K1 的 crypto-engine（简称 CE）通过硬件实现了 AES 加密算法，支持 ECB、CBC、XTS 等加密模式。

### 源码结构介绍

CE 驱动代码位于 `drivers/crypto/spacemit` 目录下：  

```  
drivers/crypto/spacemit
|--spacemit_ce_engine.c            #CE 驱动代码
|--spacemit-ce-glue.c              #基于 CE 驱动实现的加密算法
|--spacemit_engine.h 
```  

内核 crypto 框架的通用实现位于内核 crypto 路径中，此处不再赘述。

## 关键特性  

### 特性

支持 AES 加密算法，支持 ECB / CBC / XTS 模式。

### 性能参数

- 纯硬件性能可达 **500MB/s**
- 通过内核实现的加密流程性能可达 **280MB/s** (对 128KB 以上的数据)

**测试方法说明：**

OpenSSL 的 `speed` 工具默认支持最大 16KB 的数据，可通过二次开发扩展为 128KB。

```
openssl speed -elapsed -async_jobs 1 -engine afalg -evp aes-128-cbc -multi 1
```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### CONFIG配置

`CONFIG_CRYPTO`：为内核平台 crypto 框架提供支持，支持 K1 CE 驱动情况下，应为 `Y`

```
CONFIG_CRYPTO=y
CONFIG_SPACEMIT_REE_AES=y
CONFIG_SPACEMIT_REE_ENGINE=y
```

### DTS 配置

CE 模块无外设引脚，因此仅需配置其时钟和复位资源。

#### dtsi 配置示例

dtsi 中配置 CE 控制器基地址和时钟复位资源，一般无需改动：

```dts
 spacemit_crypto_engine@d8600000 {
  compatible = "spacemit,crypto_engine";
  spacemit-crypto-engine-0 = <0xd8600000 0x00100000>;
  interrupt-parent = <&intc>;
  interrupts = <113>;
  num-engines = <1>;
  clocks = <&ccu CLK_AES>;
  resets = <&reset RESET_AES>;
  interconnects = <&dram_range5>;
  interconnect-names = "dma-mem";
  status = "okay";
 };
```

## 接口介绍

### API介绍

AES 驱动注册了加密与解密接口到 crypto 框架中。

以 CBC 模式为例：
- 该接口实现了通过 CE 硬件执行 CBC 模式的加密。
```
static int cbc_encrypt(struct skcipher_request *req)
```

- 实现了通过 CE 硬件执行 CBC 模式的解密。
```
static int cbc_decrypt(struct skcipher_request *req)
```

## 测试介绍

首先验证 AES 加密算法是否注册成功

```
cat /proc/crypto
结果中如下
name         : xts(aes)
driver       : __driver-xts-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 1
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 32
max keysize  : 64
ivsize       : 16
chunksize    : 16
walksize     : 16

name         : cbc(aes)
driver       : __driver-cbc-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 1
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 16
max keysize  : 32
ivsize       : 16
chunksize    : 16
walksize     : 16

name         : ecb(aes)
driver       : __driver-ecb-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 2
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 16
max keysize  : 32
ivsize       : 16
chunksize    : 16
walksize     : 16
```

加密算法功能测试可通过 OpenSSL 工具测试，用法示例如下：

```
echo "hello,world" | openssl enc -aes128 -e -a -salt -engine afalg  //加密字符串
echo "加密自动生成的密钥" | openssl enc -engine afalg -aes128 -a -d -salt   //解密字符串
openssl enc -aes128 -e -engine afalg -in data.txt -out encrypt.txt -pass pass:12345678   //使用密钥进行加密
openssl enc -aes-cbc -d -engine afalg -in encrypt.txt -out data.txt -pass pass:12345678   //使用密钥进行解密
可将解密得到的内容与原始数据进行比对，一致即表示加密/解密功能正常。
```

## FAQ
