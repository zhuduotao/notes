---
created: '2026-04-17'
updated: '2026-04-17'
tags:
  - kafka
  - transaction
  - exactly-once
  - message-queue
  - middleware
  - streaming
aliases:
  - Kafka 事务
  - Kafka transactional messaging
  - Kafka exactly-once semantics
  - Kafka 事务型 Producer
  - Kafka 精确一次语义
source_type: official-doc
source_urls:
  - >-
    https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging
  - >-
    https://cwiki.apache.org/confluence/display/KAFKA/Transactional+Messaging+in+Kafka
  - 'https://docs.confluent.io/kafka/design/delivery-semantics.html'
status: verified
---

## 核心结论

Kafka 自 **0.11.0.0** 版本起引入事务型 Producer（Transactional Producer），支持**跨多个 Partition 的原子写入** —— 要么所有消息全部提交，要么全部回滚。结合幂等性 Producer 和 Consumer 的 `isolation.level` 配置，Kafka 可以实现 **Exactly-Once Semantics（精确一次语义）**，解决传统 At-least-once 语义下因重试导致的重复消息问题 [^kip-98]。

## 是什么 / 为什么需要事务

### Kafka 原有的投递语义局限

Kafka 默认提供 **At-least-once**（至少一次）语义：消息保证不丢失，但 Producer 重试可能导致重复 [^kip-98]。

| 语义 | 消息丢失 | 消息重复 | 说明 |
|------|----------|----------|------|
| At-most-once | 可能 | 不会 | 不重试，消息可能丢失 |
| At-least-once | 不会 | 可能 | 重试保证送达，但可能重复 |
| Exactly-once | 不会 | 不会 | 每条消息恰好被处理一次 |

### 事务解决的核心问题

1. **跨 Partition 原子性**：普通 Producer 无法保证写入多个 Partition 时要么全成功、要么全失败
2. **Consume-Transform-Produces 模式**：流处理应用需要"消费 → 处理 → 生产"作为一个原子操作，避免部分成功导致的数据不一致
3. **Producer 故障恢复**：Producer 重启后能自动恢复或中止上一次未完成的事务，保证状态一致性 [^kip-98]

## 核心概念

### TransactionalId

事务型 Producer 必须配置一个全局唯一的 `transactional.id`，它是事务语义的基石：

- **跨会话持久化**：不同 Producer 实例使用相同的 `transactional.id` 时，Kafka 能保证它们不会同时活跃
- **Zombie Fencing（僵尸隔离）**：新实例启动时会自动"隔离"旧实例，旧实例的任何后续操作都会抛出 `ProducerFencedException` [^kip-98]
- **事务恢复**：新实例调用 `initTransactions()` 时，会自动完成或回滚旧实例遗留的未决事务

### Producer Epoch（生产者纪元）

每个 `transactional.id` 对应一个内部 Producer ID（PID），PID 附带一个 Epoch 值：

- 每次新 Producer 实例调用 `initTransactions()` 时，Epoch 递增
- Broker 拒绝 Epoch 较低的旧实例的请求，实现 Fencing [^kip-98]

### Transaction Coordinator（事务协调器）

Kafka 内部组件，负责：

- 分配 Producer ID 和 Epoch
- 管理事务生命周期（Begin → Commit/Abort）
- 维护 `__transaction_state` 内部 Topic 作为事务日志 [^kip-98]

### Control Messages（控制消息）

事务提交或中止时，Coordinator 向每个参与事务的 Partition 写入特殊的 **COMMIT** 或 **ABORT** 控制标记。这些消息对应用层不可见，仅用于通知 Consumer 事务状态 [^kip-98]。

### Last Stable Offset（LSO）

在 `read_committed` 模式下，Consumer 使用 LSO 而非 High Watermark 来计算消费进度。LSO 是**第一个未决事务的起始 Offset**，保证 Consumer 不会读到未完成事务的消息 [^kip-98]。

## 怎么用

### 前提条件

使用事务前，必须满足以下配置要求 [^kip-98]：

| 配置项 | 要求 | 说明 |
|--------|------|------|
| `transactional.id` | **必须设置** | 全局唯一标识符，跨 Producer 会话持久化 |
| `enable.idempotence` | **必须为 true** | 事务依赖幂等性，设置 `transactional.id` 时自动开启 |
| `acks` | 必须为 `all` | 开启幂等性时自动设置 |
| `retries` | 必须 > 1 | 开启幂等性时自动设为 `Integer.MAX_VALUE` |
| `max.in.flight.requests.per.connection` | 必须为 1 | 开启幂等性时自动设置，保证有序 |

> 如果未显式覆盖上述配置，Kafka 会在开启幂等性/事务时**自动设置**这些值 [^kip-98]。

### Producer 端事务 API

KafkaProducer 提供 5 个事务相关方法，调用顺序严格固定 [^kip-98]：

```
initTransactions()
  ↓
beginTransaction()
  ↓
send() × N          ← 发送消息
sendOffsetsToTransaction()  ← 可选：提交消费位移
  ↓
commitTransaction() 或 abortTransaction()
```

#### API 详细说明

| 方法 | 作用 | 异常 |
|------|------|------|
| `initTransactions()` | 初始化事务环境，恢复/中止旧实例的未决事务，获取 PID 和 Epoch。每个 Producer 实例调用一次 | `IllegalStateException`（未配置 transactional.id） |
| `beginTransaction()` | 标记新事务开始 | `ProducerFencedException` |
| `send(record)` | 在事务内发送消息。消息不会立即可见，直到事务提交 | 异步异常通过 Future/Callback 返回 |
| `sendOffsetsToTransaction(offsets, groupId)` | 将 Consumer 的位移提交作为事务的一部分，实现 consume-transform-produce 原子性 | `ProducerFencedException` |
| `commitTransaction()` | 提交事务，使所有消息对 `read_committed` Consumer 可见 | `ProducerFencedException` |
| `abortTransaction()` | 中止事务，所有消息对 Consumer 不可见（被丢弃） | `ProducerFencedException` |

### 完整示例：Consume-Transform-Produces 模式

```java
public class KafkaTransactionsExample {
 
  public static void main(String[] args) {
    // Consumer 配置
    Properties consumerConfig = new Properties();
    consumerConfig.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    consumerConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
    consumerConfig.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
        StringDeserializer.class.getName());
    consumerConfig.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
        StringDeserializer.class.getName());
    // 注意：Consumer 不需要设置 isolation.level，除非需要 read_committed
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
    consumer.subscribe(Arrays.asList("input-topic"));

    // Producer 配置 —— 必须设置 transactional.id
    Properties producerConfig = new Properties();
    producerConfig.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    producerConfig.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
        StringSerializer.class.getName());
    producerConfig.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
        StringSerializer.class.getName());
    producerConfig.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-producer");
    KafkaProducer<String, String> producer = new KafkaProducer<>(producerConfig);

    // 1. 初始化事务（每个 Producer 实例仅调用一次）
    // 此方法会恢复或中止之前同名 transactional.id 的未决事务
    producer.initTransactions();
    
    while (true) {
      ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
      if (!records.isEmpty()) {
        // 2. 开始新事务
        producer.beginTransaction();
        
        try {
          // 3. 处理输入记录并发送到输出 Topic
          for (ConsumerRecord<String, String> record : records) {
            String processedValue = process(record.value());
            producer.send(new ProducerRecord<>("output-topic", 
                record.key(), processedValue));
          }
          
          // 4. 将消费位移作为事务的一部分提交
          // 这保证了"消费"和"生产"的原子性
          producer.sendOffsetsToTransaction(
              getUncommittedOffsets(consumer), 
              "my-consumer-group");
          
          // 5. 提交事务
          producer.commitTransaction();
          
        } catch (ProducerFencedException e) {
          // 另一个同名 Producer 实例已上线，当前实例被隔离
          // 需要关闭当前 Producer 并退出
          producer.close();
          throw e;
        } catch (Exception e) {
          // 处理失败，中止事务
          producer.abortTransaction();
        }
      }
    }
  }
}
```

### Consumer 端配置

Consumer 通过 `isolation.level` 控制事务消息的可见性 [^kip-98]：

| isolation.level | 行为 | 适用场景 |
|-----------------|------|----------|
| `read_uncommitted`（默认） | 消费所有消息，包括未提交和中止事务中的消息 | 对重复/无效消息不敏感的场景；流处理拓扑中的推测执行 |
| `read_committed` | 仅消费已提交事务的消息和非事务消息；中止事务的消息被丢弃 | 需要精确一次语义的场景 |

```properties
# Consumer 配置 —— 仅读取已提交的事务消息
isolation.level=read_committed
```

> **注意**：在 `read_committed` 模式下，Consumer 需要**缓冲**未决事务的消息，直到看到 COMMIT 或 ABORT 标记，这会增加一定的内存开销和延迟 [^kip-98]。

## 事务工作流程

事务的完整生命周期涉及 Producer、Transaction Coordinator 和 Broker 三方协作 [^kip-98]：

```
1. Producer → Coordinator: FindCoordinatorRequest（发现事务协调器）
2. Producer → Coordinator: InitPidRequest（获取 PID + Epoch，恢复旧事务）
3. Producer: beginTransaction()（本地标记事务开始）
4. Producer → Coordinator: AddPartitionsToTxnRequest（首次写入某 Partition 时）
5. Producer → Broker: ProduceRequest（发送事务消息，携带 PID/Epoch/Sequence）
6. Producer → Coordinator: AddOffsetsToTxnRequest（可选：将位移提交纳入事务）
7. Producer → Consumer Coordinator: TxnOffsetCommitRequest（提交位移）
8. Producer → Coordinator: EndTxnRequest（请求提交或中止）
9. Coordinator → 各 Broker: WriteTxnMarkersRequest（写入 COMMIT/ABORT 控制标记）
10. Coordinator: 在事务日志中写入最终状态（COMMITTED/ABORTED）
```

## 限制与注意事项

### 事务超时

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `transaction.timeout.ms`（Producer） | 60000（1 分钟） | Producer 等待事务状态更新的最大时间，超时后 Coordinator 自动中止事务 |
| `max.transaction.timeout.ms`（Broker） | 900000（15 分钟） | Broker 允许的最大事务超时，Producer 请求超时超过此值会被拒绝 |
| `transactional.id.timeout.ms`（Broker） | 604800000（7 天） | Producer 不活跃时，TransactionalId 的过期时间 |

> 事务超时不宜设置过大，否则会阻塞 `read_committed` Consumer 的进度（Consumer 需等待事务完成才能越过 LSO）[^kip-98]。

### 性能影响

- **吞吐量**：事务型 Producer 的吞吐量低于非事务 Producer，因为需要与 Coordinator 多次交互
- **延迟**：`read_committed` Consumer 需要缓冲未决事务消息，增加消费延迟
- **短事务 vs 长事务**：
  - 短事务：Consumer 缓冲少，但 Producer 吞吐量低（频繁与 Coordinator 交互）
  - 长事务：Producer 吞吐量高，但 Consumer 需要更多缓冲内存 [^txn-wiki]

### 不支持减少 Partition

Kafka **不支持减少 Topic 的 Partition 数量**。增加 Partition 后，已有 Key 的 Hash 映射可能改变，影响事务内消息的路径一致性。

### Compacted Topic 的注意事项

在压缩 Topic 中使用事务时存在一个 caveat：

- 事务中的部分消息可能在压缩过程中被覆盖
- 落后的 Consumer 可能只看到事务中的部分消息
- 如果应用依赖事务的原子性更新数据库，这种部分可见性可能导致不一致状态 [^txn-wiki]

### 单写者约束

同一时刻，同一个 `transactional.id` 只能有一个活跃的 Producer 实例。新实例启动时会 Fencing 旧实例，旧实例后续操作抛出 `ProducerFencedException` [^kip-98]。

### 异常处理

| 异常 | 含义 | 处理方式 |
|------|------|----------|
| `ProducerFencedException` | 另一个同名 Producer 已上线，当前实例被隔离 | **不可恢复**，关闭 Producer，退出应用 |
| `OutOfOrderSequenceException` | Broker 检测到数据丢失（序列号不连续） | **不可恢复**，关闭 Producer |
| `InvalidTxnStateException` | 事务状态不匹配（如未 begin 就 commit） | 检查代码逻辑，确保 API 调用顺序正确 |
| `ConcurrentTransactionsException` | 上一个事务尚未完成就开始了新事务 | 等待上一个事务完成 |
| `TimeoutException` | 事务超时 | 可重试，或中止后重新开始 |

## 常见误区

### 误区 1：事务 = 数据库事务

Kafka 事务**不是**数据库事务的等价物：

- Kafka 事务仅保证消息写入的原子性，不提供 ACID 中的隔离性（Isolation）
- Consumer 仍然可以按各自的速度消费，不保证所有 Consumer 同时看到事务提交
- Compacted Topic 中事务消息可能被部分覆盖 [^txn-wiki]

### 误区 2：开启事务就自动 Exactly-Once

Exactly-Once 需要** Producer + Consumer 两端配合**：

- Producer 端：配置 `transactional.id`，使用事务 API
- Consumer 端：配置 `isolation.level=read_committed`
- 仅 Producer 端开启事务而 Consumer 使用 `read_uncommitted`，仍然可能读到未提交/中止的消息

### 误区 3：事务可以跨 Producer 实例共享

每个 Producer 实例必须**独立调用** `initTransactions()`。`transactional.id` 仅用于标识和 Fencing，不共享事务状态 [^kip-98]。

### 误区 4：事务内消息数量无限制

虽然 Kafka 没有显式限制事务内消息数量，但受以下因素约束：

- `transaction.timeout.ms`：事务必须在此时间内完成
- Consumer 缓冲能力：`read_committed` Consumer 需要缓冲整个事务的消息
- 内存限制：Producer 需要在内存中缓冲事务消息直到提交

## 与幂等性 Producer 的关系

| 特性 | 幂等性 Producer | 事务型 Producer |
|------|-----------------|-----------------|
| 配置 | `enable.idempotence=true` | `transactional.id=xxx`（自动开启幂等） |
| 保证范围 | 单个 Producer 会话内，单 Partition | 跨 Producer 会话，跨多个 Partition |
| 原子性 | 单条消息不重复 | 多条消息要么全提交、要么全回滚 |
| 跨会话 | 不支持（每次启动分配新 PID） | 支持（通过 transactional.id 恢复） |
| 依赖 | 无 | 依赖幂等性 |

> 事务型 Producer **内置幂等性**。设置 `transactional.id` 时，`enable.idempotence` 自动为 `true` [^kip-98]。

## 适用场景

| 场景 | 是否适合使用事务 | 说明 |
|------|------------------|------|
| 日志/指标采集 | 不需要 | 容忍少量重复，追求高吞吐 |
| 订单处理 | 推荐 | 需要保证订单状态变更的原子性 |
| 流处理（Kafka Streams / Flink） | 必须 | consume-transform-produce 模式需要原子性 |
| 数据同步（CDC） | 推荐 | 保证源端和目标端数据一致性 |
| 金融交易 | 必须 | 不允许消息丢失或重复 |

## 相关概念

- [[Kafka 如何保证消息的顺序消费]]：Partition 级别顺序保证、幂等性 Producer
- [[Kafka 是如何实现高性能的]]：日志结构、零拷贝、批量发送
- [[数据库事务的隔离级别]]：ACID 事务的对比参考

## 参考资料

[^kip-98]: [KIP-98 - Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)
[^txn-wiki]: [Transactional Messaging in Kafka](https://cwiki.apache.org/confluence/display/KAFKA/Transactional+Messaging+in+Kafka)
[^delivery-semantics]: [Kafka Message Delivery Guarantees - Confluent Documentation](https://docs.confluent.io/kafka/design/delivery-semantics.html)
