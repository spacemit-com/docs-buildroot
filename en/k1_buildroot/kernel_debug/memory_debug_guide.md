sidebar_position: 3

# Linux Memory Debugging Guide

## 1. Overall Memory Status

These tools are used to quickly assess memory pressure and performance bottlenecks.

### 1.1 free

#### Example Output

```text
free
               total        used        free      shared  buff/cache   available
Memory:       3505164     1079472     1232024        4492     1354252     2425692
Swap:             0           0           0
```

- total: total memory
- used: used memory
- free: free memory
- shared: shared memory (mainly used by tmpfs, etc)
- buff/cache: buffer/cache memory
- available: available memory for applications
- core relationship
  - `total` ≈ `used` + `free` + `buff/cache`
  - `available` ≈ `free` + (most of the `buff/cache`)

#### Common Options

- `-h`: human-readable format
- `-s <interval>`: continuously monitor and refresh every interval second 

#### Purpose
 It is the first choice for quickly checking memory usage and Swap status in daily use. **Note**: You should check `available` rather than `free` to determine whether memory is sufficient.

### 1.2 /proc/meminfo

#### Example Output

```text
cat /proc/meminfo
MemTotal:        3505164 kB
MemFree:         1199524 kB
MemAvailable:    2425724 kB
Buffers:           56024 kB
Cached:          1280484 kB
SwapCached:            0 kB
Active:          1299720 kB
Inactive:         571476 kB
Active(anon):     541312 kB
Inactive(anon):        0 kB
Active(file):     758408 kB
Inactive(file):   571476 kB
Unevictable:          48 kB
Mlocked:              48 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        534828 kB
Mapped:           300448 kB
Shmem:              6608 kB
KReclaimable:      53124 kB
Slab:             137832 kB
SReclaimable:      53124 kB
SUnreclaim:        84708 kB
KernelStack:       11008 kB
PageTables:        14860 kB
SecPageTables:         0 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     1752580 kB
Committed_AS:    4488468 kB
VmallocTotal:   67108864 kB
VmallocUsed:       89224 kB
VmallocChunk:          0 kB
Percpu:             5728 kB
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:         393216 kB
CmaFree:          270004 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```

- `Active`: Recently accessed memory is not prioritized for reclamation by the kernel, since it is deemed likely to be reused in the short term.
- `Inactive`: Memory that has not been accessed for a long time is the primary target for memory reclamation.
- `Active(anon)`: Recently accessed anonymous memory. This is the most common area for memory leaks. 
- `Slab`: Caches for kernel data structure (such as inode, dentry, socket buffer, etc.). 
- `SReclaimable`: Safely reclaimable portion of the `Slab`.
- `SUnreclaim`: Unreclaimable portion of the `Slab`.
- `CommitLimit`: Total memory the system can currently commit to allocating for all processes. 
- `Committed_AS`: Total virtual memory already requested by all processes. This memory is just reserved rather than used.
- Core relationship
  - `Active` + `Inactive` roughly constitute the memory pages currently in use by the system. 
  - `CommitLimit` ≈ `SwapTotal` + (`MemTotal` * `vm.overcommit_ratio` / 100)
  - `Committed_AS` (4.5 GB) > `CommitLimit` (1.75 GB). This indicates that the system is in an overcommit state. Linux permits processes to request more memory than the actual physical memory, betting that these processes will not use all their requested memory simultaneously.
  - `MemTotal` ≈ (`AnonPages` used by applications) + (`Slab` + `KernelStack` + `PageTables` + ... used by kernel) + (`Cached` + `Buffers` file cache) + `MemFree`
  - `MemAvailable` ≈ `MemFree` + (most reclaimable `Cached`) + (reclaimable kernel memory `SReclaimable`)

#### Purpose

Provide a detailed, real-time snapshot of system memory usage. Because the output of the `free` command is limited, see "/proc/meminfo" for more detailed information. 

When the system is slow to respond, it may indicate insufficient memory. If `MemAvailable` remains continuously low (e.g. below 10% of total memory), the available system memory is insufficient; if Swap is enabled and its usage is high and growing continuously, it means physical memory is exhausted, and the system is frequently using slow disk as memory.

When suspecting a memory leak, run `watch -n 1 cat /proc/meminfo` to check whether MemAvailable decreases steadily and slowly without launching new programs. 

### 1.3 top 

#### Example Output

```text
top - 10:36:11 up  1:06,  2 users,  load average: 2.00, 2.00, 2.03
Task: 308 total,   1 running, 307 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.4 us,  0.4 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
MiB Mem :   3423.0 total,   1127.9 free,   1070.9 used,   1392.0 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2352.1 avail Mem 

 Process ID USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND                                                                                                                                                                                       
   3029 root      20   0 1224740 119232  72168 S   2.3   3.4   5:26.08 Xorg                                                                                                                                                                                          
   5021 root      20   0   16432   5248   3072 R   1.3   0.1   0:00.32 top                                                                                                                                                                                           
    723 avahi     20   0    7544   3712   2944 S   0.7   0.1   0:21.87 avahi-daemon                                                                                                                                                                                  
     31 root      20   0       0      0      0 S   0.3   0.0   0:04.17 ksoftirqd/3                                                                                                                                                                                   
    152 root      20   0       0      0      0 D   0.3   0.0   0:01.42 vq0                                                                                                                                                                                           
    648 systemd+  20   0   14580   6016   5248 S   0.3   0.2   0:05.20 systemd-oomd                                                                                                                                                                                  
    677 root      -2   0       0      0      0 S   0.3   0.0   0:00.14 sdma1                                                                                                                                                                                         
    825 root      20   0  336432  17264  14064 S   0.3   0.5   0:08.19 NetworkManager                                                                                                                                                                                
   1011 root      -2   0       0      0      0 S   0.3   0.0   0:19.97 ksdioirqd/mmc1                                                                                                                                                                                
   1113 root      20   0       0      0      0 S   0.3   0.0   0:17.89 disp_eng_share_thread                                                                                                                                                                         
   1116 root      20   0       0      0      0 S   0.3   0.0   0:10.36 RTW_RX_CB_THREAD                                                                                                                                                                              
   3636 bianbu    20   0 3101528 252440 123968 S   0.3   7.2   2:19.30 gnome-shell                                                                                                                                                                                   
   4471 bianbu    20   0  736896  57240  44428 S   0.3   1.6   0:04.27 gnome-terminal-                                                                                                                                                                               
      1 root      20   0   19304  11368   7528 S   0.0   0.3   0:10.78 systemd                                                                                                                                                                                       
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.02 kthreadd                                                                                                                                                                                      
      3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_workqueue_release                                                                                                                                                                        
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/R-rcu_g                                                                                                                                                                               
```

- `KiB Mem : ...`: Physical memory usage. This line is a simplified `free` command. 
- `KiB Swap: ...`: Swap usage. `avail Mem` is the key indicator for assessing available system memory. 
- Process list area:
  - `VIRT`: Total memory "requested" or "reserved" by process. It includes actual memory used by the process, swapped memory and memory from shared libraries linked to the process.
  - `RES`: It represents physical memory (RAM) currently occupied by the process.
  - `SHR`: It represents portion of `RES` that is shared with other processes (primarily shared libraries).
  - `%MEM`: It represents the percentage of total system physical memory (`MemTotal`) used by the process. 
- Core relationship
  - `RES` - `SHR` estimates the private physical memory used by the process. 

#### Purpose

The `top` command can be seen as a dynamic, interactive "dashboard" for /proc/meminfo and free. It not only manifests the overall system resource but also ranks processes by resource consumption. 

In the top interface, press "M" ("Shift + m") to sort processes by %MEM in descending order. 

Press "k", and top will prompt you to enter the PID of the process to be killed. Press "ENTER", then enter a signal (default: 15 for normal termination). Press "ENTER" again to kill the process. 

### 1.4 echo m > /proc/sysrq-trigger

#### Example Output

```text
echo m > /proc/sysrq-trigger
[ 5091.850787] Mem-Info:
[ 5091.850796] active_anon:134760 inactive_anon:0 isolated_anon:0
                active_file:189965 inactive_file:147427 isolated_file:0
                unevictable:12 dirty:1 writeback:0
                slab_reclaimable:17517 slab_unreclaimable:21343
                mapped:74158 shmem:593 pagetables:3727
                sec_pagetables:0 bounce:0
                kernel_misc_reclaimable:0
                free:291765 free_pcp:8452 free_cma:67501
[ 5091.850817] Node 0 active_anon:539040kB inactive_anon:0kB active_file:759860kB inactive_file:589708kB unevictable:48kB isolated(anon):0kB isolated(file):0kB mapped:296632kB dirty:4kB writeback:0kB shmem:2372kB shmem_thp:0kB shmem_pmdmapped:0kB anon_thp:0kB writeback_tmp:0kB kernel_stack:11072kB pagetables:14908kB sec_pagetables:0kB all_unreclaimable? no
[ 5091.850834] DMA32 free:1134516kB boost:0kB min:24436kB low:30544kB high:36652kB reserved_highatomic:0KB active_anon:54744kB inactive_anon:0kB active_file:208028kB inactive_file:349756kB unevictable:0kB writepending:0kB present:2097152kB managed:1908640kB mlocked:0kB bounce:0kB free_pcp:15376kB local_pcp:3736kB free_cma:270004kB
[ 5091.850851] lowmem_reserve[]: 0 1559 1559
[ 5091.850865] Normal free:32544kB boost:0kB min:20616kB low:25768kB high:30920kB reserved_highatomic:0KB active_anon:484296kB inactive_anon:0kB active_file:551832kB inactive_file:239952kB unevictable:48kB writepending:4kB present:2097152kB managed:1596524kB mlocked:48kB bounce:0kB free_pcp:18432kB local_pcp:2760kB free_cma:0kB
[ 5091.850877] lowmem_reserve[]: 0 0 0
[ 5091.850887] DMA32: 779*4kB (UMEC) 485*8kB (UME) 430*16kB (UMEC) 188*32kB (UMEC) 846*64kB (UME) 613*128kB (UMEC) 196*256kB (UM) 76*512kB (UMC) 50*1024kB (UMC) 25*2048kB (UMC) 11*4096kB (MEC) 91*8192kB (MC) = 1134516kB
[ 5091.850950] Normal: 588*4kB (UME) 296*8kB (UME) 189*16kB (UME) 73*32kB (UME) 81*64kB (UME) 35*128kB (UM) 6*256kB (M) 16*512kB (M) 3*1024kB (M) 0*2048kB 0*4096kB 0*8192kB = 32544kB
[ 5091.851006] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=1048576kB
[ 5091.851012] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=64kB
[ 5091.851016] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=2048kB
[ 5091.851021] 337987 total pagecache pages
[ 5091.851025] 0 pages in swap cache
[ 5091.851028] Free swap  = 0kB
[ 5091.851032] Total swap = 0kB
[ 5091.851035] 1048576 pages RAM
[ 5091.851038] 0 pages HighMem/MovableOnly
[ 5091.851041] 172285 pages reserved
[ 5091.851044] 98304 pages cma reserved 
```

- Part 1: Global memory page statistics
  - `isolated_anon/isolated_file`: Isolated pages. This is a temporary status during memory reclamation. When the kernel scans the inactive list to reclaim pages, it first moves selected pages to the isolated list to avoid contention. 
  - `unreclaimable`: Kernel data structures that the kernel considers "in use" and cannot be freed. 
  - `unevictable`: Unreclaimable pages. Memory locked by the mlock() system call, certain special kernel allocations, etc. These pages are never swapped out or reclaimed. 
  - `pagetables`: Memory used by page tables themselves, which manage virtual-to-physical address mappings. This value increases with more processes and more fragmented memory allocations.   
- Part 2： Nodes and Zones
  - `Node`: Under a NUMA architecture, it represents a CPU node and its local memory. On ordinary PCs, there is usually only one Node 0.
  - `DMA32 / Normal`: Memory zone (Zone). DMA32 is used for DMA operations with 32-bit devices that can only address below 4GB.
  - `Watermarks`: min: minimum watermark; low: low watermark; high: high watermark.
  - `588*4KB`:It indicates there are 588 contiguous free memory blocks of size 4KB in the Normal zone.

#### Purpose

This command immediately trigger the kernel to dump a highly detailed, low-level snapshot of memory state into the kernel log buffer. It is a point-in-time snapshot. It is very useful for locating transient system issues. You can invoke it right away when a problem occurs. 

If the system becomes unresponsive, immediately trigger `echo m` and then check `dmesg`. If the free value of any zone is lower than its low or even min threshold, it turns out that the lag is caused by kernel memory reclamation.

If the total `free` memory is high, but the histogram consists almost entirely of small blocks such as *4KB, *8KB without large blocks like *2048KB, it turns out the issue is caused by memory fragmentation.

## 2. Page Allocator 

Analyze the buddy system and diagnose memory fragmentation issues.

### 2.1 /proc/buddyinfo

#### Example output

```text
cat /proc/buddyinfo 
Node 0, zone    DMA32    753    750    565    400    893    433    167     93     40     13      1     71 
Node 0, zone   Normal     22      3      7    119     96     37      6     16      3      0      0      0  
```
- Each column represents an order, whose size is 2^order * page_size. On x86 systems, one page is 4KB. 

#### Purpose

It is primarily used to identify memory fragmentation. When severe fragmentation is suspected, you can manually trigger memory compaction with `echo 1 > /proc/sys/vm/compact_memory`. Then you will observe that the kernel successfully compacts small fragments into large blocks.  

### 2.2 /proc/pagetypeinfo

#### Example output

```text
cat /proc/pagetypeinfo 
Page block order: 9
Pages per block:  512

Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10     11 
Node    0, zone    DMA32, type    Unmovable      1     11     89     63     50     14      6      2      1      1      0      0 
Node    0, zone    DMA32, type      Movable      1      1      1      1      1      0      1      0      3     39      8     50 
Node    0, zone    DMA32, type  Reclaimable      4     17     13      1      8      3      1      1      1      1      0      0 
Node    0, zone    DMA32, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone    DMA32, type          CMA      1      0      1      1      0      0      0      0      0      0      0     30 
Node    0, zone    DMA32, type      Isolate      0      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone   Normal, type    Unmovable    502    306    105    102     45      9      1      0      1      0      0      0 
Node    0, zone   Normal, type      Movable    110     26      5     14     15      4      3      4      3      2      0      0 
Node    0, zone   Normal, type  Reclaimable      3      3      5      6      2      0      0      0      0      0      0      0 
Node    0, zone   Normal, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone   Normal, type          CMA      0      0      0      0      0      0      0      0      0      0      0      0 
Node    0, zone   Normal, type      Isolate      0      0      0      0      0      0      0      0      0      0      0      0 

Number of blocks type     Unmovable      Movable  Reclaimable   HighAtomic          CMA      Isolate 
Node 0, zone    DMA32           16          804           12            0          192            0 
Node 0, zone   Normal         1166          856           26            0            0            0  
```

- Part 1: Header information  
  - `Page block order\Pages per block`: Indicates that one pageblock consists of  2^9 = 512 pages. If one page is 4KB, one pageblock is 512 * 4KB = 2048KB = 2MB.
- Part 2: Free pages counts by type
  - Further breaks down each line from `buddyinfo` according to the page migratetypes.
  - `Unmovable`: Unmovable pages used for core kernel data structures, memory directly referenced by hardware, etc.
  - `Movable`: Movable pages. Memory for most userspace applications.
  - `Reclaimable`: Reclaimable pages. Pages cannot be moved but can be safely discarded. The most typical type is file page cache.
  - `HighAtomic`: The final reserve pool for atomic allocations.
  - `CMA`: Reserved zone for device driver that requires physical contiguous memory.
  - `Isolate`: A temporary state during memory compaction, which should normally be 0.
- Part 3: Pageblock type summary 
  - The number of 2MB pageblocks in each zone classified into each type.

#### Purpose

It is used to identify failures in memory compaction. Numerous unmovable blocks act as "roadblocks", preventing the kernel from effectively compacting movable pages.

### 2.3 /proc/zoneinfo

#### Example Usage

```text
cat /proc/zoneinfo 
```

- `per-node stats`: Provides memory statistics aggregated for an entire NUMA Node. It combines data from all memory Zones (DMA32, Normal) under the node. 
  - `nr_kernel_stack`: Total pages occupied by all processes/threads. This is an indicator for estimating the number of system tasks running on the system.
  - `nr_page_table_pages`: Number of memory pages used for pages themselves. This value increases significantly if a process has a very large and sparse virtual memory space.
- `Per-CPU page cache`: The kernel maintains a small private memory cache pageset for each CPU. When the number of pages in pageset exceeds its high watermark (805), the excess pages are returned in batches (batch=63) to overall buddy system in batches (batch=63).

#### Purpose

`/proc/zoneinfo` provides the most detailed, raw and comprehensive memory state information. Data from almost all previous memory-related `/proc` files (e.g. meminfo, buddyinfo) is summarized or derived from here. It provides precise categorized counts, allowing exact inspection of how much memory of each type exists in the Normal zone and DMA32 zone. 

## 3. Object Allocator 

Monitors Slab/Slub caches and debugs the kernel object leaks.

### 3.1 /proc/slabinfo

#### Example Usage

```text
cat /proc/slabinfo 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables ... : slabdata <active_slabs> <num_slabs> <sharedavail>
inode_cache        13879  14025    640   25    4 : tunables    0    0    0 : slabdata    561    561      0
```

#### Purpose

Used to identify kernel memory leaks. After the system has been running for a long time, free memory gradually decreases, but tools such as `top` cannot identify which user process is responsible. `meminfo` manifests that SUnreclaim (unreclaimable Slab) keeps growing. If `active_objs` and `num_slabs` of a particular slab only increase and never decrease, this is highly likely the source of the memory leak. 

### 3.2 slabtop

#### Example Usage

```text
slabtop 
```

- Press `c` to sort the list by CACHE SIZE, bringing the largest memory-consuming caches to the top. 

#### Purpose

`slabtop` can be seen as an interactive real-time view of `/proc/slabinfo`, making kernel memory issue diagnosis more intuitive. If the number of ACTIVE object and CACHE SIZE of a cache at the top of the screen keep growing steadily without ever decreasing, it is likely the source of the leak.

## 4. Virtual Memory Management

Analyzes virtual memory activities and process memory mappings.

### 4.1 vmstat

#### Usage example

```text
vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b swap free buffer cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0 736276  66928 1503792    0    0    46    19 1533    2  1  0 99  0  0  0
 
```

#### Purpose

Identifies memory bottlenecks. For example, when physical memory is insufficient to support the current application load, the system will frequently use the slow swap partition. 

### 4.2 /proc/vmallocinfo

#### Example Usage

```text
cat /proc/vmallocinfo 
```

#### Purpose

In the kernel, memory can be allocated using `vmalloc` in addition to `slab`. This file lists all allocated memory blocks in the current `vmalloc` zone in detail. If memory in a specific area only increases and never decreases, and there are repeated allocation records from the same caller (such as some_driver_alloc), the driver or kernel module is very likely suffering from a memory leak.

### 4.3 pmap

#### Example Usage

```text
pmap [options] PID [PID ...] 
```

#### Common Options

- `-x`: displays details
- `-X`: displays more details
- `-XX`: displays all informtion from the kernel 

#### Purpose

Displays virtual memory mapping information of a specific process. Program crashes or abnormal behavior are sometimes related to memory access errors (such as segmentation fault), and `pmap` can provide contextual information. 

## 5. Debugging File Nodes and Tools

Tools for in-depth debugging. 

### 5.1 /sys/kernel/debug/kmemleak

#### Usage

- Enable kmemleak

```text
CONFIG_DEBUG_KMEMLEAK=y
```

- Manually trigger a scan 

```text
echo scan > /sys/kernel/debug/kmemleak
```

- Acquire the report

```text
cat /sys/kernel/debug/kmemleak
```

#### Purpose

Identify memory leak issues, which do not lead to an immediate crash, but gradually exhaust system resources.

### 5.2 KASAN

#### Usage

-  Enable KASAN

```text
CONFIG_KASAN=y 
```

- For software mode, `CONFIG_KASAN_OUTLINE` and `CONFIG_KASAN_INLINE` are available. `Outline` and `inline` are compiler instrumentation types. `Outline` produces a small binary file, while `inline` is twice as fast. times faster. 

#### Purpose

Used to detect immediately destructive illegal memory accesses, such as out-of-bounds access, use-after-free, double-free, etc.

### 5.3 /proc/sys/vm/drop_caches

#### Usage

```text
echo 1 > /proc/sys/vm/drop_caches
```

-  Free page cache

```text
echo 2 > /proc/sys/vm/drop_caches
```

- Free dentry caches and inode caches.


#### Purpose

An interface used to manually trigger the kernel to drop page caches, dentry caches and inode caches. 

### 5.4 /proc/sys/vm/compact_memory

#### Usage

```text
 echo 1 > /proc/sys/vm/compact_memory
```

#### Purpose

Usedd to manually trigger memory compaction to reduce memory fragments and create larger contiguous physical memory blocks.

