# Linux DMA-BUF 调试指南

## 简介

DMA-BUF  是 Linux 内核为解决一个核心问题而设计的基础设施：如何在不同设备驱动、子系统之间高效、安全地共享大量内存缓冲区。DMA-BUF 出现之前，每个需要共享内存的驱动（如图形驱动、视频编解码器、摄像头驱动等）都有自己私有的共享机制，这导致了大量的重复开发和兼容性问题。

以 drm_gem 举例，DMA-BUF 整体结构如下：

![DMA-BUF 架构示意图](static/dma_buf_arch.png)

## 核心机制

DMA-BUF 框架通过提供一套标准的 API 和对象，将缓冲区的“所有权”和“使用权”分离，从而建立了一个统一的共享模型。

- 文件描述符抽象：DMA-BUF 将物理缓冲区与文件描述符（fd）绑定，通过 fd 实现跨进程/设备的共享传递。
- 角色分离架构：
  - Exporter（生产者）：负责分配内存并实现 `dma_buf_ops` 操作集，包括缓冲区映射（`map_dma_buf`）、缓存同步（`begin_cpu_access`）等接口。
  - Importer（消费者）：通过 `dma_buf_attach` 和 `dma_buf_map_attachment` 获取缓冲区的设备地址（如 IOVA），支持直接硬件访问。
- 同步机制：通过 `dma-fence` 和 `dma-resv` 实现异步操作同步，确保多设备访问时的数据一致性。

## 使用方法

### 内核配置

```ini
CONFIG_DMA_SHARED_BUFFER=y
```

### 导出者

```c
 static struct dma_buf *jpu_exp_alloc(struct jpu_alloc_dma_buf *alloc_data)
  {
      // 计算所需页面数
      size = alloc_data->size;
      npages = PAGE_ALIGN(size) / PAGE_SIZE;

      // 分配数据结构
      data = kmalloc(sizeof(*data) + npages * sizeof(struct page *), GFP_KERNEL);

      // 分配物理页面
      for (i = 0; i < npages; i++) {
          data->pages[i] = alloc_page(GFP_KERNEL);
      }

      // 配置导出信息
      exp_info.ops = &jpu_dmabuf_ops;
      exp_info.size = npages * PAGE_SIZE;
      exp_info.priv = data;

      // 导出dma-buf
      dmabuf = dma_buf_export(&exp_info);
      return dmabuf;
  }
```

上述示例实现了 JPU 的内存分配逻辑，根据请求大小分配相应数量的物理页面。分配完成后，调用 `dma_buf_export` 将这些页面封装成 dma-buf 对象并导出给系统。

### 导入者

```c
  static int get_addr_from_dmabuf(struct v2d_info *v2dinfo, 
                                 struct v2d_dma_buf_info *pInfo, 
                                 dma_addr_t *paddr)
  {
      // 附加dma-buf到设备
      pInfo->attach = dma_buf_attach(pInfo->dmabuf, dev);

      // 映射附件
      pInfo->sgtable = dma_buf_map_attachment_unlocked(pInfo->attach, DMA_BIDIRECTIONAL);

      if (sgt->nents == 1) {
          addr = sg_dma_address(sgt->sgl);  // 连续内存
      } else {
          // 多段内存使用IOMMU
          addr = V2D_TBU_BASE_VA + (dma_addr_t)(pInfo->tbu_id)*V2D_TBU_VA_STEP;
          size = v2d_iommu_map_sg(addr, sgt->sgl, sgt->orig_nents, flags);
      }

      *paddr = addr;
      return 0;
  }
```

V2D 接收一个 dma-buf，并将其“附加（attach）”到自己的设备上，然后“映射（map）”以获得可供其硬件访问的物理地址（或 IOMMU 地址）。

### 交互逻辑

![DMA-BUF 交互流程图](static/dma_buf_interaction.png)

dma-buf 的核心是通过一个文件描述符（fd）在不同驱动间实现“零拷贝”内存共享。

- **导出与传递**: JPU（生产者）分配内存，并导出一个 fd 给用户程序。程序再将此 fd 传递给 V2D（消费者）。
- **导入与使用**: V2D 使用该 fd 找到并映射（map）内存，以获取其硬件可直接访问的物理地址（或 IOMMU 生成的 IOVA），从而直接处理数据。
- **释放**: 当 V2D 和用户程序都释放了对 fd 的引用后，JPU 会自动回调并释放最初分配的内存。

## 调试方法

### /sys/kernel/dmabuf/buffers/

#### 使用方法

开启 `CONFIG_DMABUF_SYSFS_STATS` 后，会在 `/sys/kernel/dmabuf/buffers/` 创建接口。

- 每个子目录中包含两个文件： 
  - `exporter_name`: 记录导出这个 `dma-buf` 的驱动名称（例如 `dma-buf-heap` 或 `i915`）。
  - `size`: 记录这个 `dma-buf` 的大小（以字节为单位）。
- 当 `dma-buf` 被销毁时，对应的目录也会被自动删除。

#### 问题定位

- 可以通过观察 `/sys/kernel/dmabuf/buffers/` 目录下的条目数量来确认是否存在 `dma-buf` 泄漏。
- 统计每个导出者分别创建了多少个 dma-buf，找出泄漏的源头。

### /sys/kernel/debug/dma_buf/bufinfo

#### 输出示例

```text
size            flags           mode            count           exp_name        ino             name
08847360        00000002        00080007        00000003        drm                00000089        1769-gnome-shell
        write fence:drm_sched gfx seq 12146 signalled
        Attached Devices:
Total 0 devices attached
```

- size: 缓冲区的大小；
- flags: 文件描述符的标志（`00000002` 对应 `O_RDWR`，表示该缓冲区以读写模式打开）；
- mode: 文件的访问权限模式；
- count: 缓冲区的引用计数；
- exp_name: 导出该 DMA-BUF 的驱动名称；
- ino: 文件的 inode 号；
- name: 用户空间为该缓冲区设置的名称；
- `write fence: ... signalled` 表示一个写 fence，`signalled` 说明对应的异步操作（如 GPU 渲染）已完成；若长期未完成（unsignalled），可能存在 GPU hang。
- `Attached Devices:` 列出所有附加到该 DMA-BUF 的设备。示例中为 0。

#### 问题定位

- 调试系统卡死或应用冻结：当显示缓冲区上的 write fence 长时间处于 unsignalled 状态，通常意味着 GPU hang。
