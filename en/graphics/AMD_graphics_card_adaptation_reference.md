# Adapting AMD Graphics Cards on the K1 Platform

This document provides guidance for developers adapting AMD graphics cards for the **K1 platform**, using the **Radeon HD7350** as an example. It outlines key modifications to the Linux kernel and explains the rationale behind each change.

## Background Introduction

The discrete GPU drivers in the Linux kernel rely on the **DRM (Direct Rendering Manager) subsystem**.  
DRM is a kernel module used to manage GPU (Graphics Processing Unit) resources, providing a unified interface to support graphics cards from different vendors.  
The overall DRM structure consists of the following layers:

- **User Space Interface**  
  Interacts with **user-space graphics libraries** (such as **Mesa, X.Org, Wayland**) via interfaces like `ioctl`.  
  **Applications** can access the GPU through these libraries.  
  Interaction flow: Application → Graphics Library → DRM `ioctl` → DRM Driver → GPU

- **DRM Core Layer**  
  Provides a standardized framework and common functionalities, including GPU device initialization, resource allocation, memory management, task scheduling and synchronization, as well as communication interfaces between user space and kernel space.

  - **GEM (Graphics Execution Manager)**
    - Manages graphics memory objects
    - Supports efficient allocation and sharing of video memory
    - Allows user-space applications to create and manage graphic buffers for efficient rendering

  - **TTM (Translation Table Maps)**
    - Responsible for mapping and managing video memory and system memory; can serve as the backend for GEM
    - Provides resource scheduling and allocation strategies under memory pressure

  - **Command Scheduling**
    - Manages GPU multitasking scheduling
    - Coordinates the order of operations between kernel space and user space using mechanisms like Fence to avoid resource conflicts

  - **VRAM Management**
    - Handles allocation, mapping, and recycling of video memory
    - Supports **Zero-Copy** technology to reduce data transfer overhead between kernel and user space

- **Hardware-Specific Driver Modules**
  - Responsible for hardware control of GPUs from different vendors and models
  - Typical drivers: `nouveau` (open-source NVIDIA driver), `amdgpu` and `radeon` (AMD drivers)


Structure diagram of the `radeon` driver:

![](static/radeon-driver.png)

## Configuration and Modifications

When adapting an AMD graphics card to the K1 platform, modifications and adjustments are required in areas such as the **graphics driver, Device Tree, address mapping, cache strategy, DMA bit width, and interrupt management**.  
For **RISC-V architecture platforms**, these adjustments should be made with a thorough understanding of the **memory management mechanism** and **PCIe address space mapping**.

This reference document will cover the following aspects:

1. Device Tree parsing and address mapping  
2. Adding write-combining support  
3. Cache attributes and access permission adjustments  
4. DMA address width adaptation  
5. MSI interrupt support limitations  
6. Linux kernel GPU driver configuration

### Device Tree and Address Mapping

**Device Tree Parsing:**

- In the Device Tree, the `ranges` property defines the mapping relationship between the address space of child devices and that of the parent device.
- In a PCIe host controller node, the `ranges` property typically contains four parts:  
  `<address properties, PCIe address, CPU address, address length>`

Example modification:

```c
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

- **Before the change**:

  - Source address `0xa0000000` → Destination address `0xa0000000`  
  - Size `0x17000000` (368 MB), attribute: **MEM**

- **After the change**:

  - 1st segment: Source address `0xa0000000` → Destination address `0xa0000000`, size `0x10000000` (256 MB), attribute: **MEM + Prefetchable**
  - 2nd segment: Source address `0xb0000000` → Destination address `0xb0000000`, size `0x7000000` (112 MB), attribute: **MEM**

This adjustment splits the large VRAM region, with part of it set as prefetchable to improve access performance.

**Note**

- The **total available address space for pcie2 on the K1 platform is 384MB**, including configuration space and BAR space.
- When modifying the Device Tree, refer to the BAR address range of the existing GPU and appropriately divide the mapped regions.

### Adding wc (write-combining) Support

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

- Enable **write-combining** (WC) in the PCI address mapping on the K1 platform.
- WC mode requires the resource to have the **IORESOURCE_PREFETCH** flag set.
- Together with Device Tree modifications, WC can be enabled when PCI resources are mapped to user space, improving VRAM access efficiency.

### Cache Attributes and Access Permission Settings

In AMD GPU adaptation, the **cache strategy** directly affects both VRAM access efficiency and system stability.  
By default, the driver selects from several caching modes based on `rbo->flags`, such as:

- **uncached** (no caching)  
- **cached** (normal caching)  
- **write-combined (WC)**

However, the default strategy is not always optimal for GPU VRAM access patterns, so modifications are necessary.

**Modification Example**

In the `radeon_ttm_tt_create` function, force the use of **write-combined (WC)**:

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

**Extend RISC-V Architecture Support**

Add support for **RISC-V** in the TTM module to ensure the WC mode works correctly:

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

**Explanation:**

- On the RISC-V platform, when the cache mode is **write-combined** (i.e., `caching = ttm_write_combined`), the PTE (Page Table Entry) is correctly set to **non-cached or write-combined attributes**.
- This avoids cache incoherence issues during VRAM access and improves rendering and data transfer efficiency.

RISC-V architecture support is introduced in the `ttm_prot_from_caching` function. When `caching = ttm_write_combined`, the corresponding page table attributes are set to write-combined mode. On RISC-V platforms, when the cache attribute is write-combined, the corresponding PTE is correctly set as non-cached or write-combined.

### DMA Address Width (`dma_bits`) Adjustment

During driver initialization, `dma_bits` is used to specify the size of the physical memory space addressable by the GPU.

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

- During driver initialization, the original code defaults to `dma_bits = 40`, meaning the GPU can access up to **1TB** of memory space (40-bit addressing).
- For adaptation on the K1 platform, the value is changed to `dma_bits = 32` to improve stability, limiting the accessible range to within **4GB** (32-bit addressing).

This modification improves compatibility and stability when loading the Radeon driver on the K1 platform. It is recommended to closely monitor related address access behavior during debugging.

> **Note:**  
For other AMD GPUs, if the GPU itself does not require accessing a large physical memory space, the `dma_bits` value can be adjusted accordingly to ensure proper device operation.

### MSI Interrupt Adaptation

On the K1 platform, some older AMD GPU architectures (especially **GCN1 and earlier**) may experience issues where **MSI (Message Signaled Interrupts)** fail to trigger interrupts correctly.  
To ensure system stability, MSI can be disabled in the driver, reverting to the traditional legacy interrupt method.

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

**Note:**
- If the GPU cannot trigger interrupts, or system interrupts behave abnormally after inserting the GPU, try disabling MSI.
- Disable MSI only when necessary:  
  - **Disabling MSI** → higher stability, but performance and latency may be slightly affected.  
  - **Enabling MSI** → better performance and latency, but may be unstable on the K1 platform.

### Enable GPU Driver Configuration

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

Enable Radeon driver support in the kernel configuration file `k1_defconfig`:

```c
+CONFIG_DRM_RADEON=m
+CONFIG_DRM_RADEON_USERPTR=y
```

These configurations can be selected in `make menuconfig` using the following options:

- `Y` → built into the kernel (built-in)  
- `M` → compiled as a module

Currently, on the K1 platform:

- The **radeon driver** supports AMD GPUs with architectures **before GCN 1.0**;  
- For **GCN 1.0 and later architectures** (such as the Radeon Rx series), the **amdgpu driver** needs to be adapted and enabled.

> **Note:** After modifying or adapting the driver code, make sure the corresponding kernel configuration is enabled.
