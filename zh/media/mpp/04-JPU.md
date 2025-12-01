# JPU

JPU（Jpeg Processing Unit）是专门用于 JPEG 图像编解码的硬件模块，能够有效提升 JPEG 编解码效率并降低 CPU 负载。K1 的 JPU 提供了完整的测试程序作为参考。

## 1 规格（待补充）

## 2 JPU 测试程序

`k1x-jpu` 内部封装了面向应用层的 API，并基于该 API 集成了一套用于测试和验证 K1 芯片 JPU 功能的测试程序集。该测试集不仅用于功能验证，也可供客户在开发对接 JPU 进行硬件编解码的应用时参考。

### 2.1 安装说明

#### 2.1.1 Bianbu 桌面系统

`k1x-jpu` 已集成于软件源，直接通过 apt 命令安装：

```shell
sudo apt update
sudo apt install k1x-jpu
```

#### 2.1.2 Buildroot 系统

2 种方法将 `k1x-jpu` 集成到系统中：

- 在编译镜像时开启 `k1x-jpu` 的编译集成选项（默认已开启），这样生成的镜像中包含所有相关测试程序；
- 若镜像未集成 k1x-jpu，可手动编译后将生成的二进制文件复制到系统的 `/usr/bin/` 目录使用，具体包含的二进制文件在下文说明。

### 2.2 使用说明

`k1x-jpu` 的测试程序集中主要包含以下测试程序：

- **`jpu_dec_test`**：用于 JPEG 的解码测试
- **`jpu_enc_test`**：用于 JPEG 的编码测试
- **`libjpu.so`**：JPU 的 API 封装库

#### 2.2.1 `jpu_dec_test`

基本用法示例：

```shell
//将input.jpeg解码为output.yuv
./jpu_dec_test --input=input.jpeg   --output=output.yuv

//将input.jpeg解码为output.yuv，output.yuv的像素格式为NV12
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=420 --ordering=nv12

//将input.jpeg解码为output.yuv，output.yuv的像素格式为YUYV
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=422 --ordering=yuyv
```

参数说明

```shell
bianbu@k1:~$ jpu_dec_test -h
[JPU/6261] ------------------------------------------------------------------------------
[JPU/6261]  CODAJ12 Decoder
[JPU/6261] ------------------------------------------------------------------------------
[JPU/6261] jpu_dec_test [options] --input=jpg_file_path
[JPU/6261] -h                      help
[JPU/6261] --input=FILE            jpeg filepath
[JPU/6261] --output=FILE           output file path
[JPU/6261] --stream-endian=ENDIAN  bitstream endianness. refer to datasheet Chapter 4.
[JPU/6261] --frame-endian=ENDIAN   pixel endianness of 16bit input source. refer to datasheet Chapter 4.
[JPU/6261] --pixelj=JUSTIFICATION  16bit-pixel justification. 0(default) - msb justified, 1 - lsb justified in little-endianness
[JPU/6261] --bs-size=SIZE          bitstream buffer size in byte
[JPU/6261] --roi=x,y,w,h           ROI region（未适配）
[JPU/6261] --subsample             conversion sub-sample(ignore case): NONE, 420, 422, 444
[JPU/6261] --ordering              conversion ordering(ingore-case): NONE, NV12, NV21, YUYV, YVYU, UYVY, VYUY, AYUV
[JPU/6261]                         NONE - planar format
[JPU/6261]                         NV12, NV21 - semi-planar format for all the subsamples.
[JPU/6261]                                      If subsample isn't defined or is none, the sub-sample depends on jpeg information
[JPU/6261]                                      The subsample 440 can be converted to the semi-planar format. It means that the encoded sub-sample should be 440.
[JPU/6261]                         YUVV..VYUY - packed format. subsample be ignored.
[JPU/6261]                         AYUV       - packed format. subsample be ignored.
[JPU/6261] --rotation              0, 90, 180, 270(未适配)
[JPU/6261] --mirror                0(none), 1(V), 2(H), 3(VH)（未适配）
[JPU/6261] --scaleH                Horizontal downscale: 0(none), 1(1/2), 2(1/4), 3(1/8)（未适配）
[JPU/6261] --scaleV                Vertical downscale  : 0(none), 1(1/2), 2(1/4), 3(1/8)（未适配）
[JPU/6261] --profiling             0: performance output will not be printed 1:print performance output 
[JPU/6261] --loop_count            loop count
```

#### 2.2.2 jpu_enc_test

基本用法示例：

```shell
//将input.yuv编码为quality为10的output.jpeg,使用enc.cfg中的配置
./jpu_enc_test --cfg-dir=xxx --yuv-dir=xxx --input=enc.cfg --output=output.jpeg --profiling=1 --loop_count=1 --quality=10


//env.cfg的配置参考如下：
;-----------------------------------------------------------------
; Configuration File for MJPEG BP @ Encoder
;
; NOTE.
; If a line begins with ;, the line is treated as a comment-line.
;-----------------------------------------------------------------
;----------------------------
; Sequence PARAMETERs
;----------------------------
; source YUV image file
YUV_SRC_IMG                  output.yuv
FRAME_FORMAT                 0
                            ; 0-planar, 1-NV12,NV16(CbCr interleave) 2-NV21,NV61(CbCr alternative)
                            ; 3-YUYV, 4-UYVY, 5-YVYU, 6-VYUY, 7-YUV packed (444 only)
PICTURE_WIDTH                1280
PICTURE_HEIGHT               720
FRAME_NUMBER_ENCODED         1
                            ; number of frames to be encoded (#>0)
;---------------------------------------
; MJPEG Encodeing PARAMETERs
;---------------------------------------
VERSION_ID                  3
RESTART_INTERVAL            0
IMG_FORMAT                  0
                            ; Source Format (0 : 4:2:0, 1 : 4:2:2, 2 : 4:4:0, 3 : 4:4:4, 4 : 4:0:0)
```

参数说明

```shell
bianbu@k1:~$ jpu_enc_test -h
[JPU/6272] ------------------------------------------------------------------------------
[JPU/6272]  JPU Encoder 
[JPU/6272] ------------------------------------------------------------------------------
[JPU/6272] jpu_enc_test [option] cfg_file 
[JPU/6272] -h                      help
[JPU/6272] --output=FILE           output file path
[JPU/6272] --cfg-dir=DIR           folder that has encode parameters default: ./cfg
[JPU/6272] --yuv-dir=DIR           folder that has an input source image. default: ./yuv
[JPU/6272] --yuv=FILE              use given yuv file instead of yuv file in cfg file
[JPU/6272] --bs-size=SIZE          bitstream buffer size in byte
[JPU/6272] --quality=PERCENTAGE    quality factor(1..100)
```

### 2.3 代码结构

`k1x-jpu` 的代码位置在：

```shell
package-src/k1x-jpu
```

代码结构及简要说明如下：

```shell
|-- CMakeLists.txt                //cmake构建脚本
|-- debian                        //deb包构建的相关配置和脚本
|   |-- bianbu.conf
|   |-- changelog
|   |-- compat
|   |-- control
|   |-- copyright
|   |-- install
|   |-- postinst
|   |-- README.Debian
|   |-- rules
|   |-- source
|   |   |-- format
|   |   `-- local-options
|   `-- watch
|-- etc
|   `-- init.d
|       `-- jpu.sh
|-- format.sh
|-- jpuapi                        //jpu API
|   |-- include
|   |   |-- jpuapi.h
|   |   |-- jpuconfig.h
|   |   |-- jpudecapi.h
|   |   |-- jpuencapi.h
|   |   `-- jputypes.h
|   |-- jdi.c
|   |-- jdi.h
|   |-- jpuapi.c
|   |-- jpuapifunc.c
|   |-- jpuapifunc.h
|   |-- jpudecapi.c
|   |-- jpuencapi.c
|   |-- jpu.h
|   |-- jputable.h
|   |-- list.h
|   |-- mm.c
|   |-- mm.h
|   `-- regdefine.h
|-- sample
|   |-- dmabufheap                           //dmabuf分配管理
|   |   |-- BufferAllocator.cpp
|   |   |-- BufferAllocator.h
|   |   |-- BufferAllocatorWrapper.cpp
|   |   |-- BufferAllocatorWrapper.h
|   |   `-- dmabufheap-defs.h
|   |-- helper                               //一些utils
|   |   |-- bitstreamfeeder.c
|   |   |-- bitstreamwriter.c
|   |   |-- bsfeeder_fixedsize_impl.c
|   |   |-- datastructure.c
|   |   |-- datastructure.h
|   |   |-- jpuhelper.c
|   |   |-- jpulog.c
|   |   |-- jpulog.h
|   |   |-- main_helper.h
|   |   |-- platform.c
|   |   |-- platform.h
|   |   |-- yuv_feeder.c
|   |   `-- yuv_feeder.h
|   |-- main_dec_test.c                      //解码测试程序实现
|   `-- main_enc_test.c                      //编码测试程序实现
`-- usr
    `-- lib
        `-- systemd
            `-- system
                `-- jpu.service
```

### 2.4 编译说明

**Bianbu 桌面系统**

```shll
cd k1x-jpu
sudo apt-get build-dep k1x-jpu    #安装依赖
dpkg-buildpackage -us -uc -nc -b -j32
```

**Buildroot 系统**

```shell
cd k1x-jpu
mkdir out
cd out
cmake ..
make
make install
```
