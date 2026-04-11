sidebar_position: 2

# FFmpeg 用户使用指南

## FFmpeg 介绍

FFmpeg 是一个开源的多媒体处理工具集，提供了音视频采集、编解码、转封装、滤镜处理、推拉流、截图、转码等能力。

官方网址：[https://ffmpeg.org](https://ffmpeg.org/)

### FFmpeg 组件说明

FFmpeg 并不是单一程序，而是由多个命令行工具和底层库组成，常见组件如下：

| 组件名      | 功能说明 |
| ----------- | -------- |
| `ffmpeg`    | 音视频转码、转封装、滤镜处理、截图、推拉流等主命令 |
| `ffplay`    | 基于 SDL 的轻量级媒体播放器，用于快速播放和调试媒体文件或流 |
| `ffprobe`   | 用于查看媒体文件或流的详细信息，例如编码格式、分辨率、时长、码率等 |
| `libavcodec` | 编解码库，负责音视频编码和解码 |
| `libavformat` | 封装/解封装库，负责音视频容器格式处理 |
| `libavfilter` | 滤镜库，负责缩放、裁剪、叠加、水印等音视频处理 |
| `libswscale`  | 图像缩放与像素格式转换库 |
| `libswresample` | 音频重采样与格式转换库 |

### FFmpeg 在 K3 平台上的使用说明

在 K3 平台中，FFmpeg 主要用于以下场景：

- 媒体文件信息查看
- 音视频格式转换
- H.264/H.265/JPEG 等码流封装与解封装
- 本地文件播放链路调试
- 摄像头视频采集验证
- 网络推流与拉流验证

对于纯软件处理场景，可直接使用 FFmpeg 的常规命令；对于硬件多媒体能力验证，建议结合平台提供的 MPP、GStreamer 等组件进行联调。

### FFmpeg 安装

#### Bianbu OS

安装 FFmpeg 需运行命令：

```shell
sudo apt update
sudo apt install ffmpeg
```

检查 FFmpeg 版本：

```shell
ffmpeg -version
```

查看 FFprobe 版本：

```shell
ffprobe -version
```

#### Buildroot 系统

在编译系统镜像时，开启 `ffmpeg` 的编译集成选项（默认已开启）;
若系统镜像中已使能 FFmpeg 包，则可直接使用以下命令检查：

```shell
ffmpeg -version
```

若系统镜像中未使能 `ffmpeg` 包，则需要在 Buildroot 配置中打开对应软件包后重新编译系统镜像。

### K3 平台硬件编解码插件说明

K3 平台上的 FFmpeg 已对接平台硬件编解码器，插件名称为 `stcodec`。在 FFmpeg 中，相关硬件编解码能力以 `xxx_stcodec` 的形式注册，可用于调用平台硬件视频编解码能力。

可以通过以下命令查询 `stcodec` 相关能力：

```shell
ffmpeg -codecs | grep stcodec
```

当前 `stcodec` 支持的编解码格式如下：

| 编码格式 | 解码器 | 编码器 |
| -------- | ------ | ------ |
| AVS | `avs_stcodec` | - |
| AVS2 | `avs2_stcodec` | - |
| H.264 | `h264_stcodec` | `h264_stcodec` |
| H.265 / HEVC | `hevc_stcodec` | `hevc_stcodec` |
| MJPEG | `mjpeg_stcodec` | `mjpeg_stcodec` |
| MPEG-2 | `mpeg2_stcodec` | - |
| MPEG-4 | `mpeg4_stcodec` | `mpeg4_stcodec` |
| VC-1 | `vc1_stcodec` | - |
| VP8 | `vp8_stcodec` | `vp8_stcodec` |
| VP9 | `vp9_stcodec` | `vp9_stcodec` |

说明：

- `xxx_stcodec` 表示平台硬件编解码器。
- 若命令中未显式指定 `-c:v xxx_stcodec`，FFmpeg 可能会自动选择软件编解码器。
- 建议在验证硬件能力时，显式指定 `stcodec` 对应的编解码器名称。


### FFmpeg 功能查询

可以通过以下命令查看当前 FFmpeg 支持的能力：

#### 查看编译配置

```shell
ffmpeg -buildconf
```

#### 查看支持的封装格式

```shell
ffmpeg -formats
```

#### 查看支持的编解码器

```shell
ffmpeg -codecs
```

#### 查看 `stcodec` 硬件编解码器

```shell
ffmpeg -codecs | grep stcodec
```

#### 查看指定 `stcodec` 编解码器帮助信息

```shell
ffmpeg -h decoder=h264_stcodec
ffmpeg -h encoder=h264_stcodec
```

#### 查看支持的复用器

```shell
ffmpeg -muxers
```

#### 查看支持的解复用器

```shell
ffmpeg -demuxers
```

#### 查看支持的协议

```shell
ffmpeg -protocols
```

#### 查看支持的滤镜

```shell
ffmpeg -filters
```

#### 查看支持的像素格式

```shell
ffmpeg -pix_fmts
```

#### 查看支持的采样格式

```shell
ffmpeg -sample_fmts
```

## FFmpeg 基本命令

### `ffmpeg`

`ffmpeg` 是 FFmpeg 的核心命令，用于执行转码、封装、解封装、截图、录制、推流和拉流等操作。

基本命令格式如下：

```shell
ffmpeg [全局选项] {[输入文件选项] -i 输入文件} ... {[输出文件选项] 输出文件} ...
```

常用参数如下：

| 参数 | 说明 |
| ---- | ---- |
| `-i` | 指定输入文件、设备或网络流 |
| `-c:v` | 指定视频编码器 |
| `-c:a` | 指定音频编码器 |
| `-an` | 禁用音频 |
| `-vn` | 禁用视频 |
| `-f` | 指定输出封装格式 |
| `-r` | 指定帧率 |
| `-s` | 指定分辨率 |
| `-b:v` | 指定视频码率 |
| `-b:a` | 指定音频码率 |
| `-pix_fmt` | 指定像素格式 |
| `-ss` | 指定起始时间，可用于快进或截图 |
| `-t` | 指定处理时长 |
| `-to` | 指定结束时间 |
| `-y` | 覆盖输出文件而不提示 |
| `-re` | 按实际时间速率读取输入，常用于推流 |

### `ffprobe`

`ffprobe` 用于查看媒体文件或流的详细信息，适合在转码、播放、调试前快速确认输入内容。

示例：

```shell
ffprobe test.mp4
```

如果只想查看流信息，可执行：

```shell
ffprobe -show_streams test.mp4
```

如果只想查看封装信息，可执行：

```shell
ffprobe -show_format test.mp4
```

## 常见应用场景

### 文件信息查看

#### 查看媒体基本信息

```shell
ffprobe input.mp4
```

#### 以更清晰的格式输出信息

```shell
ffprobe -hide_banner input.mp4
```

#### 查看视频流详细信息

```shell
ffprobe -select_streams v:0 -show_streams input.mp4
```

#### 查看音频流详细信息

```shell
ffprobe -select_streams a:0 -show_streams input.mp4
```

### 视频转封装

转封装不会重新编码，速度快，适用于容器格式转换。

#### MP4 转 TS

```shell
ffmpeg -i input.mp4 -c copy output.ts
```

#### MKV 转 MP4

```shell
ffmpeg -i input.mkv -c copy output.mp4
```

### 视频转码

#### 转码为 H.264 MP4 文件

```shell
ffmpeg -i input.mkv -c:v libx264 -c:a aac output.mp4
```

#### 使用 `h264_stcodec` 进行 H.264 硬件编码

```shell
ffmpeg -i input.mkv -c:v h264_stcodec -c:a aac output.mp4
```

#### 转码为 H.265 文件

```shell
ffmpeg -i input.mp4 -c:v libx265 -c:a copy output_h265.mp4
```

#### 使用 `hevc_stcodec` 进行 H.265 硬件编码

```shell
ffmpeg -i input.mp4 -c:v hevc_stcodec -c:a copy output_h265.mp4
```

#### 仅转码视频，去除音频

```shell
ffmpeg -i input.mp4 -c:v libx264 -an output_noaudio.mp4
```

#### 仅保留音频

```shell
ffmpeg -i input.mp4 -vn -c:a aac output_audio.m4a
```

### 硬件解码与硬件编码

当需要验证平台 `stcodec` 硬件能力时，建议显式指定对应的硬件编解码器。

#### 使用 `h264_stcodec` 解码 H.264 视频

```shell
ffmpeg -c:v h264_stcodec -i input.264 -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 使用 `hevc_stcodec` 解码 H.265 视频

```shell
ffmpeg -c:v hevc_stcodec -i input.265 -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 使用 `mpeg2_stcodec` 解码 MPEG-2 视频

```shell
ffmpeg -c:v mpeg2_stcodec -i input.mpeg -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 使用 `mpeg4_stcodec` 解码 MPEG-4 视频

```shell
ffmpeg -c:v mpeg4_stcodec -i input.mpeg -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 使用 `vp8_stcodec` 解码 VP8 视频

```shell
ffmpeg -c:v vp8_stcodec -i input.webm -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 使用 `vp9_stcodec` 解码 VP9 视频

```shell
ffmpeg -c:v vp9_stcodec -i input.webm -pix_fmt yuv420p -f rawvideo output.yuv
```

### 分辨率与帧率调整

#### 调整分辨率为 1280x720

```shell
ffmpeg -i input.mp4 -vf scale=1280:720 output_720p.mp4
```

#### 调整帧率为 30fps

```shell
ffmpeg -i input.mp4 -r 30 output_30fps.mp4
```

### 裸流处理

#### 将 H.264 裸流封装为 MP4

```shell
ffmpeg -f h264 -i input.264 -c copy output.mp4
```

#### 使用 `h264_stcodec` 解码 H.264 裸流并输出原始数据

```shell
ffmpeg -c:v h264_stcodec -f h264 -i input.264 -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 将 H.265 裸流封装为 MP4

```shell
ffmpeg -f hevc -i input.265 -c copy output.mp4
```

#### 使用 `hevc_stcodec` 解码 H.265 裸流并输出原始数据

```shell
ffmpeg -c:v hevc_stcodec -f hevc -i input.265 -pix_fmt yuv420p -f rawvideo output.yuv
```

#### 将 MJPEG 序列封装为 MP4

```shell
ffmpeg -framerate 30 -i frame_%04d.jpg -c:v libx264 output.mp4
```

### YUV 原始数据处理

处理 YUV 原始数据时，需要明确指定像素格式、分辨率和帧率。

#### 将 YUV420P 原始数据编码为 H.264

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v libx264 output.mp4
```

#### 使用 `h264_stcodec` 将 YUV420P 原始数据编码为 H.264

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v h264_stcodec output.mp4
```

#### 使用 `hevc_stcodec` 将 NV12 原始数据编码为 H.265

```shell
ffmpeg -f rawvideo -pix_fmt nv12 -s 1920x1080 -r 30 -i input.yuv -c:v hevc_stcodec output.mp4
```

#### 使用 `mjpeg_stcodec` 对 MJPEG 进行硬件编解码

```shell
ffmpeg -c:v mjpeg_stcodec -i input.mjpg -pix_fmt yuv420p -f rawvideo output.yuv
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v mjpeg_stcodec output.mjpg
```

### 音频处理

#### 提取 AAC 音频

```shell
ffmpeg -i input.mp4 -vn -c:a copy output.aac
```

#### 将音频转码为 MP3

```shell
ffmpeg -i input.wav -c:a libmp3lame output.mp3
```

#### 将音频转码为 WAV

```shell
ffmpeg -i input.mp3 output.wav
```

### 网络推流与拉流

#### 推送 RTMP 流

```shell
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://<server>/<app>/<stream>
```

#### 推送 RTP/UDP 码流

```shell
ffmpeg -re -i input.mp4 -an -c:v copy -f rtp rtp://<ip>:<port>
```

#### 拉取 RTSP 流并保存到本地

```shell
ffmpeg -i rtsp://<server>/<stream> -c copy output.mp4
```

#### 拉取网络流并播放调试

```shell
ffplay rtsp://<server>/<stream>
```

### 摄像头采集场景

在 Linux 环境下，可通过 V4L2 设备节点采集摄像头数据。使用前建议先确认设备节点及其格式能力。

`v4l2-ctl` 是 `v4l-utils` 软件包提供的工具，用于查看 V4L2 设备信息、支持格式和控制项。

#### `v4l2-ctl` 安装说明

**Bianbu OS**

可通过 `apt` 安装：

```shell
sudo apt update
sudo apt install v4l-utils
```

安装完成后，可通过以下命令检查：

```shell
v4l2-ctl --version
```

**Buildroot 系统**

在编译系统镜像时，开启 `v4l-utils` 的编译集成选项。若系统镜像中已使能该软件包，则可直接使用 `v4l2-ctl`；若未使能，则需要在 Buildroot 配置中打开对应软件包后重新编译系统镜像。

安装完成后，可通过以下命令检查：

```shell
v4l2-ctl --version
```

#### 查看摄像头设备

```shell
v4l2-ctl --list-devices
```

#### 查看设备支持的格式

```shell
v4l2-ctl -d /dev/video0 --list-formats-ext
```

#### 采集摄像头并保存为 MP4 文件

```shell
ffmpeg -f v4l2 -framerate 30 -video_size 1280x720 -i /dev/video0 -c:v h264_stcodec output.mp4
```

#### 采集 MJPEG 摄像头并转存

```shell
ffmpeg -f v4l2 -input_format mjpeg -framerate 30 -video_size 1920x1080 -i /dev/video0 -c copy output.mjpeg
```

### 播放调试

#### 播放本地视频

```shell
ffplay input.mp4
```

#### 循环播放视频

```shell
ffplay -loop 0 input.mp4
```

#### 无音频播放

```shell
ffplay -an input.mp4
```

## 常见问题

### 如何确认当前使用的是硬件编解码器

建议在命令行中显式指定 `-c:v xxx_stcodec`，并结合以下命令确认 FFmpeg 是否包含对应编解码器：

```shell
ffmpeg -codecs | grep stcodec
```

如果日志中显示实际使用的是 `h264_stcodec`、`hevc_stcodec` 等编解码器，则说明当前已走硬件路径。

### 无法识别输入格式

如果 FFmpeg 无法自动识别输入裸流格式，可以显式指定格式，例如：

```shell
ffmpeg -f h264 -i input.264 -c copy output.mp4
```

### YUV 原始数据处理失败

处理 `.yuv` 原始数据时，必须显式指定以下参数：

- 像素格式：`-pix_fmt`
- 分辨率：`-s`
- 帧率：`-r`

例如：

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv output.mp4
```

### 覆盖输出文件提示

如果不希望每次执行时手动确认是否覆盖输出文件，可以增加 `-y` 参数：

```shell
ffmpeg -y -i input.mp4 output.mp4
```

### 查看更详细的日志

可以通过 `-loglevel` 调整日志级别，例如：

```shell
ffmpeg -loglevel debug -i input.mp4 -f null -
```

常见日志级别包括：`quiet`、`panic`、`fatal`、`error`、`warning`、`info`、`verbose`、`debug`、`trace`。

在排查 `stcodec` 硬件编解码问题时，建议优先使用如下命令查看详细日志：

```shell
ffmpeg -loglevel debug -c:v h264_stcodec -i input.264 -f null -
```
