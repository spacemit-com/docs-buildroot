sidebar_position: 2

# FFmpeg Usage Guide

## FFmpeg

FFmpeg is an open-source multimedia toolkit that provides capabilities such as audio and video capture, encoding and decoding, muxing and demuxing, filter processing, streaming (publishing/receiving), screenshot capture, transcoding, and more.

Official Website: [https://ffmpeg.org](https://ffmpeg.org/)

### FFmpeg Components

FFmpeg is a collection of command-line tools and underlying libraries rather than a single program. Common components include:

| Component      | Functionality |
| ----------- | -------- |
| `ffmpeg`    |  The main command for audio/video transcoding, remuxing, filter processing, screenshot capture, streaming, etc. |
| `ffplay`    | A lightweight media player based on SDL, used for quick playback and debugging of media files or streams |
| `ffprobe`   | Used to view detailed information about media files or streams, such as encoding format, resolution, duration, bitrate, etc. |
| `libavcodec` | Codec library, responsible for audio/video encoding and decoding |
| `libavformat` | Muxing/demuxing library, responsible for audio/video container format processing |
| `libavfilter` | Filter library, responsible for scaling, cropping, overlaying, watermarking, and other audio/video processing |
| `libswscale`  | Image scaling and pixel format conversion library |
| `libswresample` | Audio resampling and format conversion library |

### FFmpeg Installation

- Bianbu OS

Run the following commands to install FFmpeg:

```shell
sudo apt update
sudo apt install ffmpeg
```

Check FFmpeg version:

```shell
ffmpeg -version
```

Check the `ffprobe` version:

```shell
ffprobe -version
```

- Buildroot

When compiling the system image, enable the `ffmpeg` integration option, which is enabled by default.
If the FFmpeg package is already enabled in the system image, verify it with:

```shell
ffmpeg -version
```

If the FFmpeg package is not enabled in the system image, enable the corresponding package in the Buildroot configuration and rebuild the system image.

### K3 Platform Hardware Encoding/Decoding Plugins

FFmpeg on the K3 platform integrates platform hardware codecs. Plugin names follow the format `[codec_format]_stcodec`. The supported FFmpeg codec plugins on the K3 platform are as follows:

The encoding and decoding formats supported by `stcodec` are as follows:

| Encoding Format | Decoder | Encoder |
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

Query the supported hardware codec plugins with the following command:

```shell
ffmpeg -codecs | grep stcodec
```

Note:

- `xxx_stcodec` represents a platform hardware codec.
- If `-c:v xxx_stcodec` is not explicitly specified in the command, FFmpeg automatically selects the software codec.

## FFmpeg Basic Command

### Tools

- ffmpeg

`ffmpeg` is the core FFmpeg command, used for operations such as transcoding, muxing, demuxing, screenshot capture, recording, and publishing or receiving streams.

Basic command format:

```shell
ffmpeg [global options] {[input file options] -i input file} ... {[output file options] output file} ...
```

Parameters:

| Parameter | Description |
| ---- | ---- |
| `-i` | Specifies the input file, device, or network stream |
| `-c:v` | Specifies the video codec |
| `-c:a` | Specifies the audio codec |
| `-an` | Disables audio |
| `-vn` | Disables video |
| `-f` | Specifies the output container format |
| `-r` | Specifies the frame rate |
| `-s` | Specifies the resolution |
| `-b:v` | Specifies the video bitrate |
| `-b:a` | Specifies the audio bitrate |
| `-pix_fmt` | Specifies the pixel format |
| `-ss` | Specifies the start time, used for fast seeking or screenshot capture |
| `-t` | Specifies the processing duration |
| `-to` | Specifies the end time |
| `-y` | Overwrites the output file without confirmation |
| `-re` | Reads input at the native frame rate, commonly used for streaming |

- ffprobe

`ffprobe` is used to view information about media files or streams. It is useful for quickly verifying input content before transcoding, playback, or debugging.

- ffplay

`ffplay` is used for video playback.

### View File Information

- View basic media information

```shell
ffprobe input.mp4
```

- Output information in a clearer format

```shell
ffprobe -hide_banner input.mp4
```

- View video information

```shell
ffprobe -select_streams v:0 -show_streams input.mp4
```

- View audio information

```shell
ffprobe -select_streams a:0 -show_streams input.mp4
```

### Video Remuxing

Remuxing does not re-encode the stream, so it is fast and suitable for container format conversion.

- MP4 to TS

```shell
ffmpeg -i input.mp4 -c copy output.ts
```

- MKV to MP4

```shell
ffmpeg -i input.mkv -c copy output.mp4
```

### Hardware Decoding

- Use `h264_stcodec` to decode H.264 video

```shell
ffmpeg -c:v h264_stcodec -i input.264 -pix_fmt yuv420p -f rawvideo output.yuv
```

- Use `hevc_stcodec` to decode H.265 video

```shell
ffmpeg -c:v hevc_stcodec -i input.265 -pix_fmt yuv420p -f rawvideo output.yuv
```

- Use `mpeg2_stcodec` to decode MPEG-2 video

```shell
ffmpeg -c:v mpeg2_stcodec -i input.mpeg -pix_fmt yuv420p -f rawvideo output.yuv
```

- Use `mpeg4_stcodec` to decode MPEG-4 video

```shell
ffmpeg -c:v mpeg4_stcodec -i input.mpeg -pix_fmt yuv420p -f rawvideo output.yuv
```

- Use `vp8_stcodec` to decode VP8 video

```shell
ffmpeg -c:v vp8_stcodec -i input.webm -pix_fmt yuv420p -f rawvideo output.yuv
```

- Use `vp9_stcodec` to decode VP9 video

```shell
ffmpeg -c:v vp9_stcodec -i input.webm -pix_fmt yuv420p -f rawvideo output.yuv
```

### Hardware Encoding

- Use `h264_stcodec` to encode YUV420P raw data to H.264

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v h264_stcodec output.mp4
```

- Use `hevc_stcodec` to encode NV12 raw data to H.265

```shell
ffmpeg -f rawvideo -pix_fmt nv12 -s 1920x1080 -r 30 -i input.yuv -c:v hevc_stcodec output.mp4
```

- Use `mjpeg_stcodec` for MJPEG hardware encoding/decoding

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v mjpeg_stcodec output.mjpg
```

### Video Transcoding

- HEVC to H.264

```shell
ffmpeg -c:v hevc_stcodec -i input.mkv -c:v h264_stcodec -c:a copy output.mp4
```

**Note:** This command uses hardware decoding for the HEVC input and hardware encoding for the H.264 output.

### Network Streaming: Publishing and Receiving

- Publish an RTMP stream

```shell
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://<server>/<app>/<stream>
```

- Publish an RTP/UDP stream

```shell
ffmpeg -re -i input.mp4 -an -c:v copy -f rtp rtp://<ip>:<port>
```

- Receive an RTSP stream and save it locally

```shell
ffmpeg -i rtsp://<server>/<stream> -c copy output.mp4
```

- Receive a network stream and play it for debugging

```shell
ffplay rtsp://<server>/<stream>
```

### Camera Capture

On Linux, camera data can be captured through a V4L2 device node. First, confirm the device node and its supported formats.

`v4l2-ctl` is a tool provided by `v4l-utils`, used to view V4L2 device information, supported formats, and control options.

- `v4l2-ctl` Installation

**Bianbu OS**

Install it via `apt`:

```shell
sudo apt update
sudo apt install v4l-utils
```

Then verify the installation with the following command:

```shell
v4l2-ctl --version
```

**Buildroot**

When compiling the system image, enable the integration option for `v4l-utils`. If it is already enabled in the system image, `v4l2-ctl` can be used directly. Otherwise, enable the corresponding package in the Buildroot configuration and rebuild the system image.

Then verify it with the following command:

```shell
v4l2-ctl --version
```

- View camera devices

```shell
v4l2-ctl --list-devices
```

- View supported formats

```shell
v4l2-ctl -d /dev/video0 --list-formats-ext
```

- Capture camera YUV data and save it as an MP4 file

```shell
ffmpeg -f v4l2 -framerate 30 -video_size 1280x720 -i /dev/video0 -c:v h264_stcodec output.mp4
```

- Capture camera MJPEG data and save it to a file

```shell
ffmpeg -f v4l2 -input_format mjpeg -framerate 30 -video_size 1280x720 -i /dev/video0 -c copy output.mjpeg
```

- Preview camera MJPEG data

```shell
ffplay -f v4l2 -input_format mjpeg -video_size 1280x720 -i /dev/video1 -codec:v mjpeg_stcodec
```

### Video Playback

- Play a local video

```shell
ffplay -codec:v h264_stcodec input.mp4
```

- Loop playback of a video

```shell
ffplay -codec:v h264_stcodec input.mp4 -loop 0
```

**Note:** `-codec:v h264_stcodec` specifies the H.264 hardware decoder. Replace it with the corresponding `stcodec` decoder based on the actual input format.

## FAQ

### How to confirm that hardware codecs are used

Explicitly specify `-c:v xxx_stcodec` in the command line, and use the following command to verify that FFmpeg contains the corresponding codec:

```shell
ffmpeg -codecs | grep stcodec
```

If the log shows that codecs such as `h264_stcodec` or `hevc_stcodec` are being used, the hardware path is active.

### Input format not recognized

If FFmpeg fails to auto-identify raw stream format, explicitly specify the format. For example:

```shell
ffmpeg -f h264 -i input.264 -c copy output.mp4
```

### YUV raw data processing failed

When processing `.yuv` raw data, the following parameters must be explicitly specified:

- Pixel format: `-pix_fmt`
- Resolution: `-s`
- Frame rate: `-r`

For example:

```shell
ffmpeg -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i input.yuv -c:v h264_stcodec output.mp4
```

### Suppress output file prompt

To suppress the overwrite prompt for the output file, add the `-y` parameter:

```shell
ffmpeg -y -i input.mp4 output.mp4
```

### View detailed logs

You can adjust the log level via `-loglevel`, for example:

```shell
ffmpeg -loglevel debug -i input.mp4 -f null -
```

Common log levels:  

- `quiet`
- `panic`
- `fatal`
- `error`
- `warning`
- `info`
- `verbose`
- `debug`
- `trace`

When troubleshooting issues with `stcodec` hardware codecs, use the following command to view detailed logs:

```shell
ffmpeg -loglevel debug -c:v h264_stcodec -i input.264 -f null -
```
