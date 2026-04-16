---
created: 2026-04-16T00:00:00.000Z
updated: '2026-04-16'
tags:
  - database
  - redis
  - nosql
  - caching
  - vector-search
  - clustering
  - release-notes
aliases:
  - Redis 8.4 新特性
  - Redis 8.4 features
  - Redis 8.4 capabilities
  - Redis 8.4 GA
source_type: official-blog
source_urls:
  - 'https://redis.io/blog/redis-8-4-open-source-ga/'
  - 'https://github.com/redis/redis/releases/tag/8.4.0'
  - 'https://redis.io/blog/atomic-slot-migration/'
  - 'https://redis.io/blog/simplifying-streams-and-strings/'
  - >-
    https://redis.io/blog/revamping-context-oriented-retrieval-with-hybrid-search-in-redis-84/
status: verified
---

## 概述

Redis 8.4 于 **2025 年 11 月 18 日** 正式发布 GA（General Availability），是 Redis 8.x 系列中的重要版本。相比 8.2，8.4 在**混合搜索、性能、集群运维、Streams 消息处理、字符串原子操作**和 **JSON 内存优化**六个维度做了显著增强，官方定位为"最快、最简洁、最强大的 Redis"。

当前最新补丁版本为 **8.4.2**（2026-02-23 发布，安全修复）。

## 相比 Redis 8.2 的主要变化速览

| 能力领域 | 关键变化 |
|----------|----------|
| 混合搜索 | 新增 `FT.HYBRID` 命令，全文搜索 + 向量相似度融合评分 |
| 性能 | 缓存场景吞吐量提升 >30%；查询引擎 I/O 多线程化，FT.SEARCH/FT.AGGREGATE 吞吐量最高提升 4.7 倍 |
| 集群运维 | 原子槽位迁移（ASM），迁移速度提升 30 倍，客户端中断减少 98% |
| Streams | `XREADGROUP` 新增 `CLAIM` 选项，一条命令同时消费空闲待处理和新增消息 |
| 字符串原子操作 | `SET` 扩展 `IFEQ/IFNE/IFDEQ/IFDNE`、新增 `DELEX`、`DIGEST`、`MSETEX` 命令 |
| JSON 内存 | 同质数值数组内存占用降低 50%–92%；短字符串内联优化 |

---

## 一、混合搜索（Hybrid Search）— `FT.HYBRID`

### 是什么

`FT.HYBRID` 是 Redis 8.4 引入的统一混合检索 API，在**单次查询执行**中将全文搜索（BM25）和向量相似度搜索的结果通过**分数融合**合并为一个排序列表。

### 为什么重要

- 之前的混合检索需要多步操作 + 外部分数合并，增加延迟和复杂度
- 研究显示混合检索（文本 + 向量）相比单一模式可降低 **49% 的上下文失败率**，检索召回率提升 **3–3.5 倍**[^1]
- 对 RAG 系统和 AI Agent 的场景尤为关键

### 评分融合策略

支持两种融合方式：

| 策略 | 说明 |
|------|------|
| **RRF**（Reciprocal Rank Fusion） | 倒数排名融合，不依赖分数绝对值，鲁棒性强 |
| **Linear Combination** | 线性加权组合，可调节全文和向量分数的权重比例 |

### 典型使用场景

- **优先近期记忆**：结合时间过滤 + 向量相似度，让 Agent 优先获取语义相近且最新的信息
- **地理上下文检索**：通过 GEO/GEOSHAPE 过滤 + 语义匹配，实现"附近 + 相关"的检索
- **混合精确与模糊匹配**：同时使用精确关键词、模糊匹配（`%`）、可选匹配（`~`）和向量匹配

### 已知限制

- `FT.HYBRID` 不支持 `EXPLAINSCORE`、`SHARD_K_RATIO`、`YIELD_DISTANCE_AS`、`WITHCURSOR` 选项
- `FT.HYBRID` 不支持 `COMBINE` 步骤后的 `FILTER` 后过滤
- 默认响应格式仅返回 `key_id` 和 `score`，未来可能扩展为返回完整文档内容
- `FT.PROFILE`、`FT.EXPLAIN`、`FT.EXPLAINCLI` 不包含 `FT.HYBRID` 选项
- `FT.HYBRID` 的指标不会显示在 `FT.INFO` 和 `INFO` 中

---

## 二、性能提升

### 缓存场景吞吐量

在典型缓存工作负载（10% SET + 90% GET，1 KB 字符串值，4 核）下，Redis 8.4 相比 8.2 **吞吐量提升超过 30%**[^2]。

### 查询引擎 I/O 多线程化

Redis 8.4 为分布式查询引入了**多线程 I/O 处理**：

- 多个分片的响应现在由并发 I/O 线程并行处理，而非顺序处理
- 消除了之前单线程瓶颈导致的 CPU 饱和和大队列等待问题

**性能数据**（5 个并行 I/O 线程）：

| 操作 | 吞吐量提升 | 说明 |
|------|-----------|------|
| `FT.SEARCH` | 最高 **4.7 倍** | 大规模搜索工作负载 |
| `FT.AGGREGATE` | 约 **1.4 倍** | 聚合操作含额外后处理 |

### 内存优化

- **JSON 短字符串内联**：≤7 字节的短字符串直接内联存储，500 个键值对（均为短字符串）的 JSON 对象内存减少 **37%**
- **JSON 同质数值数组优化**：当数组元素类型一致时，类型信息只存储一次。100 万元素数组的内存减少幅度：

| 数组元素类型 | 8.2 内存 | 8.4 内存 | 减少幅度 |
|-------------|---------|---------|---------|
| 8 位整数 | 8.42 MB | 1.14 MB | **87%** |
| 16 位整数 | 8.43 MB | 2.19 MB | **74%** |
| 32 位整数 | 8.46 MB | 4.26 MB | **50%** |
| 64 位整数 | 24.46 MB | 8.43 MB | **66%** |
| BF16/FP16 浮点 | 24.43 MB | 2.16–2.19 MB | **92%** |
| FP32 浮点 | 24.46 MB | 4.26 MB | **83%** |
| FP64 浮点 | 24.46 MB | 8.43 MB | **66%** |

### OOM 行为可控

新增 `search-on-oom` 配置参数，允许开发者定义查询引擎在内存不足时的行为策略。

---

## 三、原子槽位迁移（Atomic Slot Migration, ASM）

### 是什么

ASM 是 Redis 8.4 引入的集群槽位迁移新机制，通过**原子化批量传输**替代之前的逐键迁移方式。

### 解决了什么问题

旧版槽位迁移（`CLUSTER GETKEYSINSLOT` + `MIGRATE`）存在以下问题：

| 问题 | 旧版表现 | ASM 改进 |
|------|---------|---------|
| 客户端重定向 | 迁移期间产生大量 `-ASK`/`-MOVED` 重定向（最高 241.6 次/秒） | 仅 2.1 次/秒，**减少 98%** |
| 多键操作可靠性 | 部分键已迁移时返回 `TRYAGAIN` | 迁移期间无此问题 |
| 迁移失败风险 | 中途失败可能导致数据不一致 | 原子手递手，要么全成功要么不切换 |
| 迁移速度 | 约 21 槽/秒 | 约 640 槽/秒，**快 30 倍** |
| 延迟峰值 | 最高 127 ms | 最高 <70 ms，**降低 40%+** |
| 集群消息开销 | 最高 5.4K 条消息 | 仅 212 条，**减少 94%** |

### 工作原理

ASM 类似分片级别的全量同步复制：

1. **从目标节点发起**：向目标节点发送 `CLUSTER MIGRATION IMPORT <start-slot> <end-slot>`
2. **建立复制连接**：目标节点向源节点请求槽位复制，建立两条并行连接（快照 + 增量流）
3. **源节点发送数据**：fork 后发送槽位快照，同时流式传输增量写入
4. **目标节点消费数据**：应用快照命令并累积增量更新
5. **原子切换所有权**：源节点短暂暂停写入，转发剩余更新后，目标节点接管槽位所有权
6. **源节点恢复**：收到配置更新后恢复写流量，客户端收到 `-MOVED` 转向新所有者
7. **后台清理**：源节点在后台线程异步删除已迁移的槽位数据

### 使用命令

| 命令 | 说明 | 发送目标 |
|------|------|---------|
| `CLUSTER MIGRATION IMPORT <start-slot> <end-slot> [...]` | 发起迁移，返回任务 ID | 目标节点 |
| `CLUSTER MIGRATION STATUS <ID id | ALL>` | 查看迁移状态 | 目标节点 |
| `CLUSTER MIGRATION CANCEL <ID id | ALL>` | 取消迁移任务 | 目标节点 |

### 性能实测数据

- **扩容（3 → 4 分片）**：迁移 1/3 槽位耗时 **6.4 秒**，吞吐量持续上升，平均延迟增加 <5%（持续 2 秒）
- **缩容（4 → 3 分片）**：迁移耗时 **8.6 秒**，延迟从 2.3 ms 增至 2.8 ms（持续 3 秒）

### 已知限制

- 在 ASM 迁移过程中执行 `FT.SEARCH`、`FT.AGGREGATE`、`FT.CURSOR`、`FT.HYBRID`、`TS.MGET`、`TS.MRANGE`、`TS.MREVRANGE`、`TS.QUERYINDEX` 时，结果可能**不完整或包含重复**

---

## 四、Streams 简化 — `XREADGROUP CLAIM`

### 背景

在 Redis Streams 中，**待处理消息**（pending messages）指已投递给消费者但尚未确认的消息。如果消息长时间未被确认，通常意味着消费者崩溃、消息有问题（如死锁）或通信故障。

### 旧版问题

之前消费者需要分别处理两类消息，逻辑复杂：
1. 调用 `XPENDING` + `XCLAIM`/`XAUTOCLAIM` 获取并认领空闲待处理消息
2. 调用 `XREADGROUP` 获取新消息

且 `XPENDING`、`XCLAIM`、`XAUTOCLAIM` **不支持多流**，消费多个流时需要逐个调用。

### 8.4 改进

`XREADGROUP` 新增 `CLAIM` 选项：

```
XREADGROUP GROUP group consumer CLAIM <min-idle-time> STREAMS key [key ...] id [id ...]
```

- 当指定 `CLAIM <min-idle-time>` 且 `id` 为 `>` 时，Redis 会先尝试认领空闲时间 ≥ `min-idle-time` 毫秒的待处理消息（空闲时间最长的优先）
- 如果没有符合条件的待处理消息，则正常消费新消息
- **支持多流**：一条命令可同时对多个流执行此操作
- 返回结果中包含额外的待处理消息元信息（类似 `XPENDING`），包括投递次数，可用于检测"毒丸消息"（poison pill）

---

## 五、字符串原子操作

### 原子比较并设置（Compare-and-Set）

`SET` 命令新增 4 个条件选项，实现单键乐观并发控制：

| 选项 | 含义 |
|------|------|
| `IFEQ <value>` | 仅当键当前值**等于**指定值时才设置 |
| `IFNE <value>` | 仅当键当前值**不等于**指定值时才设置 |
| `IFDEQ <digest>` | 仅当键当前值的 **XXH3 摘要**等于指定摘要时才设置 |
| `IFDNE <digest>` | 仅当键当前值的 **XXH3 摘要**不等于指定摘要时才设置 |

**使用场景**：产品描述编辑——获取旧值 → 用户编辑 → `SET key newValue IFEQ oldValue`，仅当值未被其他客户端修改时才更新。

`IFDEQ`/`IFDNE` 使用 **XXH3** 哈希函数计算摘要，适合值较大时避免在客户端保留完整旧值。

### 新增 `DIGEST` 命令

```
DIGEST <key>
```

返回键值的 XXH3 哈希摘要。可配合 `IFDEQ`/`IFDNE` 使用，也可用于定期检测键值是否被修改。

### 新增 `DELEX` 命令（Compare-and-Delete）

```
DELEX <key> [IFEQ <value> | IFNE <value> | IFDEQ <digest> | IFDNE <digest>]
```

原子比较并删除——仅当键值满足条件时才删除。与 `SET` 的条件选项语义一致。

### 新增 `MSETEX` 命令

```
MSETEX [NX | XX] [EX <seconds> | PX <ms> | EXAT <unix-time> | PXAT <unix-time-ms> | PERSIST] <key> <value> [<key> <value> ...]
```

原子设置多个字符串键并统一设置过期时间：

| 选项 | 行为 |
|------|------|
| `XX` | 仅当**所有**指定键已存在时才设置 |
| `NX` | 仅当**所有**指定键都不存在时才设置 |
| `EX`/`PX`/`EXAT`/`PXAT` | 为所有键设置统一的过期时间 |
| `PERSIST` | 移除所有键的过期时间 |

 effectively 替代并扩展了 `MSET` 和 `MSETNX`。

---

## 六、集群槽位统计增强

`CLUSTER SLOT-STATS` 命令在 8.2 引入，8.4 新增 **`MEMORY-BYTES`** 指标：

| 指标 | 说明 |
|------|------|
| `KEY-COUNT` | 槽位中的键数量 |
| `CPU-USEC` | 处理该槽位消耗的 CPU 时间（微秒） |
| `NETWORK-BYTES-IN` | 该槽位接收的入站网络流量（字节） |
| `NETWORK-BYTES-OUT` | 该槽位发送的出站网络流量（字节） |
| `MEMORY-BYTES` | **8.4 新增**：该槽位中键占用的总内存 |

可用于检测热点槽位、规划槽位迁移以均衡负载。

---

## 七、查询引擎 OOM 管理

新增 `search-on-oom` 配置参数，允许定义查询引擎在内存不足时的行为。这使得大规模搜索工作负载在内存受限环境下更加可控和稳健。

---

## 八、测试平台与兼容性

### 测试操作系统

- Ubuntu 22.04、24.04
- Rocky Linux 8.10、9.5
- AlmaLinux 8.10、9.5
- Debian 12、13
- macOS 13、14、15

### 获取方式

- Docker：https://hub.docker.com/_/redis
- Snap：https://github.com/redis/redis-snap
- Homebrew：https://github.com/redis/homebrew-redis
- RPM：https://github.com/redis/redis-rpm
- Debian APT：https://github.com/redis/redis-debian

---

## 已知 Bug 与限制汇总

| 类别 | 限制说明 |
|------|---------|
| `FT.HYBRID` | 不支持 `EXPLAINSCORE`、`SHARD_K_RATIO`、`YIELD_DISTANCE_AS`、`WITHCURSOR` |
| `FT.HYBRID` | 不支持 `COMBINE` 后的 `FILTER` |
| `FT.HYBRID` | 默认响应仅返回 `key_id` 和 `score` |
| `FT.HYBRID` | 指标不显示在 `FT.INFO` 和 `INFO` 中 |
| `FT.PROFILE`/`FT.EXPLAIN` | 不包含 `FT.HYBRID` 选项 |
| ASM 迁移中 | 搜索和时间序列查询结果可能不完整或重复 |
| 许可证 | 双许可（RSALv2 / SSPLv1），不再符合 OSI 开源定义 |

---

## 与 Redis 7.2 的功能和性能对比

Redis 7.2（2023 年 8 月发布）是 **最后一个 BSD 3-Clause 许可版本**，也是 Redis 发展史上的一个重要分水岭。从 7.2 到 8.4 经历了许可证变更、模块整合、架构优化等多次重大演进。

### 许可证对比

| 维度 | Redis 7.2 | Redis 8.4 |
|------|----------|----------|
| 许可证 | **BSD 3-Clause**（OSI 认证开源） | **RSALv2 / SSPLv1**（双 source-available 许可） |
| 云托管限制 | 无限制 | RSALv2 禁止将 Redis 作为托管服务提供 |
| 开源替代 | 无必要 | Valkey（基于 7.2.4 fork，保持 BSD） |
| 官方称呼 | Redis OSS | Redis Community Edition |

### 功能对比

| 能力领域 | Redis 7.2 | Redis 8.4 | 变化说明 |
|----------|----------|----------|---------|
| **JSON 支持** | 需安装 RedisJSON 模块（Redis Stack） | **原生内置**（`JSON.GET`、`JSON.SET` 等） | 模块并入核心，无需额外安装 |
| **全文/向量搜索** | 需安装 RediSearch 模块 | **原生内置**（`FT.*` 系列命令） | 模块并入核心 |
| **混合搜索** | 需外部多步操作 + 手动分数合并 | **`FT.HYBRID` 原生支持**，RRF/线性融合 | 8.4 新增，单次查询完成 |
| **时间序列** | 需安装 RedisTimeSeries 模块 | **原生内置**（`TS.*` 系列命令） | 模块并入核心 |
| **向量数据库** | 无原生 Vector Set | **Vector Set 数据类型** | 8.0 引入 |
| **概率数据结构** | 基础 Bloom/HyperLogLog | **增强**（Bloom、Cuckoo filter 并入核心） | 模块并入核心 |
| **集群槽位迁移** | 逐键非原子迁移（`CLUSTER GETKEYSINSLOT` + `MIGRATE`） | **原子槽位迁移（ASM）** | 迁移快 30 倍，客户端中断减少 98% |
| **集群槽位统计** | 无 | `CLUSTER SLOT-STATS`（含 `MEMORY-BYTES`） | 8.2 引入，8.4 增强 |
| **Streams 消费** | `XPENDING` + `XCLAIM`/`XAUTOCLAIM` + `XREADGROUP` 多步操作 | **`XREADGROUP CLAIM` 一条命令** | 同时消费空闲待处理 + 新增消息 |
| **字符串原子操作** | 需 Lua 脚本或 `WATCH`/`MULTI`/`EXEC` 事务 | **`SET IFEQ/IFNE/IFDEQ/IFDNE`、`DELEX`、`DIGEST`、`MSETEX`** | 8.4 新增，更简单更快 |
| **JSON 内存优化** | 无特殊优化 | 短字符串内联（≤7 字节）+ 同质数值数组优化 | 100 万元素数组内存减少 50%–92% |
| **查询引擎 I/O** | 单线程处理分片响应 | **多线程 I/O**（并行处理分片响应） | FT.SEARCH 吞吐量最高提升 4.7 倍 |
| **OOM 管理** | 无查询引擎 OOM 配置 | **`search-on-oom` 配置** | 可控查询引擎内存不足行为 |
| **复制机制** | 传统全量/增量复制 | **8.0 引入的新复制机制**（ASM 也复用此机制） | 源节点处理率更高，内存需求更低 |

### 性能对比

官方博文中的吞吐量趋势图显示，从 7.2 到 8.4 缓存工作负载（10% SET + 90% GET，1 KB 字符串值）的 OPS 呈**持续增长**趋势[^3]：

| 指标 | Redis 7.2 | Redis 8.4 | 变化 |
|------|----------|----------|------|
| **缓存吞吐量**（4 核，10% SET + 90% GET） | 基线 | 相比 8.2 提升 >30%；**相比 7.2 累计提升显著**（趋势图显示持续增长） | 持续增长 |
| **FT.SEARCH 吞吐量**（大规模搜索，5 并行 I/O 线程） | 不支持（需外部模块，单线程协调） | 最高提升 **4.7 倍**（相比 8.2） | 8.4 引入多线程 I/O |
| **FT.AGGREGATE 吞吐量** | 不支持（需外部模块，单线程协调） | 约 **1.4 倍**提升（相比 8.2） | 8.4 引入多线程 I/O |
| **JSON 同质数组内存**（100 万元素） | 无优化（模块存储） | 减少 **50%–92%**（相比 8.2） | 8.4 引入同质数组优化 |
| **JSON 短字符串内存**（500 键值对） | 无优化（模块存储） | 减少 **37%**（相比 8.2） | 8.4 引入短字符串内联 |
| **槽位迁移速度** | ~21 槽/秒 | ~640 槽/秒 | **快 30 倍** |
| **槽位迁移客户端中断** | 最高 241.6 次 `-MOVED`/秒 | 2.1 次/秒 | **减少 98%** |
| **槽位迁移延迟峰值** | 最高 127 ms | 最高 <70 ms | **降低 40%+** |

### 架构演进路径

```
Redis 7.2（2023-08）          Redis 8.0                  Redis 8.2                  Redis 8.4
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ BSD 3-Clause    │      │ 双许可开始      │      │                 │      │                 │
│ 模块需单独安装  │─────>│ JSON 内置核心   │─────>│ CLUSTER         │─────>│ FT.HYBRID       │
│ (Stack 分发)    │      │ Vector Set      │      │ SLOT-STATS      │      │ 原子槽位迁移    │
│ 传统槽位迁移    │      │ 搜索引擎整合    │      │ Streams 增强    │      │ XREADGROUP CLAIM│
│ 单线程查询 I/O  │      │ 新复制机制      │      │ Bitmap 改进     │      │ SET IFEQ/DELEX  │
│                 │      │                 │      │                 │      │ MSETEX          │
│                 │      │                 │      │                 │      │ JSON 内存优化   │
│                 │      │                 │      │                 │      │ 多线程查询 I/O  │
└─────────────────┘      └─────────────────┘      └─────────────────┘      └─────────────────┘
```

### 升级建议

| 场景 | 建议 |
|------|------|
| 从 7.2 升级，需要真正开源（BSD） | 考虑 **Valkey**（基于 7.2.4 fork）而非 8.4 |
| 从 7.2 升级，内部使用/无云托管限制 | **直接升级到 8.4+**，获得全部新功能和性能提升 |
| 使用 Redis Stack 模块 | 8.4 已内置所有模块能力，升级后可**移除模块依赖** |
| 使用 Lua 脚本做乐观并发控制 | 8.4 可用 `SET IFEQ`/`DELEX` **替代 Lua 脚本**，更简单更快 |
| 集群频繁扩缩容 | 8.4 ASM 使槽位迁移**快 30 倍**，大幅降低运维成本 |
| 构建 RAG/AI Agent 应用 | 8.4 `FT.HYBRID` 提供**原生混合搜索**，无需外部分数合并 |

---

## 与前后版本的关系

| 版本 | 发布时间 | 与 8.4 的关系 |
|------|---------|-------------|
| 8.2 | 2025 年 | 8.4 的基线版本；8.2 引入了 `CLUSTER SLOT-STATS` 和 Streams/Bitmap 增强 |
| **8.4** | **2025-11-18** | **当前主题版本** |
| 8.4.1 | 2026-02-08 | 安全修复（PII 日志泄露、Bloom/Cuckoo 过滤器 RDB 加载崩溃）+ 大量 RediSearch 修复 |
| 8.4.2 | 2026-02-23 | 安全修复：错误回复中 `\r\n` 序列注入漏洞 |
| 8.6 | 2026-02-10 | 8.4 的后续版本，新增 Streams 幂等生产、LRM 驱逐策略、热点键检测等 |

---

## 参考资料

- [Redis 8.4 GA 官方公告](https://redis.io/blog/redis-8-4-open-source-ga/)（2025-11-25）
- [Redis 8.4.0 GitHub Release](https://github.com/redis/redis/releases/tag/8.4.0)
- [原子槽位迁移详解](https://redis.io/blog/atomic-slot-migration/)（2026-04-02）
- [Streams 和字符串简化](https://redis.io/blog/simplifying-streams-and-strings/)（2026-01-05）
- [混合搜索与 FT.HYBRID](https://redis.io/blog/revamping-context-oriented-retrieval-with-hybrid-search-in-redis-84/)（2025-11-17）
- [FT.HYBRID 命令文档](https://redis.io/docs/latest/commands/ft.hybrid/)
- [XREADGROUP 命令文档（CLAIM 选项）](https://redis.io/docs/latest/commands/xreadgroup/#the-claim-option)
- [SET 命令文档（IFEQ/IFNE/IFDEQ/IFDNE）](https://redis.io/docs/latest/commands/set/)
- [DELEX 命令文档](https://redis.io/docs/latest/commands/delex/)
- [DIGEST 命令文档](https://redis.io/docs/latest/commands/digest/)
- [MSETEX 命令文档](https://redis.io/docs/latest/commands/msetex/)
- [CLUSTER MIGRATION 命令文档](https://redis.io/docs/latest/commands/cluster-migration/)
- [CLUSTER SLOT-STATS 命令文档](https://redis.io/docs/latest/commands/cluster-slot-stats/)

[^1]: 数据来源：Anthropic (2025)、Apple ML Research (2024)、Blended RAG (2024)、HyPA-RAG (2024) 等研究，详见 [Revamping context-oriented retrieval with hybrid search in Redis 8.4](https://redis.io/blog/revamping-context-oriented-retrieval-with-hybrid-search-in-redis-84/)
[^2]: 基准测试条件：10% SET + 90% GET，1 KB 字符串值，4 核，详见 [Redis 8.4 GA 官方公告](https://redis.io/blog/redis-8-4-open-source-ga/)
[^3]: 吞吐量趋势图详见 [Redis 8.4 GA 官方公告](https://redis.io/blog/redis-8-4-open-source-ga/) 中 "The fastest, most resource-efficient Redis yet" 章节；官方未直接给出 7.2→8.4 的具体倍数，但趋势图显示从 7.2 到 8.4 持续增长，8.4 相比 8.2 提升 >30%，8.2 相比 7.2 也有显著提升
