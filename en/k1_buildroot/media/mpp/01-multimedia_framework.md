sidebar_position: 1

# Multimedia Framework

This introduces K1 multimedia framework and the functions of each layer.
By reading this, you will understand the relationships among applications, frames, MPP and drivers, as well as the roles of commonly used hardware module.

## Framework Hierarchy Diagram and Description

![](static/M5mAbw911oDOp2xFEtHc2lNnned.png)

The multimedia system includes 4 layers. From top to bottom, they are:

### 1. APP

This layer includes **third-party APPs** and **self-developed APPs**.

**Third party APPs**

- **mpv** (Bianbu default player)
  - Supports hardware decoding of multiple formats including H.264/ HEVC/ VP8/ VP9/ MPEG-4/ MPEG-2/ MJPEG.
  - Supports video playback up to **4K60**
  
- **totem** (Ubuntu default player)
  - Supports hardware decoding of multiple formats including H.264/ HEVC/ VP8/ VP9/ MPEG-4/ MPEG-2/ MJPEG.
  - Supports video playback up to **4k30**

- **cheese** (Defualt camera for Bianbu/Ubuntu desktop system)
  - Supports preview, photographing, video recording and other functions.
  - Enables smooth 1080P30 video recording.

- **chromium** (Bianbu default browser)
  - Supports hardware decoding of multiple formats including H.264/HEVC.
  - Supports video playback up to **4K30**

- **kodi** (open-source player)
  - Supports hardware decoding of multiple formats including  H.264/HEVC/VP8/VP9.
  - Supports video playback up to 4K60.

**Self-developed Apps**

  They are mainly API interaction and test programs provided by us, used for function verification and reference development, such as:

- **v2d-test**
  - Tests V2D module (non-compressed image format conversion, rotation, scaling)
  
- **mvx-player**
  - Tests VPU module (video encoding and decoding), operated via command line, with outputs saved as files.

- **jpu-test**
  - Tests JPU module (JPEG image encoding and decoding), operated via command line, with outputs saved as files.

- **camera-test**
  - Tests CAMERA module (CPP-ISP-MIPICSI pipeline)
  - Only applicable for platform camera, excludes USB camera (Test USB camera via v4l-utils)

### 2. Open-Source Multimedia Framework

**GStreamer** and **FFmpeg** are commonly used in this layer.
  They are complete multimedia solutions that fully include various stages of video playback:
  **muxer**/ **demuxer**/ **decoder**/ **encoder**/ **display**.
  We have implemented multiple plugins at this layer, and integrated K1 hardware encoding and decoding libraries into these frameworks via **MPP**.

- **FFmpeg**
  - Already interfaced with K1 hardware codec
  - **Decoding**: H.264/ HEVC/ VP8/ VP9/ MPEG-4/ MPEG-2/ MJPEG, with a maximum support for **4K60**
  - **Output pixel format**:AV_PIX_FMT_DRM_PRIME, AV_PIX_FMT_NV12
  - **Encoding**: H.264/ H.265/ VP8/ VP9/ MJPEG, with a maximum support for **4K30**

- **Gstreamer**
  - Already interfaced with K1 hardware codec
  - **Decoding**: H.264/ HEVC/ VP8/ VP9/ MPEG-4/ MPEG-2/ MJPEG, with maximum supports for **4K30**
  - **Encoding**: H.264/ H.265/ VP8/ VP9/ MJPEG, with a maximum supports for **1080P60**.

- **Openmax IL**
  - Under adaptation for encoding and decoding

### 3. MPP (Multimedia Processing Platform)

- For upper layer: provides a unified multimedia API
- For lower layer: dynamically loads codec library plugins of different platforms to invoke the codec libraries

### 4. Driver & Library

- Provided by IP vendors
- Includes hardware drivers and API dynamic library, which directly operate the multimedia modules on the chip.

## Terms

- **VPU (Video Processing Unit)**
  - A video processing unit for video hardware encoding and decoding that reduces CPU load.
  - K1 VPU is implemented based on the standard V4L2 framework, supporting decoding of H.264/ HEVC/ VP8/ VP9/ MJPEG/ MPEG4 and encoding of H.264/ HEVC/ VP8/ VP9/ MJPEG.

- **V2D**
  - K1 image processing hardware module, supporting functions such as **format conversion**, **scaling**, and **cropping**.

- **JPU (JPEG Processing Unit)**
  - A hardware for JPEG encoding and decoding that improves efficiency and reduces CPU load.

- **ISP**
  - An image signal processing module that optimizes raw images from sensors and improves image quality.

- **CPP**
  - An image post-processing module used for offline processing of NV12 data output by ISP.
  - Pyramid-based multi-layer time-sharing processing. Main functions include: lens distortion correction, spatial and temporal denoising, frequency domain denoising, edge enhancement, etc.

- **RVV**
  - Vector extension instruction set for the RISC‑V architecture, used to accelerate data-parallel computing, similar to ARM NEON.

- **MPP (Multimedia Processing Platform)**
  - Multimedia processing platform for interfacing hardware encoding and decoding with upper-layer architectures.

- **Gstreamer**
  - An open-source, flexible and powerful multimedia framework for building streaming media applications and processing audio/video data, used to build streaming media applications and process audio/video data.
  - It provides a set of libraries and tools for creating, processing, and playing various multimedia streams, including audio, video, and streaming media.
  - Supports multiple codecs and formats, can run on different platforms.

- **FFmpeg**
  - A cross-platform open-source audio and video processing tool that supports recording, conversion, streaming and editing.
  - It supports a wide range of audio and video formats and codecs, and can run on various operating systems including Windows, Mac and Linux.
  - Widely applied in multimedia processing.

- **V4L2 (Video for Linux 2)**
  - Linux video capture/output device API
  - It provides a unified access interface for devices such as cameras and video capture cards, facilitating video capture, processing, and display.

- **ALSA （Advanced Linux Sound Architecture）**
- The mainstream audio architecture on Linux systems, widely used in various Linux distributions, is a software architecture for processing audio and managing audio devices.
- It provides a unified audio interface that allows applications to communicate with audio hardware. It supports a wide range of audio devices and formats, and offers low-latency, high-quality audio processing.
- It provides a set of tools and libraries for configuring and managing audio devices, as well as for developing audio applications.
