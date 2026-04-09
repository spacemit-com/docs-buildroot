---
sidebar_position: 12
---

# 常见问题

## 组件

### 制作一个Linux发行版需要集成哪些组件？

制作一个Linux发行版，至少需要集成以下组件，系统才能运行到命令行。

- [opensbi](https://git.spacemit.com/buildroot-k3/opensbi)
- [uboot-2022.10](https://git.spacemit.com/buildroot-k3/uboot)
- [linux-6.18](https://git.spacemit.com/buildroot-k3/linux-6.18)
- esos

其中，`esos`是RCPU (Real-Time CPU)的firmware，负责部分硬件模块初始化，以及HDMI Audio中断转发，被Linux内核依赖，缺失会无法启动。发布在[esos](https://git.spacemit.com/buildroot-k3/esos)和[esos-lite](https://git.spacemit.com/buildroot-k3/esos-lite)仓库，编译esos后会生成`rt24_os0_rcpu.elf`和`rt24_os1_rcpu.elf`文件，需要安装到initramfs的`/lib/firmware`目录。

支持GPU，需要集成以下组件：

- [img-gpu-powervr](https://git.spacemit.com/buildroot-k3/img-gpu-powervr)
- [mesa](https://git.spacemit.com/buildroot-k3/mesa)

支持视频硬件加速，需要集成以下组件：

- [k3x-vpu-firmware](https://git.spacemit.com/buildroot-k3/k3x-vpu-firmware)
- [mpp](https://git.spacemit.com/buildroot-k3/mpp)
- FFmpeg
- GStreamer

其中，FFmpeg和GStreamer的补丁在[buildroot](https://git.spacemit.com/buildroot-k3/buildroot)仓库，路径分别是`package/ffmpeg`和`package/gstreamer1`。
