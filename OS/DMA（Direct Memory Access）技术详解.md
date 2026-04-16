---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - dma
  - os
  - hardware
  - io
  - performance
  - bus-mastering
  - cache-coherency
aliases:
  - DMA
  - Direct Memory Access
  - 直接内存访问
  - DMA 控制器
  - DMAC
source_type: mixed
source_urls:
  - 'https://en.wikipedia.org/wiki/Direct_memory_access'
  - 'https://www.kernel.org/doc/html/latest/core-api/dma-api.html'
  - 'https://wiki.osdev.org/ISA_DMA'
status: verified
---

## 定义

**DMA（Direct Memory Access，直接内存访问）** 是计算机系统的一种硬件特性，允许特定的硬件子系统（如磁盘控制器、网卡、显卡等）**独立于 CPU** 直接访问主系统内存 [^wiki-dma]。

在没有 DMA 的情况下，CPU 使用**程序化 I/O（Programmed I/O, PIO）** 进行数据读写时，需要在整个传输过程中持续参与每一个字节的搬运，无法执行其他工作。引入 DMA 后，CPU 只需发起传输请求，之后可以继续执行其他任务，待 DMA 控制器（DMAC）完成传输后通过**中断（Interrupt）** 通知 CPU [^wiki-dma]。

## 为什么重要

DMA 的核心价值在于**解放 CPU**，使其从繁重的数据搬运工作中脱身：

- **降低 CPU 占用**：数据传输由专用硬件完成，CPU 仅需初始化和收尾处理
- **提升系统吞吐**：CPU 计算与 I/O 传输可并行进行
- **降低传输延迟**：专用 DMA 引擎的传输速率通常高于 CPU 逐字节搬运
- **高吞吐场景必需**：当网络链路带宽接近或超过 CPU 处理能力时，DMA 成为避免 CPU 成为瓶颈的关键技术 [^wiki-dma]

## 工作原理

### 基本流程

一次典型的 DMA 传输包含以下步骤：

1. **初始化**：CPU 向 DMA 控制器写入传输参数：
   - 源地址（Source Address）
   - 目标地址（Destination Address）
   - 传输字节数（Byte Count）
   - 传输方向（读设备 / 写设备）
2. **启动传输**：CPU 命令外设发起数据传输
3. **传输执行**：DMA 控制器接管系统总线，直接在设备和内存之间搬运数据
4. **完成通知**：传输完成后，DMA 控制器向 CPU 发送中断信号

### 两种实现模式

| 模式 | 说明 | 典型总线 |
|------|------|----------|
| **第三方 DMA（Third-party DMA）** | 使用独立的 DMA 控制器芯片，由 DMAC 生成地址和控制信号 | ISA、早期 PATA |
| **总线主控（Bus Mastering / First-party DMA）** | 外设自身成为总线主控者，直接读写系统内存，无需中央 DMAC | PCI、PCIe、AHB |

**现代系统（PCI/PCIe）几乎全部采用 Bus Mastering 模式** [^wiki-dma]。PCI 架构没有中央 DMA 控制器，每个 PCI 设备可以向 PCI 总线控制器请求总线所有权，获得授权后直接发起内存读写 [^wiki-pci]。

## 工作模式

DMA 控制器有三种主要的工作模式 [^wiki-modes]：

### Burst Mode（突发模式）

- 一次性传输整个数据块，期间 CPU 被完全挂起
- 传输效率最高，但 CPU 会被阻塞较长时间
- 也称 "Block Transfer Mode"

### Cycle Stealing Mode（周期窃取模式）

- 每传输一个数据单元就将总线控制权交还 CPU
- 通过不断请求和释放总线，实现 CPU 指令执行与 DMA 数据传输的交错
- 传输速度较慢，但 CPU 不会被长时间阻塞
- 适用于实时监控数据的控制器

### Transparent Mode（透明模式）

- 仅在 CPU 不使用系统总线时传输数据
- CPU 从不暂停执行程序，DMA 传输在时间上"免费"
- 传输整个数据块所需时间最长，但系统整体性能最优
- 也称 "Hidden DMA data transfer mode"

## 缓存一致性（Cache Coherency）

DMA 操作可能引发**缓存一致性问题** [^wiki-cache]：

### 问题场景

1. **DMA 写入时**：设备通过 DMA 将数据写入内存，但 CPU 缓存中可能持有该内存地址的旧值（stale value）。如果缓存未及时失效，CPU 会读到过期数据。
2. **DMA 读取时**：CPU 修改了数据但仅更新了缓存（Write-back Cache），尚未写回内存。此时设备通过 DMA 读取到的仍是内存中的旧值。

### 解决方案

| 方案 | 机制 | 适用场景 |
|------|------|----------|
| **硬件方案：Bus Snooping（总线嗅探）** | 外部写入时通知缓存控制器，自动执行缓存失效或刷新 | 缓存一致性系统（Cache-coherent systems） |
| **软件方案：OS 管理** | 操作系统在 DMA 传输前刷新（flush）缓存行，传输后失效（invalidate）缓存行 | 非一致性系统（Non-coherent systems） |
| **混合方案** | L2 缓存硬件一致，L1 缓存软件管理 | 部分嵌入式系统 |

**Linux 内核中的处理**：对于非一致性系统，驱动必须使用 `dma_sync_single_for_device()` 和 `dma_sync_single_for_cpu()` 显式管理内存所有权 [^kernel-dma-api]。

## Linux 内核 DMA API

Linux 内核提供了统一的 DMA API（定义在 `<linux/dma-mapping.h>`），屏蔽了不同架构的差异 [^kernel-dma-api]。

### 核心概念

- **`dma_addr_t`**：可保存任何有效的 DMA 地址，可传递给设备作为 DMA 源/目标地址。CPU 不能直接引用 `dma_addr_t`，因为 DMA 地址空间与物理地址空间之间可能存在转换（如 IOMMU）
- **DMA Mask**：设备可寻址的位掩码，例如 32 位设备只能寻址 4 GB 以下内存

### 两类 DMA 映射

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **Coherent（一致性映射）** | CPU 和设备的读写立即可见，无需手动同步 | DMA 描述符、控制结构等小缓冲区 |
| **Streaming（流式映射）** | 映射已存在的缓冲区，传输前后需手动同步 | 大数据块的一次性传输 |

### 常用 API

**一致性内存分配**：

```c
// 分配一致性内存
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t flag);

// 释放一致性内存
void dma_free_coherent(struct device *dev, size_t size,
                       void *cpu_addr, dma_addr_t dma_handle);
```

**流式 DMA 映射**：

```c
// 映射单个缓冲区
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr,
                          size_t size, enum dma_data_direction direction);

// 解除映射
void dma_unmap_single(struct device *dev, dma_addr_t dma_addr,
                      size_t size, enum dma_data_direction direction);

// 检查映射是否成功
int dma_mapping_error(struct device *dev, dma_addr_t dma_addr);
```

**Scatter/Gather 映射**（分散/聚集 I/O）：

```c
// 映射 scatterlist
int dma_map_sg(struct device *dev, struct scatterlist *sg,
               int nents, enum dma_data_direction direction);

// 解除映射
void dma_unmap_sg(struct device *dev, struct scatterlist *sg,
                  int nents, enum dma_data_direction direction);
```

**方向枚举**：

| 值 | 含义 |
|----|------|
| `DMA_TO_DEVICE` | 数据从内存到设备 |
| `DMA_FROM_DEVICE` | 数据从设备到内存 |
| `DMA_BIDIRECTIONAL` | 双向传输（需同步两次） |

### DMA Pool（小对象分配）

对于需要大量小型 DMA 一致内存的场景（如 DMA 描述符），使用 DMA Pool 比 `dma_alloc_coherent()` 更高效 [^kernel-dma-api]：

```c
struct dma_pool *dma_pool_create(const char *name, struct device *dev,
                                  size_t size, size_t align, size_t boundary);
void *dma_pool_alloc(struct dma_pool *pool, gfp_t mem_flags, dma_addr_t *handle);
void dma_pool_free(struct dma_pool *pool, void *vaddr, dma_addr_t dma);
void dma_pool_destroy(struct dma_pool *pool);
```

## Scatter/Gather DMA

**Scatter/Gather（分散/聚集）** 也称 **Vectored I/O**，允许在单次 DMA 事务中向/从多个不连续的内存区域传输数据 [^wiki-dma]。

**工作原理**：

- 将多个简单的 DMA 请求链式连接
- 设备驱动提供一个 scatterlist，描述多个内存片段
- DMA 引擎自动依次传输每个片段

**优势**：

- 减少中断次数和数据拷贝
- 支持非连续内存的高效传输
- 现代网卡、存储控制器广泛使用

## 典型应用

### 磁盘控制器

- 硬盘读写时，数据直接在磁盘缓冲区和内存之间传输
- 现代 SATA/NVMe 控制器均使用 Bus Mastering DMA

### 网卡（NIC）

- 网络数据包直接 DMA 到内存中的 Ring Buffer
- Linux 网络驱动使用 NAPI + DMA 实现高吞吐网络 I/O

### 显卡（GPU）

- 纹理、顶点数据通过 DMA 从系统内存传输到显存
- PCIe Bus Mastering 是 GPU 与 CPU 内存通信的基础

### 声卡

- 音频流数据通过 DMA 传输，避免 CPU 中断风暴
- 传统 ISA Sound Blaster 使用 DMA 通道 1

### 内存到内存拷贝

- Intel I/OAT（I/O Acceleration Technology）：集成在 Xeon 芯片组中的 DMA 引擎，可卸载内存拷贝操作 [^wiki-ioat]
- 但实际测试表明，在网络流量拷贝场景中，I/OAT 对 CPU 利用率的改善不超过 10% [^linuxnet-ioat]

## 现代增强技术

### DDIO（Data Direct I/O）

Intel Xeon E5 处理器引入的 DDIO 技术允许 DMA 窗口驻留在 **CPU 缓存（L3 Cache）** 而非系统 RAM 中 [^wiki-ddio]：

- NIC 可直接 DMA 到本地 CPU 的末级缓存
- 避免从系统 RAM 获取 I/O 数据的开销
- 降低 I/O 处理延迟，减少 RAM 带宽瓶颈

### IOMMU

**IOMMU（I/O Memory Management Unit）** 为 DMA 提供地址转换服务 [^wiki-dma]：

- 将设备的 DMA 地址翻译为物理内存地址
- 解决 32 位设备无法寻址 4 GB 以上内存的问题
- 提供 DMA 隔离和安全保护（防止恶意设备访问任意内存）
- Linux 中对应实现为 Intel VT-d / AMD-Vi

### RDMA（Remote Direct Memory Access）

RDMA 允许一台计算机直接访问另一台计算机的内存，**无需双方操作系统内核参与**：

- 零内核旁路（Kernel Bypass）
- 极低延迟、极高吞吐
- 应用于 InfiniBand、RoCE（RDMA over Converged Ethernet）、iWARP

## 限制与注意事项

### DMA 地址限制

- 32 位设备只能寻址 4 GB 以下内存
- 操作系统可能需要使用 **Bounce Buffer（弹跳缓冲区）** 或 IOMMU 来解决 [^wiki-dma]
- Linux 中通过 SWIOTLB（Software I/O TLB）实现 bounce buffer [^kernel-swiotlb]

### 缓存一致性开销

- 非一致性系统中，每次 DMA 传输前后都需要刷新/失效缓存行
- 频繁的小块 DMA 传输可能因缓存同步开销而得不偿失

### 对齐要求

- DMA 缓冲区通常需要按缓存行大小对齐
- Linux 内核建议 DMA 映射区域按页边界对齐（页边界保证也是缓存行边界）[^kernel-dma-api]

### 安全考虑

- **DMA 攻击**：恶意外设可通过 DMA 直接读取/修改系统内存
- 现代系统通过 IOMMU 和 VT-d/AMD-Vi 提供 DMA 隔离
- Thunderbolt、FireWire 等外部高速接口是主要攻击面

## 相关概念

- [[零拷贝（Zero-Copy）技术详解]]：DMA 是零拷贝技术的硬件基础之一（sendfile 依赖 DMA 将磁盘数据直接传输到网卡）
- [[Kafka 是如何实现高性能的]]：Kafka 使用 sendfile + DMA 实现零拷贝网络传输
- IOMMU：为 DMA 提供地址转换和隔离
- Scatter/Gather I/O：DMA 的高级传输模式
- Bus Mastering：现代 DMA 的实现方式
- RDMA：跨网络的直接内存访问

## 参考资料

[^wiki-dma]: [Direct memory access - Wikipedia](https://en.wikipedia.org/wiki/Direct_memory_access)
[^wiki-pci]: [PCI - Wikipedia: Bus mastering section](https://en.wikipedia.org/wiki/Direct_memory_access#PCI)
[^wiki-cache]: [Cache coherency - Wikipedia](https://en.wikipedia.org/wiki/Direct_memory_access#Cache_coherency)
[^wiki-ioat]: [I/O Acceleration Technology - Wikipedia](https://en.wikipedia.org/wiki/Direct_memory_access#I/OAT)
[^wiki-ddio]: [Data Direct I/O - Wikipedia](https://en.wikipedia.org/wiki/Direct_memory_access#DDIO)
[^linuxnet-ioat]: [I/OAT on LinuxNet wiki - Andrew Grover (2006)](https://web.archive.org/web/20160505034410/http://www.linuxfoundation.org/collaborate/workgroups/networking/i/oat)
[^kernel-dma-api]: [Dynamic DMA mapping using the generic device - Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/core-api/dma-api.html)
[^kernel-swiotlb]: [DMA and swiotlb - Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/core-api/swiotlb.html)
[^osdev-isa-dma]: [ISA DMA - OSDev Wiki](https://wiki.osdev.org/ISA_DMA)
