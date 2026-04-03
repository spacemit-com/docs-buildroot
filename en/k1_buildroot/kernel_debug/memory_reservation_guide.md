sidebar_position: 4

# Linux Memory Reservation Guide

## 1. Memory Reservation Overview

Linux kernel memory reservation consists of two categories:

- **no map:**   
The reserved memory is completely removed from the kernel buddy system. The kernel cannot access or use this memory during normal operation, and generally no virtual address mapping is established for it. This memory is "invisible" to the kernel, providing the strongest isolation. 
- **CMA:**   
The reserved memory is nominally dedicated to a specific purpose, but when free, it can be temporarily borrowed by the kernel for ordinary movable pages.

## 2. no-map

### 2.1 Reservation via Kernel Boot Parameter

- #### Method

Use `memmp=<size>$<address>` in the kernel command line parameters passed by the bootloader.

- #### Example

```text
# Reserve 64MB memory starting at physical address 0x50000000
bootargs=... memmap=64M$0x50000000
```

### 2.2 Reservation via Device Tree

- #### Example

```c
reserved-memory {
    /* ... */
    // 1. Define a labeled no-map region
    dsp_mem: dsp_memory_region@30000000 {
        reg = <0x0 0x30000000 0x0 0x1000000>; // 16MB
        no-map;
    };
};

// ...

// 2. Reference this region in DSP device node
dsp_controller: dsp@C088C000 {
    compatible = "vendor,dsp-controller";
    /* ... */
    // The Linux driver learns the available physical memory address and size for the DSP by reading this property
    memory-region = <&dsp_mem>;
};
```

## 3. CMA

### 3.1 Reservation via Kernel Boot Parameters 

- #### Method 
1. Use `cma=<size>[@<address>] in the kernel command line parameters passed by the bootloader
2. Example

```text
# Establish a global 512MB CMA region
bootargs=... cma=512M
```

### 3.2 Reservation via Kernel Configuration

- #### Method

Set the value of CONFIG_CMA_SIZE_MBYTES in the menuconfig

- #### Example

```c
//in kernel .config
CONFIG_DMA_CMA=y
CONFIG_CMA_SIZE_SEL_MBYTES=y
CONFIG_CMA_SIZE_MBYTES=256 // Configure a global 256MB CMA region
```

### 3.3 Reservation via Device Tree

#### 3.3.1 Global Reservation

#### Method

The nodes with `linux,cma-default` property in the device tree. This is the standard method for defining a global pool via device tree.

**Note:** There can be only one global CMA pool, and the priority order should be DTS > CMDLINE > DEFCONFIG. That is, if a valid node exists in the DTS, the configuration from that DTS will be used as the final parameter. Otherwise, the kernel checks whether `cma=size@base` is present in the CMDLINE. Only if neither is present will it use the definition from DEFCONFIG. 

#### Example

```c
reserved-memory {
    /* ... */
    linux,cma {
        compatible = "shared-dma-pool";
        reusable;
        size = <0x10000000>; // 256MB
        alignment = <0x2000>;
        linux,cma-default; // <-- This property marks it as a global default pool
    };
};
```

#### 3.3.2 Device Exclusive Reservation

#### Method

- Step 1: Define a labeled CMA region in the reserved-memory. This node should not contain `linux,cma-default` property.
- Step 2: Reference this label in the corresponding device node using the `memory-region` property.

#### Example

```c
reserved-memory {
    /* ... */
    vpu_cma_pool: vpu_cma@90000000 { // <-- "vpu_cma_pool" is a label
        compatible = "shared-dma-pool";
        reusable;
        reg = <0 0x90000000 0 0x8000000>; // 128MB
    };
};

vpu_node: vpu@12340000 {
    compatible = "vendor,vpu-driver";
    /* ... */
    memory-region = <&vpu_cma_pool>; // <-- Reference the dedicated pool above
};
```

With this configuration, when the `vpu_node` driver requests contiguous memory, the kernel will allocate  from the dedicated 128MB `vpu_cma_pool` rather than from the global CMA pool.

## 4. Debugging and Troubleshooting

### 4.1 Node Debugging

#### 4.1.1 /proc/iomem

`/proc/iomem` is a physical address space map from the Linux kernel perspective. It lists all physical address segments in the system, and indicates each segment is "claimed" or "occupied" by which component.

```c
00000000-0007ffff : Reserved
2ff40000-2fffffff : Reserved 
```

 This is no-map memory, which is not managed by the kernel's buddy system.

#### 4.1.2 cat /proc/meminfo | grep "Cma"

```c
CmaTotal:         262144 kB  // Total size of all CMA regions 
CmaFree:          258048 kB  // Currently free portion of the CMA regions 
```

- CmaTotal: Total size of all CMA pools
- CmaFree: This memory is either truly free or borrowed by the kernel for movable pages. The value of `CmaTotal-CmaFree` indicates the amount of memory currently allocated by drivers for contiguous physical memory via functions like `dma_alloc_*`. 

#### 4.1.3 /sys/kernel/debug/cma

```c
root@root:/sys/kernel/debug/cma/linux,cma# ls
alloc  base_pfn  bitmap  count  free  maxchunk  order_per_bit  used
```

Key nodes introduction:

```
cat /sys/kernel/debug/cma/linux,cma/used_bytes
```

- Displays the number of byte currently allocated by drivers for DMA3. This value is highly precise, and can be used to determine whether memory leaks exists in the drivers. 

```
cat /sys/kernel/debug/cma/linux,cma/count
```

- Displays the total number of pages contained in this CMA pool.

```
cat /sys/kernel/debug/cma/linux,cma/bitmap
```

- Provides an underlying bitmap that displays the allocation status of each page block, used for advanced issues like memory fragmentation analysis.   

### 4.2 Boot log

#### 4.2.1 dmesg | grep -i reserved

```text
root@root:~# dmesg | grep -i reserved
[    0.000000] Reserved memory: created CMA memory pool at 0x0000000058000000, size 384 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] OF: reserved mem: 0x0000000058000000..0x000000006fffffff (393216 KiB) map reusable linux,cma
[    0.000000] OF: reserved mem: 0x0000000000000000..0x000000000007ffff (512 KiB) nomap non-reusable mmode_resv0@0
[    0.000000] OF: reserved mem: 0x0000000000100000..0x00000000002fffff (2048 KiB) nomap non-reusable rcpu_mem_heap@100000
[    0.000000] OF: reserved mem: 0x0000000000300000..0x0000000000302fff (12 KiB) nomap non-reusable vdev0vring0@300000
[    0.000000] OF: reserved mem: 0x0000000000303000..0x0000000000305fff (12 KiB) nomap non-reusable vdev0vring1@303000
[    0.000000] Reserved memory: created DMA memory pool at 0x0000000000306000, size 0 MiB
[    0.000000] OF: reserved mem: initialized node vdev0buffer@306000, compatible id shared-dma-pool
[    0.000000] OF: reserved mem: 0x0000000000306000..0x00000000003fbfff (984 KiB) nomap non-reusable vdev0buffer@306000
[    0.000000] OF: reserved mem: 0x00000000003fc000..0x00000000003fffff (16 KiB) nomap non-reusable rsc_table@3fc000
[    0.000000] OF: reserved mem: 0x0000000000400000..0x00000000005fffff (2048 KiB) nomap non-reusable rcpu_mem_0@400000
[    0.000000] Reserved memory: created DMA memory pool at 0x000000002ff40000, size 0 MiB
[    0.000000] OF: reserved mem: initialized node dpu_reserved@2ff40000, compatible id shared-dma-pool
[    0.000000] OF: reserved mem: 0x000000002ff40000..0x000000002fffffff (768 KiB) nomap non-reusable dpu_reserved@2ff40000
[    0.000000] OF: reserved mem: 0x000000007f000000..0x000000007fffffff (16384 KiB) map non-reusable framebuffer@7f000000
```

This command shows the reserved memory address and size of no-map and cma.

#### 4.2.2 dmesg | grep -i cma

```c
dmesg | grep -i cma
[    0.000000] Reserved memory: created CMA memory pool at 0x0000000058000000, size 384 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] OF: reserved mem: 0x0000000058000000..0x000000006fffffff (393216 KiB) map reusable linux,cma
```

This command shows the address and size of the global CMA reserved memory.
