sidebar_position: 1

# Multimedia Framework

This document introduces the K3 multimedia framework and the function of each layer, including applications, frameworks, MPP, and drivers, as well as the roles of commonly used hardware modules.

## Framework Hierarchy Diagram

![](static/K3X-SDK.png)

The multimedia system includes four layers:

### 1. APP

This layer includes **third-party Apps** and **self-developed Apps**.

**Third-Party Apps**

- **mpv** (Bianbu default player)
  - Supports hardware decoding for multiple formats, including H.264 / HEVC / VP8 / VP9 / MPEG-4 / MPEG-2 / MJPEG.
  - Supports video playback up to **4K60**.

- **totem** (Ubuntu default player)
  - Supports hardware decoding for multiple formats, including H.264 / HEVC / VP8 / VP9 / MPEG-4 / MPEG-2 / MJPEG.
  - Supports video playback up to **4K60**.

- **snapshot** (Default camera application for Bianbu and Ubuntu desktop systems)
  - Supports preview, still image capture, video recording, and related functions.
  - Supports smooth **720P30** video recording.

- **chromium** (Bianbu default browser)
  - Supports hardware decoding for formats including H.264 / HEVC.
  - Supports video playback up to **4K60**.

**Self-Developed Apps**

This category includes API interaction (by SpacemiT) and test programs provided for function verification and reference development, such as:

- **v2d-test**
  - Tests the V2D module, including uncompressed image format conversion, rotation, and scaling.

- **mvx-player**
  - Tests the VPU module for video encoding and decoding. It is operated from the command line, and outputs are saved as files.
  
- **camera-test**
  - Tests the camera module in the CPP-ISP-MIPI CSI pipeline.
  - Applicable only to platform cameras and not to USB cameras. USB cameras can be tested by using `v4l-utils`.

### 2. Open-Source Multimedia Framework

GStreamer and FFmpeg are commonly used in this layer.
They are complete multimedia solutions that cover the major stages of video playback: **muxer / demuxer / decoder / encoder / display**.
Multiple plugins are implemented at this layer, and K3 hardware encoding and decoding libraries are integrated into these frameworks through **MPP**.

- **FFmpeg**
  - Already integrated with the K3 hardware codec.
  - **Decoding:** H.264 / HEVC / VP8 / VP9 / MPEG-4 / MPEG-2 / MJPEG, with support up to **4K120**.
  - **Output pixel format:** AV_PIX_FMT_DRM_PRIME, AV_PIX_FMT_NV12
  - **Encoding:** H.264 / H.265 / VP8 / VP9 / MJPEG, with support up to **4K60**.

- **GStreamer**
  - Already integrated with the K3 hardware codec.
  - **Decoding:** H.264 / HEVC / VP8 / VP9 / MPEG-4 / MPEG-2 / MJPEG, with support up to **4K120**.
  - **Encoding:** H.264 / H.265 / VP8 / VP9 / MJPEG, with support up to **4K60**.

- **OpenMAX IL**
  - Currently being adapted for encoding and decoding.

### 3. MPP (Multimedia Processing Platform)

- For the upper layer, it provides a unified multimedia API.
- For the lower layer, it dynamically loads codec library plugins for different platforms to invoke the codec libraries.

### 4. Driver & Library

- Provided by IP vendors.
- Includes hardware drivers and API dynamic libraries that directly operate the multimedia modules on the chip.

## Terms

- **VPU (Video Processing Unit)**
  - A hardware video processing unit for video encoding and decoding that reduces CPU load.
  - K3 is based on the V4L2 framework and supports decoding of H.264 / HEVC / VP8 / VP9 / MJPEG / MPEG-4 and encoding of H.264 / HEVC / VP8 / VP9 / MJPEG.

- **V2D**
  - A K3 image processing hardware module that supports functions such as format conversion, scaling, and cropping.

- **RVV**
  - Vector extension instruction set for the RISC‑V architecture, used to accelerate data-parallel computing, similar to ARM NEON.

- **MPP (Multimedia Processing Platform)**
  - A multimedia processing platform that interfaces hardware encoding and decoding with upper-layer frameworks.

- **GStreamer**
  - An open-source, flexible, and powerful multimedia framework for building streaming media applications and processing audio and video data.
  - It provides a set of libraries and tools for creating, processing, and playing various multimedia streams, including audio, video, and streaming media.
  - It supports multiple codecs and formats and can run on different platforms.

- **FFmpeg**
  - A cross-platform open-source audio and video processing tool that supports recording, conversion, streaming, and editing.
  - It supports a wide range of audio and video formats and codecs and can run on various operating systems, including Windows, macOS, and Linux.
  - It is widely used in multimedia processing.

- **V4L2 (Video for Linux 2)**
  - The Linux video capture and output device API.
  - It provides a unified access interface for devices such as cameras and video capture cards, facilitating video capture, processing, and display.

- **ALSA (Advanced Linux Sound Architecture)**
  - The mainstream audio architecture on Linux systems, widely used in various Linux distributions. It is a software architecture for processing audio and managing audio devices.
  - It provides a unified audio interface that allows applications to communicate with audio hardware, supports a wide range of audio devices and formats, and offers low-latency, high-quality audio processing.
  - It provides a set of tools and libraries for configuring and managing audio devices, as well as for developing audio applications.
