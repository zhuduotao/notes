---
created: '2026-04-17'
updated: '2026-04-17'
tags:
  - linux
  - memory-management
  - kernel
  - virtual-memory
  - os-internals
aliases:
  - Linux Memory Management
  - 内存管理
  - Linux MM
source_type: official-doc
source_urls:
  - https://docs.kernel.org/admin-guide/mm/index.html
  - https://docs.kernel.org/mm/index.html
  - https://www.kernel.org/doc/html/latest/core-api/mm_api.html
status: verified
---

## 概述

Linux 内存管理是内核最核心的子系统之一，负责物理内存和虚拟内存的分配、回收、映射与保护。其设计目标是在多进程环境下提供高效的内存利用率、公平的分配策略以及可靠的隔离机制。

## 核心概念

### 虚拟内存（Virtual Memory）

每个进程拥有独立的虚拟地址空间，通过页表（Page Table）映射到物理内存。虚拟内存带来的好处：

- **进程隔离**：每个进程只能访问自己的地址空间
- **按需分配**：使用内存映射和缺页中断实现延迟分配
- **地址空间扩展**：通过交换（Swap）机制使用磁盘空间扩展可用内存
- **共享内存**：多个进程可映射到同一物理页面

**32 位与 64 位地址空间**：
- 32 位系统：通常用户空间 3GB，内核空间 1GB（可配置）
- 64 位系统：用户空间 128TB（4 级页表）或更大（5 级页表）

### 页面（Page）与页帧（Page Frame）

Linux 以页面为基本单位管理内存，默认页面大小为 **4KB**（x86_64），也支持大页面（Huge Page）：

| 页面类型 | 大小 | 适用场景 |
|---------|------|---------|
| 普通页面 | 4KB | 通用内存分配 |
| 透明大页面（THP） | 2MB / 1GB | 数据库、高性能计算 |
| hugetlbfs | 2MB / 1GB | 显式分配的大页面 |

### 内存区域（Memory Zones）

内核将物理内存划分为不同的区域（Zone），用于满足不同的分配需求：

| Zone 名称 | 用途 | 典型范围 |
|-----------|------|---------|
| `ZONE_DMA` | 用于 DMA 操作，需要低地址内存 | 0 ~ 16MB |
| `ZONE_DMA32` | 32 位设备 DMA 可用区域 | 16MB ~ 4GB |
| `ZONE_NORMAL` | 内核可直接映射的普通内存 | 4GB 以上 |
| `ZONE_HIGHMEM` | 高端内存，32 位系统中无法直接映射的部分 | 仅 32 位系统 |
| `ZONE_MOVABLE` | 可迁移页面，用于内存热插拔 | 动态 |

## 内存分配机制

### 伙伴系统（Buddy System）

伙伴系统是 Linux 管理物理页面的核心算法，用于解决外部碎片问题。

**工作原理**：
- 将空闲页面按 2 的幂次方分组（order 0 ~ MAX_ORDER，通常 MAX_ORDER=11）
- order n 的块包含 2^n 个连续页面
- 分配时找到满足需求的最小 order 块，若有剩余则拆分并放入更低 order 链表
- 释放时检查相邻块是否空闲，若空闲则合并为更大块

**优点**：
- 快速分配和释放（O(1) 或 O(log n)）
- 有效减少外部碎片

**缺点**：
- 不解决内部碎片问题
- 大内存分配可能失败（碎片化导致无连续大块）

### SLUB 分配器

SLUB 是 Linux 默认的 slab 分配器，用于管理内核中小对象的分配（如 inode、task_struct 等）。

**层级结构**：
1. **SLUB**：当前默认实现，简化了 SLAB/SLAB 的设计，减少元数据开销
2. **SLOB**：用于嵌入式系统，极简实现

**核心概念**：
- **kmem_cache**：缓存特定类型的对象
- **slab**：由一个或多个连续页面组成，包含多个同类型对象
- **object**：单个分配的内核对象

### vmalloc

`vmalloc` 用于分配虚拟地址连续但物理地址不连续的内存区域，适用于：

- 大内存分配（超过伙伴系统能力）
- 模块加载
- 临时缓冲区

**注意**：`vmalloc` 比 `kmalloc` 慢，因为需要建立页表映射。

## 虚拟内存管理

### 页表（Page Table）

Linux 使用多级页表实现虚拟地址到物理地址的转换：

- **4 级页表**（x86_64 默认）：PGD → P4D → PUD → PMD → PTE
- **5 级页表**（可选，支持更大地址空间）

**页表项标志**：
- `Present`：页面是否在内存中
- `Read/Write`：读写权限
- `User/Supervisor`：用户态/内核态访问权限
- `Accessed`：是否被访问过（用于页面回收）
- `Dirty`：是否被修改过（用于页面写回）

### 缺页中断（Page Fault）

当进程访问的虚拟页面不在物理内存中时触发缺页中断，内核处理流程：

1. 检查访问是否合法（地址是否在 VMA 范围内、权限是否正确）
2. 若不合法，发送 SIGSEGV 信号（段错误）
3. 若合法，分配物理页面并建立映射
4. 若页面在交换区，从磁盘读回

**缺页类型**：
- **Minor Fault**：页面在内存中但未映射，只需建立映射
- **Major Fault**：页面不在内存中，需要从磁盘读取

### 内存映射区域（VMA）

VMA（Virtual Memory Area）描述进程地址空间中的一块连续区域，通过 `mm_struct` 管理。

**VMA 类型**：
- 代码段（.text）
- 数据段（.data、.bss）
- 堆（heap）
- 栈（stack）
- 内存映射文件（mmap）
- 共享内存

可通过 `/proc/<pid>/maps` 查看进程的 VMA 信息。

## 内存回收机制

### 页面回收（Page Reclaim）

当系统内存不足时，内核通过页面回收机制释放内存：

**LRU 链表**：
- **Active List**：活跃页面，频繁访问
- **Inactive List**：不活跃页面，可能被回收
- 每个链表分为 **Anonymous**（匿名页）和 **File**（文件缓存页）

**回收策略**：
1. 页面被访问时从 Inactive 移到 Active
2. Active 列表满时，最久未使用的页面移到 Inactive
3. 回收时优先回收 File 页面（可重新从文件读取）
4. Anonymous 页面需要先写入 Swap 才能回收

### kswapd 内核线程

`kswapd` 是每个内存节点（NUMA Node）一个的内核线程，负责异步页面回收：

- 当空闲页面低于水位线（watermark）时唤醒
- 尝试回收页面直到空闲页面高于高水位线
- 避免进程在分配内存时阻塞

### 直接回收（Direct Reclaim）

当 `kswapd` 回收速度跟不上分配速度时，分配内存的进程会直接参与页面回收：

- 可能导致进程阻塞，影响性能
- 触发条件：空闲页面低于最低水位线（min watermark）

## 交换机制（Swap）

### Swap 分区与 Swap 文件

Linux 支持两种 Swap 后端：
- **Swap 分区**：专用分区，性能更好
- **Swap 文件**：普通文件，配置灵活

### Swap 策略

- **swappiness**（`/proc/sys/vm/swappiness`）：控制内核回收匿名页的倾向
  - 0：尽量不 Swap（仅在内存极度紧张时）
  - 60：默认值
  - 100：积极使用 Swap
- **Swap 优先级**：可配置多个 Swap 设备的优先级，高优先级先使用

### 透明大页面与 Swap

透明大页面（THP）在 Swap 场景下可能导致延迟问题，因为需要拆分大页面。某些场景下建议关闭 THP 的 Swap 支持。

## OOM（Out of Memory）机制

### OOM Killer

当系统内存耗尽且无法回收时，OOM Killer 会选择一个或多个进程终止以释放内存。

**选择算法**（基于 `oom_score`）：
- 使用内存越多，得分越高，越容易被杀
- 子进程使用的内存会计入父进程
- 特权进程得分较低（更不容易被杀）

**查看 OOM 分数**：
```bash
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj  # 可调整的范围：-1000 ~ 1000
```

**控制 OOM 行为**：
- `oom_score_adj = -1000`：完全禁止 OOM Killer 终止该进程
- `oom_score_adj = 1000`：优先终止该进程

### cgroup 内存限制

cgroup v2 支持对控制组设置内存限制：
- `memory.max`：硬限制，超出时触发 OOM
- `memory.high`：软限制，超出时施加回收压力
- `memory.swap.max`：限制 Swap 使用量

## NUMA 支持

### NUMA 架构

非一致性内存访问（NUMA）架构中，每个 CPU 节点有本地内存，访问本地内存比远程内存更快。

### NUMA 策略

- **默认策略**：优先分配本地节点内存，不足时分配远程节点
- **绑定策略**（`MPOL_BIND`）：仅在指定节点分配
- **交叉策略**（`MPOL_INTERLEAVE`）：在多个节点间交叉分配
- **本地策略**（`MPOL_PREFERRED`）：优先本地，可回退到远程

**查看 NUMA 信息**：
```bash
numactl --hardware
numastat
```

## 内存监控与调试

### 常用命令

| 命令 | 用途 |
|------|------|
| `free` | 查看系统内存使用情况 |
| `vmstat` | 虚拟内存统计 |
| `top` / `htop` | 进程级内存使用 |
| `pmap` | 进程内存映射详情 |
| `smem` | 更准确的进程内存统计（考虑共享内存） |

### 关键指标

- **RSS**（Resident Set Size）：进程实际使用的物理内存
- **VSZ**（Virtual Memory Size）：进程虚拟地址空间大小
- **PSS**（Proportional Set Size）：按比例分配的共享内存
- **USS**（Unique Set Size）：进程独占内存

### `/proc/meminfo` 关键字段

```
MemTotal:       总物理内存
MemFree:        完全空闲的内存
MemAvailable:   可用内存（包括可回收的缓存）
Buffers:        块设备缓冲区
Cached:         页面缓存
SwapTotal:      Swap 总量
SwapFree:       Swap 剩余量
Dirty:          待写回的脏页
Slab:           内核 slab 缓存
```

## 限制与注意事项

1. **内存碎片化**：长期运行的系统可能出现内存碎片，大内存分配失败但总空闲内存充足
2. **Swap 性能**：使用 Swap 会导致显著的性能下降，应尽量避免频繁 Swap
3. **THP 延迟**：透明大页面分配可能导致延迟尖峰，数据库等低延迟应用建议关闭
4. **OOM 不可预测性**：OOM Killer 的行为受多种因素影响，生产环境应做好内存监控和限制
5. **cgroup 内存统计**：cgroup 内存统计包含缓存，理解 `memory.current` 与 `memory.swap.current` 的区别

## 相关概念

- **DMA（Direct Memory Access）**：设备直接访问内存的技术，详见 `[[DMA（Direct Memory Access）技术详解]]`
- **Linux I/O 模型**：内存管理与 I/O 密切相关，详见 `[[Linux I_O 模型：BIO、NIO、AIO 与系统实现]]`
- **Linux 文件系统**：页面缓存与文件系统交互，详见 `[[Linux 文件系统架构与设计详解]]`
- **Java 零拷贝**：利用内存映射和 sendfile 减少数据拷贝，详见 `[[Java 零拷贝（Zero-Copy）实现详解]]`

## 参考资料

- [Linux 内核文档 - 内存管理](https://docs.kernel.org/admin-guide/mm/index.html)
- [Linux 内核内存管理子系统文档](https://docs.kernel.org/mm/index.html)
- [Linux 内核 MM API](https://www.kernel.org/doc/html/latest/core-api/mm_api.html)
- [Understanding the Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/html/understand/understand006.html) - Mel Gorman 经典著作
- [Linux man pages - swapon](https://man7.org/linux/man-pages/man2/swapon.2.html)
- [Linux man pages - mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)
- [cgroup v2 内存控制器文档](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)
