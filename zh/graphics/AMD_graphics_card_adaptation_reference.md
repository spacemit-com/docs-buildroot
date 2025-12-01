# K1 平台 AMD 显卡适配指南

本文档为在 **K1 平台** 上适配 AMD 显卡的开发者提供参考。
以 **Radeon HD7350** 为例，介绍适配过程中对 Linux 内核的主要修改，并解释原因。本指南也适用于同代及其他 AMD GPU 的适配。

## 背景介绍

Linux 内核中的独立显卡驱动依托于 **DRM（Direct Rendering Manager）子系统**。
DRM 是一个用于管理 GPU（图形处理单元）资源的内核模块，提供统一的接口，以支持不同厂商的显卡。
它的整体结构可分为以下几个层次：

- **用户空间接口**
  通过 `ioctl` 等接口与 **用户空间的图形库**（如 **Mesa、X.Org、Wayland**）进行交互，**应用程序** 可通过这些库访问 GPU。
  交互流程：应用 → 图形库 → DRM `ioctl` → DRM 驱动 → GPU

- **DRM 核心层（DRM Core）**
  提供统一的框架和通用功能，包括显卡设备初始化、资源分配、内存管理、任务调度与同步、以及用户空间和内核空间的通信接口。

  - **GEM（Graphics Execution Manager）**
    - 提供图形内存对象的管理
    - 支持显存的高效分配与共享
    - 允许用户空间应用创建和管理图形缓冲区，实现高效渲染

  - **TTM（Translation Table Maps）**
    - 负责显存与系统内存的映射和管理，可作为 GEM 的后端
    - 提供在内存压力下的资源调度与分配策略

  - **调度与同步（Command Scheduling）**
    - 管理 GPU 多任务的调度
    - 使用 Fence 等机制协调内核与用户空间操作的顺序, 避免资源冲突

  - **内存管理（VRAM Management）**
    - 负责显存分配、映射与回收
    - 支持 **零拷贝（Zero-Copy）** 技术，减少内核与用户空间之间的数据传输开销

- **硬件特定驱动模块（Driver Modules）**
  - 负责不同厂商和型号的 GPU 的硬件控制
  - 典型驱动：`nouveau`（NVIDIA 开源驱动）、`amdgpu` 和 `radeon`（AMD驱动）

radeon 驱动结构框图：

![](static/radeon-driver.png)

## 配置与修改

在将 AMD 显卡适配至 K1 平台时，需要对 **显卡驱动、设备树 (Device Tree)、地址映射、缓存策略、DMA 位宽以及中断管理** 等进行修改与调整。
对于 **RISC-V 架构平台**，这些调整应在充分理解 **内存管理机制** 和 **PCIe 地址空间映射** 的前提下进行。

本参考文档将从以下几个方面展开：

1. 设备树解析与地址映射
2. 添加 write-combining 支持
3. 缓存属性与访问权限的调整
4. DMA 地址宽度适配
5. MSI 中断支持限制
6. Linux 内核显卡驱动配置。

### 设备树与地址映射

**设备树解析：**

- 在设备树中，`ranges` 属性定义了子设备地址空间与父设备地址空间的映射关系。
- 在 PCIe 主控节点中，`ranges` 通常包含四个部分：
  `<地址属性, PCIe地址, CPU地址, 地址长度>`

示例修改如下：

``` c
diff --git a/arch/riscv/boot/dts/spacemit/k1-x.dtsi b/arch/riscv/boot/dts/spacemit/k1-x.dtsi
index 7ea166ca6fb0..fd978a379c37 100644
--- a/arch/riscv/boot/dts/spacemit/k1-x.dtsi
+++ b/arch/riscv/boot/dts/spacemit/k1-x.dtsi
@@ -2198,7 +2198,8 @@ pcie2_rc: pcie@ca800000 {
             #address-cells = <3>;
             #size-cells = <2>;
             ranges = <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>,
-                 <0x02000000 0x0 0xa0000000 0 0xa0000000 0x0 0x17000000>;
+                 <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>,
+                 <0x02000000 0x0 0xb0000000 0 0xb0000000 0x0 0x7000000>; 
             interconnects = <&dram_range2>;
             interconnect-names = "dma-mem";
```

- **改动前**：

  - 源地址 `0xa0000000` → 目标地址 `0xa0000000`
  - 大小 `0x17000000` (368 MB)，属性为 **MEM**

- **改动后**：

  - 第一段：源地址 `0xa0000000` → 目标地址 `0xa0000000`，大小 `0x10000000` (256 MB)，属性为 **MEM + 预取**
  - 第二段：源地址 `0xb0000000` → 目标地址 `0xb0000000`，大小 `0x7000000` (112 MB)，属性为 **MEM**

此调整将较大显存区间分段，其中一部分设置为预取，提高访问性能。

**提示**

- K1 平台上 **pcie2 可用地址空间总计 384MB**，包括配置空间和 BAR 空间。
- 修改设备树时，应参考现有 GPU 的 BAR 地址范围，并适当划分映射区域。

### 添加 wc(write-combining) 支持

```c
diff --git a/arch/riscv/include/asm/pci.h b/arch/riscv/include/asm/pci.h
index cc2a184cfc2e..9f6f59aff214 100644
--- a/arch/riscv/include/asm/pci.h
+++ b/arch/riscv/include/asm/pci.h
@@ -27,6 +27,10 @@ static inline int pcibus_to_node(struct pci_bus *bus)
 #endif
 #endif /* defined(CONFIG_PCI) && defined(CONFIG_NUMA) */
 
+#if defined(CONFIG_SOC_SPACEMIT_K1X)
+#define arch_can_pci_mmap_wc() 1
+#endif
+
 /* Generic PCI */
 #include <asm-generic/pci.h>
```

- 在 K1 平台 PCI 地址映射中启用 **write-combining** (WC)。
- WC 模式生效需要资源具备 **IORESOURCE\_PREFETCH** 标志。
- 结合设备树修改，可在 PCI 资源映射到用户空间时启用 WC，从而提升显存访问效率。

### 缓存属性与访问权限设置

在 AMD 显卡适配中，**缓存策略** 会直接影响显存的访问效率和系统稳定性。默认情况下，驱动会根据 `rbo->flags` 在多种模式之间选择，例如：

- **uncached**（不缓存）
- **cached**（普通缓存）
- **write-combined (WC)**（写合并）

然而，默认策略并不总是最适合 GPU 的显存访问模式，因此需要进行修改。

**修改示例**

在 `radeon_ttm_tt_create` 函数中，强制使用 **write-combined (WC)**：

```c
diff --git a/drivers/gpu/drm/radeon/radeon_ttm.c b/drivers/gpu/drm/radeon/radeon_ttm.c
index 4eb83ccc4906..4693119e2412 100644
--- a/drivers/gpu/drm/radeon/radeon_ttm.c
+++ b/drivers/gpu/drm/radeon/radeon_ttm.c
@@ -512,6 +512,8 @@ static struct ttm_tt *radeon_ttm_tt_create(struct ttm_buffer_object *bo,
     else
         caching = ttm_cached;
 
+    caching = ttm_write_combined;
     if (ttm_sg_tt_init(&gtt->ttm, bo, page_flags, caching)) {
         kfree(gtt);
         return NULL;
```

**扩展 RISC-V 架构支持**

在 TTM 模块中增加对 **RISC-V** 的支持，使 WC 模式能正确生效：

```c
diff --git a/drivers/gpu/drm/ttm/ttm_module.c b/drivers/gpu/drm/ttm/ttm_module.c
index b3fffe7b5062..1319178edf03 100644
--- a/drivers/gpu/drm/ttm/ttm_module.c
+++ b/drivers/gpu/drm/ttm/ttm_module.c
@@ -74,7 +74,8 @@ pgprot_t ttm_prot_from_caching(enum ttm_caching caching, pgprot_t tmp)
 #endif /* CONFIG_UML */
 #endif /* __i386__ || __x86_64__ */
 #if defined(__ia64__) || defined(__arm__) || defined(__aarch64__) || \
-    defined(__powerpc__) || defined(__mips__) || defined(__loongarch__)
+    defined(__powerpc__) || defined(__mips__) || defined(__loongarch__) \
+    || defined(__riscv)
     if (caching == ttm_write_combined)
         tmp = pgprot_writecombine(tmp);
     else
```

**说明**：

- 在 RISC-V 平台下，当缓存模式为 **write-combined** 时 (即 `caching = ttm_write_combined`)，PTE（页表项）会正确设置为 **非缓存或写合并属性**。
- 这样可以避免显存访问时的缓存不一致问题，提升渲染和数据传输效率。

`ttm_prot_from_caching` 函数中引入 RISC-V 架构支持。当 caching = ttm_write_combined 时，会将相应页表属性设置为写合并模式。在 RISC-V 平台下，当缓存属性为写合并时，对应 PTE 能正确设置为非缓存或写合并的属性。

### DMA 地址宽度 (`dma_bits`) 调整

在驱动初始化中，`dma_bits` 用于指定 GPU 可寻址的物理内存空间大小。

```c
diff --git a/drivers/gpu/drm/radeon/radeon_device.c b/drivers/gpu/drm/radeon/radeon_device.c
index afbb3a80c0c6..42e6510eccf0 100644
--- a/drivers/gpu/drm/radeon/radeon_device.c
+++ b/drivers/gpu/drm/radeon/radeon_device.c
@@ -1361,7 +1361,7 @@ int radeon_device_init(struct radeon_device *rdev,
      * AGP - generally dma32 is safest
      * PCI - dma32 for legacy pci gart, 40 bits on newer asics
      */
-    dma_bits = 40;
+    dma_bits = 32;
     if (rdev->flags & RADEON_IS_AGP)
         dma_bits = 32;
     if ((rdev->flags & RADEON_IS_PCI) &&
```

- 在驱动初始化时，原代码默认 `dma_bits = 40`，表示显卡最多可以访问 **1TB** 内存空间（40 位寻址）。
- 但在实际适配 K1 平台时，为提升稳定性，改为 `dma_bits = 32`，限制访问范围在 **4GB**（32 位寻址）以内。

此处的修改提升了 K1 平台在加载 Radeon 驱动时的兼容性和稳定性，建议在调试过程中重点关注相关地址访问行为。

> **注意：**
对于其他 AMD 显卡，如若 GPU 本身不需要访问过大的物理内存空间，则可相应调整 `dma_bits` 来确保设备正常工作。

### MSI 中断适配

在 K1 平台上，部分旧架构的 AMD GPU（尤其是 **GCN1 及更早架构**）使用 **MSI（Message Signaled Interrupts）** 可能导致中断无法正确触发。
为保证系统稳定性，可在驱动中禁用 MSI，退回到传统的线性中断方式。

```c
diff --git a/drivers/gpu/drm/radeon/radeon_irq_kms.c b/drivers/gpu/drm/radeon/radeon_irq_kms.c
index c4dda908666c..f540529909d3 100644
--- a/drivers/gpu/drm/radeon/radeon_irq_kms.c
+++ b/drivers/gpu/drm/radeon/radeon_irq_kms.c
@@ -244,6 +244,10 @@ static bool radeon_msi_ok(struct radeon_device *rdev)
     /* MSIs don't work on AGP */
     if (rdev->flags & RADEON_IS_AGP)
         return false;
+#if IS_ENABLED(CONFIG_SOC_SPACEMIT_K1X)
+       /* Chips <= GCN1 cannot get MSI to work on K1 */
+       return false;
+#endif
 
     /*
      * Older chips have a HW limitation, they can only generate 40 bits
```

**提示：**

- 如果 GPU 无法触发中断，或插入 GPU 后系统中断异常，可尝试禁用 MSI。
- 仅在必要时禁用 MSI：
  - **禁用 MSI** → 稳定性更高，但性能和延迟可能略受影响。
  - **启用 MSI** → 性能和延迟更优，但在 K1 平台上可能不稳定。

### 打开显卡驱动配置

``` c
diff --git a/arch/riscv/configs/k1_defconfig b/arch/riscv/configs/k1_defconfig
index 1e14a4a5c4f7..58dfdf05bfce 100644
--- a/arch/riscv/configs/k1_defconfig
+++ b/arch/riscv/configs/k1_defconfig
@@ -881,6 +881,8 @@ CONFIG_SPACEMIT_K1X_SENSOR_V2=y
 # CONFIG_DVB_CXD2099 is not set
 # CONFIG_DVB_SP2 is not set
 # CONFIG_DRM_DEBUG_MODESET_LOCK is not set
+CONFIG_DRM_RADEON=m
+CONFIG_DRM_RADEON_USERPTR=y
 CONFIG_DRM_SPACEMIT=y
 CONFIG_SPACEMIT_MIPI_PANEL=y
 CONFIG_SPACEMIT_HDMI=y
```

在内核配置文件 `k1_defconfig` 中开启 Radeon 驱动支持：

```c
+CONFIG_DRM_RADEON=m
+CONFIG_DRM_RADEON_USERPTR=y
```

这两项配置可在 `make menuconfig` 中选择，方式如下：

- `Y` → 内置到内核（build-in）
- `M` → 编译为模块（module）

目前，在 K1 平台上：

- **radeon 驱动** 已支持 **GCN 1.0 之前架构** 的 AMD 显卡；
- 对于 **GCN 1.0 及之后架构**（如 Radeon Rx 系列），则需要适配并开启 **amdgpu 驱动**。

>**注意：** 修改或适配驱动代码后，必须确保对应的内核配置已打开。
