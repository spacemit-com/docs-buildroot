sidebar_position: 2

# MPP 

**MPP (Multimedia Processing Platform)** is a component of the self-developed **Bianbu OS**. 
It is designed to encapsulate the differences of hardware encoding and decoding across multiple platforms, providing a unified set of APIs for developers.

## 1. Module Introduction

### 1.1 Terms

- **MPP (Multimedia Processing Platform)**  
  Architure of multimedia processing platform. 

- **MPI (Multimedia Processing Interface)**  
  The unified API interface provided by MPP for upper-layer access. 

- **MPP AL**  
  Abstract Layer, which unifies multimedia interfaces from different IPs, SoCs and solutions.

- **Packet**  
  Data packet, compressed video data unit (such as H.264/ H.265 video stream). It is used before decoding and after encoding. 

- **Frame**  
  Data frame, uncompressed video data unit (such as YUV420 image). It is used before encoding and after decoding. 

### 1.2 Module Function

MPP mainly consists of the following parts:

- **VDEC**  
  Video decoding submodule and open APIs, mainly used for decoding compressed data stream (Packet) to raw frame (Frame).

- **VENC**  
  Video encoding submodule and open APIs, mainly used for encoding raw frame (RGB/YUV) to compressed video stream (Packet).

- **G2D**  
  2D graphics processing acceleration submodule and open APIs, mainly used for frame operations such as format conversion, scaling, rotation, and clipping.

- **BIND System**  
  It supports dynamic binding of multiple modules.

- **AL(Abstract Layer)**  
  Multi-platform support layer that abstracts away hardware differences.

- **VI**  
  Video input submodule and open APIs. Currently, it only supports file input and standard V4L2 input.

- **VO**  
  Video output submodule and open APIs. Currently, it only supports file output and video SDL2 output. 

Parts not yet included:

- **AI/AO**  
  Audio input/output. It uses the standard path: pipewire -> alsa-lib -> alsa driver.

- **AENC/ADEC**  
  Not supported yet. It is recommended to use software implementations of GStream or FFmpeg.

### 1.3 Configuration Instructions

#### 1.3.1 Debug Configuration

| Configuration item                                 | Default Value                       | Function Description                                        |
| ----------------------------------- | ------------------------- | ------------------------------------------- |
| **MPP\_PRINT\_BUFFER**              | `0`                        | When set to `1`, it will print the real-time buffer status.                          |
| **MPP\_SAVE\_OUTPUT\_BUFFER**       | `0`                         | When set to `1`, it can save the decoded YUV buffer. (Note: YUV buffers are large, which may cause playback stuttering and occupy large storage.) |
| **MPP\_SAVE\_OUTPUT\_BUFFER\_PATH** | `/home/bianbu/output.yuv` | YUV output path only valid when `MPP_SAVE_OUTPUT_BUFFER=1`. |
| **MPP\_PRINT\_UNFREE\_PACKET**      | `0`                         | When set to `1`, it will print the allocation and release status of packets in real time.                     |
| **MPP\_PRINT\_UNFREE\_FRAME**       | `0`                         | When set to `1`, it will print the allocation and release status of frame in real time.                      |
| **MPP\_PRINT\_UNFREE\_DMABUF**      | `0`                        | When set to `1`, it will print the allocation and release status of dmabuf in real time.                     |

Examples：

```shell
# Need to print buffer status in real time
export MPP_PRINT_BUFFER=1

# Save decoded YUV data to /mnt/a.yuv
export MPP_SAVE_OUTPUT_BUFFER=1
export MPP_SAVE_OUTPUT_BUFFER_PATH=/mnt/a.yuv
```

#### 1.3.2 Module Configuration

No configuration parameters are provided yet.

### 1.4 Source Code

#### 1.4.1 Source Code Location

The source code of MPP locates at:

```shell
buildroot-sdk/package-src/mpp
```

#### 1.4.2 Source Code Compilation

In the buildroot solution, compilation is enabled by default. If you need to recompile after modifying the code, execute:

```shell
make mpp-rebuild
```

#### 1.4.3 Source Code Structure

The source code structure of MPP and its brief description are as follows (the source code structure has been simplified):

```shell
|-- al                               // AL (Abstract Layer): Code for interfacing with functional modules or drivers across various platforms.
|   |-- CMakeLists.txt
|   |-- include
|   |   |-- al_interface_base.h      // AL layer interface base calss
|   |   |-- al_interface_dec.h       // AL layer decoding interface base class, inherited from base
|   |   |-- al_interface_enc.h       // AL layer encoding interface base class, inherited from base
|   |   |-- al_interface_g2d.h       // AL layer image conversion interface base class, inherited from base
|   |   |-- al_interface_vi.h        // AL layer video input interface base class, inherited from base
|   |   `-- al_interface_vo.h        // AL layer video output interface base class, inherited from base
|   |-- vcodec                       // Interface with multi-platform encoding and decoding modules or drivers
|   |   |-- chipmedia                // Interface with chipmedia IP codec
|   |   |   |-- CMakeLists.txt
|   |   |   `-- starfive             // Interface with starfive codec API (not implemented yet)
|   |   |       |-- sfdec_plugin.c
|   |   |       `-- sfenc_plugin.c
|   |   |-- CMakeLists.txt
|   |   |-- debug                    // Virtual decoding plugin, outputs simple solid-color images for debugging
|   |   |   |-- CMakeLists.txt
|   |   |   `-- fake_dec_plugin.c
|   |   |-- ffmpeg                   // Interface with FFmpeg software encoding and decoding for debugging
|   |   |   |-- CMakeLists.txt
|   |   |   |-- ffmpegdec.c
|   |   |   |-- ffmpegenc.c
|   |   |   `-- ffmpegswscale.c
|   |   |-- k1
|   |   |   |-- CMakeLists.txt
|   |   |   `-- jpu                  // Interface with encoding and decoding of K1 JPU (not implemented yet)
|   |   |       |-- include
|   |   |       |-- jpudec.c
|   |   |       `-- jpuenc.c
|   |   |-- openh264                 // Interface with openh264 software encoding and decoding library for debugging
|   |   |   |-- CMakeLists.txt
|   |   |   |-- include
|   |   |   |   `-- wels
|   |   |   |       |-- codec_api.h
|   |   |   |       |-- codec_app_def.h
|   |   |   |       |-- codec_def.h
|   |   |   |       `-- codec_ver.h
|   |   |   |-- openh264dec.cpp
|   |   |   |-- openh264enc.cpp
|   |   |   `-- README.md
|   |   |-- openmax                  // Interface with openmax
|   |   |   |-- CMakeLists.txt
|   |   |   |-- include
|   |   |   |   |-- khronos
|   |   |   |   |   |-- OMX_Audio.h
|   |   |   |   |   |-- OMX_ComponentExt.h
|   |   |   |   |   |-- OMX_Component.h
|   |   |   |   |   |-- OMX_ContentPipe.h
|   |   |   |   |   |-- OMX_CoreExt.h
|   |   |   |   |   |-- OMX_Core.h
|   |   |   |   |   |-- OMX_ImageExt.h
|   |   |   |   |   |-- OMX_Image.h
|   |   |   |   |   |-- OMX_IndexExt.h
|   |   |   |   |   |-- OMX_Index.h
|   |   |   |   |   |-- OMX_IVCommon.h
|   |   |   |   |   |-- OMX_Other.h
|   |   |   |   |   |-- OMX_Types.h
|   |   |   |   |   |-- OMX_VideoExt.h
|   |   |   |   |   `-- OMX_Video.h
|   |   |   |   |-- sfomxil_find_dec_library.h
|   |   |   |   `-- sfomxil_find_enc_library.h
|   |   |   `-- starfive             // Interface with the starfive openmaxIL layer for video encoding and decoding
|   |   |       |-- sfomxil_dec_plugin.c
|   |   |       `-- sfomxil_enc_plugin.c
|   |   |-- v4l2                     // Interface with V4L2 encoding and decoding
|   |   |   |-- CMakeLists.txt
|   |   |   |-- linlonv5v7           // Interface with linlonv5v7 encoding and decoding（K1）
|   |   |   |   |-- include
|   |   |   |   |   |-- linlonv5v7_buffer.h
|   |   |   |   |   |-- linlonv5v7_codec.h
|   |   |   |   |   |-- linlonv5v7_constant.h
|   |   |   |   |   |-- linlonv5v7_port.h
|   |   |   |   |   `-- mvx-v4l2-controls.h
|   |   |   |   |-- linlonv5v7_buffer.c
|   |   |   |   |-- linlonv5v7_codec.c
|   |   |   |   |-- linlonv5v7_dec.c
|   |   |   |   |-- linlonv5v7_enc.c
|   |   |   |   `-- linlonv5v7_port.c
|   |   |   `-- standard
|   |   |       |-- v4l2dec.c
|   |   |       `-- v4l2enc.c
|   |   `-- verisilicon              // Interface with verisilicon encoding and decoding (not implemented yet)
|   |       |-- CMakeLists.txt
|   |       `-- vc8000.c
|   |-- vi                           // Interface with multi-platform video input modules or drivers
|   |   |-- CMakeLists.txt
|   |   |-- file                     // Video input via File
|   |   |   |-- include
|   |   |   |   |-- defaultparse.h
|   |   |   |   |-- h264parse.h
|   |   |   |   |-- h265parse.h
|   |   |   |   |-- mjpegparse.h
|   |   |   |   `-- parse.h
|   |   |   |-- parse
|   |   |   |   |-- defaultparse.c
|   |   |   |   |-- h264parse.c
|   |   |   |   |-- h265parse.c
|   |   |   |   |-- mjpegparse.c
|   |   |   |   `-- parse.c
|   |   |   `-- vi_file.c
|   |   |-- k1                       // Video input via K1 ISP (not implemented yet)
|   |   |   `-- cam
|   |   |       |-- include
|   |   |       `-- vi_k1_cam.c
|   |   `-- v4l2                     // Video output via standard V4L2
|   |       |-- include
|   |       `-- vi_v4l2.c
|   |-- vo                           // Interface with multi-platform video output modules or drivers
|   |   |-- CMakeLists.txt
|   |   |-- file                     // Video output to File
|   |   |   `-- vo_file.c
|   |   `-- sdl2                     // Video output via SDL2
|   |       |-- include
|   |       |   |-- begin_code.h
|   |       |   |-- close_code.h
|   |       |   |-- SDL_assert.h
|   |       |   |-- SDL_atomic.h
|   |       |   `-- SDL_vulkan.h
|   |       `-- vo_sdl2.c
|   `-- vps                          // Interface with multi-platform video processing modules or drivers
|       |-- CMakeLists.txt
|       `-- k1
|           |-- CMakeLists.txt
|           `-- v2d                  // Interface with K1 v2d module, implement basic framework
|               |-- include
|               |   |-- asr_v2d_api.h
|               |   `-- asr_v2d_type.h
|               `-- v2d.c
|-- cmake                            // Find system modules
|   `-- modules
|       |-- Findlibavcodec.cmake
|       |-- Findlibopenh264.cmake
|       |-- Findlibsfdec.cmake
|       |-- Findlibsfenc.cmake
|       `-- Findlibsf-omx-il.cmake
|-- CMakeLists.txt
|-- compile_install_completely.sh
|-- debian                           // Deb package build directory
|   |-- bianbu.conf
|   |-- changelog
|   |-- compat
|   |-- control
|   |-- copyright
|   |-- install
|   |-- README.Debian
|   |-- rules
|   |-- source
|   |   `-- format
|   |-- usr
|   |   `-- lib
|   |       `-- udev
|   |           `-- rules.d
|   |               `-- 99-video.rules
|   `-- watch
|-- doc                              // some documents
|   |-- C_naming_conventions.md
|   `-- MPP Module Design Document V0.1.pdf
|-- do_test.sh
|-- format.sh
|-- include                          // API header file
|   |-- data.h                       // MppData data base class
|   |-- dataqueue.h                  // Data queue management API
|   |-- dmabufwrapper.h              // dmabuf management API
|   |-- frame.h                      // frame management API
|   |-- g2d.h                        // Image processing API
|   |-- packet.h                     // packet management API
|   |-- para.h                       // Parameter struct
|   |-- processflow.h
|   |-- ringbuffer.h                 // Ring buffer management API
|   |-- sys.h                        // SYS-related API
|   |-- vdec.h                       // Video decoding API
|   `-- venc.h                       // Video decoding API
|   |-- vi.h                         // Video input API
|   `-- vo.h                         // Video output API
|-- LICENSE
|-- mpi                              // API interface implementation
|   |-- CMakeLists.txt
|   |-- g2d.c
|   |-- include
|   |   `-- module.h
|   |-- module.c
|   |-- sys.c
|   |-- vdec.c
|   |-- venc.c
|   |-- vi.c
|   `-- vo.c
|-- mpp.cppcheck
|-- pack_to_tar_gz.sh
|-- pkgconfig
|   `-- spacemit_mpp.pc.cmake
|-- remove_space_end_of_line.sh
|-- test                             // test programs, test scripts, test files, etc.
|   |-- CMakeLists.txt
|   |-- g2d_test.c
|   |-- include
|   |   |-- argument.h
|   |   |-- const.h
|   |   |-- defaultparse.h
|   |   |-- h264parse.h
|   |   |-- h265parse.h
|   |   |-- mjpegparse.h
|   |   `-- parse.h
|   |-- parse
|   |   |-- defaultparse.c
|   |   |-- h264parse.c
|   |   |-- h265parse.c
|   |   |-- mjpegparse.c
|   |   `-- parse.c
|   |-- test_script
|   |   |-- cases
|   |   |   `-- vdec.csv
|   |   |-- streams
|   |   `-- vdec_test.sh
|   |-- test_sys_vdec_venc_one_frame.c
|   |-- test_sys_vdec_venc_vdec_one_frame.c
|   |-- test_sys_venc_vdec_one_frame.c
|   |-- vi_file_vdec_venc_sync_userptr_vo_file_test.c
|   |-- vi_file_vdec_vo_test.c
|   |-- vi_file_venc_sync_userptr_vo_file_test.c
|   `-- vi_v4l2_vo_test.c
|-- thirdparty
|   |-- ffmpeg_compile_install.md
|   `-- openh264_compile_install.md
`-- utils                            // utils
    |-- CMakeLists.txt
    |-- dataqueue.c                  // Data queue implementation
    |-- dmabufwrapper.c              // dmabuf management implementation
    |-- env.c
    |-- frame.c                      // frame management implementation
    |-- include
    |   |-- env.h
    |   |-- log.h
    |   |-- resolution_utils.h
    |   |-- type.h
    |   `-- v4l2_utils.h
    |-- log.c
    |-- os
    |   |-- include
    |   |   |-- os_env.h
    |   |   `-- os_log.h
    |   `-- linux
    |       |-- os_env.c
    |       `-- os_log.c
    |-- packet.c                     // packet management implementation
    |-- resolution_utils.c
    |-- ringbuffer.c
    |-- utils.c
    `-- v4l2_utils.c
```

## 2. MPP Framework Structure Diagram

![](static/IjzAbsjbyoMgXkx1Cq5cUMNLnYe.png)

From the perspective of framework structure, it is mainly divided into 2 layers as follows.

- **MPI (Interface layer)**  
  Mainly includes the upper-layer APIs and their implementations.
- **MPP AL (Abstract layer)**  
  Abstract away differences across various platforms and hardware to achieve unified invocation.

From the perspective of functionality, it is divided into：

- **MPI**  
  External API interface layer
- **MPP AL**  
  Multi-platform hardware abstract layer
- **TESTS**  
  Test programs and test streams for function verification
- **UTILS**  
  Tool package, implementation of basic functionalities, including PACKET/FRAME management, log output, reading and writing of environment variables, etc.
- **SYS**  
  Implementation of dynamic plugin loading and the BIND system.

## 3. Key Processes

### 3.1 Decoding Process

![](static/JI81bbaDyo42MaxuLJUcFGtynkP.png)

### 3.2 Encoding Process

![](static/OeWFbJXHDo1o4px5ftTcniJynve.png)

## 4. Data Structures

### 4.1 General Data Structures

#### 4.1.1 enum MppModuleType

This enumeration defines the supported plugin types, and plugins can be selected via this enumeration. AUTO indicates that plugins are selected according to the default priority (logic not yet perfected), while CODEC_V4L2_LINLONV5V7 is generally selected for encoding and decoding on K1.

```c
/***
 * @description: all codec mpp support
 */

/*

+-----------------------+---------+---------+-----------+
|                       | DECODER | ENCODER | CONVERTER |
+=======================+=========+=========+===========+
| CODEC_OPENH264        | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_FFMPEG          | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_SFDEC           | √       | x       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_SFENC           | x       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_CODADEC         | √(jpeg) | x       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_SFOMX           | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_V4L2            | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_FAKEDEC         | √       | x       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_V4L2_LINLONV5V7 | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| CODEC_K1_JPU          | √       | √       | x         |
+-----------------------+---------+---------+-----------+
| VO_SDL2               | x       | x       | x         |
+-----------------------+---------+---------+-----------+
| VO_FILE               | x       | x       | x         |
+-----------------------+---------+---------+-----------+
| VI_V4L2               | x       | x       | x         |
+-----------------------+---------+---------+-----------+
| VI_K1_CAM             | x       | x       | x         |
+-----------------------+---------+---------+-----------+
| VI_FILE               | x       | x       | x         |
+-----------------------+---------+---------+-----------+
| VPS_K1_V2D            | x       | x       | √         |
+-----------------------+---------+---------+-----------+

*/

typedef enum _MppModuleType {
  /***
   * auto mode, mpp select suitable codec.
   */
  CODEC_AUTO = 0,

  /***
   * use openh264 soft codec api, support decoder and encoder
   */
  CODEC_OPENH264,

  /***
   * use ffmpeg avcodec api, support decoder and encoder.
   */
  CODEC_FFMPEG,

  /***
   * use starfive wave511 vpu api for video decoder.
   */
  CODEC_SFDEC,

  /***
   * use starfive wave420l vpu api for video encoder.
   */
  CODEC_SFENC,

  /***
   * use starfive codaj12 vpu api for jpeg video decoder and encoder.
   */
  CODEC_CODADEC,

  /***
   * use starfive omx-il api for video decoder and encoder.
   */
  CODEC_SFOMX,

  /***
   * use V4L2 standard codec interface for video decoder and encoder.
   */
  CODEC_V4L2,

  /***
   * a fake decoder for test, send green frame to application layer.
   */
  CODEC_FAKEDEC,

  /***
   * use ARM LINLON VPU codec interface for video decoder and encoder.(K1)
   */
  CODEC_V4L2_LINLONV5V7,

  /***
   * use jpu for jpeg decoder and encoder (K1).
   */
  CODEC_K1_JPU,

  CODEC_MAX,

  /***
   * auto mode, mpp select suitable vo.
   */
  VO_AUTO = 100,

  /***
   * use sdl2 for output
   */
  VO_SDL2,
  VO_FILE,

  VO_MAX,

  /***
   * auto mode, mpp select suitable vi.
   */
  VI_AUTO = 200,

  /***
   * use standard v4l2 framework for input
   */
  VI_V4L2,

  /***
   * use K1 ISP for input
   */
  VI_K1_CAM,
  VI_FILE,

  VI_MAX,

  /***
   * auto mode, mpp select suitable vi.
   */
  VPS_AUTO = 300,

  /***
   * use v2d for graphic 2D convert (K1).
   */
  VPS_K1_V2D,

  VPS_MAX,
} MppModuleType;
```

In the code, the plugin library for a specific codec is dynamically loaded through the following interface.

```c
/**
 * @description: dlopen the video codec library by codec_type
 * @param {MppCodecType} codec_type : input, the codec need to be opened
 * @return {MppModule*} : the module context
 */
 MppModule*  module_init(MppCodecType codec_type)
```

#### 4.1.2 enum MppCodingType

This enumeration defines the supported encoding formats, including all formats supported by decoders and encoders. Each codec may only support some of these types; for example, openh264 only supports the encoding and decoding of H264.

```c
typedef enum _MppCodingType {
  CODING_UNKNOWN = 0,
  CODING_H263,
  CODING_H264,

  /***
   * Multiview Video Coding, 3D, etc.
   */
  CODING_H264_MVC,

  /***
   * no start code
   */
  CODING_H264_NO_SC,
  CODING_H265,
  CODING_MJPEG,
  CODING_JPEG,
  CODING_VP8,
  CODING_VP9,
  CODING_AV1,
  CODING_AVS,
  CODING_AVS2,
  CODING_MPEG1,
  CODING_MPEG2,
  CODING_MPEG4,
  CODING_RV,

  /***
   * ANNEX_G, Advanced Profile
   */
  CODING_VC1,

  /***
   * ANNEX_L, Simple and Main Profiles
   */
  CODING_VC1_ANNEX_L,
  CODING_FWHT,
  CODING_MAX,
} MppCodingType;
```

**Note.** Each codec has its own defined format type, which requires format conversion. There is an example of format mapping for `ffmpeg`:

```c
#define CODING_TYPE_MAPPING_DEFINE(Type, format)  \
    typedef struct _AL##Type##CodingTypeMapping {  \
        MppCodingType eMppCodingType;  \
        format e##Type##CodingType;  \
    } AL##Type##CodingTypeMapping;

#define CODING_TYPE_MAPPING_CONVERT(Type, type, format)  \
    static MppCodingType get_##type##_mpp_coding_type(format src_type)  \
    {  \
        S32 i = 0;  \
        S32 mapping_length = NUM_OF(stAL##Type##CodingTypeMapping);  \
        for(i = 0; i < mapping_length; i ++)  \
        {  \
            if(src_type == stAL##Type##CodingTypeMapping[i].e##Type##CodingType)  \
                return stAL##Type##CodingTypeMapping[i].eMppCodingType;  \
        }  \
  \
        mpp_loge("Can not find the mapping format, please check it !");  \
        return CODING_UNKNOWN;  \
    }  \
  \
    static format get_##type##_codec_coding_type(MppCodingType src_type)  \
    {  \
        S32 i = 0;  \
        S32 mapping_length = NUM_OF(stAL##Type##CodingTypeMapping);  \
        for(i = 0; i < mapping_length; i ++)  \
        {  \
            if(src_type == stAL##Type##CodingTypeMapping[i].eMppCodingType)  \
                return stAL##Type##CodingTypeMapping[i].e##Type##CodingType;  \
        }  \
  \
        mpp_loge("Can not find the mapping coding type, please check it !");  \
        return CODING_UNKNOWN;  \
    }

...

CODING_TYPE_MAPPING_DEFINE(FFMpegDec, enum AVCodecID)
static const ALFFMpegDecCodingTypeMapping stALFFMpegDecCodingTypeMapping[] = {
    {CODING_H264, AV_CODEC_ID_H264},
    {CODING_H265, AV_CODEC_ID_H265},
    {CODING_MJPEG, AV_CODEC_ID_MJPEG},
    {CODING_VP8, AV_CODEC_ID_VP8},
    {CODING_VP9, AV_CODEC_ID_VP9},
    {CODING_AV1, AV_CODEC_ID_NONE},
    {CODING_AVS, AV_CODEC_ID_AVS},
    {CODING_AVS2, AV_CODEC_ID_AVS2},
    {CODING_MPEG1, AV_CODEC_ID_MPEG1VIDEO},
    {CODING_MPEG2, AV_CODEC_ID_MPEG2VIDEO},
    {CODING_MPEG4, AV_CODEC_ID_MPEG4},
};
CODING_TYPE_MAPPING_CONVERT(FFMpegDec, ffmpegdec, enum AVCodecID)
```

#### 4.1.3 enum MppPixelFormat

This enumeration defines the supported pixel formats, including all formats supported by decoders and encoders. Each codec may only support some of these types.

```c
/***
 * @description: pixelformat mpp or some other platform may use.
 */
typedef enum _MppPixelFormat {
  PIXEL_FORMAT_UNKNOWN = 0,

  /***
   * YYYYYYYYVVUU
   */
  PIXEL_FORMAT_YV12,

  /***
   * YYYYYYYYUUVV  YU12/YUV420P is the same
   */
  PIXEL_FORMAT_I420,

  /***
   * YYYYYYYYVUVU
   */
  PIXEL_FORMAT_NV21,

  /***
   * YYYYYYYYUVUV
   */
  PIXEL_FORMAT_NV12,

  /***
   * 11111111 11000000, 16bit only use 10bit
   */
  PIXEL_FORMAT_YV12_P010,

  /***
   * 11111111 11000000, 16bit only use 10bit
   */
  PIXEL_FORMAT_I420_P010,

  /***
   * 11111111 11000000, 16bit only use 10bit
   */
  PIXEL_FORMAT_NV21_P010,

  /***
   * 11111111 11000000, 16bit only use 10bit
   */
  PIXEL_FORMAT_NV12_P010,
  PIXEL_FORMAT_YV12_P016,
  PIXEL_FORMAT_I420_P016,
  PIXEL_FORMAT_NV21_P016,
  PIXEL_FORMAT_NV12_P016,

  /***
   * YYYYUUVV, YU16 is the same
   */
  PIXEL_FORMAT_YUV422P,

  /***
   * YYYYVVUU
   */
  PIXEL_FORMAT_YV16,

  /***
   * YYYYUVUV  NV16 is the same
   */
  PIXEL_FORMAT_YUV422SP,

  /***
   * YYYYVUVU
   */
  PIXEL_FORMAT_NV61,
  PIXEL_FORMAT_YUV422P_P010,
  PIXEL_FORMAT_YV16_P010,
  PIXEL_FORMAT_YUV422SP_P010,
  PIXEL_FORMAT_NV61_P010,

  /***
   * YYUUVV
   */
  PIXEL_FORMAT_YUV444P,

  /***
   * YYUVUV
   */
  PIXEL_FORMAT_YUV444SP,
  PIXEL_FORMAT_YUYV,
  PIXEL_FORMAT_YVYU,
  PIXEL_FORMAT_UYVY,
  PIXEL_FORMAT_VYUY,
  PIXEL_FORMAT_YUV_MB32_420,
  PIXEL_FORMAT_YUV_MB32_422,
  PIXEL_FORMAT_YUV_MB32_444,
  PIXEL_FORMAT_YUV_MAX,

  PIXEL_FORMAT_RGB_MIN,
  PIXEL_FORMAT_RGBA,
  PIXEL_FORMAT_ARGB,
  PIXEL_FORMAT_ABGR,
  PIXEL_FORMAT_BGRA,
  PIXEL_FORMAT_RGBA_5658,
  PIXEL_FORMAT_ARGB_8565,
  PIXEL_FORMAT_ABGR_8565,
  PIXEL_FORMAT_BGRA_5658,
  PIXEL_FORMAT_RGBA_5551,
  PIXEL_FORMAT_ARGB_1555,
  PIXEL_FORMAT_ABGR_1555,
  PIXEL_FORMAT_BGRA_5551,
  PIXEL_FORMAT_RGBA_4444,
  PIXEL_FORMAT_ARGB_4444,
  PIXEL_FORMAT_ABGR_4444,
  PIXEL_FORMAT_BGRA_4444,
  PIXEL_FORMAT_RGB_888,
  PIXEL_FORMAT_BGR_888,
  PIXEL_FORMAT_RGB_565,
  PIXEL_FORMAT_BGR_565,
  PIXEL_FORMAT_RGB_555,
  PIXEL_FORMAT_BGR_555,
  PIXEL_FORMAT_RGB_444,
  PIXEL_FORMAT_BGR_444,
  PIXEL_FORMAT_RGB_MAX,

  PIXEL_FORMAT_AFBC_YUV420_8,
  PIXEL_FORMAT_AFBC_YUV420_10,
  PIXEL_FORMAT_AFBC_YUV422_8,
  PIXEL_FORMAT_AFBC_YUV422_10,

  /***
   * for usb camera
   */
  PIXEL_FORMAT_H264,
  PIXEL_FORMAT_MJPEG,

  PIXEL_FORMAT_MAX,
} MppPixelFormat;
```

**Note.** Each codec has its own defined format type, which requires format mapping. There is an example of format mapping for `ffmpeg`:

```c
#define PIXEL_FORMAT_MAPPING_DEFINE(Type, format)  \
    typedef struct _AL##Type##PixelFormatMapping {  \
        MppPixelFormat eMppPixelFormat;  \
        format e##Type##PixelFormat;  \
    } AL##Type##PixelFormatMapping;

#define PIXEL_FORMAT_MAPPING_CONVERT(Type, type, format)  \
    static MppPixelFormat get_##type##_mpp_pixel_format(format src_format)  \
    {  \
        S32 i = 0;  \
        S32 mapping_length = NUM_OF(stAL##Type##PixelFormatMapping);  \
        for(i = 0; i < mapping_length; i ++)  \
        {  \
            if(src_format == stAL##Type##PixelFormatMapping[i].e##Type##PixelFormat)  \
                return stAL##Type##PixelFormatMapping[i].eMppPixelFormat;  \
        }  \
  \
        mpp_loge("Can not find the mapping format, please check it !");  \
        return PIXEL_FORMAT_UNKNOWN;  \
    }  \
  \
    static format get_##type##_codec_pixel_format(MppPixelFormat src_format)  \
    {  \
        S32 i = 0;  \
        S32 mapping_length = NUM_OF(stAL##Type##PixelFormatMapping);  \
        for(i = 0; i < mapping_length; i ++)  \
        {  \
            if(src_format == stAL##Type##PixelFormatMapping[i].eMppPixelFormat)  \
                return stAL##Type##PixelFormatMapping[i].e##Type##PixelFormat;  \
        }  \
  \
        mpp_loge("Can not find the mapping format, please check it !");  \
        return (format)0;  \
    }

...

PIXEL_FORMAT_MAPPING_DEFINE(FFMpegDec, enum AVPixelFormat)
static const ALFFMpegDecPixelFormatMapping stALFFMpegDecPixelFormatMapping[] = {
    {PIXEL_FORMAT_I420, AV_PIX_FMT_YUV420P},
    {PIXEL_FORMAT_NV12, AV_PIX_FMT_NV12},
    {PIXEL_FORMAT_YVYU, AV_PIX_FMT_YVYU422},
    {PIXEL_FORMAT_UYVY, AV_PIX_FMT_UYVY422},
    {PIXEL_FORMAT_YUYV, AV_PIX_FMT_YUYV422},
    {PIXEL_FORMAT_RGBA, AV_PIX_FMT_RGBA},
    {PIXEL_FORMAT_BGRA, AV_PIX_FMT_BGRA},
    {PIXEL_FORMAT_ARGB, AV_PIX_FMT_ARGB},
    {PIXEL_FORMAT_ABGR, AV_PIX_FMT_ABGR},
};
PIXEL_FORMAT_MAPPING_CONVERT(FFMpegDec, ffmpegdec, enum AVPixelFormat)
```

#### 4.1.4 struct MppData

Data base class, MppPacket and MppFrame inherit from MppData. 

```c
/*
 *                  +------------------------+
 *                  |       MppData          |
 *                  +------------------------+
 *                  |   eType                |
 *                  +-----------^------------+
 *                              |
 *            +-----------------+---------------+
 *            |                                 |
 * +----------+-------------+       +-----------+-----------+
 * |       MppPacket        |       |       MppFrame        |
 * +------------------------+       +-----------------------+
 * |   eBaseData            |       |   eBaseData           |
 * |   pData                |       |   nDataUsedNum        |
 * |   nLength              |       |   pData0              |
 * |                        |       |   pData1              |
 * |                        |       |   pData2              |
 * +------------------------+       +-----------------------+
 *
 */

/***
 * @description: mpp data type struct.
 *
 */
typedef enum _MppDataType {
  /***
   * stream type, bitstream, un-decoded data or encoded data maybe.
   */
  MPP_DATA_STREAM = 1,

  /***
   * frame type, YUV/RGB format, decoded data or un-encoded data maybe.
   */
  MPP_DATA_FRAME = 2,

  MPP_DATA_UNKNOWN = 1023
} MppDataType;

/***
 * @description: abstruct MppData struct.
 *
 * important struct.
 *
 * data abstruct from MppFrame and MppPacket.
 */
typedef struct _MppData {
  MppDataType eType;
} MppData;
```

#### 4.1.5 enum MppReturnValue

MPP return value：

```c
typedef enum _MppReturnValue {
  MPP_OK = 0,

  /***
   * error about memory
   */
  MPP_OUT_OF_MEM = -1,
  MPP_MALLOC_FAILED = -2,
  MPP_MMAP_FAILED = -3,
  MPP_MUNMAP_FAILED = -4,
  MPP_NULL_POINTER = -5,

  /***
   * error about file
   */
  MPP_FILE_NOT_EXIST = -100,
  MPP_OPEN_FAILED = -101,
  MPP_IOCTL_FAILED = -102,
  MPP_CLOSE_FAILED = -103,
  MPP_POLL_FAILED = -104,

  /***
   * error about codec
   */
  MPP_NO_STREAM = -200,
  MPP_NO_FRAME = -201,
  MPP_DECODER_ERROR = -202,
  MPP_ENCODER_ERROR = -203,
  MPP_CONVERTER_ERROR = -204,
  MPP_CODER_EOS = -205,
  MPP_CODER_NO_DATA = -206,
  MPP_RESOLUTION_CHANGED = -207,
  MPP_ERROR_FRAME = -208,
  MPP_CODER_NULL_DATA = -209,

  /***
   * error about dataqueue
   */
  MPP_DATAQUEUE_FULL = -300,
  MPP_DATAQUEUE_EMPTY = -301,

  /***
   * other
   */
  MPP_INIT_FAILED = -400,
  MPP_CHECK_FAILED = -401,
  MPP_BIND_NOT_MATCH = -402,
  MPP_NOT_SUPPORTED_FORMAT = -403,

  /*unknown error*/
  MPP_ERROR_UNKNOWN = -1023
} MppReturnValue;

```

### 4.2 Decoding Data Structure

#### 4.2.1 struct MppVdecCtx

Create and initialize video decoder context via the VDEC_CreateChannel and VDEC_Init.

```c
typedef struct _MppVdecCtx {
  MppProcessNode pNode;        ; Used for pipeline creation in the bind system
  MppModuleType eCodecType;    ; Selected decoding plugin
  MppModule *pModule;          ; Context of the dynamically loaded decoding plugin
  MppVdecPara stVdecPara;      ; Decoding prameter set
} MppVdecCtx;
```

#### 4.2.2 struct MppVdecPara

Decoder Parameter Structure

```c
/***
 * @description: para sent and get between application and decoder.
 */
typedef struct _MppVdecPara {
  /***
   * set to MPP
   */
  MppCodingType eCodingType;                   // Video encoding format
  S32 nProfile;                                // Video encoding profile

  /***
   * read from MPP
   */
  MppFrameBufferType eFrameBufferType;         // buffer class for frame
  MppDataTransmissinMode eDataTransmissinMode; //Transmission type of input and output buffers

  /***
   * set to MPP
   */
  S32 nWidth;                                  // Video width
  S32 nHeight;                                 // Video height
  S32 nAlign;                                  // Video align
  S32 nScale;                                  // Video scale

  /***
   * Horizontal downscale ratio, [1, 256]
   * set to MPP
   */
  S32 nHorizonScaleDownRatio;                  // Video horizon scale down ratio

  /***
   * Vertical downscale ratio, [1, 128]
   * set to MPP
   */
  S32 nVerticalScaleDownRatio;                 // Video vertical scale down ratio

  /***
   * Downscaled frame width in pixels
   * set to MPP
   */
  S32 nHorizonScaleDownFrameWidth;             // Video horizon scale down frame width

  /***
   * Downscaled frame height in pixels
   * set to MPP
   */
  S32 nVerticalScaleDownFrameHeight;           // Video vertical scale down frame height 

  /***
   * 0, 90, 180, 270
   * set to MPP
   */
  S32 nRotateDegree;                           // Video rotate degree
  S32 bThumbnailMode;                          // Not used yet
  BOOL bIsInterlaced;                          // Whether it is an i-source
  BOOL bIsFrameReordering;
  BOOL bIgnoreStreamHeaders;
  MppPixelFormat eOutputPixelFormat;           // Output pixel format
  BOOL bNoBFrames;
  BOOL bDisable3D;
  BOOL bSupportMaf;
  BOOL bDispErrorFrame;
  BOOL bInputBlockModeEnable;                  // Whether block mode is enabled for bitstream input
  BOOL bOutputBlockModeEnable;                 // Whether block mode is enabled for frame output

  /***
   * read from MPP
   */
  /***
   * input buffer num that APP can use
   */
  S32 nInputQueueLeftNum;
  S32 nOutputQueueLeftNum;
  S32 nInputBufferNum;
  S32 nOutputBufferNum;
  void* pFrame[64];
  S32 nOldWidth;
  S32 nOldHeight;
  BOOL bIsResolutionChanged;

  /***
   * used for chromium
   */
  BOOL bIsBufferInDecoder[64];
  S32 nOutputBufferFd[64];
} MppVdecPara;
```

### 4.3 Encoding Data Structure

#### 4.3.1 struct MppVencCtx

The video encoder context is created and initialized via the VENC_CreateChannel and VENC_Init interfaces.

```c
typedef struct _MppVencCtx {
  MppProcessNode pNode;        // Used for pipeline creation in the bind system
  MppModuleType eCodecType;    // Selected encoding plugin
  MppVencPara stVencPara;      // Encoding parameter set
  MppModule *pModule;          // Context of the dynamically loaded encoding plugin
} MppVencCtx;
```

#### 4.3.2 struct MppVencPara

Encoder Parameter Structure 

```c
/***
 * @description: para sent and get between application and encoder.
 */
typedef struct _MppVencPara {
  /***
   * set to MPP
   */
  MppCodingType eCodingType;                   // Video encoding format
  MppPixelFormat PixelFormat;                  // Pixel format of the input frame

  /***
   * read from MPP
   */
  MppFrameBufferType eFrameBufferType;         // Buffer type of the frame
  MppDataTransmissinMode eDataTransmissinMode; // Transmission type of input and output buffers

  /***
   * set to MPP
   */
  S32 nWidth;                                  // Video width
  S32 nHeight;                                 // Video height
  S32 nAlign;                                  // Video align
  S32 nBitrate;                                // Video bitrate
  S32 nFrameRate;                              // Video rotate tate
  S32 nRotateDegree;                           // Video rotate degree
} MppVencPara;
```

### 4.4 G2D Data Structure

#### 4.4.1 struct MppG2dCtx

Image Processor Context

```c
typedef struct _MppG2dCtx {
  MppProcessNode pNode;         // Used for pipeline creation in the bind system
  MppModuleType eCodecType;     // Selected image processing plugin
  MppModule *pModule;           // Context of the image processing plugin dynamic library
  MppG2dPara stG2dPara;         // Image processing parameter set
} MppG2dCtx;
```

#### 4.4.2 struct MppG2dPara（Under Improvement）

```c
typedef struct _MppG2dPara {
  /***
   * read from MPP
   */
  MppFrameBufferType eInputFrameBufferType;     // Input frame buffer type
  MppFrameBufferType eOutputFrameBufferType;    // Output frame buffer type
  MppDataTransmissinMode eDataTransmissinMode;  // Input and output buffer transmission type

  /***
   * set to MPP
   */
  MppG2dCmd eG2dCmd;                            // Processing command
  MppPixelFormat eInputPixelFormat;             // Input frame pixel format
  MppPixelFormat eOutputPixelFormat;            // Output frame pixel format
  S32 nInputBufFd;                              // Input frame fd
  S32 nOutputBufFd;                             // Output frame fd
  S32 nInputWidth;                              // Input frame width
  S32 nInputHeight;                             // Input frame height
  S32 nOutputWidth;                             // Output frame width
  S32 nOutputHeight;                            // Output frame height
  S32 nInputBufSize;                            // Input frame size
  S32 nOutputBufSize;                           // Output frame size
  union {
    MppG2dFillColorPara sFillColorPara;         // Fill color parameter
    MppG2dCopyPara sCopyPara;                   // Copy parameter
    MppG2dScalePara sScalePara;                 // Scale parameter
    MppG2dRotatePara sRotatePara;               // Rotate parameter
    MppG2dMaskPara sMaskPara;
    MppG2dDrawPara sDrawPara;
  };
} MppG2dPara;
```

### 4.5 VI Data Structure

#### 4.5.1 struct MppViCtx

Video input context

```c
typedef struct _MppViCtx {
  MppProcessNode pNode;         // Used for pipeline creation in the bind system
  MppModuleType eViType;        // Selected video input plugin
  MppModule *pModule;           // Context of the video input plugin dynamic library
  MppViPara stViPara;           // Video input parameter set
} MppViCtx;
```

#### 4.5.2 struct MppViPara

```c
typedef struct _MppViPara {
  MppFrameBufferType eFrameBufferType;          // Input frame buffer type
  MppDataTransmissinMode eDataTransmissinMode;  // Input and output transmission type
  BOOL bIsFrame;

  /***
   * for frame
   */
  MppPixelFormat ePixelFormat;                  // Input frame pixel format
  S32 nWidth;                                   // Input frame width
  S32 nHeight;                                  // Input frame height
  S32 nStride;                                  // Input frame stride

  /***
   * for packet
   */
  MppCodingType eCodingType;                    // Input packet video encoding format

  /***
   * for vi v4l2
   */
  S32 nBufferNum;                               // Requested buffer number
  U8* pVideoDeviceName;                         // V4L2 device code
  /***
   * for vi file
   */
  U8* pInputFileName;                           // Input file path
} MppViPara;
```

### 4.6 VO Data Structure

#### 4.6.1 struct MppVoCtx

Video input context

```c
typedef struct _MppVoCtx {
  MppProcessNode pNode;         // Used for pipeline creation in the bind system
  MppModuleType eVoType;        // Selected video output plugin
  MppModule *pModule;           // Context of the video outputput plugin dynamic library
  MppVoPara stVoPara;           // Video output parameter set
} MppVoCtx;
```

#### 4.6.2 struct MppVoPara

```c
typedef struct _MppVoPara {
  MppFrameBufferType eFrameBufferType;          // Output frame buffer type
  MppDataTransmissinMode eDataTransmissinMode;  // Input and output buffer transmission type
  BOOL bIsFrame;

  /***
   * for frame
   */
  MppPixelFormat ePixelFormat;                  // Output frame pixel format
  S32 nWidth;                                   // Output frame width
  S32 nHeight;                                  // Output frame height
  S32 nStride;                                  // Output frame stride

  /***
   * for vo file
   */
  U8* pOutputFileName;                          // Output file path
} MppVoPara;
```

### 4.7 SYS Data Structure（Under Improvement）

#### 4.7.1 struct MppProcessFlowCtx

BIND system pipeline context.

```c
/***
 * @description: main process flow struct
 *
 * manage the whole process flow, maybe include some process nodes.
 */
typedef struct _MppProcessFlowCtx {
  /***
   * the total node num of the flow.
   */
  S32 nNodeNum;

  /***
   * every node in this flow.
   */
  MppProcessNode *pNode[MAX_NODE_NUM];

  /***
   * data transmission thread between nodes.
   */
  pthread_t pthread[MAX_NODE_NUM];
} MppProcessFlowCtx;
```

#### 4.7.2 struct MppProcessNode

Definition of each node in the BIND system pipeline

```c
/***
 * @description: main process node struct
 *
 * manage every process node.
 */
typedef struct _MppProcessNode {
  S32 nNodeId;
  MppProcessNodeType eType;
  ALBaseContext *pAlBaseContext;
  MppOps *ops;
} MppProcessNode;
```

#### 4.7.3 struct MppOps

Definition of node operation set

```c
/***
 * @description: common ops for bind node
 *
 */
typedef struct _MppOps {
  /***
   * @description: send unhandled data to process node
   * @param {ALBaseContext} *base_context
   * @param {MppData} *sink_data
   * @return {*}
   */
  S32 (*handle_data)(ALBaseContext *base_context, MppData *sink_data);
  /***
   * @description: get handled result from process node
   * @param {ALBaseContext} *base_context
   * @param {MppData} *src_data
   * @return {*}
   */
  S32 (*get_result)(ALBaseContext *base_context, MppData *src_data);
  /***
   * @description: return the buffer to process node if needed
   * @param {ALBaseContext} *base_context
   * @param {MppData} *src_data
   * @return {*}
   */
  S32 (*return_result)(ALBaseContext *base_context, MppData *src_data);
} MppOps;
```

#### 4.7.4 struct MppProcessNodeType

This enumeration defines node type.

```c
/***
 * @description: process node type enum
 *
 */
typedef enum _MppProcessNodeType {
  /***
   * video decoder node, bitstream in and frame out
   */
  VDEC = 1,

  /***
   * video encoder node, frame in and bitstream out
   */
  VENC = 2,

  /***
   * graphic 2d node, frame in and frame out
   */
  G2D = 3,
} MppProcessNodeType;
```

### 4.8 Internal Key Data Structures

#### 4.8.1 struct MppFrame

```c
struct _MppFrame {
  /**
   * parent class
   */
  MppData eBaseData;              // MppData base class

  /**
   * video parameter
   */
  MppPixelFormat ePixelFormat;    // Video pixel format
  S32 nWidth;                     // Video width
  S32 nHeight;                    // Video height
  S32 nLineStride;                // Video aligned width
  S32 nFrameRate;                 // Video frame

  /**
   * frame parameter
   */
  S64 nPts;                       // frame pts
  BOOL bEos;                      // frame eos flag
  MppFrameBufferType eBufferType; // frame buffer type
  S32 nDataUsedNum;               // planner number
  S32 nID;                        // frame ID
  U8 *pData0;                     // planner0 address
  U8 *pData1;                     // planner1 address
  U8 *pData2;                     // planner2 address
  U8 *pData3;                     // planner3 address
  void *pMetaData;
  S32 nFd[MPP_MAX_PLANES];
  U32 refCount;
  DmaBufWrapper *pDmaBufWrapper;

  // environment variable
  BOOL bEnableUnfreeFrameDebug;
};
```

#### 4.8.2 struct MppPacket

```c
struct _MppPacket {
  /**
   * parent class
   */
  MppData eBaseData;              // MppData base class

  /**
   * video parameter
   */
  MppPixelFormat ePixelFormat;    // Video pixel format
  S32 nWidth;                     // Video width
  S32 nHeight;                    // Video height
  S32 nLineStride;                // Video aligned stride
  S32 nFrameRate;                 // Video frame rate

  /**
   * packet parameter
   */
  U8 *pData;                      // packet address
  S32 nTotalSize;  // total size that PACKET_Alloc
  S32 nLength;     // data length, nLength <= nTotalSize
  void *pMetaData;
  S32 nID;                        // packet ID
  S64 nPts;                       // packet pts
  S64 nDts;                       // packet dts
  BOOL bEos;                      // packet eos flag

  // environment variable
  BOOL bEnableUnfreePacketDebug;
};
```

#### 4.8.3 struct ALBaseContex/ALXxxBaseContext

```c
typedef struct _ALBaseContext ALBaseContext;
typedef struct _ALDecBaseContext ALDecBaseContext;
typedef struct _ALEncBaseContext ALEncBaseContext;
typedef struct _ALG2dBaseContext ALG2dBaseContext;
typedef struct _ALVoBaseContext ALVoBaseContext;
typedef struct _ALViBaseContext ALViBaseContext;

struct _ALBaseContext {};

struct _ALDecBaseContext {
  ALBaseContext stAlBaseContext;
};

struct _ALEncBaseContext {
  ALBaseContext stAlBaseContext; 
};

struct _ALG2dBaseContext {
  ALBaseContext stAlBaseContext;
};

struct _ALVoBaseContext {
  ALBaseContext stAlBaseContext;
};

struct _ALViBaseContext {
  ALBaseContext stAlBaseContext;
```

## 5. Interface Description

### 5.1 VDEC

| Interface                    | Description               | Parameters                                                         | Return Value                   |
| ----------------------- | ------------------ | ------------------------------------------------------------ | ------------------------- |
| VDEC_CreateChannel      | Create decoder         | None                                                           | MppVdecCtx*：Decoder context |
| VDEC_Init               | Initialize decoder       | MppVdecCtx *ctx：decoder context                                | 0: Success Non-0: Error code value      |
| VDEC_SetParam           | Set decoder parameters     | MppVdecCtx *ctx：decoder context                                | 0: Success Non-0: Error code value       |
| VDEC_GetParam           | Get decoder parameters     | MppVdecCtx *ctx：decoder context MppVdecPara **stVdecPara：parameter | 0: Success Non-0: Error code value       |
| VDEC_GetDefaultParam    | Get default parameters | MppVdecCtx *ctx：decoder context                                | 0: Success Non-0: Error code value       |
| VDEC_Decode             | Send bistream to decoder   | MppVdecCtx *ctx：decoder context MppData *sink_data：buffer     | 0: Success Non-0: Error code value       |
| VDEC_RequestOutputFrame | Request decoded output frame         | MppVdecCtx *ctx：decoder context MppData *src_data：Output decoded frame | 0: Success Non-0: Error code value       |
| VDEC_ReturnOutputFrame  | return decoded output frame         | MppVdecCtx *ctx：decoder context MppData *src_data：Output decoded frame | 0: Success Non-0: Error code value       |
| VDEC_DestroyChannel     | Destroy decoder         | MppVdecCtx *ctx：decoder context                                | 0: Success Non-0: Error code value      |
| VDEC_ResetChannel       | Reset decoder         | MppVdecCtx *ctx：decoder context                                | 0: Succes Non-0: Error code value       |

### 5.2 VENC

| Interface                       | Description                 | Parameter                                                         | Return Value                    |
| -------------------------- | -------------------- | ------------------------------------------------------------ | ------------------------- |
| VENC_CreateChannel         | Create encoder           | None                                                           | MppVencCtx*: encoder context |
| VENC_Init                  | Initialize encoder         | MppVencCtx *ctx: encoder context                                | 0: Success Non-0: Error code value       |
| VENC_SetParam              | Set encoder parameter       | MppVencCtx *ctx: encoder context MppVencPara *para: encoder parameter  | 0: Success Non-0: Error code value       |
| VENC_GetParam              | Get encoder parameter       | MppVencCtx *ctx: encoder context MppVencPara *para: encoder parameter  | 0: Success Non-0: Error code value       |
| VENC_SendInputFrame        | Send input frame to encoder         | MppVencCtx *ctx: encoder context MppData *sink_data: encoder frame     | 0: Success Non-0: Error code value       |
| VENC_ReturnInputFrame      | Return input frame to encoder       | MppVencCtx *ctx: encoder context MppData *sink_data: encoder frame     | 0: Success Non-0: Error code value       |
| VENC_GetOutputStreamBuffer | Get encoded output stream buffer       | MppVencCtx *ctx: encoder context MppData *src_data: encoded bitstream | 0: Success Non-0: Error code value       |
| VENC_DestroyChannel        | Destroy encoder channel           | MppVencCtx *ctx: encoder context                                | 0: Success Non-0: Error code value       |
| VENC_ResetChannel          | Reset encoder            | MppVencCtx *ctx: encoder context                                | 0: Success Non-0: Error code value       |
| VENC_Flush                 | Flush encoder internal buffer | MppVencCtx *ctx: encoder context                               | 0 0: Success Non-0: Error code value       |

### 5.3 G2D（Under Improvement）

| Interface                   | Description           | Parameter                                                   | Return Value                |
| ---------------------- | -------------- | ------------------------------------------------------ | --------------------- |
| G2D_CreateChannel      | Create G2D        | None                                                     | MppG2dCtx*: G2D context |
| G2D_Init               | Initialize G2D     | MppG2dCtx *ctx: G2D context                              |  0: Success Non-0: Error code value   |
| G2D_SetParam           | Set G2D parameter   | MppG2dCtx*ctx: G2D context MppG2dPara *para: G2D parameter     |  0: Success Non-0: Error code value   |
| G2D_GetParam           | Get G2D parameter    | MppG2dCtx*ctx: G2D context MppG2dPara *para: G2D parameter     | 0: Success Non-0: Error code value    |
| G2D_SendInputFrame     | Send input frame   | MppG2dCtx*ctx: G2D context MppData *sink_data: Frame to be processed  |0: Success Non-0: Error code value    |
| G2D_ReturnInputFrame   | Return input frame | MppG2dCtx*ctx: G2D context MppData *sink_data: Frame to be processed | 0: Success Non-0: Error code value    |
| G2D_RequestOutputFrame | Request output frame | MppG2dCtx*ctx: G2D context MppData *src_data: Processed frame | 0: Success Non-0: Error code value    |
| G2D_ReturnOutputFrame  | Release processed frame | MppG2dCtx*ctx: G2D context MppData *src_data: Processed frame |  0: Success Non-0: Error code value  |
| G2D_DestoryChannel     | Destroy G2D        | MppG2dCtx*ctx: G2D context                             | 0: Success Non-0: Error code value   |

### 5.4 VI

| Interface                 | Description         | Prameter                                                | Return Value              |
| -------------------- | ------------ | --------------------------------------------------- | ------------------- |
| VI_CreateChannel     | Create VI       | None                                                  | MppViCtx*: VI context |
| VI_Init              | Initialize VI     | MppViCtx *ctx: VI context                             | 0: Success Non-0: Error code value |
| VI_SetParam          | Set VI parameter   | MppViCtx *ctx: VI context MppViPara *para: VI parameter     | 0: Success Non-0: Error code value |
| VI_GetParam          | Get VI parameter   | MppViCtx *ctx: VIcontext MppViPara *para: VI parameter     | 0: Success Non-0: Error code value |
| VI_RequestOutputData | Get input data | MppViCtx *ctx: VIcontext MppData *src_data: Input data | 0: Success Non-0: Error code value|
| VI_ReturnOutputData  | Release input data | MppViCtx *ctx: VIcontext MppData *src_data: Input data| 0: Success Non-0: Error code value |
| VI_DestoryChannel    | Destroy VI       | MppViCtx *ctx: VIcontext                            | 0: Success Non-0: Error code value |

### 5.5 VO

| Interface              | Description       | Parameter                                                 | Return Value                     |
| ----------------- | ---------- | ---------------------------------------------------- | -------------------------- |
| VO_CreateChannel  | Create VO     | None                                                  | MppVoCtx *: Image processing context |
| VO_Init           | Initialize VI   | MppVoCtx *ctx: encoder context                          | 0: Success Non-0: Error code value        |
| VO_SetParam       | Set VI parameter | MppVoCtx *ctx: VO context MppVoPara *para: VO parameter      | 0: Success Non-0: Error code value        |
| VO_GetParam       | Get VI parameter | MppVoCtx *ctx: VO context MppVoPara **para: VO parameter     | 0: Success Non-0: Error code value        |
| VO_Process        | Output data   | MppVoCtx *ctx: VO context MppData *sink_data: output data | 0: Success Non-0: Error code value      |
| VO_DestoryChannel | Destroy VO     | MppVoCtx *ctx: VO context                              | 0: Success Non-0: Error code value        |

### 5.6 SYS

| Interface           | Description                       | Parameter                                                         | Return Value                         |
| -------------- | -------------------------- | ------------------------------------------------------------ | ------------------------------ |
| SYS_GetVersion | Get MPP version              | MppVersion *version：MPP version                               | 0: Success Non-0: Error code value            |
| SYS_CreateFlow | Create BIND flow              | None                                                           | MppProcessFlowCtx*: flow context |
| SYS_CreateNode | Create BIND node      | MppProcessNodeType type: node type                            | MppProcessNode*: node context    |
| SYS_Init       | Initialize BIND flow            | MppProcessFlowCtx *ctx: flow context                           | None                             |
| SYS_Destory    | Destroy BIND flow              | MppProcessFlowCtx *ctx: flow context                           | None                             |
| SYS_Bind       | Data source bind data receiver       | MppProcessFlowCtx *ctx: flow context MppProcessNode *src_ctx: data source MppProcessNode *sink_ctx: Data receiver | 0: Success Non-0: Error code value            |
| SYS_UnBind     | Unbind all data source and data receiver | MppProcessFlowCtx *ctx: flow context                           | None                             |
| SYS_Handledata | Process data                   | MppProcessFlowCtx *ctx：flow context MppData *sink_data：Data to be processed | None                             |
| SYS_Getresult  | Return result                   | MppProcessFlowCtx *ctx：flow context MppData *src_data：processed data | None                             |


## 6. Test Program

### 6.1 Single-Channel Decoding Test（vi_file_vdec_vo_test）

```shell
VI(file) --> VDEC(linlonv5v7) --> VO(file or sdl2)
```

#### 6.1.1 Test Program Instruction

```shell
bianbu@k1:~$ vi_file_vdec_vo_test -H
Usage:
-H       --help                    Print help
-i       --input                   Input file path
-c       --codingtype              Coding type
-m       --moduletype              Module type
-o       --save_frame_file         Saving picture file path
-w       --width                   Video width
-h       --height                  Video height
-f       --format                  Video PixelFormat
--codectype:
0        CODEC_AUTO
1        CODEC_OPENH264
2        CODEC_FFMPEG
3        CODEC_SFDEC
4        CODEC_SFENC
5        CODEC_CODADEC
6        CODEC_SFOMX
7        CODEC_V4L2
8        CODEC_FAKEDEC
9        CODEC_V4L2_LINLONV5V7
10       CODEC_K1_JPU
100      UNKNOWN
101      VO_SDL2
102      VO_FILE
200      UNKNOWN
201      VI_V4L2
202      VI_K1_CAM
203      VI_FILE
300      UNKNOWN
301      VPS_K1_V2D
--codingtype:
0        CODING_UNKNOWN
1        CODING_H263
2        CODING_H264
3        CODING_H264_MVC
4        CODING_H264_NO_SC
5        CODING_H265
6        CODING_MJPEG
7        CODING_JPEG
8        CODING_VP8
9        CODING_VP9
10       CODING_AV1
11       CODING_AVS
12       CODING_AVS2
13       CODING_MPEG1
14       CODING_MPEG2
15       CODING_MPEG4
16       CODING_RV
17       CODING_VC1
18       CODING_VC1_ANNEX_L
19       CODING_FWHT
--format:
0        PIXEL_FORMAT_UNKNOWN
1        PIXEL_FORMAT_YV12
2        PIXEL_FORMAT_I420
3        PIXEL_FORMAT_NV21
4        PIXEL_FORMAT_NV12
5        PIXEL_FORMAT_YV12_P010
6        PIXEL_FORMAT_I420_P010
7        PIXEL_FORMAT_NV21_P010
8        PIXEL_FORMAT_NV12_P010
9        PIXEL_FORMAT_YV12_P016
10       PIXEL_FORMAT_I420_P016
11       PIXEL_FORMAT_NV21_P016
12       PIXEL_FORMAT_NV12_P016
13       PIXEL_FORMAT_YUV422P
14       PIXEL_FORMAT_YV16
15       PIXEL_FORMAT_YUV422SP
16       PIXEL_FORMAT_NV61
17       PIXEL_FORMAT_YUV422P_P010
18       PIXEL_FORMAT_YV16_P010
19       PIXEL_FORMAT_YUV422SP_P010
20       PIXEL_FORMAT_NV61_P010
21       PIXEL_FORMAT_YUV444P
22       PIXEL_FORMAT_YUV444SP
23       PIXEL_FORMAT_YUYV
24       PIXEL_FORMAT_YVYU
25       PIXEL_FORMAT_UYVY
26       PIXEL_FORMAT_VYUY
27       PIXEL_FORMAT_YUV_MB32_420
28       PIXEL_FORMAT_YUV_MB32_422
29       PIXEL_FORMAT_YUV_MB32_444
31       UNKNOWN
32       PIXEL_FORMAT_RGBA
33       PIXEL_FORMAT_ARGB
34       PIXEL_FORMAT_ABGR
35       PIXEL_FORMAT_BGRA
36       PIXEL_FORMAT_RGBA_5658
37       PIXEL_FORMAT_ARGB_8565
38       PIXEL_FORMAT_ABGR_8565
39       PIXEL_FORMAT_BGRA_5658
40       PIXEL_FORMAT_RGBA_5551
41       PIXEL_FORMAT_ARGB_1555
42       PIXEL_FORMAT_ABGR_1555
43       PIXEL_FORMAT_BGRA_5551
44       PIXEL_FORMAT_RGBA_4444
45       PIXEL_FORMAT_ARGB_4444
46       PIXEL_FORMAT_ABGR_4444
47       PIXEL_FORMAT_BGRA_4444
48       PIXEL_FORMAT_RGB_888
49       PIXEL_FORMAT_BGR_888
50       PIXEL_FORMAT_RGB_565
51       PIXEL_FORMAT_BGR_565
52       PIXEL_FORMAT_RGB_555
53       PIXEL_FORMAT_BGR_555
54       PIXEL_FORMAT_RGB_444
55       PIXEL_FORMAT_BGR_444
```

Use the LINLONV5V7 decoder to decode the H.264 bitstream (input.264) read from the file into NV12-formatted output.yuv, and write the result to a file, as follows:

```shell
vi_file_vdec_vo_test -i input.264 -m 203,9,102 -c 2 -f 4 -w 1280 -h 720 -o output.yuv

// -m 203,9,102
// 203 represents VI_FILE
// 9 represents CODEC_V4L2_LINLONV5V7
// 102 represents VO_FILE
```

#### 6.1.2 Test Program Code Flow

The test program flow is simple and not detailed here. Please refer to the source code at:

```shell
mpp/test/vi_file_vdec_vo_test.c
```

### 6.2 Encoding Test（vi_file_venc_sync_userptr_vo_file_test）

```shell
VI(file) --> VENC(linlonv5v7) --> VO(file)
```

#### 6.2.1 Test Program Instruction

```shell
bianbu@k1:~$ vi_file_venc_sync_userptr_vo_file_test -H
Usage:
-H       --help                    Print help
-i       --input                   Input file path
-c       --codingtype              Coding type
-m       --moduletype              Module type
-o       --save_frame_file         Saving picture file path
-w       --width                   Video width
-h       --height                  Video height
-f       --format                  Video PixelFormat
--codectype:
0        CODEC_AUTO
1        CODEC_OPENH264
2        CODEC_FFMPEG
3        CODEC_SFDEC
4        CODEC_SFENC
5        CODEC_CODADEC
6        CODEC_SFOMX
7        CODEC_V4L2
8        CODEC_FAKEDEC
9        CODEC_V4L2_LINLONV5V7
10       CODEC_K1_JPU
100      UNKNOWN
101      VO_SDL2
102      VO_FILE
200      UNKNOWN
201      VI_V4L2
202      VI_K1_CAM
203      VI_FILE
300      UNKNOWN
301      VPS_K1_V2D
--codingtype:
0        CODING_UNKNOWN
1        CODING_H263
2        CODING_H264
3        CODING_H264_MVC
4        CODING_H264_NO_SC
5        CODING_H265
6        CODING_MJPEG
7        CODING_JPEG
8        CODING_VP8
9        CODING_VP9
10       CODING_AV1
11       CODING_AVS
12       CODING_AVS2
13       CODING_MPEG1
14       CODING_MPEG2
15       CODING_MPEG4
16       CODING_RV
17       CODING_VC1
18       CODING_VC1_ANNEX_L
19       CODING_FWHT
--format:
0        PIXEL_FORMAT_UNKNOWN
1        PIXEL_FORMAT_YV12
2        PIXEL_FORMAT_I420
3        PIXEL_FORMAT_NV21
4        PIXEL_FORMAT_NV12
5        PIXEL_FORMAT_YV12_P010
6        PIXEL_FORMAT_I420_P010
7        PIXEL_FORMAT_NV21_P010
8        PIXEL_FORMAT_NV12_P010
9        PIXEL_FORMAT_YV12_P016
10       PIXEL_FORMAT_I420_P016
11       PIXEL_FORMAT_NV21_P016
12       PIXEL_FORMAT_NV12_P016
13       PIXEL_FORMAT_YUV422P
14       PIXEL_FORMAT_YV16
15       PIXEL_FORMAT_YUV422SP
16       PIXEL_FORMAT_NV61
17       PIXEL_FORMAT_YUV422P_P010
18       PIXEL_FORMAT_YV16_P010
19       PIXEL_FORMAT_YUV422SP_P010
20       PIXEL_FORMAT_NV61_P010
21       PIXEL_FORMAT_YUV444P
22       PIXEL_FORMAT_YUV444SP
23       PIXEL_FORMAT_YUYV
24       PIXEL_FORMAT_YVYU
25       PIXEL_FORMAT_UYVY
26       PIXEL_FORMAT_VYUY
27       PIXEL_FORMAT_YUV_MB32_420
28       PIXEL_FORMAT_YUV_MB32_422
29       PIXEL_FORMAT_YUV_MB32_444
31       UNKNOWN
32       PIXEL_FORMAT_RGBA
33       PIXEL_FORMAT_ARGB
34       PIXEL_FORMAT_ABGR
35       PIXEL_FORMAT_BGRA
36       PIXEL_FORMAT_RGBA_5658
37       PIXEL_FORMAT_ARGB_8565
38       PIXEL_FORMAT_ABGR_8565
39       PIXEL_FORMAT_BGRA_5658
40       PIXEL_FORMAT_RGBA_5551
41       PIXEL_FORMAT_ARGB_1555
42       PIXEL_FORMAT_ABGR_1555
43       PIXEL_FORMAT_BGRA_5551
44       PIXEL_FORMAT_RGBA_4444
45       PIXEL_FORMAT_ARGB_4444
46       PIXEL_FORMAT_ABGR_4444
47       PIXEL_FORMAT_BGRA_4444
48       PIXEL_FORMAT_RGB_888
49       PIXEL_FORMAT_BGR_888
50       PIXEL_FORMAT_RGB_565
51       PIXEL_FORMAT_BGR_565
52       PIXEL_FORMAT_RGB_555
53       PIXEL_FORMAT_BGR_555
54       PIXEL_FORMAT_RGB_444
55       PIXEL_FORMAT_BGR_444
```

Use the LINLONV5V7 encoder to encode the 1280×720 NV12-formatted input.yuv file into an H.264 bitstream (output.264), and save the result to a file, as follows:

```shell
vi_file_venc_sync_userptr_vo_file_test -i input.yuv -m 293,9,102 -c 2 -f 4 -w 1280 -h 720 -o output.264

// -m 203,9,102
// 203 represents VI_FILE
// 9 represents CODEC_V4L2_LINLONV5V7
// 102 represents VO_FILE
```

#### 6.2.2 Test Program Code Flow

The code flow of the test program is relatively simple and not detailed here. You can check the source code at the following path:

```shell
mpp/test/vi_file_venc_sync_userptr_vo_file_test.c
```

### 6.3 Decode-Encode Cost（vi_file_vdec_venc_sync_userptr_vo_file_test）

```shell
VI(file) --> VDEC(linlonv5v7) --> VENC(linlonv5v7) --> VO(file)
```

#### 6.3.1 Test Program Instruction

```shell
bianbu@k1:~$ vi_file_vdec_venc_sync_userptr_vo_file_test -H
Usage:
-H       --help                    Print help
-i       --input                   Input file path
-c       --codingtype              Coding type
-m       --moduletype              Module type
-o       --save_frame_file         Saving picture file path
-w       --width                   Video width
-h       --height                  Video height
-f       --format                  Video PixelFormat
--codectype:
0        CODEC_AUTO
1        CODEC_OPENH264
2        CODEC_FFMPEG
3        CODEC_SFDEC
4        CODEC_SFENC
5        CODEC_CODADEC
6        CODEC_SFOMX
7        CODEC_V4L2
8        CODEC_FAKEDEC
9        CODEC_V4L2_LINLONV5V7
10       CODEC_K1_JPU
100      UNKNOWN
101      VO_SDL2
102      VO_FILE
200      UNKNOWN
201      VI_V4L2
202      VI_K1_CAM
203      VI_FILE
300      UNKNOWN
301      VPS_K1_V2D
--codingtype:
0        CODING_UNKNOWN
1        CODING_H263
2        CODING_H264
3        CODING_H264_MVC
4        CODING_H264_NO_SC
5        CODING_H265
6        CODING_MJPEG
7        CODING_JPEG
8        CODING_VP8
9        CODING_VP9
10       CODING_AV1
11       CODING_AVS
12       CODING_AVS2
13       CODING_MPEG1
14       CODING_MPEG2
15       CODING_MPEG4
16       CODING_RV
17       CODING_VC1
18       CODING_VC1_ANNEX_L
19       CODING_FWHT
--format:
0        PIXEL_FORMAT_UNKNOWN
1        PIXEL_FORMAT_YV12
2        PIXEL_FORMAT_I420
3        PIXEL_FORMAT_NV21
4        PIXEL_FORMAT_NV12
5        PIXEL_FORMAT_YV12_P010
6        PIXEL_FORMAT_I420_P010
7        PIXEL_FORMAT_NV21_P010
8        PIXEL_FORMAT_NV12_P010
9        PIXEL_FORMAT_YV12_P016
10       PIXEL_FORMAT_I420_P016
11       PIXEL_FORMAT_NV21_P016
12       PIXEL_FORMAT_NV12_P016
13       PIXEL_FORMAT_YUV422P
14       PIXEL_FORMAT_YV16
15       PIXEL_FORMAT_YUV422SP
16       PIXEL_FORMAT_NV61
17       PIXEL_FORMAT_YUV422P_P010
18       PIXEL_FORMAT_YV16_P010
19       PIXEL_FORMAT_YUV422SP_P010
20       PIXEL_FORMAT_NV61_P010
21       PIXEL_FORMAT_YUV444P
22       PIXEL_FORMAT_YUV444SP
23       PIXEL_FORMAT_YUYV
24       PIXEL_FORMAT_YVYU
25       PIXEL_FORMAT_UYVY
26       PIXEL_FORMAT_VYUY
27       PIXEL_FORMAT_YUV_MB32_420
28       PIXEL_FORMAT_YUV_MB32_422
29       PIXEL_FORMAT_YUV_MB32_444
31       UNKNOWN
32       PIXEL_FORMAT_RGBA
33       PIXEL_FORMAT_ARGB
34       PIXEL_FORMAT_ABGR
35       PIXEL_FORMAT_BGRA
36       PIXEL_FORMAT_RGBA_5658
37       PIXEL_FORMAT_ARGB_8565
38       PIXEL_FORMAT_ABGR_8565
39       PIXEL_FORMAT_BGRA_5658
40       PIXEL_FORMAT_RGBA_5551
41       PIXEL_FORMAT_ARGB_1555
42       PIXEL_FORMAT_ABGR_1555
43       PIXEL_FORMAT_BGRA_5551
44       PIXEL_FORMAT_RGBA_4444
45       PIXEL_FORMAT_ARGB_4444
46       PIXEL_FORMAT_ABGR_4444
47       PIXEL_FORMAT_BGRA_4444
48       PIXEL_FORMAT_RGB_888
49       PIXEL_FORMAT_BGR_888
50       PIXEL_FORMAT_RGB_565
51       PIXEL_FORMAT_BGR_565
52       PIXEL_FORMAT_RGB_555
53       PIXEL_FORMAT_BGR_555
54       PIXEL_FORMAT_RGB_444
55       PIXEL_FORMAT_BGR_444
```

Use the LINLONV5V7 codec to decode the 1280×720 H.264 bitstream (input.264) read from the file into NV12 format, then re-encode it into an H.264 bitstream (output.264), as follows:

```shell
vi_file_vdec_venc_sync_userptr_vo_file_test -i input.264 -m 203,9,9,102 -c 2 -f 4 -w 1280 -h 720 -o output.264

// -m 203,9,9,102
// 203 represents VI_FILE
// 9 represents CODEC_V4L2_LINLONV5V7
// 102 represents VO_FILE
```

#### 6.3.2 Test Program Code Flow

The code flow of the test program is relatively simple and will not be elaborated on here. You may refer to the source code at the following path:

```shell
mpp/test/vi_file_vdec_venc_sync_userptr_vo_file_test.c
```

## 7. Hardware Decoding and Verification Method

### 7.1 Bianbu Desktop System

#### 7.1.1 mpv player

mpv supports hardware decoding for formats such as H.264/ HEVC/ VP8/ VP9/ MJPEG/ MPEG4, with a maximum support for 4K60fps. The verification method is as follows:

- In the desktop system, locate and right-click on the video, and select "Play with mpv".
- In the terminal, play the video with the command line.

```
mpv  xxx.mp4
mpv -fs xxx.mp4  // play in fullscreen
mpv --loop xxx.mp4  //play in a loop
```

#### 7.1.2 totem player

totem supports hardware decoding for formats such as H.264/ HEVC/ VP8/ VP9/ MJPEG, with a maximum support of 4K30fps. The verification method is as follows:

- In the desktop system, locate and right-click on the video, and select "Play with totem".
- In the terminal, play the video with the command line. 

```
totem  xxx.mp4
```

#### 7.1.3 chromium browser

chromium supports hardware decoding for formats such as H.264/ HEVC, with a maximum support of 4K30fps. The verification method is as follows:

- Open chromium, play Bilibili's video
- Open chromium, play Youku's video
- Open chromium, play Sina-sport's video
- Open chromium, play other website's video

#### 7.1.4 kodi player

kodi supports hardware decoding for formats such as H.264/ HEVC/ VP8/ VP9, with a maximum support of 4K60fps. The verification method is as follows:

- Open kodi, select the video and click Play.

#### 7.1.5 ffplay command line

```
ffplay -codec:v h264_stcodec xxx.mp4(H.264 video codec)
ffplay -codec:v hevc_stcodec xxx.mp4(HEVC video codec)
...
```

#### 7.1.6 Gstreamer command line

```
gst-launch-1.0 playbin uri=file:///path/to/some/media/file.mp4 (H.264 video codec)
gst-launch-1.0 playbin uri=file:///path/to/some/media/file.mp4 (HEVC video codec)
```

### 7.2 Buildroot System

#### 7.2.1 FFmpeg

```
ffplay -codec:v h264_stcodec xxx.mp4(H.264 video codec)
ffplay -codec:v hevc_stcodec xxx.mp4(HEVC video codec)
...
```

#### 7.2.2 Gstreamer

```sql
gst-launch-1.0 playbin uri=file:///path/to/some/media/file.mp4 (H.264 video codec)
gst-launch-1.0 playbin uri=file:///path/to/some/media/file.mp4 (HEVC video codec)
```
