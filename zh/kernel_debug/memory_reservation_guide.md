# Linux 内存预留指南

## 内存预留概览

Linux 内核的内存预留可以大致分为两类：

- no-map：被预留的内存完全从内核伙伴系统中移除，内核在正常运行时无法访问或使用这部分内存，通常不会为其建立虚拟地址映射。这块内存对内核来说是“不可见”的，隔离性最强。
- CMA：被预留的内存名义上属于某个用途，但在其未被使用时，可以被内核借用给普通的可移动页使用。

## no-map

### 内核启动参数预留

#### 方法

在 bootloader 传递给内核的命令行参数中，使用 `memmap=<size>$<address>`。

#### 示例

```text
# 从物理地址 0x50000000 开始，预留 64MB 内存
bootargs=... memmap=64M$0x50000000
```

### 设备树预留

#### 示例

```c
reserved-memory {
    /* ... */
    // 1. 定义一个带标签的 no-map 区域
    dsp_mem: dsp_memory_region@30000000 {
        reg = <0x0 0x30000000 0x0 0x1000000>; // 16MB
        no-map;
    };
};

// ...

// 2. 在 DSP 设备节点中引用这个区域
dsp_controller: dsp@C088C000 {
    compatible = "vendor,dsp-controller";
    /* ... */
    // Linux 驱动通过读取此属性，得知 DSP 可用的物理内存地址和大小
    memory-region = <&dsp_mem>;
};
```

## CMA

### 内核启动参数预留

#### 方法
   1.   在 bootloader 传递给内核的命令行参数中，使用 `cma=<size>[@<address>]`
2. 示例

```text
# 创建一个 512MB 大小的全局 CMA 区域
bootargs=... cma=512M
```

### 配置内核预留

#### 方法

在 menuconfig 中设置 CONFIG_CMA_SIZE_MBYTES 的值。

#### 示例

```c
//in kernel .config
CONFIG_DMA_CMA=y
CONFIG_CMA_SIZE_SEL_MBYTES=y
CONFIG_CMA_SIZE_MBYTES=256 // 设置一个 256MB 的全局 CMA 区域
```

### 设备树预留

#### 全局预留

#### 方法

设备树中带有 linux,cma-default 属性的节点。这是通过设备树定义全局池的标准方法。

注意，全局 CMA 池只能有一个，优先顺序是DTS > CMDLINE > DEFCONFIG，即如果dts有合法的节点，会

以dts节点配置为最终参数，否则看cmdline是否有cma=size@base，最后才看defconfig是否有定义。

#### 示例

```c
reserved-memory {
    /* ... */
    linux,cma {
        compatible = "shared-dma-pool";
        reusable;
        size = <0x10000000>; // 256MB
        alignment = <0x2000>;
        linux,cma-default; // <-- 此属性将其标记为全局默认池
    };
};
```

#### 设备独占预留

#### 方法

- 第一步：在 reserved-memory 中定义一个带标签的 CMA 区域。这个节点不能有 linux,cma-default 属性。
- 第二步：在具体的设备节点中，通过 memory-region 属性引用该标签。

#### 示例

```c
reserved-memory {
    /* ... */
    vpu_cma_pool: vpu_cma@90000000 { // <-- "vpu_cma_pool" 是标签
        compatible = "shared-dma-pool";
        reusable;
        reg = <0 0x90000000 0 0x8000000>; // 128MB
    };
};

vpu_node: vpu@12340000 {
    compatible = "vendor,vpu-driver";
    /* ... */
    memory-region = <&vpu_cma_pool>; // <-- 引用上面定义的独占池
};
```

通过这种方式，`vpu_node` 的驱动在申请连续内存时，内核会定向到 `vpu_cma_pool` 这个专用的 128MB 池中进行分配，而不会使用全局的 CMA 池。

## 调试与问题定位

### 调试节点

#### /proc/iomem

/proc/iomem 是 Linux 内核视角下的物理地址空间地图。它详细列出了系统中所有的物理地址段，并标明了每个地址段当前被哪个组件“声明”或“占用”。

```c
00000000-0007ffff : Reserved
2ff40000-2fffffff : Reserved 
```

 这是 no-map 内存，内核的伙伴系统不会管理这部分内存。

1. cat /proc/meminfo | grep "Cma"

```c
CmaTotal:         262144 kB  // 所有 CMA 区域的总大小
CmaFree:          258048 kB  // CMA 区域中当前“空闲”的部分 
```

- CmaTotal：所有 CMA 池大小的总和。
- CmaFree：这部分内存或者是真正空闲的，或者是被内核“借走”用于可移动页。CmaTotal - CmaFree 的值，代表当前有多少内存被驱动程序通过 dma_alloc_* 等函数实际分配用于连续内存。

#### /sys/kernel/debug/cma

```c
root@root:/sys/kernel/debug/cma/linux,cma# ls
alloc  base_pfn  bitmap  count  free  maxchunk  order_per_bit  used
```

关键节点解读：

```
cat /sys/kernel/debug/cma/linux,cma/used_bytes
```

- 显示当前被驱动程序实际分配用于 DMA 的字节数。这个值非常精确，可以判断驱动是否存在内存泄漏。

```
cat /sys/kernel/debug/cma/linux,cma/count
```

- 显示该 CMA 池总共包含的页面数量。

```
cat /sys/kernel/debug/cma/linux,cma/bitmap
```

- 提供一个底层的位图，显示池中每个页块的分配状态，用于分析内存碎片化等高级问题。

### 开机 log

1.  dmesg | grep -i reserved

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

可以查看 no-map、cma 的预留内存地址与大小。

#### dmesg | grep -i cma

```c
dmesg | grep -i cma
[    0.000000] Reserved memory: created CMA memory pool at 0x0000000058000000, size 384 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] OF: reserved mem: 0x0000000058000000..0x000000006fffffff (393216 KiB) map reusable linux,cma
```

可以查看 cma 全局预留内存的地址与大小。
