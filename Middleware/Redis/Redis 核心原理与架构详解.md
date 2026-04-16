
---
created: 2026-04-16
updated: 2026-04-16
tags:
  - database
  - redis
  - architecture
  - in-memory
  - data-structures
  - persistence
  - replication
  - clustering
  - single-threaded
  - event-loop
aliases:
  - Redis 原理
  - Redis 架构
  - Redis 底层原理
  - How Redis works
  - Redis internals
  - Redis architecture
source_type: official-doc
source_urls:
  - https://redis.io/docs/latest/develop/data-types/
  - https://redis.io/docs/latest/develop/interact/transactions/
  - https://redis.io/docs/latest/develop/interact/programmability/eval-intro/
  - https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
  - https://redis.io/docs/latest/operate/oss_and_stack/management/replication/
  - https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
  - https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/
  - https://github.com/redis/redis/blob/unstable/README.md
status: verified
---

## 是什么

Redis（Remote Dictionary Server）是一个**内存数据结构存储系统**，可用作数据库、缓存和消息代理。其核心设计哲学是：**将所有数据保存在内存中，通过高效的数据结构和单线程事件循环模型实现亚毫秒级延迟**。

## 为什么重要

| 维度 | 说明 |
|------|------|
| **极致性能** | 数据主要在内存中操作，避免了磁盘 I/O 瓶颈；单线程模型消除了锁竞争和上下文切换开销 |
| **丰富的数据结构** | 原生支持字符串、哈希、列表、集合、有序集合、位图、HyperLogLog、流、地理空间、JSON、向量等，远超传统 key-value 存储 |
| **原子操作** | 所有单个命令都是原子的，无需额外加锁即可实现安全的并发操作 |
| **持久化可选** | 提供 RDB 快照和 AOF 日志两种持久化方式，可在性能和数据安全之间灵活权衡 |
| **高可用与分布式** | 内置主从复制、Sentinel 自动故障转移、Cluster 分布式分片，支持水平扩展 |

## 核心架构

### 整体设计

```
┌─────────────────────────────────────────────────────────┐
│                    Redis Server                          │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Network  │  │  Event Loop  │  │   Data Store     │  │
│  │   Layer   │─>│ (Single      │─>│   (In-Memory)    │  │
│  │  (epoll)  │  │  Thread)     │  │                  │  │
│  └───────────┘  └──────────────┘  └──────────────────┘  │
│                                    │                     │
│                    ┌───────────────┼───────────────┐     │
│                    │               │               │     │
│              ┌─────▼─────┐ ┌──────▼──────┐ ┌─────▼────┐│
│              │   RDB     │ │    AOF      │ │  Modules ││
│              │ Snapshot  │ │    Log      │ │ (JSON,   ││
│              │           │ │             │ │ Search,  ││
│              │           │ │             │ │ etc.)    ││
│              └───────────┘ └─────────────┘ └──────────┘│
└─────────────────────────────────────────────────────────┘
         │                              │
    ┌────▼────┐                   ┌─────▼─────┐
    │ Clients │                   │ Replicas  │
    │ (TCP)   │                   │ (异步复制) │
    └─────────┘                   └───────────┘
```

### 关键组件

| 组件 | 职责 |
|------|------|
| **网络层** | 基于 TCP 协议，使用 I/O 多路复用（Linux epoll / macOS kqueue）处理并发连接 |
| **事件循环** | 单线程事件驱动器，顺序处理文件事件（网络 I/O）和时间事件（定时任务如过期键清理） |
| **数据存储** | 内存中的键值对字典，键为字符串，值为各种数据结构的实例 |
| **持久化引擎** | RDB（fork 子进程生成快照）和 AOF（记录写命令日志） |
| **复制模块** | 主从异步复制，支持全量同步（RDB）和增量同步（复制缓冲区） |
| **模块系统** | 通过 Modules API 扩展 Redis 功能（JSON、Search、TimeSeries 等已并入核心） |

## 单线程模型详解

### 为什么采用单线程

Redis 选择单线程处理命令执行，主要基于以下原因：

1. **CPU 不是瓶颈**：Redis 操作主要在内存中完成，单次命令执行时间极短（通常 < 1ms），单核即可处理数十万 ops/sec
2. **避免锁竞争**：多线程需要复杂的锁机制来保护共享数据结构，锁竞争反而可能降低性能
3. **避免上下文切换**：线程切换会消耗 CPU 时间，单线程完全消除此开销
4. **简化实现**：单线程模型使代码更简单、更可预测、更容易调试

### 单线程 ≠ 整个进程只有一个线程

从 Redis 6.0 开始，引入了**多线程 I/O**来处理网络读写操作：

| 线程类型 | 职责 | 引入版本 |
|----------|------|----------|
| **主线程** | 执行所有命令、处理事件循环、管理数据结构 | 始终存在 |
| **I/O 线程**（可选） | 并行读取客户端请求和写入响应，减轻主线程网络 I/O 负担 | 6.0+ |
| **后台线程** | RDB 快照（fork 子进程）、AOF 重写、惰性删除过期键、异步删除大键 | 4.0+ |

**关键点**：即使启用了多线程 I/O，**命令执行仍然是单线程的**，保证了操作的原子性和一致性。

### 事件循环机制

Redis 的事件循环基于 **ae 事件库**（封装了 epoll/kqueue/evport/select），处理两类事件：

```
while (true) {
    // 1. 处理文件事件（网络 I/O）
    aeProcessFileEvents(eventLoop);
    
    // 2. 处理时间事件（定时任务）
    aeProcessTimeEvents(eventLoop);
}
```

**文件事件**包括：
- 客户端连接请求（accept）
- 客户端数据可读（read）
- 响应数据可写（write）

**时间事件**包括：
- 过期键清理（serverCron，默认每 100ms 执行一次）
- AOF 持久化触发
- RDB 快照调度
- 客户端超时检测

### 单线程的限制与应对

| 限制 | 说明 | 应对方案 |
|------|------|----------|
| **阻塞命令** | `KEYS *`、`SORT`、`FLUSHALL` 等 O(N) 命令会阻塞主线程 | 使用 `SCAN` 替代 `KEYS`；生产环境禁用危险命令（`rename-command`） |
| **大键操作** | 操作包含百万元素的集合/列表/哈希可能耗时较长 | 避免创建大键；使用 `UNLINK` 替代 `DEL` 异步删除 |
| **CPU 密集型任务** | Lua 脚本中的复杂计算会阻塞其他命令 | 将计算移到客户端；限制脚本执行时间（`lua-time-limit`） |
| **单核性能上限** | 单线程只能利用一个 CPU 核心 | 多实例部署；使用 Redis Cluster 分片 |

## 数据结构底层实现

### String（字符串）

**底层编码**：`int`（整数）、`embstr`（短字符串 ≤ 44 字节）、`raw`（长字符串）

| 特性 | 说明 |
|------|------|
| 最大长度 | 512 MB |
| 时间复杂度 | GET/SET 为 O(1) |
| 内存布局 | 使用 SDS（Simple Dynamic String），预分配冗余空间减少内存重分配次数 |
| 二进制安全 | 可以存储任意字节序列，包括图片、序列化对象等 |

**SDS 结构**（Redis 自定义字符串）：

```c
struct sdshdr {
    int len;    // 已使用长度
    int free;   // 剩余可用长度
    char buf[]; // 字节数组
};
```

相比 C 字符串的优势：
- O(1) 获取长度（无需遍历）
- 防止缓冲区溢出（操作前检查空间）
- 减少内存重分配（预分配冗余空间）
- 二进制安全（不依赖 `\0` 终止符）

### List（列表）

**底层编码**：`quicklist`（双向链表 + 压缩节点）

| 特性 | 说明 |
|------|------|
| 插入/删除 | 两端 O(1)；中间 O(N) |
| 访问 | 按索引 O(N)；两端 O(1) |
| 内存优化 | quicklist 节点使用 ziplist 压缩存储小元素 |

**演进历史**：
- Redis 3.2 之前：`ziplist`（小列表）或 `linkedlist`（大列表）
- Redis 3.2+：统一使用 `quicklist`，结合了两者的优点

### Hash（哈希）

**底层编码**：`ziplist`（小哈希）或 `hashtable`（大哈希）

| 编码 | 触发条件 | 特点 |
|------|----------|------|
| `ziplist` | 字段数 < `hash-max-ziplist-entries`（默认 512）且所有值长度 < `hash-max-ziplist-value`（默认 64） | 连续内存块，缓存友好，但修改时需 realloc |
| `hashtable` | 超过上述任一阈值 | 标准哈希表，O(1) 平均访问，但内存开销更大 |

Redis 8.0+ 将 Hash 模块并入核心，原生支持 `HGET`、`HSET`、`HMGET` 等命令。

### Set（集合）

**底层编码**：`intset`（整数集合）或 `hashtable`

| 编码 | 触发条件 | 特点 |
|------|----------|------|
| `intset` | 所有元素都是整数且数量 < `set-max-intset-entries`（默认 512） | 有序数组，二分查找 O(log N)，内存紧凑 |
| `hashtable` | 包含非整数元素或超过数量阈值 | 字典实现，value 恒为 NULL，仅用 key 存储 |

集合操作（交集、并集、差集）时间复杂度为 O(N)，N 为最小集合的大小。

### Sorted Set（有序集合）

**底层编码**：`ziplist`（小有序集合）或 `skiplist + hashtable`

| 编码 | 触发条件 |
|------|----------|
| `ziplist` | 元素数 < `zset-max-ziplist-entries`（默认 128）且所有元素长度 < `zset-max-ziplist-value`（默认 64） |
| `skiplist + hashtable` | 超过上述任一阈值 |

**跳表（Skiplist）结构**：
- 多层链表，每层是下一层的"快速通道"
- 平均查找/插入/删除时间复杂度 O(log N)
- 支持范围查询（`ZRANGEBYSCORE`）和排名查询（`ZRANK`）

**为什么不用平衡树**：
- 跳表实现更简单
- 范围查询更高效（平衡树需要中序遍历）
- 内存局部性更好
- 并发友好（虽然 Redis 是单线程）

### Stream（流）

**底层编码**：`radix tree`（基数树）+ `listpack`

| 特性 | 说明 |
|------|------|
| 数据结构 | 类似 append-only log，每个消息有唯一 ID（时间戳-序列号） |
| 消费模型 | 支持消费者组（Consumer Group），类似 Kafka 的消费组概念 |
| 持久化 | 消息持久化在内存中，可配合 AOF/RDB 持久化到磁盘 |
| 典型用途 | 事件溯源、消息队列、传感器数据记录 |

### 其他数据类型

| 类型 | 底层实现 | 典型用途 |
|------|----------|----------|
| **Bitmap** | 基于 String 的位操作 | 用户签到、在线状态、布隆过滤器预计算 |
| **HyperLogLog** | 概率数据结构，固定 12 KB | 基数估算（UV 统计），标准误差 0.81% |
| **Geospatial** | 基于 Sorted Set（GeoHash 编码） | 附近的人、地理围栏 |
| **JSON** | 树形结构，8.0+ 内置核心 | 嵌套文档存储、索引和查询 |
| **Vector Set** | HNSW 算法，8.0+ 引入 | 向量相似度搜索、RAG 检索 |
| **Probabilistic** | Bloom/Cuckoo filter、t-digest、Top-K、Count-Min Sketch | 去重、分位数估算、频率统计 |

## 内存管理与过期策略

### 内存分配器

| 平台 | 默认分配器 | 选择原因 |
|------|-----------|----------|
| Linux | **jemalloc** | 内存碎片率更低，性能优于 libc malloc |
| macOS / 其他 | libc malloc | 平台兼容性 |

可通过 `MALLOC` 环境变量强制指定：`make MALLOC=jemalloc`

### 过期键删除策略

Redis 采用**惰性删除 + 定期删除**的组合策略：

| 策略 | 机制 | 优缺点 |
|------|------|--------|
| **惰性删除** | 访问键时检查是否过期，过期则删除 | 保证不会返回过期数据；但已过期未访问的键会持续占用内存 |
| **定期删除** | 事件循环中定期抽样检查（默认每秒 10 次，每次随机抽样 20 个键） | 主动清理过期键；但抽样可能遗漏部分过期键 |

**配合驱逐策略**：当内存达到 `maxmemory` 限制时，根据配置的淘汰策略主动删除键：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `noeviction` | 拒绝写入（默认） | 不允许数据丢失的场景 |
| `allkeys-lru` | 淘汰最近最少使用的键（全量） | 缓存场景，推荐 |
| `volatile-lru` | 淘汰最近最少使用的键（仅带 TTL 的） | 混合缓存和持久数据 |
| `allkeys-lfu` | 淘汰最不经常使用的键（全量） | 访问频率差异明显的场景 |
| `volatile-lfu` | 淘汰最不经常使用的键（仅带 TTL 的） | 同上，但仅影响有过期时间的键 |
| `allkeys-random` | 随机淘汰 | 所有键访问概率相同时 |
| `volatile-random` | 随机淘汰（仅带 TTL 的） | 同上 |
| `volatile-ttl` | 淘汰剩余 TTL 最短的键 | 需要优先保留长生命周期数据 |

Redis 8.6+ 新增 `volatile-lrm` 和 `allkeys-lrm`（Least Recently Modified），仅写操作刷新键的活跃度，适合缓存响应和聚合数据场景。

### 大键问题

**什么是大键**：包含大量元素或占用大量内存的单个键（如百万元素的 Sorted Set、10 MB 的 String）。

**危害**：
- 删除大键会阻塞主线程（`DEL` 是同步操作）
- 迁移大键会导致网络拥塞和延迟抖动
- RDB 快照和 AOF 重写时占用更多内存

**解决方案**：
- 使用 `UNLINK` 替代 `DEL`（异步删除，后台线程回收内存）
- 使用 `SCAN` 逐步拆分大键操作
- 合理设计数据结构，避免单个键过大
- 使用 `MEMORY USAGE <key>` 监控键的内存占用

## 持久化机制

### RDB（Redis Database）

**原理**：在指定时间间隔内生成内存数据集的快照。

**触发方式**：
- 手动触发：`SAVE`（阻塞主线程）或 `BGSAVE`（fork 子进程，不阻塞）
- 自动触发：配置 `save` 规则，如 `save 900 1`（900 秒内至少 1 次修改则触发）

**工作流程**（BGSAVE）：
1. 主进程 fork 子进程
2. 子进程遍历内存数据，写入临时 RDB 文件
3. 写入完成后，原子替换旧 RDB 文件
4. 子进程退出

**优点**：
- 文件紧凑，适合备份和灾难恢复
- 恢复速度快（直接加载二进制数据）
- 对性能影响小（fork 后子进程独立工作）

**缺点**：
- 可能丢失最后一次快照后的数据
- fork 子进程时需要复制页表，数据量大时可能阻塞主进程数百毫秒
- 使用操作系统的 Copy-on-Write 机制，写入频繁时内存开销增加

### AOF（Append Only File）

**原理**：记录每个写操作命令，重启时重放这些命令恢复数据。

**刷盘策略**（`appendfsync`）：

| 策略 | 行为 | 数据安全性 | 性能 |
|------|------|-----------|------|
| `always` | 每次写操作都同步到磁盘 | 最多丢失 1 条命令 | 最慢 |
| `everysec`（默认） | 每秒同步一次 | 最多丢失 1 秒数据 | 推荐 |
| `no` | 由操作系统决定何时同步 | 不确定 | 最快 |

**AOF 重写**：
- AOF 文件会随写操作不断增长
- 当 AOF 文件大小超过上次重写后的 100% 且至少 64 MB 时，触发 `BGREWRITEAOF`
- 重写过程：fork 子进程，遍历当前内存数据生成最小化的 AOF 文件
- 重写期间的新写操作记录到 AOF 重写缓冲区，完成后追加到新文件

**混合持久化**（Redis 4.0+）：
- 同时启用 RDB 和 AOF（`aof-use-rdb-preamble yes`，默认开启）
- AOF 重写时，先写入 RDB 格式的全量数据，再追加 AOF 格式的增量数据
- 兼顾 RDB 的快速恢复和 AOF 的数据安全性

### 持久化配置建议

| 场景 | 推荐配置 | 理由 |
|------|----------|------|
| **缓存**（数据可丢失） | 关闭持久化或仅用 RDB | 性能优先，重启后从源数据库重建 |
| **一般业务** | RDB + AOF（everysec）+ 混合持久化 | 平衡性能和数据安全 |
| **金融/关键数据** | AOF（always）+ RDB 定期备份 | 数据安全性优先 |
| **大规模数据** | RDB 定期快照 + AOF（everysec） | 避免 AOF 文件过大影响恢复速度 |

## 复制与高可用

### 主从复制（Replication）

**原理**：从节点（replica）异步复制主节点（primary）的数据。

**复制过程**：
1. 从节点连接主节点，发送 `SYNC` 或 `PSYNC` 命令
2. 主节点执行 `BGSAVE` 生成 RDB 快照
3. 快照生成期间的新写操作记录到复制缓冲区
4. 主节点将 RDB 快照发送给从节点
5. 从节点加载快照
6. 主节点发送复制缓冲区中的增量命令
7. 此后主节点持续将写操作传播给从节点

**全量复制 vs 增量复制**：
- **全量复制**：首次连接或复制断开时间过长时触发，传输完整 RDB 快照
- **增量复制**（Redis 2.8+）：短暂断开后通过 `PSYNC` 命令和复制偏移量恢复，仅传输缺失的命令

**复制拓扑**：
- 一主多从：主节点可连接多个从节点
- 级联复制：从节点也可以有自己的从节点（链式 A → B → C）
- 从节点默认只读（可配置 `replica-read-only no` 允许写入，但不推荐）

### Sentinel（哨兵）

**职责**：
- **监控**：定期检查主从节点是否存活
- **自动故障转移**：主节点宕机时，选举一个从节点提升为新主节点
- **配置提供者**：客户端向 Sentinel 询问当前主节点地址
- **通知**：故障转移时通知客户端和应用程序

**故障转移流程**：
1. Sentinel 集群发现主节点主观下线（SDOWN）
2. 达到法定数量（quorum）后确认客观下线（ODOWN）
3. 选举一个 Sentinel 作为领导者执行故障转移
4. 从候选从节点中选举新的主节点（优先级、复制偏移量、运行 ID）
5. 将其他从节点重新指向新主节点
6. 通知客户端新主节点地址

**部署建议**：
- 至少部署 3 个 Sentinel 节点（奇数，避免脑裂）
- Sentinel 节点应分布在不同的物理机器上
- 客户端需使用支持 Sentinel 的客户端库

### Cluster（集群）

**原理**：将数据分片到多个节点，每个节点负责 16384 个哈希槽（hash slot）中的一部分。

**数据分片**：
- 键通过 CRC16 算法计算哈希值，再对 16384 取模确定所属槽位
- 公式：`HASH_SLOT = CRC16(key) % 16384`
- 每个节点负责一部分槽位，支持动态迁移

**Hash Tag**：
- 使用 `{}` 包裹键的一部分，强制相关键分配到同一槽位
- 例如：`{user:1000}:name` 和 `{user:1000}:email` 都在 `user:1000` 的槽位
- 使得多键操作（如 `MSET`、`SINTER`）可以在集群中执行

**客户端路由**：
- 客户端发送命令到任意节点
- 如果键不在该节点负责的槽位，返回 `-MOVED` 重定向到正确节点
- 智能客户端会缓存槽位映射，减少重定向

**集群通信**：
- 节点间通过 Gossip 协议交换状态信息
- 使用专用的集群总线端口（客户端端口 + 10000）

**限制**：
- 多键操作要求所有键在同一槽位（使用 Hash Tag）
- 事务（`MULTI`/`EXEC`）仅支持单槽位
- 编号数据库（`SELECT`）在集群模式下仅支持 db 0（Redis 7.2 及之前）
- 从节点不提供写服务，仅在主节点故障时参与选举

## 事务与可编程性

### 事务（Transactions）

**机制**：`MULTI` / `EXEC` / `DISCARD` / `WATCH`

```
MULTI          # 开启事务
SET key1 val1  # 命令入队，返回 QUEUED
SET key2 val2  # 命令入队，返回 QUEUED
EXEC           # 执行所有入队命令
```

**特点**：
- **顺序执行**：事务中的命令按入队顺序执行，不会被其他客户端的命令打断
- **无回滚**：即使某个命令执行失败（如类型错误），其他命令仍会继续执行
- **原子性**：事务要么全部执行，要么全部不执行（`DISCARD` 或连接断开时）

**乐观锁**（WATCH）：
```
WATCH key      # 监视键
MULTI
SET key val    # 如果 key 被其他客户端修改过，EXEC 返回 nil，事务不执行
EXEC
```

### Lua 脚本（Programmability）

**机制**：通过 `EVAL` 或 `EVALSHA` 在服务器端执行 Lua 脚本。

**优势**：
- 多条命令作为单个原子操作执行（脚本执行期间不会处理其他命令）
- 减少网络往返（多条命令一次发送）
- 可以读取命令执行结果并决定后续操作

**限制**：
- 脚本执行时间受 `lua-time-limit` 限制（默认 5 秒）
- 脚本中只能调用 Redis 命令，不能使用外部 I/O 操作
- 脚本执行期间无法被中断（除非达到时间限制）

**Redis 8.0+ 的 Functions**：
- 替代传统 Lua 脚本的更结构化方案
- 支持函数库、持久化、版本管理
- 可通过 `FUNCTION LOAD` 预加载，通过 `FCALL` 调用

## 安全与访问控制

### ACL（Access Control List）

Redis 6.0+ 引入 ACL 系统，替代了之前仅靠 `requirepass` 的简单认证：

| 能力 | 说明 |
|------|------|
| **多用户** | 可创建多个用户，每个用户有不同的权限 |
| **命令权限** | 可限制用户能执行的命令（如 `+get -set` 允许 GET 禁止 SET） |
| **键权限** | 可限制用户能访问的键（如 `~user:*` 仅允许访问 `user:` 前缀的键） |
| **通道权限** | 可限制用户能订阅/发布的 Pub/Sub 通道 |
| **无密码用户** | 可配置无需密码的用户（适合内网可信环境） |

**默认用户**：
- `default` 用户默认无密码且拥有所有权限（兼容旧版本行为）
- 生产环境应禁用 `default` 用户或限制其权限

### 网络安全建议

| 措施 | 说明 |
|------|------|
| **绑定地址** | `bind 127.0.0.1` 或内网 IP，不暴露到公网 |
| **修改默认端口** | `port 6380` 替代默认的 6379 |
| **设置密码** | `requirepass <strong-password>` 或配置 ACL |
| **禁用危险命令** | `rename-command FLUSHALL ""`、`rename-command CONFIG ""` |
| **启用 TLS** | Redis 6.0+ 支持 TLS 加密传输 |
| **防火墙规则** | 仅允许应用服务器 IP 访问 Redis 端口 |

## 协议与通信

### RESP（Redis Serialization Protocol）

Redis 使用 RESP 协议进行客户端-服务器通信，当前主流版本为 RESP2，RESP3 在 Redis 6.0 引入。

**RESP2 数据类型**：

| 类型 | 前缀 | 示例 |
|------|------|------|
| 简单字符串 | `+` | `+OK\r\n` |
| 错误 | `-` | `-ERR unknown command 'foobar'\r\n` |
| 整数 | `:` | `:1000\r\n` |
| 批量字符串 | `$` | `$6\r\nfoobar\r\n` |
| 数组 | `*` | `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n` |

**RESP3 新增类型**（Redis 6.0+）：
- Null（`_`）、Boolean（`#`）、Double（`,`）、Map（`%`）、Set（`~`）、Attribute（`|`）等
- 提供更丰富的类型系统，客户端可以更精确地解析响应

### Pipeline（管道）

**原理**：客户端将多个命令一次性发送给服务器，服务器按顺序执行后一次性返回所有结果。

**优势**：
- 减少网络往返次数（RTT）
- 显著提升批量操作的性能（可达数倍提升）

**注意**：
- Pipeline 不是事务，命令之间不保证原子性
- 管道过大可能占用过多内存，建议分批发送

## 适用场景

| 场景 | 说明 | 推荐数据结构 |
|------|------|-------------|
| **缓存** | 数据库查询结果、API 响应、会话数据 | String、Hash，配合 TTL 和驱逐策略 |
| **会话存储** | Web 应用的用户会话管理 | Hash（存储用户属性）、String（存储序列化会话） |
| **排行榜** | 游戏积分、热门内容排序 | Sorted Set（分数排序） |
| **消息队列** | 任务分发、事件处理 | List（`LPUSH`/`BRPOP`）、Stream（消费者组） |
| **计数器** | 页面访问、API 调用、限流 | String（`INCR`/`DECR` 原子操作） |
| **社交关系** | 好友列表、共同关注、标签 | Set（交集、并集、差集） |
| **实时分析** | UV 统计、在线用户、热度排行 | HyperLogLog、Bitmap、Sorted Set |
| **地理服务** | 附近搜索、地理围栏 | Geospatial（`GEOADD`/`GEORADIUS`） |
| **向量搜索** | RAG 检索、语义缓存、推荐系统 | Vector Set（HNSW 索引） |
| **文档存储** | JSON 文档的存储和查询 | JSON 数据类型 + Redis Query Engine |

## 限制与注意事项

| 限制 | 说明 |
|------|------|
| **内存容量** | 数据集必须能装入内存；单实例理论上限 2^32 个键，实测至少 2.5 亿键/实例 |
| **单线程阻塞** | 慢命令会阻塞所有后续命令，需避免使用 `KEYS *`、大键操作 |
| **事务无回滚** | `MULTI`/`EXEC` 中命令失败不会回滚已执行的命令 |
| **集群多键限制** | 多键操作要求所有键在同一哈希槽（使用 Hash Tag 可强制同槽） |
| **编号数据库** | 官方不推荐使用 `SELECT` 切换数据库，应使用命名空间前缀（如 `user:1000:settings`） |
| **持久化性能影响** | `BGSAVE` 和 `BGREWRITEAOF` 时 fork 子进程可能引起短暂延迟 |
| **复制延迟** | 异步复制可能导致从节点数据滞后，不适合强一致性场景 |
| **许可证变更** | Redis 7.4+ 采用 RSALv2/SSPLv1 双许可，不再符合 OSI 开源定义；如需 BSD 许可可考虑 Valkey |

## 常见误区

| 误区 | 事实 |
|------|------|
| "Redis 是单线程所以很慢" | 单线程消除了锁竞争和上下文切换，内存操作亚毫秒级延迟，单核可达数十万 ops/sec |
| "Redis 只能做缓存" | Redis 是功能丰富的数据结构存储，支持持久化、事务、脚本、搜索、向量数据库等 |
| "Redis 数据不安全，重启就没了" | RDB 和 AOF 持久化可保证数据安全，混合持久化兼顾恢复速度和数据完整性 |
| "Redis 事务支持回滚" | Redis 事务不支持回滚，命令执行失败不会撤销已执行的命令 |
| "Redis Cluster 自动处理所有多键操作" | 集群中多键操作要求所有键在同一哈希槽，否则返回错误 |
| "Redis 7.4+ 仍然是开源软件" | 7.4+ 采用 RSALv2/SSPLv1 双许可，不再满足 OSI 开源定义，官方改称"Community Edition" |

## 相关概念

- **Valkey**：Linux Foundation 管理的 BSD 许可 fork，基于 Redis 7.2.4，许可证变更后的开源替代
- **RESP**：Redis Serialization Protocol，客户端-服务器通信协议
- **RDB/AOF**：Redis 的两种持久化格式
- **Hash Slot**：Redis Cluster 使用 16384 个哈希槽进行数据分片
- **jemalloc**：Redis 在 Linux 上的默认内存分配器，碎片率低于 libc malloc
- **Copy-on-Write**：操作系统机制，RDB 快照和 AOF 重写时 fork 子进程依赖此机制

## 参考资料

- [Redis 官方文档 — 数据类型](https://redis.io/docs/latest/develop/data-types/)
- [Redis 官方文档 — 事务](https://redis.io/docs/latest/develop/interact/transactions/)
- [Redis 官方文档 — 可编程性（Lua 脚本）](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/)
- [Redis 官方文档 — 持久化](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis 官方文档 — 复制](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis 官方文档 — Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [Redis 官方文档 — ACL 安全](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/)
- [Redis GitHub 仓库 README](https://github.com/redis/redis/blob/unstable/README.md)
- [Redis 许可证变更公告](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)（2024-03-20）
