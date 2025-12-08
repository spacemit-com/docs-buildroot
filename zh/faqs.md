---
sidebar_position: 14
---

# 常见问题

## 组件

### 制作一个Linux发行版需要集成哪些组件？

制作一个Linux发行版，至少需要集成以下组件，系统才能运行到命令行。

- [opensbi](https://gitee.com/spacemit-buildroot/opensbi)
- [uboot-2022.10](https://gitee.com/spacemit-buildroot/uboot-2022.10)
- [linux-6.1](https://gitee.com/spacemit-buildroot/linux-6.1)或[linux-6.6](https://gitee.com/spacemit-buildroot/linux-6.6)
- esos.elf

其中，`esos.elf`是RCPU (Real-Time CPU)的firmware，负责部分硬件模块初始化，以及HDMI Audio中断转发，被Linux内核依赖，缺失会无法启动。发布在[buildroot-ext](https://gitee.com/spacemit-buildroot/buildroot-ext)仓库，路径`board/spacemit/k1/target_overlay/lib/firmware/esos.elf`，需要安装到initramfs `/lib/firmware`目录。

支持GPU，需要集成以下组件：

- [img-gpu-powervr](https://gitee.com/spacemit-buildroot/img-gpu-powervr)
- [mesa3d](https://gitee.com/spacemit-buildroot/mesa3d)

支持视频硬件加速，需要集成以下组件：

- [k1x-vpu-firmware](https://gitee.com/spacemit-buildroot/k1x-vpu-firmware)
- [mpp](https://gitee.com/spacemit-buildroot/mpp)
- FFmpeg
- GStreamer

其中，FFmpeg和GStreamer的补丁在[buildroot](https://gitee.com/spacemit-buildroot/buildroot)仓库，路径分别是`package/ffmpeg`和`package/gstreamer1`。