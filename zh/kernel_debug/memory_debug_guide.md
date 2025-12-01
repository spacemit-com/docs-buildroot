# Linux 内存调试指南

## 整体内存情况

这些工具用于快速评估内存压力和性能瓶颈。

### free

#### 输出示例

```text
free
               total        used        free      shared  buff/cache   available
内存：       3505164     1079472     1232024        4492     1354252     2425692
交换：             0           0           0
```

- total: 总量
- used: 已使用量
- free: 完全空闲量
- shared: 共享内存（主要由 tmpfs 等使用）
- buff/cache: 用作缓冲区和缓存的内存
- available: 对应用程序来说，可用的内存
- 核心关系解读
  - `total` ≈ `used` + `free` + `buff/cache`
  - `available` ≈ `free` + (大部分的 `buff/cache`)

#### 常用选项

- `-h`：人类可读格式
- `-s <interval>`：持续监控，每 interval 秒刷新

#### 作用
   1.  作为日常快速检查内存使用率和 Swap 情况的首选工具。注意，判断内存是否够用，应该看 available，而不是 free。

### /proc/meminfo

#### 输出示例

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

- `Active`: 最近被访问过的内存，内核认为短期内可能还会被用到，不优先回收。
- `Inactive`: 较长时间未被访问的内存，是内存回收时的首选目标。
- `Active(anon)`: 最近被访问过的匿名内存。这是内存泄漏最常发生的区域。
- `Slab`: 内核数据结构（如 inode、dentry、socket 缓冲区等）的缓存。
- `SReclaimable`: `Slab` 中可以被安全回收的部分。
- `SUnreclaim`: `Slab` 中不能被回收的部分 。
- `CommitLimit`: 系统当前能够承诺分配给所有进程的总内存大小。
- `Committed_AS`: 所有进程已经申请的虚拟内存总和。这部分内存不一定真的被使用了，只是“预定”了。
- 核心关系解读
  - `Active` + `Inactive` 大致构成了系统当前正在使用的内存页。
  - `CommitLimit` ≈ `SwapTotal` + (`MemTotal` * `vm.overcommit_ratio` / 100)
  - `Committed_AS` (4.5 GB) > `CommitLimit` (1.75 GB)。这说明系统处于内存超售的状态。Linux 允许进程申请比实际物理内存更多的内存，它在赌这些进程不会同时使用它们申请的所有内存。
  - `MemTotal` ≈ (应用程序占用的 `AnonPages`) + (内核占用的 `Slab` + `KernelStack` + `PageTables` + ...) + (文件缓存 `Cached` + `Buffers`) + `MemFree`
  - `MemAvailable` ≈ `MemFree` + (大部分可回收的 `Cached`) + (可回收的内核内存 `SReclaimable`)

#### 作用

提供一个关于系统内存使用情况的、极其详尽的实时快照。因为free命令的输出信息有限，如果要获取更详细的内容，应查看"/proc/meminfo"。

系统响应缓慢，怀疑内存不足时，如果`MemAvailable`持续很低（比如低于总内存的 10%），说明系统可用的内存确实紧张；如果系统有 Swap，而这个值很高且在持续增长，说明物理内存已经耗尽，系统正在频繁地使用慢速的硬盘充当内存。

怀疑存在内存泄漏时，运行 `watch -n 1 cat /proc/meminfo`，观察 MemAvailable 是否在没有运行新程序的情况下，持续、缓慢地下降。

### top 

#### 输出示例

```text
top - 10:36:11 up  1:06,  2 users,  load average: 2.00, 2.00, 2.03
任务: 308 total,   1 running, 307 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.4 us,  0.4 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st 
MiB Mem :   3423.0 total,   1127.9 free,   1070.9 used,   1392.0 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2352.1 avail Mem 

 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND                                                                                                                                                                                       
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

- `KiB Mem : ...`: 物理内存使用情况。这一行就是 free 命令的简化版。
- `KiB Swap: ...`: 交换空间使用情况。`avail Mem`是评估系统可用内存的关键指标。
-  进程列表区：
  - `VIRT`: 进程“申请”或“预定”的总内存大小。它包括了进程实际使用的内存、被交换出去的内存、以及它所链接的共享库的内存。
  - `RES`: 它表示该进程当前实际占用了多少物理内存（RAM）。
  - `SHR`: 代表`RES` 中，有多少是与其他进程共享的内存（主要是共享库）。
  - `%MEM`: 代表该进程的 `RES` 占系统总物理内存 (`MemTotal`) 的百分比。
- 核心关系解读
  -  `RES` - `SHR` 可以估算出这个进程“独占”的物理内存。

#### 作用

top 命令可以看作是 /proc/meminfo 和 free 的一个动态、交互式的“仪表盘”，它不仅显示了系统的总体资源情况，并列出了消耗资源的进程排行榜。

在 top 界面中，直接按下大写的 M (Shift + m)。top 会立刻按照 %MEM 列从高到低对进程进行排序。

按下小写的 k，top 会提示你输入要杀死的进程 PID，输入后按回车，再输入信号（默认15，表示正常终止），再次回车即可杀死进程。

### echo m > /proc/sysrq-trigger

1.  输出示例

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

- 第一部分：全局内存页统计
  - `isolated_anon/isolated_file`: 隔离页。这是内存回收过程中的一个临时状态。当内核扫描 inactive 列表准备回收时，会先把选中的页移动到 isolated 列表中，以避免争用。
  - `unreclaimable`: 内核认为“正在使用中”、不可释放的内核数据结构。
  - `unevictable`: 不可回收页。被 mlock() 系统调用锁定的内存、某些特殊的内核分配等。这些页面绝对不会被交换或回收。
  - `pagetables`: 用于管理虚拟地址到物理地址映射的页表本身所占用的内存。进程越多、内存分配越碎片化，这个值就越大。
- 第二部分：Node 和 Zones 
  - `Node`: 在 NUMA 架构下代表一个 CPU 节点及其本地内存。在普通PC上，通常只有一个 Node 0。
  - `DMA32 / Normal`: 内存域 (Zone)。DMA32 用于与只能在 4GB 以下地址寻址的 32 位设备进行 DMA 交互的内存区域。
  - `Watermarks`: min: 最低水位线。low: 低水位线。high: 高水位线。
- 第三部分：内存碎片化直方图
  - `588*4kB`: 表示在 Normal 域中，有 588 个大小为 4KB 的连续空闲内存块。

#### 作用

该命令会立即触发内核，让它将一份极其详细的、底层的内存状态快照转储到内核日志缓冲区中。它是一个时间点的快照。这对于捕捉瞬时发生的系统问题非常有用，可以在问题发生时立即触发它。

如果发现系统卡顿，立即触发 echo m，然后查看 dmesg。如果发现任何一个 Zone 的 free 值低于它的 low 甚至 min 值，就直接证明了卡顿是由内核进行内存回收导致的。

如果看到总 free 很高，但直方图里全是 *4kB, *8kB 这样的小块，而没有 *2048kB 这样的大块，这就直接证明了问题是内存碎片化导致的。

## 页分配器

分析伙伴系统，诊断内存碎片问题。

### /proc/buddyinfo

#### 输出示例

```text
cat /proc/buddyinfo 
Node 0, zone    DMA32    753    750    565    400    893    433    167     93     40     13      1     71 
Node 0, zone   Normal     22      3      7    119     96     37      6     16      3      0      0      0  
```

- 每一列代表一个“阶”（order），其大小为 2^order * page_size。在 x86 系统上，一个 page 是 4KB。

#### 作用

主要定位内存碎片化问题。当怀疑碎片化严重时，你可以手动触发内存整理：`echo 1 > /proc/sys/vm/compact_memory`。可以看到内核成功地将小碎片合并成了大块内存。

### /proc/pagetypeinfo

#### 输出示例

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

- 第一部分：头部信息
  - `Page block order\Pages per block`: 表示一个 pageblock 的大小是 2^9 = 512 个页，如果一个页是 4KB，那么一个 pageblock 就是 512 * 4KB = 2048KB = 2MB。
- 第二部分：按类型细分的空闲页统计
  - 把 buddyinfo 的每一行，按照页面的可迁移类型进行了再次分解。
  - `Unmovable`: 不可移动页，内核核心数据结构、被硬件直接引用的内存等。
  - `Movable`: 可移动页，绝大多数用户态应用程序的内存。
  - `Reclaimable`: 可回收页，不能移动，但可以被安全地丢弃，最典型的是文件页缓存。
  - `HighAtomic`: 作为原子分配的终极备用池。
  - `CMA`: 为需要物理连续内存的设备驱动程序预留的区域。
  - `Isolate`: 内存整理过程中的一个临时状态，通常应为 0。
- 第三部分：Pageblock 类型摘要
  - 在每个 zone 中，有多少个 2MB 的 pageblock 被归类为各种类型。

#### 作用

可以用于定位内存整理失效问题，大量的 Unmovable 内存块充当了“路障”，会使得内核无法将可移动的内存页有效地挪到一起。

### /proc/zoneinfo

#### 使用示例

```text
cat /proc/zoneinfo 
```

- `per-node stats`: 提供了针对整个 NUMA 节点（Node）的内存统计数据。它将该节点下所有内存域（DMA32, Normal）的数据聚合在一起。
  - `nr_kernel_stack`: 所有进程/线程的内核栈所占用的总页数。这是一个估算系统任务数量的指标。
  - `nr_page_table_pages`: 用于页表本身的内存页数。如果一个进程的虚拟内存空间非常大且稀疏，这个值会显著增长。
- `Per-CPU 页缓存`：内核为每个 CPU 都维护了一个小型的“私有”内存缓存 pageset。当 pageset 的页数超过了它的 high 水位线（805），多余的页才会被批量（batch=63）地还给全局的伙伴系统。

#### 作用

/proc/zoneinfo 提供了最详尽、最原始、最全面的内存状态数据，基本上之前所有内存相关 /proc 文件（如 meminfo, buddyinfo）的数据都是从这里汇总或提取出来的。它提供了最精确的分类计数，能精确地知道在 Normal 区域和 DMA32 区域，各种类型的内存分别是多少。

## 对象分配器

监控 Slab/Slub 缓存，排查内核对象泄漏。

### /proc/slabinfo

#### 使用示例

```text
cat /proc/slabinfo 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables ... : slabdata <active_slabs> <num_slabs> <sharedavail>
inode_cache        13879  14025    640   25    4 : tunables    0    0    0 : slabdata    561    561      0
```

#### 作用

用于定位内核内存泄漏 ，系统长时间运行后，可用内存越来越少，但通过 top 等工具找不到是哪个用户进程的错。meminfo 显示 SUnreclaim (不可回收的 Slab) 持续增长。如果发现某个特定的 slab 的 active_objs 和 num_slabs 数量在只增不减，就极有可能找到了内存泄漏的源头。

### slabtop

#### 使用示例

```text
slabtop 
```

- 按下 c 键，这会使列表按 CACHE SIZE 排序，将内存消耗最大的缓存池置顶。 

#### 作用

slabtop 可以看作是 /proc/slabinfo 的一个交互式的实时视图，它让诊断内核内存问题变得更加直观。如果屏幕上最顶部的某个缓存池的 ACTIVE 对象数和 CACHE SIZE 在不断稳定增长，从不下降，很可能就是泄漏的源头。

## 虚拟内存管理

分析虚拟内存活动和进程内存映射。

### vmstat

#### 使用示例

```text
vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b 交换 空闲 缓冲 缓存   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0 736276  66928 1503792    0    0    46    19 1533    2  1  0 99  0  0  0
 
```

#### 作用

可以定位内存瓶颈，比如当物理内存不足以支撑当前的应用负载，系统正在频繁使用慢速的交换分区。

### /proc/vmallocinfo

#### 使用示例

```text
cat /proc/vmallocinfo 
```

#### 作用

内核中申请内存，除了使用slab，还可能使用vmallo申请。该文件详细列出了当前 vmalloc 区域中所有已分配的内存块。如果发现某个区域的内存只增不减，并且反复出现由同一个调用者（例如 some_driver_alloc）分配的记录，很可能是这个驱动程序或内核模块存在内存泄漏。

### pmap

#### 使用示例

```text
pmap [options] PID [PID ...] 
```

#### 常用选项

- `-x`: 显示细节
- `-X`: 显示更多细节
- `-XX`: 显示内核提供的所有信息

#### 作用

用于显示指定进程的虚拟内存映射信息。有时候程序崩溃或行为异常可能与内存访问错误有关（如段错误），pmap 可以提供上下文信息。

## 调试文件节点与工具

用于深度调试的专用工具。

### /sys/kernel/debug/kmemleak

#### 使用方法

- 开启 kmemleak

```text
CONFIG_DEBUG_KMEMLEAK=y
```

- 手动触发一次扫描

```text
echo scan > /sys/kernel/debug/kmemleak
```

- 获取报告

```text
cat /sys/kernel/debug/kmemleak
```

#### 作用

用于定位内存泄漏问题，这类问题不会立即导致崩溃，但会慢慢耗尽系统资源。

### KASAN

#### 使用方法

-  开启 KASAN

```text
CONFIG_KASAN=y 
```

- 对于软件模式，还可以在 `CONFIG_KASAN_OUTLINE` 和 `CONFIG_KASAN_INLINE` 之间进行选择。outline和inline是编译器插桩类型。前者产生较小的二进制文件， 而后者快2倍。

#### 作用

用于发现具有立即破坏性的、非法的内存访问行为。如越界访问、释放后使用、重复释放等。

### /proc/sys/vm/drop_caches

#### 使用方法

```text
echo 1 > /proc/sys/vm/drop_caches
```

-  释放页面缓存（Page Cache）

```text
echo 2 > /proc/sys/vm/drop_caches
```

- 释放目录项缓存（dentries）和索引节点缓存（inodes）。

#### 作用

用于手动触发内核清空Page Cache、dentry caches和inode caches的接口。

### /proc/sys/vm/compact_memory

#### 使用方法

```text
 echo 1 > /proc/sys/vm/compact_memory
```

#### 作用

用于手动触发内存规整，减少内存碎片、创建更大连续物理内存块。

