sidebar_position: 4

# JPU

As a hardware module for JPEG image encoding and decoding, JPU (JPEG Processing Unit) improves encoding and decoding efficiency and reduces CPU load. K1's CPU offers complete test programs for reference.

## 1. Specification（To be supplement）

## 2. JPU Test Program

`k1x-jpu` encapsulates APIs for the application layer. Based on these APIs, it also integrates a set of programs for testing and verifying the K1's JPU functions, which also provide reference for application development (interface with the JPU for hardware encoding and decoding).

### 2.1 Installation Instruction

#### 2.1.1 Bianbu Desktop System

`k1x-jpu` is pre-integrated in the repository and can be installed via `apt` command. 

```shell
sudo apt update
sudo apt install k1x-jpu
```

#### 2.1.2 Buildroot System

Two methods for intergrating `k1x-jpu` into the system: 

- Enable the compilation and integration option for k1x-jpu when compiling the image (enabled by default), so that all related test programs are included in the generated image.
- If the `k1x-jpu` is not integrated into the img, you can only compile `k1x-jpu` manually, then copy the generated `bin` files to the `/usr/bin/` directory of the system for use. The specific `bin` files are described below.

### 2.2 Usage Instruction

The main test programs of `k1x-jpu`: 

- **jpu_dec_test**: Used for JPEG decoding test
- **jpu_enc_test**: Used for JPEG encoding test
- **libjpu.so**：JPU wrapper library

#### 2.2.1 jpu_dec_test

1. Basic usage examples:

```shell
//Decode input.jpeg to output.yuv
./jpu_dec_test --input=input.jpeg   --output=output.yuv

//Decode input.jpeg to output.yuv, with pixel format of output.yuv set to NV12. 
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=420 --ordering=nv12

//Decode input.jpeg to output.yuv, with pixel format of output.yuv set to YUYV将input.jpeg
./jpu_dec_test --input=input.jpeg   --output=output.yuv --subsample=422 --ordering=yuyv
```

2. Parameter Description

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

1. Basic usage examples:
 
```shell
//Encode imput.yuv to output.jpeg with a quality of 10, using the configurations in enc.cfg.
./jpu_enc_test --cfg-dir=xxx --yuv-dir=xxx --input=enc.cfg --output=output.jpeg --profiling=1 --loop_count=1 --quality=10


//reference for env.cfg configuration:
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

2. Parameter Description

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

### 2.3 Code Structure

Code of `k1x-jpu` locates at:

```shell
package-src/k1x-jpu
```

Code structure and brief description are as follows:

```shell
|-- CMakeLists.txt                //cmake build script
|-- debian                        //deb package build configration and script
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
|   |-- dmabufheap                           //dmabuf allocation and management
|   |   |-- BufferAllocator.cpp
|   |   |-- BufferAllocator.h
|   |   |-- BufferAllocatorWrapper.cpp
|   |   |-- BufferAllocatorWrapper.h
|   |   `-- dmabufheap-defs.h
|   |-- helper                               //utils
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
|   |-- main_dec_test.c                      //decoding test program implementation
|   `-- main_enc_test.c                      //encoding test program implementation
`-- usr
    `-- lib
        `-- systemd
            `-- system
                `-- jpu.service
```

### 2.4 Compilation Instruction

**Bianbu Desktop System**

```shll
cd k1x-jpu
sudo apt-get build-dep k1x-jpu    #Install Dependencies
dpkg-buildpackage -us -uc -nc -b -j32
```

**Buildroot System**

```shell
cd k1x-jpu
mkdir out
cd out
cmake ..
make
make install
```
