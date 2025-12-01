# JPU

JPU（Jpeg Processing Unit）是进行 Jpeg 图像编解码的硬件，能够提高 Jpeg 的编解码效率并减少 CPU 负荷。K1 的 JPU 提供了完整的测试程序供参考。

## 1 规格（待补充）

## 2 JPU 测试程序

k1x-jpu 内部封装了给应用层的 API，同时基于该 API 集成一套用于测试验证 K1 芯片的 JPU（Jpeg Processing Unit，负责视频的编解码工作）功能的程序集，也可以作为客户开发自己的应用程序（需要对接 JPU 进行硬件编解码）的参考。

### 2.1 安装说明

#### 2.1.1 Bianbu 桌面系统

源中已经集成了 k1x-jpu，直接使用 apt 命令来安装即可。

```shell
sudo apt update
sudo apt install k1x-jpu
```

#### 2.1.2 Buildroot 系统

2 种方法将 k1x-jpu 集成到系统中：

- 在编译 img 的时候，将 k1x-vpu-test 的编译集成选项打开**（默认已经打开）**，这样，编译的 img 中默认就有 k1x-jpu 相关的测试程序
- 如果编译 img 的时候，没有打开 k1x-jpu 的编译集成选项，img 中没有 k1x-jpu 相关的测试程序，只能手动编译 k1x-jpu，然后将生成的 bin 拷贝到系统的/usr/bin/目录中来使用，具体包括 bin，下面有说明

### 2.2 使用说明

k1x-jpu 的测试程序集中主要包含下面几个测试程序：

- **jpu_dec_test**：用于 JPEG 的解码测试
- **jpu_enc_test**：用于 JPEG 的编码测试
- **libjpu.so**：JPU 的 API 封装库

#### 2.2.1 jpu_dec_test

1. 一些基本用法

```shell
//将input.jpeg解码为output.yuv
./jpu_dec_test --input=input.jpeg   --output=output.yuv

//将input.jpeg解码为output.yuv，output.yuv的像素格式为NV12
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=420 --ordering=nv12

//将input.jpeg解码为output.yuv，output.yuv的像素格式为YUYV
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=422 --ordering=yuyv
```

2. 参数说明

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

1. 一些基本用法

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

2. 参数说明

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

k1x-jpu 的代码位置在：

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

Bianbu 桌面系统

```shll
cd k1x-jpu
sudo apt-get build-dep k1x-jpu    #安装依赖
dpkg-buildpackage -us -uc -nc -b -j32
```

Buildroot 系统

```shell
cd k1x-jpu
mkdir out
cd out
cmake ..
make
make install
```
