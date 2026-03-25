sidebar_position: 2

# GStreamer User Guide

## GStreamer Intruduction

GStreamer is an open-source multimedia framework. Designed based on plugins, it enables all plugins to be linked to any predefined data stream pipeline.

Official website: [https://gstreamer.freedesktop.org](https://gstreamer.freedesktop.org/)

### GStreamer Framework

By creating and linking elements, GStreamer creates a pipeline that enables data streams to flow between these linked elements, thereby accomplishing a specific task such as media playback or audio recording.

Gstreamer framework:

![](static/Jk4JbKgVlonB2txkiBHctjycnXf.png)

### GStreamer Source Code Distribution Structure

GStreamer codes are divided into different code repos according to functional modules with each repo responsible for different functions. The brief description of each repo are as follows:

| Repo Name            | Function Description                                                                 |
|-------------------|------------------------------------------------------------------------|
| `gstreamer`      | Framework and base repo                                     |
| `gst-plugins-base` | Framework and base repo                                    |
| `gst-plugins-good` | Mature and stable plugins                                |
| `gst-plugins-bad`  | Plugins under development, potentially unstable                     |
| `gst-plugins-ugly` | Plugins with license issues, which are optional for users according to local laws and regulations                              |
| `gst-libav`        | Codec plugins based on libav                     |

This structure ensures that each repo is independent, yet all depend on `gstreamer` and `get-plugins-base`.

### GStreamer Installation

Run the following command to install Gstreamer-1.0:

```
sudo apt-get update

sudo apt-get install gstreamer1.0-tools gstreamer1.0-alsa gstreamer1.0-plugins-base gstreamer1.0-plugins-good  gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav

sudo apt-get install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-good1.0-dev libgstreamer-plugins-bad1.0-dev  
```

Run the following command to inspect the version of Gstreamer-1.0:

```
gst-inspect-1.0 --version  
```

### Gstreamer Plugin Description

Run the following command to inspect the default GStreamer plugins supported in current **Bianbu OS/Buildroot** system:

```
gst-inspect-1.0
```

Running the `gst-inspect-1.0` command followed by a specific plugin name will display detailed information about that plugin.

#### Video Decoder Plugins

Video decoder converts the source video format into a raw format that can be interpreted by target sink (e.g., display device). **SpacemiT GStreamer supports the spacemitdec proprietary plugin, which helps users achieve superior results.**

| Video Decoder | Package            | Description                                             | Bianbu OS(Y/N) | Buildroot(Y/N) |
|---------------|--------------------|---------------------------------------------------------|----------------|-------------------|
| decodebin     | gst-plugins-base   | Autoplug and decode to raw media                        | Y              | Y                 |
| spacemitdec   | gst-plugins-bad    | Decodes H264/H265/MJPEG/VP8/VP9/MPEG2/MPEG4 via MPP API | Y              | Y                 |
| avdec_xxxx    | gstreamer1.0-libav | ffmpeg plugin for GStreamer                             | Y              | Y                 |
| mpeg2dec      | gst-plugins-ugly   | mpeg1 and mpeg2 video decoder                           | Y              | N                 |
| openh264dec   | gst-plugins-bad    | OpenH264 video decoder                                  | Y              | N                 |
| jpegdec       | gst-plugins-good   | Decode images from JPEG format                          | Y              | N                 |
| vp8dec        | gst-plugins-good   | On2 VP8 Decoder                                         | Y              | N                 |
| vp9dec        | gst-plugins-good   | On2 VP9 Decoder                                         | Y              | N                 |
|               |                    |                                                         |                |                   |

#### Video Encoder Plugins

Video encoder converts raw data into an encoded video format, such as H.264. **SpacemiT GStreamer supports the spacemitenc proprietary plugins, which help users achieve superior results.**

| Video Encoder    | Package            | Description                         | Bianbu OS(Y/N) | Buildroot(Y/N) |
|------------------|--------------------|-------------------------------------|----------------|-------------------|
| encodebin        | gst-plugins-base   | Convenience encoding/muxing element | Y              | N                 |
| spacemith264enc  | gst-plugins-bad    | Encodes H264 via MPP API            | Y              | Y                 |
| spacemith265enc  | gst-plugins-bad    | Encodes H265 via MPP API            | Y              | Y                 |
| spacemitmjpegenc | gst-plugins-bad    | Encodes MJPEG via MPP API           | Y              | Y                 |
| spacemitmpegenc  | gst-plugins-bad    | Encodes MPEG2/MPEG4 via MPP API     | Y              | Y                 |
| spacemitvp8enc   | gst-plugins-bad    | Encodes vp8 via MPP API             | Y              | Y                 |
| spacemitvp9enc   | gst-plugins-bad    | Encodes vp9 via MPP API             | Y              | Y                 |
| avenc_xxxx       | gstreamer1.0-libav | ffmpeg plugin for GStreamer         | Y              | Y                 |
| mpeg2enc         | gst-plugins-ugly   | mpeg2enc video encoder              | Y              | N                 |
| openh264enc      | gst-plugins-bad    | OpenH264 video encoder              | Y              | N                 |
| jpegenc          | gst-plugins-good   | JPEG image encoder                  | Y              | N                 |
| vp8enc           | gst-plugins-good   | On2 VP8 Encoder                     | Y              | N                 |
| vp9enc           | gst-plugins-good   | On2 VP9 Encoder                     | Y              | N                 |
|                  |                    |                                     |                |                   |

#### Video Sink Plugins

Video sink plugins displays processed data by visual output. **SpacemiT GStreamer optimizes glimagesink/gtkglsink/way landsink plugins, whch help users achieve superior results.**

| Video Encoder  | Package          | Description                                             | Bianbu OS(Y/N) | Buildroot(Y/N) |
|----------------|------------------|---------------------------------------------------------|----------------|-------------------|
| autovideosink  | gst-plugins-good | Wrapper video sink for automatically detected videosink | Y              | Y                 |
| glimagesink    | gst-plugins-base | Infrastructure to process GL textures                   | Y              | N                 |
| waylandsink    | gst-plugins-bad  | Output to wayland surface                               | Y              | Y                 |
| gtkglsink      | gst-plugins-good | A video sink that renders to a GtkWidget using OpenGL   | Y              | N                 |
| fpsdisplaysink | gst-plugins-bad  | Video sink with current and average framerate           | Y              | N                 |

#### Demux Plugins

Demux plugins convert different video/audio format into a raw format.

| Video Demux   | Package          | Description                        | Bianbu OS(Y/N) | Buildroot(Y/N) |
|---------------|------------------|------------------------------------|----------------|-------------------|
| qtdemux       | gst-plugins-good | Demux a .mov/.mp4 file to raw data | Y              | Y                 |
| matroskedemux | gst-plugins-good | Demux a .mkv file to raw data      | Y              | N                 |
| flvdemux      | gst-plugins-good | Demux a .flv file to raw data      | Y              | N                 |
| avidemux      | gst-plugins-good | Demux a .avi file to raw data      | Y              | Y                 |

#### Mux Plugins

Mux plugins convert undermuxed raw data into specific video/audio data.

| Video Mux   | Package          | Description                   | Bianbu OS(Y/N) | Buildroot(Y/N) |
|-------------|------------------|-------------------------------|----------------|-------------------|
| qtmux       | gst-plugins-good | Mux a raw data to a .mov file | Y              | Y                 |
| matroskemux | gst-plugins-good | Mux a raw data to a .mkv file | Y              | N                 |
| flvmux      | gst-plugins-good | Mux a raw data to a .flv file | Y              | N                 |
| avimux      | gst-plugins-good | Mux a raw data to a .avi file | Y              | Y                 |
| mp4mux      | gst-plugins-good | Mux a raw data to a .mp4 file | Y              | Y                 |
|             |                  |                               |                |                   |

#### Audio Plugins

Audio plugins process data from raw audio formats or specific audio formats (e.g. WAV).

| Audio Plugin   | Package          | Description                                     | Bianbu OS(Y/N) | Buildroot(Y/N) |
|----------------|------------------|-------------------------------------------------|----------------|-------------------|
| mpg123audiodec | gst-plugins-good | MP3 decoding plugin based on the mpg123 library | Y              | N                 |
| vorbisdec      | gst-plugins-base | Decodes raw vorbis streams to float audio       | Y              | N                 |
| vorbisenc      | gst-plugins-base | Encodes audio in Vorbis format                  | Y              | N                 |
| alsasink       | gst-plugins-base | Output to a sound card via ALSA                 | Y              | N                 |
| pulsesink      | gst-plugins-good | Plays audio to a PulseAudio server              | Y              | N                 |

#### Image Plugins

Image plugins process data from raw image formats or specific data formats (e.g. JPEG).

| Image Plugin     | Package          | Description                                             | Bianbu OS(Y/N) | Buildroot(Y/N) |
|------------------|------------------|---------------------------------------------------------|----------------|-------------------|
| spacemitdec      | gst-plugins-bad  | Decodes H264/H265/MJPEG/VP8/VP9/MPEG2/MPEG4 via MPP API | Y              | Y                 |
| spacemitmjpegenc | gst-plugins-bad  | Encodes MJPEG via MPP API                               | Y              | Y                 |
| imagefreeze      | gst-plugins-good | Generates a still frame stream from an image            | Y              | N                 |
| jpegdec          | gst-plugins-good | Decode images from JPEG format                          | Y              | N                 |
| jpegenc          | gst-plugins-good | JPEG image encoder                                      | Y              | N                 |
| pngdec           | gst-plugins-good | Decode a png video frame to a raw image                 | Y              | N                 |
| pngenc           | gst-plugins-good |  Encode a video frame to a .png image                   | Y              | N                 |
|                  |                  |                                                         |                |                   |

#### Network Protocol Plugins

Network protocol plugins establish connection between devices.

| Network Plugins | Package          | Description                                                      | Bianbu OS(Y/N) | Buildroot(Y/N) |
|-----------------|------------------|------------------------------------------------------------------|----------------|-------------------|
| udpsink         | gst-plugins-good | Send data over the network via UDP                               | Y              | Y                 |
| multiudpsink    | gst-plugins-good | Send data over the network via UDP to one or multiple recipients | Y              | Y                 |
| udpsrc          | gst-plugins-good | Receive data over the network via UDP                            | Y              | Y                 |
| tcpserversink   | gst-plugins-base | Send data as a server over the network via TCP                   | Y              | N                 |
| tcpclientsrc    | gst-plugins-base | Receive data as a client over the network via TCP                | Y              | N                 |
| rtspsrc         | gst-plugins-good | Receive data over the network via RTSP                           | Y              | N                 |

#### Payload/Depayload Plugins

Payload plugins transmit data over a network, while depayload plugins are used in conjunction with them to receive and unpack the data.

| Payload/Depayload Plugins | Package          | Description                                                           | Bianbu OS(Y/N) | Buildroot(Y/N) |
|-----------------|------------------|-----------------------------------------------------------------------|----------------|-------------------|
| gdppay          | gst-plugins-bad  | Payloads GStreamer Data Protocol buffers                              | Y              | N                 |
| gdpdepay        | gst-plugins-bad  | Depayloads GStreamer Data Protocol buffers                            | Y              | N                 |
| rtpvrawpay      | gst-plugins-good | Payload raw video as RTP packets                                      | Y              | Y                 |
| rtpvrawdepay    | gst-plugins-good | Extracts raw video as RTP packets                                     | Y              | Y                 |
| rtph264pay      | gst-plugins-good | Payload-encode H264 video into RTP packets                            | Y              | Y                 |
| rtph264depay    | gst-plugins-good | Extracts H264 video from RTP packets                                  | Y              | Y                 |
| rtpmpapay       | gst-plugins-good | Payload MPEG audio as RTP packets                                     | Y              | Y                 |
| rtpmpadepay     | gst-plugins-good | Extracts MPEG audio from RTP packets                                  | Y              | Y                 |
| rtpjitterbuffer | gst-plugins-good | A buffer that deals with network jitter and other transmission faults | Y              | Y                 |

## Gstreamer Basic Command

### `gst-launch-1.0`

`gst-launch-1.0`: Used to initiate a pipeline to perform multimedia tasks, such as media playback and audio recording.

There are some common usage examples (mainly includes GStreamer plugins adapted by SpacemiT):

#### Camera Application Scenario

##### UVC Camera

- UVC camera information can be obtained via the following command：

  ```
  $ gst-device-monitor-1.0
  Device found:

        name  : UvcH264 HD Pro Webcam C920
        class : Video/CameraSource
        caps  : video/x-raw, format=YUY2, width=2304, height=1536, pixel-aspect-ratio=1/1, framerate=2/1
                video/x-raw, format=YUY2, width=2304, height=1296, pixel-aspect-ratio=1/1, framerate=2/1
                video/x-raw, format=YUY2, width=1920, height=1080, pixel-aspect-ratio=1/1, framerate=5/1
                video/x-raw, format=YUY2, width=1600, height=896, pixel-aspect-ratio=1/1, framerate={ (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=1280, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=960, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=1024, height=576, pixel-aspect-ratio=1/1, framerate={ (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=800, height=600, pixel-aspect-ratio=1/1, framerate={ (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=864, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=800, height=448, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=640, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=640, height=360, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=432, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=352, height=288, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=320, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=320, height=180, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=176, height=144, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=160, height=120, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-raw, format=YUY2, width=160, height=90, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=1920, height=1080, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=1600, height=896, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=1280, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=960, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=1024, height=576, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=800, height=600, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=864, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=800, height=448, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=640, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=640, height=360, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=432, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=352, height=288, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=320, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=320, height=180, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=176, height=144, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=160, height=120, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                image/jpeg, parsed=true, width=160, height=90, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=1920, height=1080, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=1600, height=896, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=1280, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=960, height=720, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=1024, height=576, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=800, height=600, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=864, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=800, height=448, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=640, height=480, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=640, height=360, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=432, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=352, height=288, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=320, height=240, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=320, height=180, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=176, height=144, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=160, height=120, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
                video/x-h264, stream-format=byte-stream, alignment=au, width=160, height=90, pixel-aspect-ratio=1/1, framerate={ (fraction)30/1, (fraction)24/1, (fraction)20/1, (fraction)15/1, (fraction)10/1, (fraction)15/2, (fraction)5/1 }
        properties:
                device.path = /dev/video20
                udev-probed = false
                device.api = uvch264
                v4l2.device.driver = uvcvideo
                v4l2.device.card = HD Pro Webcam C920
                v4l2.device.bus_info = usb-xhci-hcd.0.auto-1.3
                v4l2.device.version = 394815 (0x0006063f)
                v4l2.device.capabilities = 2225078273 (0x84a00001)
                v4l2.device.device_caps = 69206017 (0x04200001)
                device.is-camerasrc = true
        gst-launch-1.0 uvch264src device=/dev/video20.vfsrc name=camerasrc ! ... camerasrc.vidsrc ! [video/x-h264] ...
  ```

  As shown, the command outputs various critical information, such as camera resolution, frame rate, supported formats and corresponding video capture codes for the UVC camera.

  Alternatively, relevent information can be obtained using `v412-ct1`, which will not be detailed here.

  ```
  $ v4l2-ctl --list-devices
  HD Pro Webcam C920 (usb-xhci-hcd.0.auto-1.3):
        /dev/video20
        /dev/video21
        /dev/media1
  ```

  There are some examples of using GStreamer to capture images from a UVC camera, including displaying, discarding, and saving the images. These examples use `/dev/video20` as the video device to capture YUY2 formatted images with a resolution of 640x480 at 30fps.

  - Capture the image and send it to the display.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "video/x-raw,framerate=30/1,format=YUY2,width=640,height=480" ! videoconvert ! glsinkbin sink=gtkglsink
    ```

  - Capture the image and send it to the display while showing frame rate.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "video/x-raw,framerate=30/1,format=YUY2,width=640,height=480" ! videoconvert ! fpsdisplaysink  video-sink='glsinkbin sink='gtkglsink''
    ```

  - Capture the image and then discard it.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "video/x-raw,framerate=30/1,format=YUY2,width=640,height=480" ! fakesink
    ```

  - Capture the image and then save it as file.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "video/x-raw,framerate=30/1,format=YUY2,width=640,height=480" ! filesink location=output.yuv
    ```

- Using /dev/video20 as the UVC camera device, capture 600 frames of 480p JPEG format images and decode them.(Resolution and frame rate can be adjusted according to needs, provided that the camera itself supports output in these specifications.)

  - Decode the image and send it to the display.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "image/jpeg,framerate=30/1,width=640,height=480" ! typefind ! spacemitdec ! waylandsink sync=0 render-rectangle="<0,0,1280,720>"
    ```

  - Decode the image and re-encoded it before saving it as file.

    ```
    gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "image/jpeg,framerate=30/1,width=640,height=480" ! typefind ! spacemitdec !  spacemith264enc ! filesink location=test.h264
    ```

##### MIPI Camera Usage Examples

There are examples using GStream to output 1080P@NV12 from MIPI camera with OV16A10. These examples assume that the JSON configuration file for spascemitsrc has been correctly set up. For detailed configurations, please refer to [Camera developement guide](/en/k1_buildroot/camera/camera_development_guide.md).

- After image capture, the frame is sent for display at a resolution of 720p. (The display position cannot be configured for now.)

  ```
  gst-launch-1.0  spacemitsrc location=/usr/share/camera_json/csi1_camera_auto.json close-dmabuf=0 ! "video/x-raw(memory:DMABuf),format=NV12,width=1920,height=1080" ! waylandsink sync=0 render-rectangle="<0,0,1280,720>"
  ```

- Capture image and then discard it.

  ```
  gst-launch-1.0  spacemitsrc location=/usr/share/camera_json/csi1_camera_auto.json close-dmabuf=0 ! "video/x-raw(memory:DMABuf),format=NV12,width=1920,height=1080" ! fakesink
  ```

- Capture 10 frames and save them to a file.

  ```
  gst-launch-1.0  spacemitsrc location=/usr/share/camera_json/csi1_camera_auto.json close-dmabuf=0 num-buffers=10 ! "video/x-raw(memory:DMABuf),format=NV12,width=1920,height=1080" ! filesink location=test.yuv
  ```

- Capture 1000 frames, encode them in H.264 format, and save to a file.

  ```
  gst-launch-1.0  spacemitsrc location=/usr/share/camera_json/csi1_camera_auto.json close-dmabuf=0 num-buffers=1000 ! "video/x-raw(memory:DMABuf),format=NV12,width=1920,height=1080" ! spacemith264enc ! filesink location=test.h264
  ```

On Bianbu, use OpenCV to capture and display video from a MIPI camera via GStreamer. The detailed steps are as follows:

1. Install required tools and libraries

   ```
   sudo apt install libopencv-dev python3 python3-opencv
   ```

2. Create the py script `capture_video_opencv.py`

   ```
   import cv2

   gst_str = 'spacemitsrc location=/home/bianbu/camtest_ov16a10.json close-dmabuf=1 ! video/x-raw,format=NV12,width=1280,height=720 ! appsink'

   cap = cv2.VideoCapture(gst_str, cv2.CAP_GSTREAMER)  # open default camera

   while True:
    ret, frame = cap.read()  # read video frame
    frame = cv2.cvtColor(frame, cv2.COLOR_YUV2BGR_NV12)
    cv2.imshow('Video', frame)  # show video frame

    if cv2.waitKey(1) & 0xFF == ord('q'):  # press the 'q' key to break the loop
        break

   cap.release()  # release camera
   cv2.destroyAllWindows()  # destroy all windows

   ```

3. Execute the script

   ```
   python3 capture_video_opencv.py
   ```

In the above demo, OpenCV leverages GStreamer for image capture, outputting data in 720p@NV12 format. After acquiring the data, OpenCV converts it to RGB format and then displays it.

#### Decoding Application Scenarios

- Raw stream video decoding

  - h264 decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/h264/h264_w1280_h720_f30_r4_p1_8bit_300f_2112kb_high_cabac.264 ! h264parse ! spacemitdec ! queue ! waylandsink render-rectangle="<0,0,1280,720>"
   ```

  - h265 decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/hevc/hevc_w1920_h1080_f25_r_p1_8bit_200f_1878kb_main.265 ! queue ! h265parse ! spacemitdec ! queue ! waylandsink render-rectangle="<0,0,1280,720>"
   ```

  - vp8, vp9 decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/vp9/vp9_w1280_h720_f25_r_p1_8bit_120f_1996kb.ivf ! typefind ! ivfparse ! spacemitdec ! queue ! waylandsink render-rectangle="<0,0,1280,720>"
   ```

  - mjpeg decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/mjpeg/mjpeg_w1280_h720_f_r_p1_8bit_120f_kb_yuv420.mjpeg ! typefind ! spacemitdec ! queue ! waylandsink render-rectangle="<0,0,1280,720>"
   ```

  - mpeg2 decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/mpeg2/mpeg2_w1920_h1080_f30_r_p1_8bit_120f_6236kb_main.mpg ! mpegpsdemux ! mpegvideoparse ! spacemitdec ! queue ! waylandsink render-rectangle="<0,0,1280,720>"
   ```

  - mpeg4 decoding and display

   ```
   gst-launch-1.0  filesrc location=/root/compressed/mpeg4/mpeg4_w1280_h720_f_r_p1_8bit_120f_3429kb_simple.mpeg4 ! mpeg4videoparse ! spacemitdec ! queue ! waylandsink sync=0 render-rectangle="<0,0,1280,720>"
   ```

- video decoding for container formats

  - H.264/H.265/VP8/VP9/MJPEG/MPEG decoding and display

   ```
   gst-launch-1.0 filesrc location=C079_1080P_AVC_AAC_8M_24F.mp4 ! qtdemux name=d d.video_0 ! queue ! **h264parse** ! spacemitdec ! queue ! waylandsink  render-rectangle="<0,0,1280,720>"
   ```

`spacemitdec` supports multiple video formats. To use it correctly, please be sure to invoke the appropriate parser (for example: use h264parse for H.264 format, h265parse for the H.265 format, etc.).

#### Encoding Application Scenarios

Testing video source: NV12 format, 720p (1280×720), 25fps

- **encoded as H.264**

  ```
  gst-launch-1.0 videotestsrc num-buffers=100 ! 'video/x-raw,format=NV12, width=1280, height=720, framerate=25/1' ! spacemith264enc ! filesink location=test.264
  ```

  or input from YUV file:

  ```
  gst-launch-1.0 filesrc location=nv12_720p_100f.yuv ! videoparse format=23 width=1280 height=720 framerate=30/1 ! spacemith264enc ! filesink location=test.264 
  ```

- **encoded as H.265**

  ```
  gst-launch-1.0 videotestsrc num-buffers=100 ! 'video/x-raw,format=NV12, width=1280, height=720, framerate=25/1' ! spacemith265enc ! filesink location=test.265
  ```

- **encoded as VP9 (encapsulated in WebM)**

  ```
  gst-launch-1.0 -v videotestsrc num-buffers=1000 ! spacemitvp9enc ! webmmux ! filesink location=videotestsrc.webm
  //corresponding decoding command is
  gst-launch-1.0 -v filesrc location=videotestsrc.webm ! matroskademux ! vp9dec ! videoconvert ! videoscale ! autovideosink
  ```

- **encoded as VP8 (encapsulated in WebM)**

  ```
  gst-launch-1.0 -v videotestsrc num-buffers=1000 ! spacemitvp8enc ! webmmux ! filesink location=videotestsrc.webm
  //corresponding decoding command is
  gst-launch-1.0 -v filesrc location=videotestsrc.webm ! matroskademux ! vp8dec ! videoconvert ! videoscale ! autovideosink
  ```

- **encoded as MJPEG**

  ```
  gst-launch-1.0 videotestsrc num-buffers=100 ! 'video/x-raw,format=NV12, width=1280, height=720, framerate=25/1' ! spacemitmjpegenc ! filesink location=test.mjpeg
  ```

#### Mux/Demux Application Scenarios

##### Mux plugins(encapsulate stream into file)

- **qtmux**
  encapsulates camera JPEG strem into `.mov` file:

   ```
   gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=600  ! "image/jpeg,framerate=30/1,width=640,height=480" ! qtmux ! filesink location=video.mov
   ```

- **matroskamux**  
  encapsulates MP3 audio into `.mkv` file:

   ```
   gst-launch-1.0 filesrc location=test.mp3 ! mpegaudioparse ! matroskamux ! filesink location=test.mkv
   ```

- **mp4mux**  
  encodes camera video as H.264 and encapsulates it into `.mp4`:

   ```
   gst-launch-1.0 v4l2src num-buffers=50 ! queue ! x264enc ! mp4mux ! filesink location=video.mp4
   ```

- **flvmux**  
  merge the audio and video into a `.flv` file:

   ```
   gst-launch-1.0 filesrc location=/root/K001-MPEG-16bit-44.1kHz-CBR-192kbps-stereo.mp3 ! decodebin ! queue !  flvmux name=mux ! filesink location=test.flv  filesrc location=../mp4/480p.mp4 ! decodebin ! queue ! mux.
  ```

- **avimux**  
  generate a test video in `.avi` format:  

   ```
   gst-launch-1.0 videotestsrc num-buffers=100 ! 'video/x-raw,format=I420,width=640,height=480,framerate=30/1' ! avimux ! filesink location=test.avi
   ```

##### Demux plugins

- qtdemux

   ```
   gst-launch-1.0 filesrc location=test.mov ! qtdemux name=demux  demux.audio_0 ! queue ! decodebin ! audioconvert ! audioresample ! autoaudiosink   demux.video_0 ! queue ! decodebin ! videoconvert ! videoscale ! autovideosink
   //If the video source contains only video, use the following command to perform demuxing:
   gst-launch-1.0 filesrc location=video.mov ! qtdemux name=demux   demux.video_0 ! queue ! decodebin ! videoconvert ! videoscale ! autovideosink
   ```

- matroskademux

   ```
   gst-launch-1.0 -v filesrc location=/path/to/mkv ! matroskademux ! vorbisdec ! audioconvert ! audioresample ! autoaudiosink
   ```

- flvdemux

   ```
   gst-launch-1.0 -v filesrc location=/path/to/flv ! flvdemux ! audioconvert ! autoaudiosink
   ```

- avidemux

   ```
  gst-launch-1.0 filesrc location=test.avi ! avidemux name=demux  demux.audio_00 ! decodebin ! audioconvert ! audioresample ! autoaudiosink   demux.video_00 ! queue ! decodebin ! videoconvert ! videoscale ! autovideosink
  ```

#### Audio Application Scenarios

This describes some basic pipelines for audio output using GStreamer.

- Audio Playback

  Audio playback refers to the process of playing a specific audio file according to its corresponding format. The pipeline below uses the `audiotestscr` plugin to output standard audio to the headphone jack.

  ```
  gst-launch-1.0 audiotestsrc wave=5 ! alsasink device=plughw:1  
  ```

- Audio decoding

  - play mp3 format file

     ```
     gst-launch-1.0 filesrc location=test.mp3 ! mpegaudioparse ! mpg123audiodec
     ! audioconvert ! audioresample ! autoaudiosink
     ```

  - play ogg vorbis format file

     ```
     gst-launch-1.0 -v filesrc location=test.ogg ! oggdemux ! vorbisdec ! audioconvert ! audioresample ! autoaudiosink
     ```

- Audio format conversion

   Audio conversion refers to the process of changing audio file format to another desired format, such as changing convert `.wav` to `.aac`.

   ```
   gst-launch-1.0 -v autoaudiosrc ! audioconvert ! vorbisenc ! oggmux ! filesink location=alsasrc.ogg
   ```

#### Image Application Scenarios

This describes some basic pipelines for image output using GStreamer.

- Image output

   Image output includes all processes of displaying the desired image file on the required screen or any other type of output source.

- Display PNG image

   ```
   gst-launch-1.0 -v filesrc location=some.png ! decodebin ! videoconvert ! imagefreeze ! autovideosink
   ```

- Display JPEG image

   ```
   gst-launch-1.0 -v filesrc location=<output_image>.jpeg ! jpegdec ! imagefreeze ! videoconvert ! autovideosink
   ```

- Image capture
  For image capture, image can be acquisited from the camera.

  - JPG format

    ```
    gst-launch-1.0 v4l2src num-buffers=1 ! jpegenc ! filesink location=capture.jpg  
    ```

  - PNG format

    ```
    gst-launch-1.0 v4l2src num-buffers=1 ! pngenc ! filesink location=capture.png  
    ```

  - JPEG format

    ```
    gst-launch-1.0 v4l2src num-buffers=1 ! jpegenc ! filesink location=capture.jpeg  
    ```

#### Transcoding Application Scenarios

This describes how to configure and run basic transcoding pipelines.

- **Video transcoding**
  Transcodes MJPEG data from camera output into an MKV file:

  ```
  gst-launch-1.0 v4l2src device=/dev/video20 ! jpegparse ! spacemitdec ! queue ! videoconvert ! spacemith264enc ! h264parse ! matroskamux ! filesink location=out.mkv
  ```

#### Video Streaming Scenarios

**Rtsp**

1. Download source code  
  Visit [https://github.com/GStreamer/gst-rtsp-server](https://github.com/GStreamer/gst-rtsp-server)，and switch to the 1.18 branch.

2. Compilation installation

3. Start the RTSP server
   Run the following command on the server side to start the RTSP server and publish the video stream:

   ```
   ./test-launch "( spacemitsrc location=/usr/share/camera_json/csi1_camera_auto.json close-dmabuf=0 ! spacemith264enc ! rtph264pay name=pay0 pt=96 )"
   ```

4. Connect the client to the RTSP server to start video playback

#### Video Composition Scenarios

- **Multichannel data mixing and output**

   ```
   gst-launch-1.0 videotestsrc ! video/x-raw,width=1280,height=720 ! tee name=testsrc ! queue ! compositor name=comp sink_0::xpos=0 sink_0::ypos=0 \
   sink_1::xpos=100 sink_1::ypos=100 sink_1::width=200 sink_1::height=200 \
   sink_2::xpos=300 sink_2::ypos=300 sink_2::width=100 sink_2::height=200 \
   sink_3::xpos=400 sink_3::ypos=600 sink_3::width=100 sink_3::height=100 ! videoconvert ! autovideosink testsrc. ! queue ! comp.sink_1 testsrc. ! queue ! comp.sink_2 testsrc. ! queue ! comp.sink_3
   ```

- **Dual-camera mixing and output**

   ```
   gst-launch-1.0 -v compositor name=comp sink_0::xpos=0 sink_0::ypos=0 sink_0::width=640 sink_0::height=480 sink_1::xpos=0 sink_1::ypos=480 sink_1::width=640 sink_1::height=480 ! autovideosink v4l2src device=/dev/video20 ! video/x-raw,width=640,height=480 ! comp.sink_0  v4l2src device=/dev/video22 ! video/x-raw,width=640,height=480 ! comp.sink_1
   ```

## Gstreamer Debugging Method

This introduces common GStreamer debugging tools and thier usage scenarios.

### Use GStreamer Logging system

When a pipeline encounters errors or behaves unexpectedly, GStreamer's built-in logging system is the primary tool for debugging. By analyzing key information in the logs, you can quickly locate issues.

- `GST_DEBUG`

   GStreamer framework and its plugins provide different levels of log information, which includes timestampes, process IDs, thread IDs, types, source code line numbers, function names, Element information and corresponding log messages. For example:

   ```
   $ GST_DEBUG=2 gst-launch-1.0 playbin uri=file:///x.mp3Setting pipeline to PAUSED ...
   0:00:00.014898047 47333      0x2159d80 WARN                 filesrc gstfilesrc.c:530:gst_file_src_start:<source> error: No such file "/x.mp3"
   ...
   ```

   The corresponding log information can be retrieved by simply setting the `GST_DEBUG` environment variable and specifying the desired log level when running. Since GStreamer logs are highly detailed, enabling full logging can impact system performance. Therefore, the system provides eight distinct log levels to output information at varying degrees of detail as needed.

   - Level 0：No log information is output
   - Level 1：ERROR information
   - Level 2：WARNING information
   - Level 3：FIXME information
   - Level 4：INFO information
   - Level 5：DEBUG information
   - Level 6：LOG information
   - Level 7：TRACE information
   - Level 8：MEMDUMP information, the highest log level

   When in use, only set `GST_DEBUG` to specified level, all log messages at or below this level will be output. For example: `GST_DEBUG=2` will display logs of ERROR level and WAENING level.

   The above settings apply when all modules use the same level. If you need to set level for specified plugins individually, use the format of  **module name:level**. For example:
   `GST_DEBUG=2,audiotestsrc:6` indicates that the global level is set to 2, and only the log level of the audiotestsrc element is set to 6.

   In this case, the value of GST\_DEBUG consists of "module name:level" key-value pairs separated by commas. You can add a default log level for all unspecified modules at the very beginning, and multiple module names can be separated by commas. Additionally, the value of GST\_DEBUG also supports the "\*" wildcard character.

   The value of `GST_DEBUG` consists of "module nameC:Level" seperated by commas, and supports the following feratures:

   - A default level can be set at the beginning (unspecified modules will use this level);
   - Supports seperate settings dor multiple modules;
   - Supports `*` wildcard for fuzzy matching.

   Example:
   `GST_DEBUG=2,audio*:6` — All modules starting with `audio` use level 6, and all other modules use level 2.

   Equivalent syntax:  
   `GST_DEBUG=*:2` has the same effect as `GST_DEBUG=2`, meaning all modules use level 2.

- `GST_DEBUG_FILE`

   In actual debugging, to facilitate subsequent analysis, log output is usually saved to a file. You can specify the log file path by setting the `GST_DEBUG_FILE` environment variable, and GStreamer will automatically write debug information to this file.

   ```
   GST_DEBUG=2 GST_DEBUG_FILE=pipeline.log GST_DEBUG=5 gst-launch-1.0 audiotestsrc ! autoaudiosink
   ```

### Use Graphviz Tools

When a pipeline structure becomes complex, it is essential to verify if it is running as expected and to identify which Elements are being utilized—especially when using `playbin` or `uridecodebin`. To facilitate this, GStreamer provides a feature that exports the current state of all elements and their interconnections into a `.dot` file. This file can then be converted into an image using tools such as Graphviz.

To obtain `.dot` files, simply set the `GST_DEBUG_DUMP_DOT_DIR` environment variable to specify the output directory. `gst-launch-1.0` will generate a `.dot` file for each state. For example, running the following command, we can obtain the Pipeline generated when using `playbin` to play a network file:

```
$ GST_DEBUG_DUMP_DOT_DIR=. gst-launch-1.0 playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm
$ ls *.dot
0.00.00.013715494-gst-launch.NULL_READY.dot    
0.00.00.170999259-gst-launch.PAUSED_PLAYING.dot  
0.00.07.642049256-gst-launch.PAUSED_READY.dot
0.00.00.162033239-gst-launch.READY_PAUSED.dot  
0.00.07.606477348-gst-launch.PLAYING_PAUSED.dot

$ apt-get install graphviz  
$ dot 0.00.00.170999259-gst-launch.PAUSED_PLAYING.dot -Tpng -o play.png
```

Generated `play.png` is as follows (results vary according to the plugins installed):

![](static/Ney4bGKADoSCj3xVK9EcSoCvnWt.png)

**Note:** In custom applications, setting only the `GST_DEBUG_DUMP_DOT_DIR` environment variable is insufficient. To generate .dot files, you must actively call the `GST_DEBUG_BIN_TO_DOT_FILE()` or `GST_DEBUG_BIN_TO_DOT_FILE_WITH_TS()` functions in the code to output the structural information of the Pipeline.

### Other Debugging Methods

There are some common methods for debugging GStreamer display and decoding issues in some specific embedded environments.

1. **Run the preview by specifying the Wayland display environment**

   - **Buildroot**
     Executes the command via the serial port to enable preview.
     You can add `WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/root/` before the GStreamer command. For example:

     ```
     WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/root/  gst-launch-1.0 spacemitsrc l
     ocation=k1-x_MUSE-Paper_sensor0_gc08a8.json ! waylandsink sync=0 render-rectangl
     e="<0,0,1280,720>"
     ```

   - **Bianbu OS**
     Executes the command via the serial port to enable preview.
     You need to log into the desktop first, then add `WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/1000` before the GStremer command. For example:

     ```
     WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/1000 gst-launch-1.0 filesrc location=/root/3840x2160_24bits_30fps_266p.h265 ! h265parse ! spacemitdec ! fpsdisplaysink video-sink='glsinkbin sink='gtkglsink sync=0''
     ```

2. **Troubleshooting decoding issues: rule out source stream abnomalies**

   When decoding fails with dedicated hardware decoder plugins such as `spacemitdec`, you should first confirm whether the source stream (raw stream) itself is problematic. It is recommended to troubleshoot in the following order:
   - Use a general GStreamer plugin to replace the SpacemiT decoder plugin for debugging.
   - Use `ffplay` to decade the source stream and check for issues.
   - Use the built-in test tool of MPP to decode the source stream and check for issues.
    Refer to [SpacemiT MPP](./mpp/02-MPP.md)  to perform decoding tests using test tools it provides.
