---
created: '2026-04-17'
updated: '2026-04-17'
tags:
  - kafka
  - message-queue
  - reliability
  - data-durability
  - middleware
  - delivery-semantics
aliases:
  - Kafka 消息丢失
  - Kafka message loss
  - Kafka 消息可靠性
  - Kafka reliability
  - Kafka 消息不丢失配置
  - Kafka exactly once
source_type: official-doc
source_urls:
  - 'https://docs.confluent.io/kafka/design/delivery-semantics.html'
  - 'https://docs.confluent.io/kafka/design/producer-design.html'
  - 'https://docs.confluent.io/kafka/design/consumer-design.html'
  - 'https://docs.confluent.io/kafka/design/replication.html'
status: verified
---

## 核心结论

Kafka 通过 **Producer 端确认机制（acks）**、**Broker 端副本同步（ISR）** 和 **Consumer 端 Offset 管理** 三层机制协同工作来避免消息丢失。默认情况下 Kafka 保证 **At-least-once（至少一次）** 投递语义；通过开启幂等性 Producer 和事务型 Producer，可进一步实现 **Exactly-once（精确一次）** 语义 [^delivery-semantics]。

消息丢失可能发生在三个环节：

| 环节 | 丢失原因 | 防护机制 |
|------|----------|----------|
| **Producer → Broker** | 发送失败未重试、未等待 Broker 确认 | `acks=all` + `retries` + 幂等性 |
| **Broker 内部** | Leader 宕机且数据未同步到 Follower | ISR + `min.insync.replicas` |
| **Broker → Consumer** | Consumer 崩溃后 Offset 已提交但消息未处理 | 先处理后提交 Offset |

## 消息投递语义概述

Kafka 提供三种消息投递语义 [^delivery-semantics]：

| 语义 | 定义 | 消息丢失 | 消息重复 | 适用场景 |
|------|------|----------|----------|----------|
| **At most once** | 消息最多投递一次 | 可能丢失 | 不会重复 | 日志采集、指标监控（可容忍丢失） |
| **At least once** | 消息至少投递一次 | 不会丢失 | 可能重复 | 大多数业务场景（推荐默认） |
| **Exactly once** | 消息精确投递一次 | 不会丢失 | 不会重复 | 金融交易、计费等强一致性场景 |

## Producer 端：如何避免消息发送丢失

### acks 配置（核心机制）

`acks` 参数控制 Producer 发送消息后需要等待多少副本确认才认为发送成功 [^replication]：

| acks 值 | 行为 | 持久性保证 | 延迟 | 丢失风险 |
|---------|------|-----------|------|----------|
| `acks=0` | Producer 发送后不等待任何确认 | 无 | 最低 | **最高**：Leader 宕机或网络故障时消息丢失 |
| `acks=1`（默认） | Leader 写入本地日志后即确认 | 部分：Leader 写入成功即确认，但可能未同步到 Follower | 低 | **中等**：Leader 写入成功后宕机且 Follower 未同步时丢失 |
| `acks=all`（或 `acks=-1`） | 所有 ISR（In-Sync Replica）都写入成功后才确认 | **最高**：只要至少一个 ISR 存活，消息不丢失 | 最高 | **最低**：仅当所有 ISR 同时宕机时才可能丢失 |

```java
Properties props = new Properties();
props.put("acks", "all");              // 等待所有 ISR 确认
props.put("retries", Integer.MAX_VALUE); // 无限重试
props.put("enable.idempotence", true);   // 开启幂等性
```

> **推荐配置**：对数据可靠性要求高的场景，必须使用 `acks=all` [^replication]。

### 重试机制（retries）

Producer 发送失败时（如网络超时、Leader 选举中），通过重试避免消息丢失：

```properties
retries=Integer.MAX_VALUE    # 最大重试次数（默认值因版本而异）
retry.backoff.ms=100         # 重试间隔（ms）
```

**未开启幂等性时的风险**：

- 原始消息可能已成功写入，重试导致**重复消息**
- 重试消息可能晚于后续消息到达，造成**乱序**

### 幂等性 Producer（Idempotent Producer）

自 Kafka **0.11.0.0** 起，Producer 支持幂等性配置 [^delivery-semantics]：

```properties
enable.idempotence=true
```

**工作原理**：

1. Broker 为每个 Producer 分配一个唯一的 Producer ID（PID）
2. Producer 发送的每条消息携带递增的 Sequence Number
3. Broker 通过 `PID + Partition + Sequence Number` 去重
4. **重试不会产生重复消息，且日志顺序不被打乱** [^delivery-semantics]

**自动联动配置**：开启 `enable.idempotence=true` 时，Kafka 自动设置 [^delivery-semantics]：

- `acks=all`（确保所有 ISR 确认）
- `retries=Integer.MAX_VALUE`（确保无限重试）
- `max.in.flight.requests.per.connection=5`（允许最多 5 个未确认请求，Kafka 3.0+ 之前为 1）

> **注意**：幂等性 Producer 仅保证 **单 Partition 单会话** 内的幂等。如果 Producer 重启（PID 改变），跨会话的幂等需要事务型 Producer [^delivery-semantics]。

### 事务型 Producer（Transactional Producer）

同样自 **0.11.0.0** 起，Producer 支持事务 [^delivery-semantics]：

```properties
transactional.id=my-transactional-producer  # 必须设置唯一的 transactional.id
```

**事务型 Producer 提供**：

- 跨 Partition、跨 Session 的原子写入（要么全部成功，要么全部失败）
- 结合幂等性，保证重试不产生重复且顺序不变
- 支持 **Read-Process-Write** 模式下的 Exactly-once 语义

**Consumer 端配合配置**：

```properties
isolation.level=read_committed  # 仅读取已提交事务的消息（默认 read_uncommitted）
```

| isolation.level | 行为 |
|-----------------|------|
| `read_uncommitted`（默认） | 可见所有消息，包括中止事务中的消息 |
| `read_committed` | 仅可见已提交事务的消息，中止事务的消息对 Consumer 不可见 |

### 同步发送 vs 异步发送

```java
// 同步发送（阻塞等待确认）
Future<RecordMetadata> future = producer.send(record);
RecordMetadata metadata = future.get(); // 阻塞直到确认

// 异步发送（带回调）
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 发送失败，记录日志或重试
        log.error("Send failed", exception);
    }
});

// Fire-and-forget（不推荐用于重要数据）
producer.send(record); // 不等待确认，可能丢失
```

> **最佳实践**：重要数据不要使用 fire-and-forget 模式。至少使用异步发送 + 回调处理失败 [^delivery-semantics]。

## Broker 端：如何避免存储丢失

### 副本机制（Replication）

Kafka 将每个 Partition 的日志复制到多个 Broker 上 [^replication]：

- **Leader**：每个 Partition 有一个 Leader，所有读写请求都走 Leader
- **Follower**：零个或多个 Follower，从 Leader 拉取数据保持同步
- **Replication Factor**：Topic 级别的副本数量（包含 Leader），建议 ≥ 3

```
Topic: orders (replication.factor=3)

Broker 1:  P0 (Leader), P1 (Follower), P2 (Follower)
Broker 2:  P0 (Follower), P1 (Leader), P2 (Follower)
Broker 3:  P0 (Follower), P1 (Follower), P2 (Leader)
```

### ISR（In-Sync Replica）

ISR 是与 Leader 保持同步的副本集合 [^replication]：

**Broker 被视为"存活"的条件**：

1. 能够与 Controller 维持会话
2. 如果是 Follower，必须持续从 Leader 复制数据且不能落后太多

**ISR 判定标准**：

```properties
replica.lag.time.max.ms=30000  # Follower 超过 30 秒未向 Leader 发送请求，则移出 ISR
```

**关键保证**：Kafka 保证只要 **至少一个 ISR 存活**，已提交的消息就不会丢失 [^replication]。

### min.insync.replicas

Topic 级别的配置，控制写入成功所需的最小 ISR 数量 [^replication]：

```properties
# Topic 级别配置
min.insync.replicas=2  # 至少需要 2 个 ISR 确认写入成功
```

**与 acks=all 配合使用**：

| 场景 | acks=all + min.insync.replicas=1 | acks=all + min.insync.replicas=2 |
|------|----------------------------------|----------------------------------|
| ISR 数量 = 3 | 1 个确认即可 | 需要 2 个确认 |
| ISR 数量 = 2 | 2 个确认即可 | 2 个确认即可 |
| ISR 数量 = 1 | 1 个确认即可 | **写入失败**（Producer 收到 NotEnoughReplicasException） |
| ISR 数量 = 0 | **写入失败** | **写入失败** |

> **推荐配置**：`replication.factor=3` + `min.insync.replicas=2` + `acks=all`，可容忍 **1 个 Broker 宕机** 而不丢失消息 [^replication]。

### Unclean Leader Election

当所有 ISR 都宕机时，Kafka 面临一致性与可用性的权衡 [^replication]：

| 策略 | 配置 | 行为 | 数据丢失风险 |
|------|------|------|-------------|
| 等待 ISR 恢复（默认） | `unclean.leader.election.enable=false` | 等待 ISR 中的副本恢复并选举为 Leader | 无丢失，但 Topic 不可用 |
| 允许非 ISR 成为 Leader | `unclean.leader.election.enable=true` | 选择第一个恢复的非 ISR 副本为 Leader | **可能丢失**：非 ISR 副本缺少部分已提交消息 |

> **默认行为**：自 Kafka 0.11.0.0 起，默认 `unclean.leader.election.enable=false`，优先保证一致性 [^replication]。

### 刷盘策略（Flush Policy）

Kafka **不依赖 fsync 来保证持久性**，而是依赖副本机制 [^replication]：

- 数据写入后先进入 OS PageCache，由 OS 决定何时刷盘
- 每条消息的"已提交"定义是：**所有 ISR 都已应用到其日志**（不一定已 fsync 到磁盘）
- 如果 Broker 崩溃，未刷盘的数据可通过从其他 ISR 重新同步恢复

```properties
# 不推荐手动配置刷盘策略，依赖 OS 和副本机制即可
log.flush.interval.messages=Long.MaxValue  # 默认：不强制刷盘
log.flush.interval.ms=Long.MaxValue        # 默认：不强制刷盘
```

> **为什么不用 fsync**：每次写入都 fsync 会导致性能下降 **2-3 个数量级** [^replication]。Kafka 的设计选择是用副本机制替代 fsync。

## Consumer 端：如何避免消费丢失

### Offset 提交时机（核心机制）

Consumer 通过 Offset 记录消费进度，Offset 提交时机决定了消息是否会丢失或重复 [^consumer-design]：

**At most once（可能丢失）**：

```java
// 先提交 Offset，再处理消息
consumer.poll(Duration.ofMillis(1000));
consumer.commitSync();  // 先提交
processRecords(records); // 后处理 → 如果处理时崩溃，消息丢失
```

**At least once（不会丢失，可能重复）**：

```java
// 先处理消息，再提交 Offset
consumer.poll(Duration.ofMillis(1000));
processRecords(records); // 先处理
consumer.commitSync();   // 后提交 → 如果提交前崩溃，消息被重复消费
```

**Exactly once（不会丢失，不会重复）**：

- 使用 Kafka Streams 或 Kafka Connect 时，Offset 与处理结果写入同一事务
- 使用外部系统时，需自行实现幂等写入或两阶段提交

### 自动提交 vs 手动提交

| 配置 | 行为 | 丢失风险 | 适用场景 |
|------|------|----------|----------|
| `enable.auto.commit=true`（默认） | 按 `auto.commit.interval.ms` 自动提交 | **有**：提交间隔内 Consumer 崩溃，消息丢失 | 可容忍重复或丢失的场景 |
| `enable.auto.commit=false` | 手动调用 `commitSync()` 或 `commitAsync()` | **无**（配合先处理后提交） | 大多数业务场景 |

```properties
enable.auto.commit=false          # 关闭自动提交
auto.commit.interval.ms=5000      # 自动提交间隔（仅 auto.commit=true 时生效）
```

### 手动提交方式

```java
// 同步提交（阻塞直到确认）
consumer.commitSync();

// 异步提交（不阻塞，带回调）
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed", exception);
    }
});

// 提交指定 Offset
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(new TopicPartition("topic", 0), new OffsetAndMetadata(currentOffset + 1));
consumer.commitSync(offsets);
```

> **最佳实践**：处理完消息后使用 `commitSync()` 确保 Offset 已提交；高吞吐场景可使用 `commitAsync()` + 失败重试 [^consumer-design]。

### Consumer Rebalance 期间的 Offset 保护

Consumer Group 发生 Rebalance 时（成员加入/退出），新 Consumer 从上次提交的 Offset 开始消费 [^consumer-design]：

```java
// 使用 ConsumerRebalanceListener 在 Rebalance 前提交 Offset
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync(); // Rebalance 前提交，避免丢失
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 可选：从指定 Offset 开始消费
    }
});
```

Kafka 4.0 引入了新的 Consumer Rebalance 协议，支持增量 Rebalance，减少 Rebalance 期间的消费中断 [^consumer-design]。

## 端到端 Exactly-once 配置

要实现从 Producer 到 Consumer 的端到端 Exactly-once 语义，需要以下配置组合 [^delivery-semantics]：

### Producer 端

```properties
enable.idempotence=true          # 幂等性（单会话）
# 或
transactional.id=my-producer     # 事务型（跨会话、跨 Partition）
acks=all                         # 等待所有 ISR 确认
retries=Integer.MAX_VALUE        # 无限重试
```

### Broker 端

```properties
# Topic 级别
replication.factor=3             # 3 个副本
min.insync.replicas=2            # 至少 2 个 ISR 确认
unclean.leader.election.enable=false  # 禁止非 ISR 选举
```

### Consumer 端

```properties
enable.auto.commit=false         # 关闭自动提交
isolation.level=read_committed   # 仅读取已提交事务（配合事务型 Producer）
```

### 处理逻辑

```java
// 先处理，后提交
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        process(record);  // 处理消息（建议下游系统做幂等）
    }
    consumer.commitSync();  // 处理完成后提交 Offset
}
```

## 消息丢失的常见场景与排查

### 场景 1：acks=0 或 acks=1 时 Leader 宕机

**现象**：Producer 发送成功，但消息丢失。

**原因**：

- `acks=0`：Producer 不等待确认，网络故障或 Broker 宕机时消息丢失
- `acks=1`：Leader 写入成功即确认，但 Follower 未同步时 Leader 宕机，消息丢失

**解决**：使用 `acks=all` + `min.insync.replicas=2`。

### 场景 2：ISR 全部宕机且开启 Unclean Leader Election

**现象**：非 ISR 副本成为新 Leader，部分已提交消息丢失。

**原因**：`unclean.leader.election.enable=true` 时，数据落后的副本成为 Leader。

**解决**：设置 `unclean.leader.election.enable=false`（默认值）。

### 场景 3：Consumer 自动提交 Offset 后崩溃

**现象**：消息被标记为已消费，但实际未处理。

**原因**：`enable.auto.commit=true` 时，Offset 按固定间隔提交，与消息处理不同步。

**解决**：设置 `enable.auto.commit=false`，在处理完成后手动提交。

### 场景 4：Producer 重试导致重复（未开启幂等性）

**现象**：同一条消息在 Consumer 端出现多次。

**原因**：未开启 `enable.idempotence` 时，超时重试可能导致重复写入。

**解决**：开启 `enable.idempotence=true`，或在下游系统实现幂等处理（如使用唯一键去重）。

### 场景 5：Consumer Rebalance 期间 Offset 提交失败

**现象**：Rebalance 后部分消息被跳过（丢失）或重复消费。

**原因**：Rebalance 触发时，旧 Consumer 的 Offset 未及时提交。

**解决**：使用 `ConsumerRebalanceListener.onPartitionsRevoked()` 在 Rebalance 前提交 Offset。

## 最佳实践总结

### 配置推荐矩阵

| 场景 | acks | min.insync.replicas | enable.idempotence | enable.auto.commit | 说明 |
|------|------|---------------------|-------------------|-------------------|------|
| **日志采集/指标** | 0 或 1 | 1 | false | true | 可容忍丢失，追求低延迟 |
| **一般业务** | all | 2 | true | false | 推荐默认配置，At-least-once |
| **金融/计费** | all | 2 | true（或事务） | false | Exactly-once，下游需幂等 |
| **跨 Topic 处理** | all | 2 | 事务型 Producer | false | Kafka Streams / Connect 内置 EOS |

### 关键原则

1. **不要使用 `acks=0`**：除非你明确接受消息丢失
2. **开启幂等性**：`enable.idempotence=true` 几乎无额外开销，但能防止重复和乱序
3. **先处理后提交**：Consumer 端永远先处理消息，再提交 Offset
4. **下游幂等设计**：即使 Kafka 保证 Exactly-once，下游系统（数据库、外部 API）也应设计为幂等
5. **监控 ISR 数量**：ISR 数量低于 `min.insync.replicas` 时写入会失败，需及时告警

## 相关概念

- [[Kafka 核心概念]]：Topic、Partition、Broker、Replica 等基础概念
- [[Kafka 如何保证消息的顺序消费]]：Partition 级别顺序保证机制
- [[Kafka 是如何实现高性能的]]：顺序 I/O、零拷贝、PageCache 等性能机制
- [[Kafka 消息投递语义]]：At-most-once、At-least-once、Exactly-once 语义详解

## 参考资料

[^delivery-semantics]: [Kafka Message Delivery Guarantees - Confluent Documentation](https://docs.confluent.io/kafka/design/delivery-semantics.html)
[^producer-design]: [Kafka Producer Design - Confluent Documentation](https://docs.confluent.io/kafka/design/producer-design.html)
[^consumer-design]: [Kafka Consumer Design - Confluent Documentation](https://docs.confluent.io/kafka/design/consumer-design.html)
[^replication]: [Kafka Replication and Committed Messages - Confluent Documentation](https://docs.confluent.io/kafka/design/replication.html)
