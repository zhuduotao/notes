---
created: '2026-04-22'
updated: '2026-04-22'
tags:
  - redis
  - message-queue
  - pubsub
  - streams
  - async
  - middleware
aliases:
  - Redis 消息队列
  - Redis Message Queue
  - Redis Streams
  - Redis Pub/Sub
  - 基于 Redis 的消息队列
source_type: mixed
source_urls:
  - 'https://redis.io/docs/latest/develop/data-types/streams/'
  - 'https://redis.io/docs/latest/develop/pubsub/'
  - >-
    https://www.dragonflydb.io/guides/use-redis-as-a-message-queue-and-a-quick-tutorial-to-get-started
  - 'https://github.com/taskforcesh/bullmq'
status: verified
---
## 概述

基于 Redis 的消息队列是利用 Redis 内置数据结构实现异步消息传递的一种轻量级方案。Redis 提供三种主要方式实现消息队列：**List 数据结构**、**Pub/Sub（发布/订阅）** 和 **Streams（流）**。

与专用消息中间件（如 Kafka、RabbitMQ）相比，Redis 消息队列的优势在于部署简单、性能高、无需引入额外基础设施；但在消息持久化、消费保证、回溯能力等方面存在不同程度的局限。

## 三种实现方式对比

| 特性 | List | Pub/Sub | Streams |
|------|------|---------|---------|
| 引入版本 | 早期版本 | 早期版本 | Redis 5.0+ |
| 消息持久化 | ✅（需手动管理） | ❌（消息不存储） | ✅（自动持久化） |
| 投递语义 | 至多一次 | 至多一次 | 至多一次 / 至少一次 |
| 消息回溯 | ❌（已消费即删除） | ❌ | ✅（支持按 ID 范围读取） |
| 消费者组 | ❌ | ❌ | ✅ |
| 消息确认机制 | ❌ | ❌ | ✅（XACK） |
| 扇出（Fan-out） | 需多 List | ✅（原生支持） | ✅（多消费者组） |
| 适用场景 | 简单任务队列 | 实时事件通知 | 生产级消息队列 |

## 方案一：基于 List 的消息队列

### 核心命令

List 提供了 FIFO 队列的基础操作：

```redis
# 生产者：从左侧入队
LPUSH myqueue '{"task_id": "1", "action": "process"}'

# 消费者：从右侧出队（非阻塞）
RPOP myqueue

# 消费者：从右侧出队（阻塞式，最多等待 60 秒）
BRPOP myqueue 60
```

### 工作模式

- **LPUSH + RPOP**：实现 FIFO 队列
- **LPUSH + LPOP**：实现 LIFO 栈
- **BRPOP / BLPOP**：阻塞式弹出，避免轮询开销

### 可靠队列模式（RPOPLPUSH）

为防止消费者处理失败导致消息丢失，可使用 `RPOPLPUSH` 将消息转移到"处理中"队列：

```redis
# 从队列取出消息并推入处理中队列（原子操作）
RPOPLPUSH myqueue myqueue:processing

# 处理完成后从处理中队列删除
LREM myqueue:processing 1 "message_content"
```

若消费者崩溃，可通过监控 `myqueue:processing` 队列将超时未处理的消息重新放回主队列。

### 限制

- **无内置消息确认**：消费者弹出消息后若崩溃，消息永久丢失
- **不支持多消费者协调**：多个消费者竞争时无法保证负载均衡
- **无消息回溯**：已消费的消息无法重新读取
- **需手动管理内存**：无自动过期机制，需配合 `EXPIRE` 或定期清理

### 适用场景

- 轻量级任务队列（如异步发送邮件、生成报表）
- 对消息丢失不敏感的场景
- 单一消费者或简单竞争消费模式

## 方案二：基于 Pub/Sub 的消息队列

### 核心命令

```redis
# 订阅频道
SUBSCRIBE channel1 channel2

# 模式匹配订阅
PSUBSCRIBE news.*

# 发布消息
PUBLISH channel1 "Hello World"
```

### 工作模式

Pub/Sub 实现的是**发布/订阅消息范式**：
- 发布者将消息发送到频道，无需知道订阅者是谁
- 订阅者接收感兴趣频道的所有消息
- 消息按发布顺序推送给订阅者

### 投递语义

Redis Pub/Sub 提供 **至多一次（at-most-once）** 投递保证：
- 消息发送后不会持久化
- 如果订阅者离线或处理失败，消息永久丢失
- 无法重播历史消息

### Sharded Pub/Sub（Redis 7.0+）

Redis 7.0 引入分片 Pub/Sub，将频道分配到与 key 相同的哈希槽：

```redis
SSUBSCRIBE shard-channel
SPUBLISH shard-channel "message"
SUNSUBSCRIBE shard-channel
```

优势：
- 消息传播限制在集群分片内，减少集群总线流量
- 支持水平扩展 Pub/Sub 使用量
- 客户端可连接到主节点或其副本进行订阅

### 限制

- **消息不持久化**：离线订阅者无法接收历史消息
- **无消息确认**：无法知道消息是否被成功处理
- **订阅者状态无关**：发布者无法获知有多少订阅者
- **数据库无关**：Pub/Sub 与 keyspace 和数据库编号无关，db 10 发布的消息会被 db 1 的订阅者收到

### 适用场景

- 实时事件通知（如聊天室、实时仪表盘）
- 配置变更广播
- 服务间轻量级事件通信
- 对消息丢失可容忍的场景

## 方案三：基于 Streams 的消息队列（推荐）

### 核心概念

Redis Streams 是 Redis 5.0 引入的**仅追加（append-only）日志型数据结构**，专为消息队列场景设计。

每个 Stream 条目包含：
- **消息 ID**：格式为 `timestamp-sequence`（如 `1697012345678-0`），保证有序性
- **字段-值对**：类似 Hash 结构的键值对数据

### 核心命令

#### 写入消息

```redis
# 添加消息（* 表示自动生成 ID）
XADD mystream * sensor_id 1234 temperature 25.6

# 指定 ID 添加
XADD mystream 1697012345678-0 sensor_id 1234 temperature 25.6

# 限制 Stream 长度（自动裁剪旧消息）
XADD mystream MAXLEN ~ 1000 * field value
```

#### 读取消息

```redis
# 读取全部消息
XRANGE mystream - +

# 按 ID 范围读取
XRANGE mystream 1697012345678-0 1697012345679-0

# 读取最新消息
XREVRANGE mystream + - COUNT 10
```

#### 消费者组

```redis
# 创建消费者组（$ 表示从最新消息开始）
XGROUP CREATE mystream mygroup $ MKSTREAM

# 从消费者组读取消息（BLOCK 表示阻塞等待）
XREADGROUP GROUP mygroup consumer1 COUNT 10 BLOCK 5000 STREAMS mystream >

# 确认消息已处理
XACK mystream mygroup 1697012345678-0

# 查看待处理消息（PEL - Pending Entries List）
XPENDING mystream mygroup

# 获取待处理消息详情
XPENDING mystream mygroup - + 10 consumer1
```

#### 消息转移与修复

```redis
# 将超时未确认的消息转移给其他消费者
XCLAIM mystream mygroup consumer2 30000 1697012345678-0

# 自动读取 PEL 中超时的消息并转移（Redis 6.2+）
XAUTOCLAIM mystream mygroup consumer2 30000 0-0 COUNT 10
```

### 投递语义

Streams 支持两种投递语义：

| 语义 | 实现方式 | 说明 |
|------|----------|------|
| 至多一次 | `XREAD` 不使用消费者组 | 读取后不确认，消息不会被重新投递 |
| 至少一次 | `XREADGROUP` + `XACK` | 消息进入 PEL，确认后才会被移除；未确认的消息可被重新投递 |

### 内存管理

Stream 默认不自动删除旧消息，需主动管理内存：

```redis
# 写入时限制最大长度（近似裁剪，性能更好）
XADD mystream MAXLEN ~ 10000 * field value

# 写入时限制最大占用空间（Redis 7.0+）
XADD mystream MINID ~ 1697000000000-0 * field value

# 手动裁剪
XTRIM mystream MAXLEN ~ 5000
```

`~` 符号表示近似裁剪，允许第一个宏节点不被删除以提升性能。

### 消费者组工作原理

1. **消息分配**：每个消息只会被组内一个消费者读取
2. **待处理列表（PEL）**：读取但未确认的消息记录在 PEL 中
3. **消息确认**：消费者处理完成后调用 `XACK` 从 PEL 移除
4. **故障恢复**：通过 `XCLAIM` 或 `XAUTOCLAIM` 将超时未确认的消息重新分配

### 限制

- **无内置死信队列**：需自行实现失败消息处理逻辑
- **内存占用**：消息持久化在内存中，需配合 `MAXLEN`/`MINID` 管理
- **单线程限制**：单实例写入吞吐量受限于 Redis 单线程模型
- **不如 Kafka 功能丰富**：无分区、偏移量管理、精确一次语义等高级特性

### 适用场景

- 生产级异步任务处理
- 事件溯源（Event Sourcing）
- 微服务间消息通信
- 需要消息持久化和回溯的场景
- 需要多消费者并行处理的场景

## 主流框架封装

直接使用 Redis 命令构建消息队列需要处理大量边界情况。生产环境推荐使用成熟框架：

| 框架 | 语言 | 特点 |
|------|------|------|
| **BullMQ** | Node.js / Python | 基于 Redis 数据和 Lua 脚本，支持延迟任务、重试、并发控制、优先级 |
| **Sidekiq** | Ruby | 支持重试、失败追踪、调度、中间件扩展 |
| **Celery** | Python | 分布式任务队列，支持调度、重试、结果追踪，Redis 是最常用 Broker |
| **RQ** | Python | 轻量级 Python 任务队列，支持 Worker、重试、监控 |
| **Resque** | Ruby | Redis 后端，关注可靠性和进程隔离 |

### BullMQ 示例（Node.js）

```typescript
import { Queue, Worker } from 'bullmq';

// 创建队列
const queue = new Queue('email-queue', {
  connection: { host: 'localhost', port: 6379 }
});

// 添加任务
await queue.add('send-email', {
  to: 'user@example.com',
  subject: 'Hello',
  body: 'World'
}, {
  attempts: 3,        // 重试次数
  backoff: { type: 'exponential', delay: 1000 },  // 退避策略
  removeOnComplete: 100,  // 完成后保留最近 100 条
  removeOnFail: 5000      // 失败后保留最近 5000 条
});

// 创建 Worker 处理任务
const worker = new Worker('email-queue', async (job) => {
  console.log(`Processing job ${job.id}:`, job.data);
  // 发送邮件逻辑
}, {
  connection: { host: 'localhost', port: 6379 },
  concurrency: 10  // 并发处理数
});

worker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`);
});

worker.on('failed', (job, err) => {
  console.log(`Job ${job.id} failed:`, err.message);
});
```

## 方案选型建议

| 需求 | 推荐方案 |
|------|----------|
| 简单任务队列，快速实现 | List（LPUSH + BRPOP） |
| 实时事件广播，不关心持久化 | Pub/Sub |
| 集群环境下大规模 Pub/Sub | Sharded Pub/Sub（Redis 7.0+） |
| 需要消息持久化和确认 | Streams + 消费者组 |
| 生产级任务调度，需要重试/优先级 | BullMQ / Celery / Sidekiq |
| 高可靠消息系统（金融级） | 考虑 Kafka / RabbitMQ |

## 最佳实践

1. **消息序列化**：使用 JSON 或 Protocol Buffers，确保跨语言兼容
2. **消息 ID 设计**：Streams 自动生成 ID；List 方案建议在消息体内包含业务 ID
3. **内存管理**：
   - List：配合 `LTRIM` 或 TTL 防止无限增长
   - Streams：使用 `MAXLEN ~` 或 `MINID ~` 近似裁剪
4. **错误处理**：
   - 实现重试机制（指数退避）
   - 记录失败消息到死信队列（可用单独的 Stream 或 List）
5. **监控指标**：
   - `LLEN` 监控 List 队列长度
   - `XPENDING` 监控 Streams 待处理消息
   - `PUBSUB NUMSUB` 查看 Pub/Sub 订阅数
6. **高可用**：
   - 使用 Redis 集群或哨兵模式
   - Streams 消费者组在集群模式下需连接正确的分片节点
7. **避免大消息**：Redis 是内存数据库，单条消息建议不超过 1MB

## 常见问题

### Q: Redis 消息队列会丢消息吗？

- **List**：消费者 `RPOP` 后崩溃则消息丢失；使用 `RPOPLPUSH` 可降低风险
- **Pub/Sub**：订阅者离线期间的消息全部丢失
- **Streams**：使用消费者组 + `XACK` 可实现至少一次投递，不会因消费者崩溃丢消息

### Q: 如何保证消息顺序？

- **List**：天然 FIFO 顺序
- **Pub/Sub**：消息按发布顺序推送
- **Streams**：消息 ID 保证全局有序；消费者组内同一消费者按序处理，但多消费者并行时不保证全局顺序

### Q: Redis 消息队列能替代 Kafka 吗？

不能完全替代。Redis 适合：
- 中小规模消息量
- 已有 Redis 基础设施，希望减少组件数量
- 对精确一次语义、消息回溯、分区等高级特性无强需求

以下场景应使用 Kafka：
- 海量消息吞吐（百万级 QPS）
- 需要长时间消息保留（天/周级）
- 需要精确一次语义（exactly-once）
- 复杂的分区和消费者偏移量管理

### Q: Streams 和 Pub/Sub 可以同时使用吗？

可以。常见模式是：
- Streams 用于持久化消息和可靠消费
- Pub/Sub 用于实时通知（如通知消费者有新消息到达）

## 相关概念

- **消息队列（Message Queue）**：异步通信模式，解耦生产者和消费者
- **投递语义**：至多一次（at-most-once）、至少一次（at-least-once）、精确一次（exactly-once）
- **消费者组（Consumer Group）**：多个消费者协同消费同一消息源，每条消息只被组内一个消费者处理
- **扇出（Fan-out）**：一条消息被多个消费者独立消费
- **死信队列（Dead Letter Queue）**：存储多次处理失败的消息
- **背压（Backpressure）**：消费者处理速度跟不上生产者时的流量控制机制

## 参考资料

- [Redis 官方文档：Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis 官方文档：Pub/Sub](https://redis.io/docs/latest/develop/pubsub/)
- [Redis 官方文档：Sharded Pub/Sub](https://redis.io/docs/latest/develop/pubsub/#sharded-pubsub)
- [Redis 命令：XADD](https://redis.io/commands/xadd/)
- [Redis 命令：XREADGROUP](https://redis.io/commands/xreadgroup/)
- [Redis 命令：XACK](https://redis.io/commands/xack/)
- [Redis 命令：PUBLISH](https://redis.io/commands/publish/)
- [Redis 命令：SUBSCRIBE](https://redis.io/commands/subscribe/)
- [Redis 命令：LPUSH](https://redis.io/commands/lpush/)
- [Redis 命令：BRPOP](https://redis.io/commands/brpop/)
- [Dragonfly: Use Redis as a Message Queue](https://www.dragonflydb.io/guides/use-redis-as-a-message-queue-and-a-quick-tutorial-to-get-started)
- [BullMQ 官方仓库](https://github.com/taskforcesh/bullmq)
