---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - kafka
  - message-queue
  - streaming
  - ordering
  - middleware
aliases:
  - Kafka 消息顺序
  - Kafka message ordering
  - Kafka 顺序消费
  - Kafka partition ordering
source_type: official-doc
source_urls:
  - 'https://docs.confluent.io/kafka/design/producer-design.html'
  - 'https://docs.confluent.io/kafka/design/consumer-design.html'
  - 'https://docs.confluent.io/kafka/design/delivery-semantics.html'
status: verified
---

## 核心结论

Kafka 保证的是 **Partition 级别**的消息顺序，而非 Topic 级别。同一 Partition 内的消息严格有序，但跨 Partition 的消息不保证全局顺序。

## 前置概念

### Topic 与 Partition

- **Topic**：消息的逻辑分类，类似数据库中的表
- **Partition**：Topic 的物理分片，每个 Partition 是一个 **append-only 的有序日志**（ordered log）
- 每条消息在 Partition 内有一个唯一的递增序号，称为 **Offset**

> 每个 Partition 的副本（Replica）具有相同的日志和相同的 Offset 序列 [^consumer-design]。

### Producer 与 Consumer

- **Producer**：向 Topic 发布消息的客户端
- **Consumer**：从 Broker 拉取（pull）消息的客户端
- **Consumer Group**：同一应用中协同消费的一组 Consumer，每个 Partition 在同一时刻只被 Group 内的一个 Consumer 消费 [^consumer-design]

## 顺序保证的三层机制

### 1. Producer 端：消息路由到正确的 Partition

Producer 通过 **Partitioning 策略** 决定消息写入哪个 Partition：

| 策略 | 行为 | 顺序影响 |
|------|------|----------|
| 指定 Key | Kafka 对 Key 做 Hash，相同 Key 始终路由到同一 Partition | 保证相同 Key 的消息有序 |
| 不指定 Key | Round-robin 或随机分配 | 不保证顺序 |

```java
// 指定 Key 确保同一用户订单消息进入同一 Partition
ProducerRecord<String, String> record = 
    new ProducerRecord<>("orders", "user-123", "order-data");
```

Producer 还支持自定义 Partitioner 覆盖默认的 Hash 分区逻辑 [^producer-design]。

### 2. Broker 端：Partition 内有序写入

每个 Partition 是一个 **append-only log**，消息按到达顺序追加写入：

- 消息在 Partition 内按 Offset 严格递增排列
- 同一时刻只有一个 Leader 副本接受写入请求
- Follower 副本从 Leader 同步数据，保持相同的 Offset 序列

这是 Kafka 顺序保证的 **核心基础** —— Partition 内的日志结构天然有序。

### 3. Consumer 端：单 Partition 单 Consumer 消费

Kafka 的 Consumer Group 协议保证：

- **每个 Partition 在同一时刻只被 Group 内的一个 Consumer 消费** [^consumer-design]
- Consumer 按 Offset 顺序拉取消息
- Consumer 通过 `__consumer_offsets` 内部 Topic 记录消费进度 [^consumer-design]

这意味着只要消息进入了同一 Partition，就只有一个 Consumer 按顺序处理它，不会出现并发消费导致的乱序。

#### Partition 与 Consumer 的数量关系

| 场景 | 结果 |
|------|------|
| **同一 Consumer Group 内** | 一个 Partition **只能被 1 个 Consumer 消费** |
| **跨 Consumer Group** | 一个 Partition 可以被**任意多个** Consumer 消费（每个 Group 独立维护 Offset） |
| **Consumer 数量 > Partition 数量** | 多余的 Consumer 空闲（idle），不分配任何 Partition |

```
Topic: orders (3 partitions)

Consumer Group A:
  P0 → Consumer-A1
  P1 → Consumer-A2
  P2 → Consumer-A3

Consumer Group B:
  P0 → Consumer-B1  ← 同样消费 P0，但 Offset 独立
  P1 → Consumer-B2
  P2 → Consumer-B3
```

**推论**：在同一 Consumer Group 内，**Consumer 数量 ≤ Partition 数量** 才有意义。如果需要提高消费并行度，应增加 Partition 数量。

## 增强顺序保证的机制

### 幂等性 Producer（Idempotent Producer）

自 Kafka 0.11.0.0 起，Producer 支持幂等性配置 [^delivery-semantics]：

```properties
enable.idempotence=true
```

幂等性 Producer 的作用：

- Broker 为每个 Producer 分配一个 Producer ID（PID）
- Producer 发送的每条消息携带序列号（Sequence Number）
- Broker 通过 PID + Sequence Number 去重，**重试不会产生重复消息**
- **保证日志顺序不被重试打乱** [^delivery-semantics]

开启 `enable.idempotence=true` 时，`acks` 自动设为 `all`，`retries` 设为 `Integer.MAX_VALUE` [^delivery-semantics]。

### 事务型 Producer（Transactional Producer）

同样自 0.11.0.0 起，Producer 支持事务 [^delivery-semantics]：

```properties
transactional.id=my-transactional-producer
```

事务型 Producer 提供：

- 跨 Partition 的原子写入（要么全部成功，要么全部失败）
- 结合幂等性，保证重试不产生重复且顺序不变
- Consumer 通过 `isolation.level` 控制是否读取未提交的消息：
  - `read_uncommitted`（默认）：可见所有消息，包括中止事务中的消息
  - `read_committed`：仅可见已提交事务的消息 [^delivery-semantics]

## 顺序被破坏的常见场景

### 1. 增加 Partition 数量

**Kafka 不支持减少 Partition 数量**，增加 Partition 后：

- 已有 Key 的 Hash 映射可能改变，相同 Key 的消息可能路由到不同 Partition
- 新增 Partition 后写入的消息与之前写入的消息 **无法保证全局顺序**

> 增加 Topic 的 Partition 数量后，如果 Producer 使用带 Key 的自定义分区器，消息的路由方式可能改变 [^partition-determination]。

### 2. Producer 重试导致乱序（未开启幂等性时）

未开启 `enable.idempotence` 时，如果 Producer 发送超时后重试：

- 原始消息可能已成功写入，重试导致重复
- 重试消息可能比后续消息晚到达，造成乱序

### 3. Consumer Rebalance

Consumer Group 发生 Rebalance 时（成员加入/退出、Topic 元数据变化）：

- Partition 重新分配给不同的 Consumer
- 新 Consumer 从上次提交的 Offset 开始消费
- 如果 Offset 提交不及时，可能导致部分消息被重复消费（但不会乱序，因为 Partition 内仍然有序）

Kafka 4.0 引入了新的 Consumer Rebalance 协议，支持增量 Rebalance，减少 Rebalance 期间的消费中断 [^consumer-design]。

### 4. 多 Consumer 并发处理

即使消息按顺序到达 Consumer，如果 Consumer 内部使用多线程异步处理：

- 处理完成的顺序可能与接收顺序不同
- 需要在 Consumer 端自行保证处理逻辑的顺序性

## 最佳实践

### 需要严格顺序的场景

1. **使用单 Partition Topic**：将 Topic 的 Partition 数量设为 1，保证全局有序（牺牲吞吐量和并行度）
2. **按业务 Key 分区**：将需要保序的消息使用相同的 Key，确保进入同一 Partition
3. **开启幂等性 Producer**：`enable.idempotence=true`，防止重试导致乱序和重复
4. **Consumer 端单线程处理**：或使用有序队列保证处理顺序

### 需要权衡的因素

| 需求 | 配置建议 | 代价 |
|------|----------|------|
| 全局严格有序 | `partitions=1` | 吞吐量低，无并行消费 |
| 局部有序（按 Key） | 指定 Key + 多 Partition | 同 Key 有序，跨 Key 无序 |
| 高吞吐 + 容错 | 多 Partition + 幂等 Producer | 仅保证 Partition 内有序 |
| 跨 Partition 有序 | 事务型 Producer + `read_committed` | 延迟增加，吞吐量降低 |

## 相关概念

- [[Kafka 核心概念]]：Topic、Partition、Broker、Consumer Group 等基础概念
- [[Kafka 消息投递语义]]：At-most-once、At-least-once、Exactly-once 语义
- [[Kafka 副本机制]]：ISR、Leader/Follower、Unclean Leader Election

## 参考资料

[^producer-design]: [Kafka Producer Design - Confluent Documentation](https://docs.confluent.io/kafka/design/producer-design.html)
[^consumer-design]: [Kafka Consumer Design - Confluent Documentation](https://docs.confluent.io/kafka/design/consumer-design.html)
[^delivery-semantics]: [Kafka Message Delivery Guarantees - Confluent Documentation](https://docs.confluent.io/kafka/design/delivery-semantics.html)
[^partition-determination]: [Choose and Change Partition Count - Confluent Documentation](https://docs.confluent.io/kafka/operations-tools/partition-determination.html)
