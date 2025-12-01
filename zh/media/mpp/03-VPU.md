# VPU

VPU（Video Processing Unit，视频处理单元）具有视频编解码功能的硬件，能够提高编解码效率并减少 CPU 负荷。
K1 平台的 VPU 基于 **V4L2** 框架实现
- **解码支持**：H.264 / HEVC / VP8 / VP9 / MJPEG / MPEG-4
- **编码支持**：H.264 / HEVC / VP8 / VP9 / MJPEG
- **特点**：提供完整的测试程序，便于开发与功能验证。

## 1 规格

### 1.1 解码规格（2cores@819MHz）

| 格式  | profile                   | 最大分辨率 | 最大码率 | 规格        | 多路规格        |
| ----- | ------------------------- | ---------- | -------- | ----------- | --------------- |
| HEVC  | Main/Main10               | 4096×4096  | 200Mbps  | 4k@60fps    | 8路 1080P@30fps |
| H.264 | BP/MP/HP/High10           | 4096×4096  | 200Mbps  | 4k@60fps    | 8路 1080P@30fps |
| VP8   | /                         | 2048×2048  | 100Mbps  | 1080p@60fps | 2路 1080P@30fps |
| VP9   | Profile0/Profile 2 10-bit | 4096×4096  | 120Mbps  | 4k@30fps    | 4路 1080P@30fps |
| JPEG  | Baseline sequential       | 8192×8192  | 80Mbps   | 4k@30fps    | 4路 1080P@30fps |
| VC-1  | SP/MP/AP                  | 2048×4096  | 80Mbps   | 1080p@60fps | 2路 1080P@30fps |
| MPEG4 | SP/ASP                    | 2048×2048  | 40Mbps   | 1080p@60fps | 2路 1080P@30fps |
| MPEG2 | MP                        | 4096×4096  | 40Mbps   | 1080p@60fps | 2路 1080P@30fps |

### 1.2 编码规格（2cores@819MHz）

| 格式  | profile                   | 最大分辨率 | 最大码率 | 规格        | 多路规格        |
| ----- | ------------------------- | ---------- | -------- | ----------- | --------------- |
| HEVC  | Main/Main10               | 4096×4096  | 200Mbps  | 4k@30fps    | 4路 1080P@30fps |
| H.264 | BP/MP/HP/High10           | 4096×4096  | 200Mbps  | 4k@30fps    | 4路 1080P@30fps |
| VP8   | /                         | 2048×2048  | 100Mbps  | 1080p@60fps | 2路 1080P@30fps |
| VP9   | Profile0/Profile 2 10-bit | 4096×4096  | 200Mbps  | 4k@30fps    | 4路 1080P@30fps |
| JPEG  | Baseline sequential       | 8192×8192  | 200Mbps  | 4k@30fps    | 4路 1080P@30fps |

## 2 VPU 测试程序

`k1x-vpu-test` 是一套用于测试和验证 K1 芯片 VPU 性能的程序集合，也可以作为客户开发基于 VPU 硬件编解码的应用的参考。

`k1x-vpu-test` 是一套用于测试和验证 K1 芯片 VPU 性能的程序集合，也可以作为客户开发基于 VPU 硬件编解码的应用的参考。

### 2.1 安装说明

#### 2.1.1 Bianbu 桌面系统

`k1x-vpu-test` 已集成在软件源中，可直接通过 `apt` 安装：

```shell
sudo apt update
sudo apt install k1x-vpu-test
```

#### 2.1.2 Buildroot 系统

有两种方法将 `k1x-vpu-test` 集成到系统中：

- 在编译系统镜像时，开启 `k1x-vpu-test` 的编译集成选项（默认已开启），这样，编译的系统镜像中默认就包含 `k1x-vpu-test` 相关的测试程序;
- 若未开启此选项，则需手动编译 `k1x-vpu-test`，将生成的二进制文件复制到系统 `/usr/bin/` 目录下使用。具体包含的二进制文件在下文说明。

### 2.2 使用说明

`k1x-vpu-test` 的测试程序集中主要包含以下测试程序：

- **`mvx_decoder`**：用于单路视频码流的解码测试
- **`mvx_decoder_multi`**：用于多路视频码流的解码测试（多路视频码流必须为同一个视频）
- **`mvx_encoder`**：用于单路视频流的视频编码测试
- **`mvx_encoder_multi`**：用于多路 YUV 流的视频编码测试（多路 YUV 流必须为同一个流）
- **`mvx_logd`**：用于抓取 firmware 的 log 分析定位问题

#### 2.2.1 mvx_decoder

基本用法示例：

```shell
//将input.264的H.264视频流（带startcode）解码为output.yuv
//-f raw ： 表示视频裸流
//输入码流的编码格式默认为H.264
//输出YUV数据的像素格式默认为YUV420P
mvx_decoder -f raw /mnt/streams/input.264 /mnt/test/output.yuv

//将input.264的H.264视频流（带startcode）解码为output.yuv，output.yuv的像素格式为NV12
mvx_decoder -f raw -o yuv420_nv12 /mnt/streams/input.264 /mnt/test/output.yuv

//将input.265的HEVC视频流（带startcode）解码为output.yuv
mvx_decoder -f raw -i hevc /mnt/streams/input.265 /mnt/test/output.yuv
```

参数说明

```shell
usage: ./mvx_decoder [optional] [positional]

positional arguments:
    input
        Input file.
    output
        Output file.

optional arguments:
    --dev
        Device.
        Default: /dev/video0
    -i, --inputformat
        Pixel format.
        Default: h264
    -o, --outputformat
        Output pixel format.
        Default: yuv420
    -f, --format
        Input container format. [ivf, rcv, raw]
        For ivf input format will be taken from IVF header.
        Default: ivf
    -s, --stride
        Stride alignment.
        Default: 1
    -y, --intbuf
        Limit of intermediate buffer size
        Default: 1000000
    -m, --md5
        Output md5 file
    -e, --md5ref
        Input ref md5 file
    -u, --nalu
        Nalu format, START_CODES (0) and ONE_NALU_PER_BUFFER (1),                                                      ONE_BYTE_LENGTH_FIELD (2),                                                    TWO_BYTE_LENGTH_FIELD (3),                                                    FOUR_BYTE_LENGTH_FIELD (3).
        Default: 0
    -r, --rotate
        Rotation, 0 | 90 | 180 | 270
        Default: 0
    -d, --downscale
        Down Scale, 1 | 2 | 4
        Default: 1
    -v, --fps
        Frame rate.
        Default: 24
    --dsl_ratio_hor
        Horizontal downscale ratio, [1, 256]
        Default: 0
    --dsl_ratio_ver
        Vertical downscale ratio, [1, 128]
        Default: 0
    --dsl_frame_width
        Downscaled frame width in pixels
        Default: 0
    --dsl_frame_height
        Downscaled frame height in pixels
        Default: 0
    --dsl_pos_mode
        Flexible Downscaled original position mode [0, 2], 
        only availble in high precision mode. 
        Value: 0 [default:x_original=(x_resized + 0.5)/scale - 0.5]         
        Value: 1 [x_original=x_reized/scale]
        Value: 2 [x_original=(x_resized+0.5)/scale]
        Default: 0
    --frames
        nr of frames to process
        Default: 0
    --fro
        Frame reordering 1 is on (default), 0 is off
        Default: 1
    --ish
        Ignore Stream Headers 1 is on, 0 is off (default)
        Default: 0
    --trystop
        Try if Decoding Stop Command exixts
        Default: 0
    --one_frame_per_packet
        Each input buffer contains one frame.
        Default: 0
    -n, --interlaced
        Frames are interlaced
        Default: 0
    --tiled
        Use tiles for AFBC formats.
        Default: disabled
    --preload
        preload the input stream to memory.
        the size for input file should be less than 15MBytes.
        Default: 0
    --fw_timeout
        timeout value[secs] for watchdog timeout. range: 5~60.
        Default: 5
    --profiling
        enable profiling for bandwidth statistics . 0:disable; 1:enable.
        Default: 0
```

#### 2.2.2 mvx_decoder_multi

基本用法示例：

```shell
//将input.264的H.264视频流（带startcode）解码为output.yuv，同时4路解码并行
//输入码流的编码格式默认为H.264
//输出YUV数据的像素格式默认为YUV420P
mvx_decoder_multi -n 4 /mnt/streams/input.264 /mnt/test/output.yuv

//将input.265的HEVC视频流（带startcode）解码为output.yuv，4路同时进行，output.yuv的像素格式为NV21
mvx_decoder_multi -n 4 -i hevc -o yuv420_nv21 /mnt/streams/input.265 /mnt/test/output.yuv
```

参数说明

```shell
usage: ./mvx_decoder_multi [optional] [positional]

positional arguments:
    input
        Input file.
    output
        Output file.

optional arguments:
    --dev
        Device.
        Default: /dev/video0
    -i, --inputformat
        Pixel format.
        Default: h264
    -o, --outputformat
        Output pixel format.
        Default: yuv420
    -s, --stride
        Stride alignment.
        Default: 1
    -n, --nsessions
        Number of sessions.
        Default: 1
```

#### 2.2.3 mvx_encoder

基本用法示例：

```shell
//将input.yuv的YUV420P流编码为output.264的H.264视频流（带startcode）
//输入YUV流的像素格式默认为YUV420P
//输出视频流的编码格式默认为H.264
mvx_encoder -f raw -w 1280 -h 720 /mnt/streams/input.yuv /mnt/test/output.264

//将input.yuv的NV21流编码为output.265的HEVC视频流(带startcode)
mvx_encoder -f raw -w 1280 -h 720 -i yuv420_nv21 -o hevc /mnt/streams/input.yuv /mnt/test/output.265
```

参数说明

```shell
usage: ./mvx_encoder [optional] [positional]

positional arguments:
    input
        Input file.
    output
        Output file.

optional arguments:
    --dev
        Device.
        Default: /dev/video0
    -i, --inputformat
        Pixel format.
        Default: yuv420
    -o, --outputformat
        Output pixel format.
        Default: h264
    -f, --format
        Output container format. [ivf, raw]
        Default: ivf
    -w, --width
        Width.
        Default: 1920
    -h, --height
        Height.
        Default: 1080
    -s, --stride
        Stride alignment.
        Default: 1
    --mirror
        mirror, 1 : horizontal; 2 : vertical.
        Default: 0
    --roi_cfg
        ROI config file.
    --frames
        nr of frames to process
        Default: 0
    --epr_cfg
        Encode Parameter Records config file name
    --rate_control
        Selects rate control type, constant/variable/off
        Default: off
    --target_bitrate
        If rate control is enabled, this option sets target bitrate
        Default: 0
    --max_bitrate
        If rate control is enabled, this option sets maximum bitrate
        Default: 0
    --gop
        GOP: 0 is None, 1 is Bidi, 2 is Low delay, 3 is Pyramid
        Default: 0
    --pframes
        Number of P frames
        Default: 0
    --bframes
        Number of B frames
        Default: 0
    -n, --minqp
        H264 min QP
        Default: 0
    -m, --maxqp
        H264 max QP
        Default: 51
    -t, --tier
        Profile.
        Default: 2
    -l, --level
        Level.
        Default: 1
    -v, --fps
        Frame rate.
        Default: 24
    --ecm
        0 is CAVLC, 1 is CABAC
        Default: 1
    --bitdepth
        Set other bitdepth
        Default: 8
    -q, --fixedqp
        H264 fixed QP for I P B frames. 
        If it is combined with -x, the value will later be increased with 2.
        Default: 20
    --qpi
        H264 fixed QP for I frames.
        Default: 20
    --qpb
        H264 fixed QP for B frames.
        Default: 20
    --qpp
        H264 fixed QP for P frames.
        Default: 20
    --crop_left
        encoder SPS crop param, left offset
        Default: 0
    --crop_right
        encoder SPS crop param, right offset
        Default: 0
    --crop_top
        encoder SPS crop param, top offset
        Default: 0
    --crop_bottom
        encoder SPS crop param, bottom offset
        Default: 0
    --colour_description_range
        VUI param: Colour description; range
        Value: 0=Unspecified, 1=Limited, 2=Full
        Default: 0
    --colour_primaries
        VUI param: Colour description; 
        colour primaries (0-255, see hevc spec. E.3.1)
        Value: 0=Unspecified, 
               1=BT709, 
               2=BT470M, 
               3=BT601_625, 
               4=T601_525, 
               5=GENERIC_FILM, 
               6=BT2020
        Default: 0
    --transfer_characteristics
        VUI param: Colour description; 
        transfer characteristics (0-255, see hevc spec. E.3.1)
        Value: 0=Unspecified, 
               1=LINEAR, 
               2=SRGB, 
               3=SMPTE170M, 
               4=GAMMA22, 
               5=GAMMA28, 
               6=ST2084, 
               7=HLG, 
               8=SMPTE240M, 
               9=XVYCC, 
               10=BT1361, 
               11=ST428
        Default: 0
    --matrix_coeff
        VUI param: Colour description; 
        matrix coefficients (0-255, see hevc spec. E.3.1)
        Value: 0=Unspecified, 
               1=BT709, 
               2=BT470M, 
               3=BT601, 
               4=SMPTE240M, 
               5=T2020, 
               6=BT2020Constant
        Default: 0
    --time_scale
        VUI param: vui_time_scale
        Default: 0
    --num_units_in_tick
        VUI param: vui_num_units_in_tick
        Default: 0
    --aspect_ratio_idc
        VUI param: aspect_ratio_idc. [0,255]
        Default: 0
    --sar_width
        VUI param: sar_width
        Default: 0
    --sar_height
        VUI param: sar_height
        Default: 0
    --video_format
        VUI param: video_format. (0-5, see hevc spec. E.3.1)
        Value: 0=Component, 2=PAL, 2=NTSC, 3=SECAM, 4=MAC, 5=Unspecified
        Default: 0
    --sei_mastering_display
        SEI param : mastering display 's parameters
        Default: 0
    --sei_content_light
        SEI param : sei_content_light
        Default: 0
    --sei_user_data_unregistered
        SEI param : user data unregisterd
        Default: 0
    --hrd_buffer_size
        Hypothetical Reference Decoder buffer size 
        relative to the bitrate (in seconds) for rate control                         Value: should bigger than target_bitrate/fps on normal case
        Default: 0
    --ltr_mode
        encoder long term reference mode,range from 1 to 8 (inclusive)
        1: LDP-method-1 | 
        2: LDP-method-2 | 
        3: LDB-method-1 | 
        4: LDB-method-2
        5: BiDirection-method-1 | 
        6: BiDirection-method-2 | 
        7: Pyrimid-method-1 | 
        8: Pyrimid-method-2
        Default: 0
    --ltr_period
        encoder long term reference period, range from 2 to 254 (inclusive)
        Default: 0
    --trystop
        Try if Encoding Stop Command exixts
        Default: 0
    --restart_interval
        JPEG restart interval.
        Default: -1
    --quality
        JPEG compression quality. [1-100, 0 - default]
        Default: 0
    --preload
        preload the first 5 yuv frames to memory.
        Default: 0
    --fw_timeout
        timeout value[secs] for watchdog timeout. range: 5~60.
        Default: 5
    --profiling
        enable profiling for bandwidth statistics . 0:disable; 1:enable.
        Default: 0
```

#### 2.2.4 mvx_encoder_multi

基本用法示例：

```shell
//将input.yuv的YUV420P流编码为output.264的H.264视频流（带startcode），4路同时进行
mvx_encoder_multi -n 4 -w 1280 -h 720 /mnt/streams/input.yuv /mnt/test/output.264

//将input.yuv的NV21流编码为output.265的HEVC视频流(带startcode)，4路同时进行
mvx_encoder_multi -n 4 -w 1280 -h 720 -i yuv420_nv21 -o hevc /mnt/streams/input.yuv /mnt/test/output.265
```

参数说明

```shell
usage: ./mvx_encoder_multi [optional] [positional]

positional arguments:
    input
        Input file.
    output
        Output file.

optional arguments:
    --dev
        Device.
        Default: /dev/video0
    -i, --inputformat
        Pixel format.
        Default: yuv420
    -o, --outputformat
        Output pixel format.
        Default: h264
    -s, --stride
        Stride alignment.
        Default: 1
    -w, --width
        Stride alignment.
        Default: 1280
    -h, --height
        Stride alignment.
        Default: 720
    -f, --frames
        Specfied frame count to be processed.
        Default: 0
    -n, --nsessions
        Number of sessions.
        Default: 1
```

#### 2.2.5 mvx_logd

基本用法示例：

```shell
//挂载debugfs，保存fw的log到文件
mount -t debugfs none /sys/kernel/debug
//清除之前的log
mvx_logd -C
//设置好保存路径
mvx_logd -d -f text xxx.log
//开始播放视频
//查看xxx.log
```

参数说明

```shell
Usage: ./mvx_logd [OPTIONS] [DST]

Positional arguments:
    DST
        Output file (default stdout).

Optional arguments:
    -h
        --help This help message.
    -C
        Clear ring buffer.
    -c
        Clear ring buffer after first printing its contents.
    -d
        Daemonize process. DST file is expected in this case.
    --follow
        Keep on reading.
    -f
        Format.
        text  Text format (default).
        bin   Binary format.
        json  Java Script Object Notation.
    -i
        Input file (default /sys/kernel/debug/amvx/log/drain/ram0/msg).
    -t
        Adjust for timezone differences.

Example:
    Print and clear log.
    # mvx_logd -c

    Run in daemon mode.
    # mvx_logd -d -f bin fw.log

    Unpack binary log and adjust for timezone differences.
    # mvx_logd -t -1 fw.log
```

### 2.3 代码结构

k1x-vpu-test 的代码位置在：

```shell
package-src/k1x-vpu-test
```

代码结构及简要说明如下：

```shell
|-- CMakeLists.txt                //cmake构建脚本
|-- debian                        //deb包构建的相关配置和脚本
|-- format.sh                     //代码风格格式化脚本，Google风格进行格式化
|-- include                       //重要头文件
|   |-- fw_v2
|   |   `-- mve_protocol_def.h    
|   |-- md5.h
|   |-- mvx_argparse.h
|   |-- mvx_list.h
|   |-- mvx_log_ram.h
|   `-- mvx-v4l2-controls.h       //VPU驱动的附加API
|-- test
|   |-- coverage
|   |-- md5                                 //md5相关的API与实现
|   |   |-- CMakeLists.txt
|   |   |-- md5.c
|   |   `-- md5.h
|   |-- mvx_player                          //测试程序集实现代码
|   |   |-- CMakeLists.txt
|   |   |-- dmabufheap                      //dmabuf管理相关API与实现
|   |   |   |-- BufferAllocator.cpp
|   |   |   |-- BufferAllocator.h
|   |   |   |-- BufferAllocatorWrapper.cpp
|   |   |   |-- BufferAllocatorWrapper.h
|   |   |   `-- dmabufheap-defs.h
|   |   |-- mvx_decoder.cpp                 //mvx_decoder实现
|   |   |-- mvx_decoder_multi.cpp           //mvx_decoder_multi实现
|   |   |-- mvx_encoder.cpp                 //mvx_encoder实现
|   |   |-- mvx_encoder_gen.cpp             //mvx_encoder_gen实现
|   |   |-- mvx_encoder_multi.cpp           //mvx_encoder_multi实现
|   |   |-- mvx_info.cpp                    //mvx_info实现
|   |   |-- mvx_player.cpp                  //核心逻辑封装实现
|   |   |-- mvx_player.hpp                  //核心逻辑封装API
|   |   `-- reader                          //码流读取API与实现
|   |       |-- parser.h
|   |       `-- read_util.h
|   |-- userptr
|   |   |-- CMakeLists.txt
|   |   `-- mvx_userptr.cpp
|   `-- utils                               //utils
|       |-- CMakeLists.txt
|       |-- mvx_argparse.c
|       |-- mvx_argparse.h
|       `-- mvx_list.h
`-- tools
    |-- logd                                //logd实现，抓取VPU的firmware的log
    |   |-- CMakeLists.txt
    |   |-- mvx_logd.cpp
    |   |-- mvx_logd_fwif_v2.cpp
    |   `-- mvx_logd.hpp
    `-- securevideo                         //用于安全视频的测试（未验证使用）
        |-- 50-mvx.rules
        |-- mvx_securehelper.cpp
        |-- mvx_secureplayer.cpp
        |-- mvx_securevideo.cpp
        `-- mvx_securevideo.hpp

```

### 2.4 编译说明

**Bianbu 桌面系统**

```shell
cd k1x-vpu-test
sudo apt-get build-dep k1x-vpu-test    #安装依赖
dpkg-buildpackage -us -uc -nc -b -j32
```

**Buildroot 系统**

```shell
cd k1x-vpu-test
mkdir out
cd out
cmake ..
make
make install
```

### 2.5 调试说明

#### 2.5.1 Log 添加

使用 `printf` 或者 `fprintf` 来添加 log，重新编译安装即可生效

#### 2.5.2 解码成功 log

1. **单路解码**

```shell
/mnt/tool # ./mvx_decoder -f raw /mnt/streams/h264dec/Zhling_1280x720.264 /mnt/test/output.yuv  //输入命令

Opening '/dev/video0'.
Enumerate frame size. index=0, pixel_format=32314d59, min_width=2, max_width=8192, step_width=2, min_height=2, max_height=8192, step_height=2
Format: type=2, format=875967048, width=0, height=0, sizeimage=1048576, bytesperline=0, interlaced:1
Format: type=9, format=842091865, width=0, height=0, nplanes=3, bytesperline=[0, 0, 0], sizeimage=[0, 0, 0], interlaced:1
Request buffers. type=2, count=6, memory=4
Query: type=2, index=0, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Query: type=2, index=1, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Query: type=2, index=2, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Query: type=2, index=3, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Query: type=2, index=4, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Query: type=2, index=5, sequence=0, timestamp={0, 0}, flags=4000, bytesused=0, length=1048576
Request buffers. type=9, count=1, memory=4
Query: type=9, index=0, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[1, 1, 1], offset=[0, 0, 0]
//申请输入输出buffer

Stream on 2
Stream on 9
//输入输出开流

Event. type=5.
source changed. should reset output stream.
Stream off 9
Request buffers. type=9, count=0, memory=4
Format: type=9, format=842091865, width=1280, height=720, nplanes=3, bytesperline=[1280, 640, 640], sizeimage=[921600, 230400, 230400], interlaced:1
Request buffers. type=9, count=4, memory=4
Query: type=9, index=0, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Query: type=9, index=1, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Query: type=9, index=2, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Query: type=9, index=3, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Stream on 9
//收到分辨率改变消息，关闭输出流，重新申请输出buffer，开启输出流

-----Decoder. set timestart_us: 15432355548 us---------
//开始解码

Capture EOS.
-----[Test Result] MVX Decode Done. frames_processed: 18, cost time: 16604176 us.
Stream off 2
Stream off 9
-----[Test Result] MVX Decode PASS. Average Framerate: 1.08.
Total size 26265600
Closing fd 5.
//解码完成，输出信息，退出
```

2. **多路解码**

```shell
/mnt/tool # ./mvx_decoder_multi -n 4 /mnt/streams/h264dec/foreman_128x64.264 /mnt/test/output.yuv
//命令

-----Decoder. set timestart_us: 16076598695 us---------
-----Decoder. set timestart_us: 16076606348 us---------
-----Decoder. set timestart_us: 16076569259 us---------
-----Decoder. set timestart_us: 16076788196 us---------
//开始解码

-----[Test Result] MVX Decode Done. frames_processed: 2, cost time: 148367 us.
-----[Test Result] MVX Decode Done. frames_processed: 2, cost time: 250563 us.
-----[Test Result] MVX Decode Done. frames_processed: 2, cost time: 217680 us.
-----[Test Result] MVX Decode Done. frames_processed: 2, cost time: 279399 us.
Total size 36864
Total size 36864
Total size 36864
Total size 36864
//解码完成，输出信息，退出
```

#### 2.5.3 编码成功 log

1. **单路编码**

```shell
/mnt/tool # ./mvx_encoder -f raw -w 1280 -h 720 /mnt/streams/yuv/zhling_1280x720.yuv /mnt/test/output.264
//命令

Opening '/dev/video0'.
setEncFramerate( 1966080 )
setRateControl( 0,0,0)
Enumerate frame size. index=0, pixel_format=34363248, min_width=2, max_width=8192, step_width=2, min_height=2, max_height=8192, step_height=2
Format: type=10, format=842091865, width=1280, height=720, nplanes=3, bytesperline=[1280, 640, 640], sizeimage=[921600, 230400, 230400], interlaced:1
Format: type=9, format=875967048, width=1280, height=720, nplanes=1, bytesperline=[0, 0, 0], sizeimage=[921600, 0, 0], interlaced:1
Request buffers. type=10, count=3, memory=4
Query: type=10, index=0, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Query: type=10, index=1, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Query: type=10, index=2, sequence=0, timestamp={0, 0}, flags=4000, num_planes=3, bytesused=[0, 0, 0], length=[921600, 230400, 230400], offset=[0, 0, 0]
Request buffers. type=9, count=9, memory=4
Query: type=9, index=0, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=1, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=2, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=3, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=4, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=5, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=6, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=7, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
Query: type=9, index=8, sequence=0, timestamp={0, 0}, flags=4000, num_planes=1, bytesused=[0], length=[921600], offset=[0]
//申请输入输出buffer

Stream on 10
Stream on 9
//输入输出开流

-----Encoder. set timestart_us: 15823672258 us---------
//开始编码

Capture EOS.
-----[Test Result] MVX Encode Done. frames_processed: 19, cost time: 28493385 us.
Stream off 10
Stream off 9
-----[Test Result] MVX Encode PASS. Average Framerate: 0.67.
Total size 140165
Closing fd 5.
//编码完成，输出信息，退出
```

2. **多路编码**

```shell
/mnt/tool # ./mvx_encoder_multi -n 4 -w 128 -h 64 /mnt/streams/yuv/foreman_128x64_3frames.yuv /mnt/test/output.264
//命令

-----Encoder. set timestart_us: 16243183563 us---------
-----Encoder. set timestart_us: 16243458316 us---------
-----Encoder. set timestart_us: 16243756086 us---------
-----Encoder. set timestart_us: 16244053547 us---------
//开始编码

-----[Test Result] MVX Encode Done. frames_processed: 3, cost time: 634710 us.
-----[Test Result] MVX Encode Done. frames_processed: 3, cost time: 952320 us.
-----[Test Result] MVX Encode Done. frames_processed: 3, cost time: 389159 us.
-----[Test Result] MVX Encode Done. frames_processed: 3, cost time: 147590 us.
Total size 1618
Total size 1618
Total size 1618
Total size 1618
//编码完成，输出信息，退出
```
