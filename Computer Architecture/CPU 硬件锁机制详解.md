---
created: 2026-04-16
updated: 2026-04-16
tags:
  - 计算机体系结构
  - 并发编程
  - CPU
  - 原子操作
  - 同步原语
  - x86
  - ARM
  - RISC-V
  - 底层原理
aliases:
  - CPU Lock Mechanisms
  - Hardware Lock Support
  - 原子指令
  - CAS
  - LL/SC
  - CPU 并发锁
source_type: mixed
source_urls:
  - https://en.wikipedia.org/wiki/Compare-and-swap
  - https://en.wikipedia.org/wiki/Load-link/store-conditional
  - https://en.wikipedia.org/wiki/MESI_protocol
  - https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
  - https://developer.arm.com/documentation/ddi0487/latest
  - https://riscv.org/technical/specifications/
status: verified
---

## 概述

并发编程中的"锁"最终需要 CPU 硬件提供原子操作支持。不同 CPU 架构（x86、ARM、RISC-V、PowerPC 等）在指令集层面采用了不同的设计哲学来实现原子读-改-写（Read-Modify-Write, RMW）操作。理解这些硬件机制是理解上层锁实现（mutex、spinlock、CAS 无锁结构）的基础。

## 前置概念

### 原子操作（Atomic Operation）

原子操作是不可分割的操作序列，在执行过程中不会被其他线程或处理器中断。在多核系统中，原子性需要两个层面的保证：

- **指令级原子**：单条指令在执行期间不被其他指令打断
- **内存级原子**：操作涉及的内存地址在多核间保持一致视图

### 缓存一致性（Cache Coherence）

现代多核 CPU 每个核心都有独立的 L1/L2 缓存。缓存一致性协议确保多个核心看到的同一内存地址的值是一致的。主流协议包括：

| 协议 | 状态数 | 特点 | 采用者 |
|------|--------|------|--------|
| MESI | 4 (Modified/Exclusive/Shared/Invalid) | 最广泛使用的写回缓存协议 | Intel、AMD |
| MESIF | 5 (MESI + Forward) | 减少 Shared 状态下的冗余响应 | Intel（多路系统） |
| MOESI | 5 (MESI + Owned) | 减少回写主存的开销 | AMD |
| Dragon | 4 | 更新型协议，适合写共享场景 | 早期 DEC Alpha |

**MESI 协议核心状态**：

- **Modified (M)**：缓存行已修改（脏数据），仅存在于当前缓存
- **Exclusive (E)**：缓存行未修改，仅存在于当前缓存（可无总线事务直接修改）
- **Shared (S)**：缓存行未修改，可能存在于多个缓存中
- **Invalid (I)**：缓存行无效

缓存一致性是硬件锁机制的底层基础——所有原子指令最终依赖缓存一致性协议来协调多核对同一内存地址的访问。

### 内存序（Memory Order）

内存序定义了内存操作在不同核心间的可见顺序。不同架构的内存序模型差异显著：

| 架构 | 内存序模型 | 特点 |
|------|-----------|------|
| x86 (Intel/AMD) | TSO (Total Store Order) | 强内存序，仅允许 Store-Load 重排序 |
| ARM | Weak Memory Order | 弱内存序，允许更多重排序，需显式屏障 |
| RISC-V | Weak Memory Order（RVWMO） | 弱内存序，通过 FENCE 指令控制 |
| PowerPC | Weak Memory Order | 最弱的内存序模型之一 |

## x86 架构（Intel / AMD）

### 核心机制：LOCK 前缀指令

x86 架构通过 `LOCK` 前缀配合特定指令实现原子操作。`LOCK` 前缀会锁定对应的缓存行（现代实现）或总线（早期实现），确保指令执行期间其他核心无法访问同一地址。

**支持 LOCK 前缀的指令**：

| 指令 | 功能 | 说明 |
|------|------|------|
| `LOCK CMPXCHG` | 比较并交换（CAS） | 最核心的原子指令，无锁编程基础 |
| `LOCK CMPXCHG8B` | 64 位 CAS（32 位模式） | 双字比较交换 |
| `LOCK CMPXCHG16B` | 128 位 CAS（64 位模式） | 四字比较交换，Windows 8.1+ 要求支持 |
| `LOCK XADD` | 交换并相加 | 用于原子加法 |
| `LOCK XCHG` | 交换 | XCHG 隐式带 LOCK 前缀 |
| `LOCK ADD/SUB/AND/OR/XOR` | 原子算术/位运算 | 原子修改内存值 |
| `LOCK BTS/BTR/BTC` | 原子位操作 | 测试并设置/复位/取反位 |

### CMPXCHG 指令详解

`CMPXCHG`（Compare and Exchange）是 x86 实现 CAS 的核心指令：

```assembly
; CMPXCHG dest, src
; 伪代码逻辑：
; if (accumulator == dest) {
;     dest = src;
;     ZF = 1;  // 成功标志
; } else {
;     accumulator = dest;
;     ZF = 0;  // 失败标志
; }
```

在多核环境下必须加 `LOCK` 前缀：`LOCK CMPXCHG`。

**Intel SDM 中的行为描述**：

- 比较目标操作数与累加器（AL/AX/EAX/RAX）的值
- 相等则将源操作数写入目标，设置 ZF=1
- 不等则将目标值加载到累加器，设置 ZF=0

### 隐式锁定的 XCHG

`XCHG` 指令在涉及内存操作数时**隐式带有 LOCK 前缀**，无需显式声明：

```assembly
XCHG EAX, [mem]  ; 等价于 LOCK XCHG EAX, [mem]
```

这一特性使 `XCHG` 成为实现简单自旋锁（spinlock）的高效选择。

### 内存屏障指令

x86 的 TSO 模型相对较强，但仍需要屏障指令控制 Store-Load 重排序：

| 指令 | 功能 |
|------|------|
| `MFENCE` | 全内存屏障（所有操作排序） |
| `SFENCE` | 存储屏障（Store 操作排序） |
| `LFENCE` | 加载屏障（Load 操作排序） |
| `XACQUIRE`/`XRELEASE` | TSX 事务内存前缀（Haswell 引入） |

### 硬件限制与注意事项

- `CMPXCHG16B` 指令需要 CPU 和主板同时支持，部分早期 Core 2 主板存在限制
- `LOCK` 前缀只能用于特定指令，用于不支持的指令会触发 `#UD` 异常
- 锁定的内存操作数必须是内存地址，不能是寄存器
- 在多路系统中，Intel 使用 MESIF 协议（增加 Forward 状态）优化共享缓存行的读取

## ARM 架构

### 演进历程

ARM 的原子操作支持经历了显著的演进：

| ARM 版本 | 原子操作机制 | 说明 |
|----------|-------------|------|
| ARMv5 及更早 | `SWP` 指令 | 已废弃，多核下性能差 |
| ARMv6 / ARMv7 | `LDREX`/`STREX` | LL/SC 风格，主流方案 |
| ARMv8-A (AArch64) | `LDXR`/`STXR` | 64 位 LL/SC |
| ARMv8.1+ | LSE (Large System Extensions) | 新增 `CAS`、`LDADD` 等直接原子指令 |

### LDREX/STREX 机制（LL/SC 风格）

ARMv6 引入的 `LDREX`（Load Exclusive）和 `STREX`（Store Exclusive）构成一对 Load-Link/Store-Conditional 指令：

```assembly
; 原子递增示例
retry:
    LDREX R0, [R1]      ; 加载并标记独占访问
    ADD   R0, R0, #1    ; 修改值
    STREX R2, R0, [R1]  ; 条件存储，R2=0 成功，R2=1 失败
    CMP   R2, #0
    BNE   retry          ; 失败则重试
```

**工作机制**：

1. `LDREX` 加载内存值并在硬件中标记该地址为"独占监视"状态
2. 如果在 `LDREX` 和 `STREX` 之间有其他核心修改了同一地址（或同一监控粒度区域），`STREX` 将失败
3. `STREX` 失败时返回非零值，需要重新执行整个序列

**重要限制**：

- ARM 的监控粒度是**实现定义的**（implementation-defined），可能是 8 字节到 2048 字节不等
- 同一监控区域内任何其他内存访问都可能导致 `STREX` 失败（虚假失败/spurious failure）
- 上下文切换、异常、甚至某些调试操作都会清除独占监视标记
- 不能在 `LDREX`/`STREX` 对之间调用可能触发上下文切换的代码

### ARMv8.1 LSE 扩展

ARMv8.1 引入的 Large System Extensions（LSE）新增了直接原子指令，更接近 x86 的 CAS 模型：

| 指令 | 功能 |
|------|------|
| `CAS`/`CASA`/`CASAL`/`CASL` | 比较并交换（带不同内存序语义） |
| `LDADD`/`LDADDA`/`LDADDAL`/`LDADDL` | 原子加并返回旧值 |
| `LDSET`/`LDSETA`/`LDSETAL`/`LDSETL` | 原子按位或并返回旧值 |
| `LDCLR`/`LDCLRA`/`LDCLRAL`/`LDCLRL` | 原子按位与（清除）并返回旧值 |
| `LDEOR`/`LDEORA`/`LDEORAL`/`LDEORL` | 原子按位异或并返回旧值 |
| `SWP`/`SWPA`/`SWPAL`/`SWPL` | 原子交换（注意：与废弃的 SWP 指令不同） |

后缀含义：

- 无后缀：无内存屏障
- `A`：Acquire 语义（后续操作不会被重排序到该操作之前）
- `L`：Release 语义（之前操作不会被重排序到该操作之后）
- `AL`：Acquire + Release 语义

LSE 指令在多核高竞争场景下性能显著优于 LL/SC 重试循环，因为它们是单条原子指令而非重试循环。

### 内存屏障指令

ARM 采用弱内存序模型，必须显式使用屏障指令：

| 指令 | 功能 |
|------|------|
| `DMB` (Data Memory Barrier) | 数据内存屏障，确保屏障前后的内存访问顺序 |
| `DSB` (Data Synchronization Barrier) | 数据同步屏障，等待所有显式内存操作完成 |
| `ISB` (Instruction Synchronization Barrier) | 指令同步屏障，刷新流水线 |

## RISC-V 架构

### 核心机制：LR/SC 指令对

RISC-V 采用 Load-Reserved/Store-Conditional 指令对实现原子操作：

| 指令 | 功能 |
|------|------|
| `LR.W` | 加载保留字（32 位） |
| `LR.D` | 加载保留双字（64 位，RV64） |
| `SC.W` | 条件存储字，成功返回 0，失败返回非 0 |
| `SC.D` | 条件存储双字（RV64） |

```assembly
# 原子递增示例（RISC-V 汇编）
retry:
    lr.w   t0, (a0)      # 加载并设置保留
    addi   t1, t0, 1     # 修改值
    sc.w   t2, t1, (a0)  # 条件存储
    bnez   t2, retry     # 失败则重试
```

### 架构保证

RISC-V 规范对 LR/SC 有明确的架构保证：

- **保留集（Reservation Set）**：LR 指令标记一个"保留集"，SC 仅在保留集未被其他 hart（hardware thread）写入时才成功
- **有限长度保证**：规范保证有限长度的 LR/SC 序列最终会成功（progress guarantee），防止活锁
- 保留集的大小是实现定义的，但必须包含被 LR 访问的地址
- 在 LR 和 SC 之间执行另一条 LR 指令会清除之前的保留

### 内存序控制

RISC-V 通过 `FENCE` 指令控制内存序：

```assembly
FENCE [pred][succ]
```

- `pred` 和 `succ` 是位掩码，指定哪些操作类型需要排序
- `I` = Input (Load), `O` = Output (Store), `R` = Read (device), `W` = Write (device)
- 常见用法：`FENCE RW, RW`（全内存屏障）、`FENCE TSO`（模拟 x86 TSO）

### A 扩展（原子操作扩展）

RISC-V 的 A 扩展（Atomic Extension）除了 LR/SC 外，还定义了**原子内存操作（AMO）**指令：

| 指令 | 功能 |
|------|------|
| `AMOSWAP.W/D` | 原子交换 |
| `AMOADD.W/D` | 原子加 |
| `AMOAND.W/D` | 原子按位与 |
| `AMOOR.W/D` | 原子按位或 |
| `AMOXOR.W/D` | 原子按位异或 |
| `AMOMAX.W/D` | 原子取最大值 |
| `AMOMINU.W/D` | 原子取最小值（无符号） |

AMO 指令是单条原子指令，不需要 LR/SC 重试循环，在低竞争场景下更高效。

## PowerPC 架构

### 核心机制：lwarx/stwcx

PowerPC 使用 LL/SC 风格的指令对：

| 指令 | 功能 |
|------|------|
| `lwarx` | Load Word And Reserve Indexed |
| `stwcx.` | Store Word Conditional Indexed |
| `ldarx` | Load Doubleword And Reserve Indexed（64 位） |
| `stdcx.` | Store Doubleword Conditional Indexed（64 位） |

PowerPC 的 LL/SC 实现相对"强"——允许在 `lwarx` 和 `stwcx.` 之间包装对其他缓存行的加载甚至存储操作，这使得它可以实现一些在其他架构上需要 DCAS（双字 CAS）才能实现的无锁算法，如无锁引用计数。

### 内存屏障

| 指令 | 功能 |
|------|------|
| `sync` ( heavyweight ) | 全系统同步屏障 |
| `lwsync` | 轻量级同步屏障 |
| `isync` | 指令同步屏障 |

## 架构对比总结

| 特性 | x86 (Intel/AMD) | ARM | RISC-V | PowerPC |
|------|-----------------|-----|--------|---------|
| 核心原子机制 | LOCK 前缀 + RMW 指令 | LL/SC (LDREX/STREX) + LSE CAS | LR/SC + AMO | LL/SC (lwarx/stwcx) |
| 直接 CAS 指令 | `CMPXCHG` | `CAS` (ARMv8.1+ LSE) | 无（用 LR/SC 模拟） | 无（用 lwarx/stwcx 模拟） |
| 内存序模型 | TSO（强） | Weak（弱） | Weak（弱） | Weak（弱） |
| 缓存一致性协议 | MESI / MESIF | 实现定义（通常 MESI 变体） | 实现定义 | 实现定义 |
| 编程模型复杂度 | 较低（强内存序减少屏障需求） | 较高（需显式屏障） | 中等（FENCE 控制） | 较高（需显式屏障） |
| 高竞争场景性能 | 较好（单条原子指令） | LSE 较好，LL/SC 重试开销大 | AMO 较好，LR/SC 重试有开销 | 较好（允许包装操作） |

## 从硬件指令到软件锁

### 自旋锁（Spinlock）

最简单的锁实现，基于原子交换或 CAS：

```c
// 基于 XCHG 的自旋锁（x86）
void lock(int *mutex) {
    while (__atomic_exchange_n(mutex, 1, __ATOMIC_ACQUIRE)) {
        while (*mutex) { /* 忙等待 */ }
    }
}

void unlock(int *mutex) {
    __atomic_store_n(mutex, 0, __ATOMIC_RELEASE);
}
```

### 互斥锁（Mutex）

操作系统级互斥锁通常结合原子操作和内核调度：

- **快速路径**：使用 CAS 尝试获取锁（用户态原子操作）
- **慢速路径**：获取失败时进入内核等待队列（如 Linux futex）

### 无锁数据结构

CAS 是构建无锁（lock-free）和无等待（wait-free）数据结构的核心原语：

- Treiber Stack（无锁栈）
- Michael-Scott Queue（无锁队列）
- 各种无锁哈希表和跳表

**ABA 问题**：CAS 的一个经典陷阱。值从 A 变为 B 再变回 A 时，CAS 无法检测到中间的变化。解决方案包括：

- 使用双字 CAS（DCAS），附加版本号/计数器
- 使用安全内存回收（SMR）/无锁垃圾回收
- 使用 LL/SC（天然不受 ABA 影响，因为监控的是写入事件而非值本身）

## 常见误区

1. **"原子操作不需要缓存一致性"**：错误。原子操作的正确性完全依赖缓存一致性协议。没有 MESI 等协议，多核间的原子性无法保证。

2. **"LL/SC 比 CAS 更强"**：部分正确。LL/SC 能检测到 A-B-A 变化中的写入事件，但实际实现中 LL/SC 可能因无关原因（如上下文切换）而虚假失败。

3. **"x86 不需要内存屏障"**：不准确。x86 的 TSO 模型虽然强，但 Store-Load 重排序仍然存在，特定场景仍需要 `MFENCE`。

4. **"LOCK 前缀锁定总线"**：过时。现代 x86 处理器使用缓存行锁定（通过缓存一致性协议），而非总线锁定。仅在访问不可缓存内存时才真正锁定总线。

5. **"ARMv8 只有 LL/SC"**：不准确。ARMv8.1 引入 LSE 扩展后，ARM 也支持直接的 CAS 指令，且在高竞争场景下性能更好。

## 参考资料

- Intel 64 and IA-32 Architectures Software Developer's Manual (SDM) — <https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html>
- ARM Architecture Reference Manual (ARMv8-A) — <https://developer.arm.com/documentation/ddi0487/latest>
- The RISC-V Instruction Set Manual, Volume I: User-Level ISA — <https://riscv.org/technical/specifications/>
- Wikipedia: Compare-and-swap — <https://en.wikipedia.org/wiki/Compare-and-swap>
- Wikipedia: Load-link/store-conditional — <https://en.wikipedia.org/wiki/Load-link/store-conditional>
- Wikipedia: MESI protocol — <https://en.wikipedia.org/wiki/MESI_protocol>
- David S. Miller: Semantics and Behavior of Atomic and Bitmask Operations (Linux Kernel Documentation) — <https://www.kernel.org/doc/Documentation/atomic_ops.txt>
- Tudor David et al.: "Everything you always wanted to know about synchronization but were afraid to ask" (SOSP 2013)
