---
created: '2026-04-23'
updated: '2026-04-23'
tags:
  - java
  - jvm
  - gc
  - garbage-collector
  - memory-management
  - performance
aliases:
  - Java GC
  - JVM GC
  - 垃圾回收器
  - Garbage Collector
source_type: mixed
source_urls:
  - 'https://docs.oracle.com/en/java/javase/17/gctuning/'
  - 'https://openjdk.org/jeps/333'
  - 'https://openjdk.org/jeps/439'
  - 'https://openjdk.org/jeps/189'
  - 'https://openjdk.org/jeps/318'
  - 'https://wiki.openjdk.org/display/shenandoah/Main'
status: verified
---
## 概述

Java HotSpot VM 提供多种垃圾收集器（Garbage Collector, GC），每种设计目标不同：有的追求吞吐量，有的追求低延迟，有的追求最小内存占用。选择合适的 GC 对应用性能至关重要。

---

## 前置概念

### 分代假说（Generational Hypothesis）

绝大多数对象"朝生夕死"——创建后很快变得不可达。少数对象会长期存活。基于此假说，现代 GC 将堆分为：

- **Young Generation（新生代）**：存放新创建对象，GC 频繁但快速
- **Old Generation（老年代）**：存放长期存活对象，GC 较少但耗时

新生代进一步分为：Eden 区、Survivor 区（通常两个：S0、S1）。

### Stop-The-World（STW）

GC 过程中，应用线程全部暂停，仅 GC 线程运行。STW 时间直接影响应用响应延迟。

### 并发 vs 并行

- **并行（Parallel）**：多 GC 线程同时工作，但应用线程暂停（STW）
- **并发（Concurrent）**：GC 线程与应用线程同时运行，尽量减少 STW

### 安全点（Safe Point）

应用线程只有在到达安全点时才能暂停。时间到安全点（Time To Safe Point, TTSP）是 GC 暂停的一部分，但非 GC 本身造成。

---

## JDK 垃圾收集器总览

| GC | 引入版本 | 默认版本 | 状态 | 暂停特性 | 适用场景 |
|---|---------|---------|------|---------|---------|
| Serial GC | JDK 1.0 | — | 生产可用 | STW，单线程 | 小内存、单核、客户端应用 |
| Parallel GC | JDK 5 | JDK 5–8 | 生产可用 | STW，多线程 | 批处理、吞吐优先 |
| CMS | JDK 5 | — | JDK 9 废弃，JDK 14 移除 | 并发标记，STW 复制 | 已废弃，不推荐 |
| G1 GC | JDK 7u4 | JDK 9+ | 生产可用 | 混合，可控延迟 | 通用服务端应用 |
| ZGC | JDK 11（实验） | — | JDK 15 生产 | <10ms，并发 | 低延迟、大堆 |
| Shenandoah | JDK 12（实验） | — | JDK 15 生产 | <10ms，并发 | 低延迟 |
| Epsilon GC | JDK 11（实验） | — | 实验性 | 无 GC | 性能测试、短生命周期任务 |

---

## Serial GC

### 原理

单线程执行所有 GC 工作。新生代采用复制算法（Eden → Survivor），老年代采用标记-压缩算法（Mark-Compact）。

### 工作流程

1. **Minor GC**：暂停应用，扫描新生代根集，复制存活对象到 Survivor，清空 Eden
2. **Major/Full GC**：暂停应用，标记老年代存活对象，压缩整理

### 特性

- **优点**：实现简单，内存占用小，无多线程协调开销
- **缺点**：STW 时间与堆大小成正比，大堆时暂停可达数秒
- **吞吐量**：单线程 GC 效率低，不适合高并发应用

### 启用方式

```bash
java -XX:+UseSerialGC ...
```

### 适用场景

- 客户端应用、小程序
- 单核机器或内存 <100MB
- 对延迟不敏感的批处理任务

---

## Parallel GC（Parallel Scavenge + Parallel Old）

### 原理

多线程并行执行 GC，追求最大吞吐量。新生代采用 Parallel Scavenge（复制算法），老年代采用 Parallel Old（标记-压缩算法）。

### 工作流程

与 Serial GC 类似，但使用多个 GC 线程并行执行标记、复制、压缩。

### 特性

- **优点**：多核环境下吞吐量最高
- **缺点**：STW 暂停仍较长，与堆大小和存活对象数量正相关
- **可配置目标**：
  - `-XX:MaxGCPauseMillis`：最大暂停时间目标
  - `-XX:GCTimeRatio`：吞吐量目标（GC 时间占比）

### 启用方式

```bash
java -XX:+UseParallelGC ...
# 同时启用 Parallel Old（JDK 6u23+ 默认）
java -XX:+UseParallelOldGC ...
```

### 适用场景

- 批处理任务、报表生成、离线分析
- 对吞吐量敏感、对延迟容忍度较高的后台服务
- JDK 8 及之前的服务端默认 GC

---

## CMS（Concurrent Mark Sweep）

> **注意**：CMS 已于 JDK 9 废弃，JDK 14 移除。不建议新项目使用。

### 原理

并发标记老年代存活对象，减少 STW 时间。采用标记-清除算法，不压缩。

### 工作流程

1. **初始标记（Initial Mark）**：STW，仅标记根集直接引用的对象
2. **并发标记（Concurrent Mark）**：并发遍历对象图
3. **预清理（Preclean）**：并发处理并发标记期间变更的引用
4. **最终标记（Final Remark）**：STW，完成标记
5. **并发清除（Concurrent Sweep）**：并发回收未标记对象
6. **重置（Reset）**：并发重置数据结构

### 特性

- **优点**：老年代 GC 大部分并发，STW 时间短
- **缺点**：
  - 不压缩，导致内存碎片
  - CPU 占用高（并发线程）
  - 浮动垃圾（并发标记期间新产生的垃圾无法回收）
  - 可能触发 Full GC（碎片严重时，使用 Serial Old 压缩）
- **已废弃原因**：维护成本高、碎片问题严重、无法满足现代低延迟需求

### 启用方式（仅限 JDK 8）

```bash
java -XX:+UseConcMarkSweepGC ...
```

---

## G1 GC（Garbage First）

### 原理

Region 化堆结构。堆被划分为多个大小相等的 Region（1MB–32MB），每个 Region 可作为 Eden、Survivor 或 Old。G1 优先回收垃圾最多的 Region（"Garbage First"）。

### 工作流程

1. **Young GC**：STW，回收 Eden + Survivor Region
2. **并发标记周期**：
   - 初始标记（STW，随 Young GC）
   - 并发标记
   - 最终标记（STW）
   - 混合回收（Mixed GC）：回收部分 Old Region + Young Region
3. **Full GC**：STW，当并发回收不足以应对分配压力时触发

### 特性

- **优点**：
  - 暂停时间可控（通过 `-XX:MaxGCPauseMillis`）
  - 无全堆压缩，按 Region 回收，减少碎片
  - 大堆（>4GB）表现良好
- **缺点**：
  - 并发标记非并发复制，混合回收仍有 STW
  - 暂停时间与回收集大小正相关，无法做到亚毫秒级
- **吞吐量损失**：约 10–15%（相比 Parallel GC）

### 启用方式

```bash
java -XX:+UseG1GC ...
# JDK 9+ 默认，无需指定
```

### 关键参数

| 参数 | 说明 | 默认值 |
|---|---|---|
| `-XX:MaxGCPauseMillis` | 目标最大暂停时间 | 200ms |
| `-XX:G1HeapRegionSize` | Region 大小 | 自动计算 |
| `-XX:InitiatingHeapOccupancyPercent` | 触发并发标记的堆占用阈值 | 45% |
| `-XX:G1NewSizePercent` | 新生代最小占比 | 5% |

### 适用场景

- JDK 9+ 服务端默认选择
- 多核机器、大堆（4–32GB）
- 需要平衡吞吐与延迟的应用
- 无法接受长时间 STW 但允许百毫秒级暂停

---

## ZGC（Z Garbage Collector）

### 原理

并发、单代（JDK 21 引入分代）、Region 化、NUMA 感知的压缩收集器。核心设计：**染色指针（Colored Pointers）** + **读屏障（Load Barrier）**。

染色指针在对象地址中嵌入元数据位，标记对象状态（如是否存活、是否被移动）。读屏障在应用线程读取引用时检查指针颜色，必要时执行修正（如更新被移动对象的地址）。

### 工作流程（非分代）

所有阶段并发执行，仅根集扫描需要 STW：

1. **Pause Init Mark**：STW，标记根集（<1ms）
2. **Concurrent Mark**：并发遍历对象图
3. **Pause Final Mark**：STW，完成标记（<1ms）
4. **Concurrent Prepare for Relocate**：并发选择回收集
5. **Pause Init Relocate**：STW，准备重定位（<1ms）
6. **Concurrent Relocate**：并发移动对象
7. **Concurrent Remap**：并发修正指针

### 分代 ZGC（JDK 21）

JEP 439 引入分代 ZGC，将堆分为 Young 和 Old 两代：

- **目标**：降低内存开销、减少分配停顿风险、降低 CPU 开销
- **设计**：
  - 每代独立回收，优先回收 Young（符合分代假说）
  - 使用 Store Barrier 记录跨代引用（Remembered Set）
  - SATB（Snapshot-At-The-Beginning）标记算法
- **优势**：相比非分代，吞吐量提升，内存占用降低（如 Cassandra 基准：仅需 1/4 堆大小，吞吐量提升 4 倍）

### 特性

- **暂停时间**：<10ms（非分代实测平均 1–2ms），与堆大小无关
- **堆范围**：支持几百 MB 到多 TB（非分代上限 16TB，分代可更高）
- **吞吐量损失**：约 15%（非分代），分代更低
- **内存开销**：染色指针需要额外位，非分代使用多映射内存（Triple Mapping），分代取消多映射

### 启用方式

```bash
# JDK 11–14（实验性）
java -XX:+UnlockExperimentalVMOptions -XX:+UseZGC ...

# JDK 15+（生产可用）
java -XX:+UseZGC ...

# JDK 21+（分代 ZGC，推荐）
java -XX:+UseZGC -XX:+ZGenerational ...

# JDK 23+（分代默认）
java -XX:+UseZGC ...
# 非分代需显式关闭：-XX:-ZGenerational
```

### 适用场景

- 低延迟刚需：金融交易、实时通信、游戏服务器
- 大堆（>32GB 甚至 TB 级）
- 对 STW 敏感，要求亚毫秒级暂停
- JDK 21+ 推荐使用分代 ZGC

---

## Shenandoah GC

### 原理

Region 化低延迟收集器。核心设计：**Brooks Pointer（转发指针）** + **读写屏障**。

每个对象头部有一个转发指针，指向对象当前地址（或转发地址）。GC 移动对象时，先在新位置复制对象，再更新转发指针。应用线程访问对象时，通过转发指针找到正确位置。

### 工作流程

1. **Pause Init Mark**：STW，初始化标记，扫描根集
2. **Concurrent Mark**：并发标记
3. **Pause Final Mark**：STW，完成标记，选择回收集
4. **Concurrent Cleanup**：并发回收空 Region
5. **Concurrent Evacuation**：并发移动对象（核心创新）
6. **Pause Init Update Refs**：STW，初始化引用更新
7. **Concurrent Update References**：并发更新所有引用
8. **Pause Final Update Refs**：STW，完成引用更新
9. **Concurrent Cleanup**：并发回收 Region

### 特性

- **暂停时间**：<10ms，实测通常 0–10ms，与堆大小无关
- **吞吐量损失**：约 0–15%（取决于屏障开销）
- **内存开销**：转发指针增加对象大小（约 8–16 字节）
- **模式**：
  - `normal/satb`（默认）：SATB 标记
  - `iu`（实验性）：增量更新标记
  - `passive`（诊断）：STW 模式，用于调试

### 启用方式

```bash
# JDK 12（实验性）
java -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC ...

# JDK 15+（生产可用）
java -XX:+UseShenandoahGC ...
```

### 适用场景

- 低延迟刚需（与 ZGC 类似）
- Red Hat、Amazon Corretto、Azul Zulu 等发行版默认包含
- Oracle JDK 不包含 Shenandoah

### ZGC vs Shenandoah

| 维度 | ZGC | Shenandoah |
|---|---|---|
| 核心技术 | 染色指针 + 读屏障 | Brooks Pointer + 读写屏障 |
| JDK 包含 | Oracle JDK 11+ | 非 Oracle 发行版（Red Hat 等） |
| 分代支持 | JDK 21 分代 ZGC | 无分代（计划中） |
| 最大堆 | 16TB（非分代），更高（分代） | 理论无限制 |
| 多映射内存 | 非分代使用，分代取消 | 不使用 |

---

## Epsilon GC

### 原理

无操作 GC：只分配内存，不回收。堆耗尽时 JVM 直接退出或抛出 `OutOfMemoryError`。

### 特性

- **无 GC 暂停**：完全不执行 GC，零延迟开销
- **无 GC Barrier**：无读/写屏障，吞吐量最高
- **用途限制**：仅适用于内存受限、短生命周期或已知不产生垃圾的应用

### 启用方式

```bash
# JDK 11+（实验性）
java -XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC ...
```

### 适用场景

- **性能测试**：排除 GC 影响，测量纯粹应用性能
- **内存压力测试**：验证内存上限（配合 `-Xmx`）
- **极短生命周期任务**：进程快速退出，无需 GC
- **已知无分配应用**：完全垃圾-free 的应用

---

## GC 选择指南

### 决策维度

1. **堆大小**：<4GB → Serial/Parallel；4–32GB → G1；>32GB → ZGC/Shenandoah
2. **延迟敏感度**：
   - 不敏感 → Parallel GC
   - 百毫秒级容忍 → G1 GC
   - 亚毫秒级刚需 → ZGC/Shenandoah
3. **吞吐优先**：Parallel GC > G1 > ZGC/Shenandoah
4. **JDK 版本**：JDK 8 → Parallel/G1；JDK 11+ → G1/ZGC/Shenandoah

### 推荐配置

| 场景 | JDK 8 | JDK 11/17 | JDK 21+ |
|---|---|---|---|
| 小堆客户端 | Serial GC | Serial GC | Serial GC |
| 批处理后台 | Parallel GC | Parallel GC | Parallel GC |
| 通用服务端 | G1 GC | G1 GC | G1 GC |
| 低延迟服务 | — | ZGC/Shenandoah | **分代 ZGC** |
| 极低延迟大堆 | — | ZGC/Shenandoah | 分代 ZGC |

### JDK 21+ 默认选择

- **通用场景**：G1 GC（仍为默认）
- **低延迟场景**：分代 ZGC（`-XX:+UseZGC`）
- **Red Hat/Azul 发行版**：Shenandoah 可选

---

## 常见误区

### 误区 1：GC 可以完全消除暂停

所有 HotSpot GC 都有 STW 阶段（至少根集扫描）。ZGC/Shenandoah 将暂停降至亚毫秒级，但不能完全消除。

### 误区 2：大堆自动解决内存问题

大堆延长 GC 暂停时间（Serial/Parallel/G1）。ZGC/Shenandoah 暂停与堆大小无关，但大堆仍需要足够 CPU 处理并发工作。

### 误区 3：调优参数万能

GC 参数仅能在设计范围内调整。如 `-XX:MaxGCPauseMillis=10` 对 G1 无效（设计无法做到）。选择合适的 GC 比参数调优更重要。

### 误区 4：CMS 仍可使用

CMS 已移除（JDK 14+），遗留系统可考虑迁移到 G1 或 ZGC/Shenandoah。

### 误区 5：低延迟 GC 无代价

ZGC/Shenandoah 使用屏障技术，增加读/写开销，吞吐量降低约 10–15%。需要评估延迟收益是否抵消吞吐损失。

---

## 调优基础

### 通用建议

1. **固定堆大小**：`-Xms` = `-Xmx`，避免动态扩容开销
2. **预触内存**：`-XX:+AlwaysPreTouch`，启动时分配物理内存
3. **启用日志**：`-Xlog:gc*`（JDK 9+）或 `-XX:+PrintGCDetails`（JDK 8）
4. **监控 GC**：使用 JMX (`GarbageCollectorMXBean`) 或日志分析工具

### 低延迟 GC 调优

1. **大页内存**：`-XX:+UseLargePages` 或 `-XX:+UseTransparentHugePages`
2. **NUMA 感知**：`-XX:+UseNUMA`
3. **禁用偏向锁**：`-XX:-UseBiasedLocking`（减少 safepoint）
4. **禁用显式 GC**：`-XX:+DisableExplicitGC`

---

## 参考资料

- [Oracle Java SE 17 GC Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)
- [JEP 333: ZGC (Experimental)](https://openjdk.org/jeps/333)
- [JEP 377: ZGC (Production)](https://openjdk.org/jeps/377)
- [JEP 439: Generational ZGC](https://openjdk.org/jeps/439)
- [JEP 189: Shenandoah (Experimental)](https://openjdk.org/jeps/189)
- [JEP 318: Epsilon GC](https://openjdk.org/jeps/318)
- [Shenandoah Wiki](https://wiki.openjdk.org/display/shenandoah/Main)
- [OpenJDK ZGC Project](https://openjdk.org/projects/zgc/)
- [OpenJDK Shenandoah Project](https://openjdk.org/projects/shenandoah/)
