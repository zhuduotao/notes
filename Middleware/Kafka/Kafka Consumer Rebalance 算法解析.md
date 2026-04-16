---
created: 2026-04-16
updated: 2026-04-16
tags:
  - kafka
  - consumer
  - rebalance
  - partition-assignment
  - middleware
  - distributed-systems
aliases:
  - Kafka 再均衡
  - Kafka 分区重分配
  - Consumer Rebalance Protocol
  - KIP-848
  - KIP-429
  - KIP-345
source_type: mixed
source_urls:
  - https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol
  - https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol
  - https://cwiki.apache.org/confluence/display/KAFKA/KIP-345%3A+Introduce+static+membership+protocol+to+reduce+consumer+rebalances
  - https://docs.confluent.io/kafka/design/consumer-design.html
status: verified
---

## 核心结论

Kafka Consumer Rebalance 是 Consumer Group 内 **Partition 所有权重新分配** 的过程。当 Group 成员变化（加入/退出/崩溃）或订阅的 Topic 元数据变化时，Broker 的 Group Coordinator 会触发 Rebalance，确保每个 Partition 在同一时刻只被 Group 内的一个 Consumer 消费。

Rebalance 是 Kafka 消费模型中 **最关键的运维痛点之一** —— 设计不当会导致消费中断、状态丢失、甚至整个 Group 的 Stop-The-World。Kafka 4.0 引入了全新的 Consumer Rebalance 协议（KIP-848），将分配逻辑从客户端移至服务端，实现了真正的增量 Rebalance。

## 前置概念

### Consumer Group 与 Group Coordinator

- **Consumer Group**：同一应用中协同消费的一组 Consumer，共享同一个 `group.id`
- **Group Coordinator**：Broker 端组件，负责管理 Group 成员关系、触发 Rebalance、维护 Partition 分配方案。每个 `group.id` 对应一个 Coordinator，由 `__consumer_offsets` 内部 Topic 的某个 Partition 的 Leader 担任
- **Member ID**：Consumer 在 Group 中的唯一标识，由 Coordinator 分配（KIP-1082 后改由客户端生成）
- **Generation / Epoch**：Group 的代数/纪元，每次 Rebalance 递增，用于防止旧成员提交 Offset

### Rebalance 的触发条件

| 触发事件 | 说明 |
|----------|------|
| 新 Consumer 加入 Group | 需要重新分配 Partition 以实现负载均衡 |
| Consumer 崩溃或主动退出 | 其持有的 Partition 需重新分配给其他成员 |
| Topic 元数据变化 | 新增 Partition、新增匹配的 Topic（正则订阅场景） |
| Consumer 更新订阅 | 订阅的 Topic 集合发生变化 |
| Consumer 超时未心跳 | 超过 `session.timeout.ms` 被 Coordinator 踢出 Group |

## 经典协议（Classic Protocol）

经典协议是 Kafka 0.9 引入的原始 Rebalance 协议，采用 **客户端 Leader 选举 + 客户端分配** 的模式。

### 协议流程

```
1. JoinGroup 阶段：
   - 所有成员向 Coordinator 发送 JoinGroupRequest
   - Coordinator 选举第一个加入的成员作为 Group Leader
   - 等待所有成员加入或达到 Rebalance Timeout

2. SyncGroup 阶段：
   - Leader 收集所有成员的订阅信息
   - Leader 在客户端执行 Partition 分配算法
   - Leader 将分配方案发送给 Coordinator
   - Coordinator 通过 SyncGroupResponse 将分配结果广播给所有成员

3. 稳定阶段（Stable）：
   - 成员开始消费分配的 Partition
   - 通过 Heartbeat 维持会话
```

### 两种分配模式

#### Eager 模式（急切模式）

- **行为**：Rebalance 开始时，**所有 Consumer 立即撤销（revoke）所有已分配的 Partition**，等待新分配完成后再恢复消费
- **特点**：Stop-The-World，整个 Group 在 Rebalance 期间完全停止消费
- **默认 Assignor**：`RangeAssignor`、`RoundRobinAssignor`、`StickyAssignor`
- **问题**：即使某个 Consumer 的分配没有变化，也必须先 revoke 再重新 assign，造成不必要的消费中断

#### Cooperative 模式（合作模式，KIP-429）

- **行为**：Rebalance 分多轮进行，Consumer **只 revoke 需要转移的 Partition**，保留不变的 Partition 继续消费
- **特点**：增量 Rebalance，显著减少消费中断时间
- **Assignor**：`CooperativeStickyAssignor`
- **流程**：
  1. 第一轮：Leader 计算目标分配，标记需要 revoke 的 Partition
  2. 相关 Consumer revoke 被标记的 Partition，触发第二轮 Rebalance
  3. 第二轮：将已 revoke 的 Partition 分配给新 Owner
- **限制**：仍需多轮 Rebalance 才能完成全部分配转移

### 内置 Partition Assignor

| Assignor | 分配策略 | 支持协议 | 特点 |
|----------|----------|----------|------|
| `RangeAssignor` | 按 Topic 独立分配，每个 Topic 的 Partition 按 Consumer 数量均分 | Eager | 默认 Assignor（Kafka 3.0 之前）；可能导致某些 Consumer 负载略高 |
| `RoundRobinAssignor` | 将所有 Topic 的 Partition 排序后轮询分配 | Eager | 比 Range 更均衡，但不考虑 Topic 间的亲和性 |
| `StickyAssignor` | 尽量保持上次的分配不变，最小化 Partition 迁移 | Eager | 减少不必要的 Partition 移动 |
| `CooperativeStickyAssignor` | 在 Sticky 基础上支持 Cooperative 协议 | Eager, Cooperative | Kafka 2.4 引入，推荐用于有状态应用 |

#### RangeAssignor 分配示例

```
Topic A: 6 partitions (A0-A5), Topic B: 3 partitions (B0-B2)
Consumer: C1, C2, C3

按 Topic 独立分配：
C1: A0, A1, B0
C2: A2, A3, B1
C3: A4, A5, B2
```

#### RoundRobinAssignor 分配示例

```
所有 Partition 排序: A0, A1, A2, A3, A4, A5, B0, B1, B2
轮询分配：
C1: A0, A3, B0
C2: A1, A4, B1
C3: A2, A5, B2
```

## 新 Consumer 协议（KIP-848，Kafka 4.0 GA）

KIP-848 是 Kafka Consumer Rebalance 协议的下一代设计，于 Kafka 4.0 正式可用（GA）。核心变化是 **将分配逻辑从客户端移至服务端**。

### 设计目标

1. **真正的增量 Rebalance**：不受影响的 Consumer 继续消费，无需全局同步屏障
2. **复杂度移至服务端**：便于调试和修复，不依赖客户端版本升级
3. **保留客户端自定义能力**：支持 Kafka Streams 等高级用户在客户端执行分配逻辑
4. **无缝升级**：支持在线滚动升级，无需停机

### 核心架构变化

| 维度 | 经典协议 | 新协议（KIP-848） |
|------|----------|-------------------|
| **Group Leader** | 选举一个 Consumer 作为 Leader 执行分配 | 无 Leader，由 Broker 端 Coordinator 统一计算 |
| **分配位置** | 客户端（Group Leader） | 服务端（Group Coordinator） |
| **Rebalance 行为** | Eager（全部 revoke）或 Cooperative（多轮） | 真正的增量分配，单轮完成 |
| **消费中断** | 高（所有 Consumer 暂停） | 低（仅受影响的 Consumer 暂停） |
| **心跳机制** | 仅维持会话 | 心跳携带分配/撤销指令（ConsumerGroupHeartbeat API） |
| **启用方式** | 默认 | 客户端设置 `group.protocol=consumer` |

### 三阶段 Epoch 机制

新协议通过三种 Epoch 驱动 Rebalance 流程：

#### 1. Group Epoch（触发）

- 由 Group Coordinator 维护，表示 Group 元数据的版本
- 以下事件会递增 Group Epoch：
  - 成员加入或离开
  - 成员更新订阅或 Assignor
  - 成员被 Fenced 或移除
  - Partition 元数据变化（新增 Partition、新 Topic 匹配正则订阅）

#### 2. Assignment Epoch（计算）

- 当 Group Epoch > Assignment Epoch 时，Coordinator 计算新的目标分配（Target Assignment）
- 分配完成后，Assignment Epoch 更新为当前的 Group Epoch
- 分配可以是 **服务端计算**（默认）或 **委托给客户端**（高级用户如 Kafka Streams）

#### 3. Member Epoch（协调）

- 每个成员独立协调自己的当前分配到目标分配
- 协调过程分三步：
  1. **撤销**：Coordinator 通过心跳响应告知成员需要 revoke 的 Partition，成员确认
  2. **更新**：Coordinator 更新成员的当前分配为目标分配
  3. **分配**：Coordinator 将新 Partition 分配给成员（确保已被其他成员 revoke）

### 服务端 Assignor

新协议内置两种服务端 Assignor：

| Assignor | 全限定名 | 策略 |
|----------|----------|------|
| `range` | `org.apache.kafka.coordinator.group.assignor.RangeAssignor` | 按 Topic 协同分区（co-partition） |
| `uniform` | `org.apache.kafka.coordinator.group.assignor.UniformAssignor` | 均匀分配，类似 Sticky Assignor |

两种 Assignor 均具有 **Sticky 特性**，即尽量减少 Partition 迁移。

### 客户端 Assignor（规划中）

> 注意：截至 Kafka 4.0，客户端 Assignor 尚未实现。

高级用户（如 Kafka Streams）仍可在客户端执行自定义分配逻辑：

1. Coordinator 选择一个成员执行分配
2. 该成员通过 `ConsumerGroupPrepareAssignment` API 获取 Group 状态
3. 执行自定义分配后，通过 `ConsumerGroupInstallAssignment` API 安装结果
4. Coordinator 验证并持久化分配方案

### Group 状态机

| 状态 | 说明 |
|------|------|
| `EMPTY` | Group 刚创建或最后一个成员离开 |
| `ASSIGNING` | Group Epoch > Assignment Epoch，正在计算新分配 |
| `RECONCILING` | 成员尚未全部收敛到目标 Epoch |
| `STABLE` | 所有成员已收敛到目标分配 |
| `DEAD` | Group 保持 EMPTY 超过配置时间后被删除 |

## 静态成员协议（KIP-345）

KIP-345 引入了 **静态成员（Static Membership）** 概念，通过持久化成员身份来减少不必要的 Rebalance。

### 核心配置

```properties
# 设置静态成员 ID（必须全局唯一）
group.instance.id=my-consumer-instance-1

# 建议增大 Session Timeout 以适应滚动重启
session.timeout.ms=300000  # 默认 45000ms，最大 1800000ms（30 分钟）
```

### 动态成员 vs 静态成员

| 特性 | 动态成员（默认） | 静态成员 |
|------|------------------|----------|
| **身份标识** | `member.id`（Broker 随机生成，每次重启变化） | `group.instance.id`（用户配置，重启后不变） |
| **重启行为** | 视为新成员，触发 Rebalance | 保留原分配，不触发 Rebalance |
| **离线处理** | 发送 LeaveGroup 请求，立即触发 Rebalance | 不发送 LeaveGroup，依赖 Session Timeout 检测 |
| **适用场景** | 无状态 Consumer、动态扩缩容 | 有状态 Consumer、滚动重启、K8s 部署 |

### 静态成员的工作机制

1. Consumer 启动时携带 `group.instance.id` 发送 JoinGroupRequest
2. Coordinator 检查该 `group.instance.id` 是否已存在：
   - **存在**：返回缓存的分配方案，不触发 Rebalance
   - **不存在**：生成新的 `member.id`，记录映射关系，触发 Rebalance
3. Consumer 离线时不发送 LeaveGroup，Coordinator 通过 Session Timeout 检测
4. 若离线时间超过 `session.timeout.ms`，Coordinator 将其移除并触发 Rebalance

### 滚动重启场景

```
场景：3 个静态成员 A、B、C 需要滚动重启

1. 重启 A → A 以相同 group.instance.id 重新加入，保留原分配，不触发 Rebalance
2. 重启 B → 同上，不触发 Rebalance
3. 重启 C → 同上，不触发 Rebalance

结果：整个滚动重启过程 0 次 Rebalance
```

### 注意事项

- `group.instance.id` 必须在 Group 内唯一，否则新加入的成员会 Fenced 旧成员（`FENCED_INSTANCE_ID` 错误）
- 静态成员离线时不发送 LeaveGroup，因此 **快速缩容** 需要手动调用 Admin API 移除成员
- 动态成员和静态成员可以共存于同一 Group

## ConsumerRebalanceListener 回调

Consumer 通过 `ConsumerRebalanceListener` 接口感知 Rebalance 事件：

```java
public interface ConsumerRebalanceListener {
    // 分区被撤销时调用（Eager 模式：全部撤销；Cooperative 模式：部分撤销）
    void onPartitionsRevoked(Collection<TopicPartition> partitions);
    
    // 分区被分配时调用
    void onPartitionsAssigned(Collection<TopicPartition> partitions);
    
    // 成员被踢出 Group 时调用（KIP-429 新增）
    default void onPartitionsLost(Collection<TopicPartition> partitions) {
        onPartitionsRevoked(partitions);
    }
}
```

### 回调语义差异

| 回调 | Eager 模式 | Cooperative 模式 |
|------|------------|------------------|
| `onPartitionsRevoked` | Rebalance 开始时调用，传入 **全部** 已分配 Partition | 仅在需要转移 Partition 时调用，传入 **部分** Partition；可能不调用 |
| `onPartitionsAssigned` | 传入 **全部** 新分配的 Partition | 传入 **新增** 的 Partition（与之前无重叠）；始终调用 |
| `onPartitionsLost` | 成员崩溃/超时被踢出时调用，传入全部 Partition | 同左 |

### Offset 提交策略

在 `onPartitionsRevoked` 中提交 Offset 是常见做法，确保新 Consumer 从正确位置开始消费：

```java
consumer.subscribe(Arrays.asList("my-topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync(); // 同步提交当前 Offset
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 可选：从特定 Offset 开始消费
    }
});
```

## Rebalance 常见问题与最佳实践

### 常见问题

#### 1. Rebalance 风暴（Rebalance Storm）

**现象**：Group 频繁触发 Rebalance，消费几乎停滞。

**常见原因**：
- `max.poll.interval.ms` 设置过小，Consumer 处理消息时间超过该值，被 Coordinator 判定为死亡
- 网络抖动导致心跳超时
- 大量 Consumer 同时重启

**解决方案**：
- 增大 `max.poll.interval.ms` 以匹配实际处理时间
- 使用静态成员协议（`group.instance.id`）避免重启触发 Rebalance
- 新协议（KIP-848）下受影响更小

#### 2. 重复消费

**原因**：Rebalance 后新 Consumer 从上次提交的 Offset 开始消费，若 Offset 提交不及时，可能重复消费已处理的消息。

**解决方案**：
- 在 `onPartitionsRevoked` 中同步提交 Offset
- 使用幂等消费逻辑（如数据库唯一键、去重表）
- 开启自动提交（`enable.auto.commit=true`）但需注意提交时机

#### 3. 消费倾斜

**原因**：Partition 数量不是 Consumer 数量的整数倍，或 Assignor 分配不均。

**解决方案**：
- 确保 Partition 数量 ≥ Consumer 数量，且最好是 Consumer 数量的倍数
- 选择更均衡的 Assignor（如 `RoundRobinAssignor` 或 `uniform`）

#### 4. 正则订阅导致的意外 Rebalance

**原因**：使用 `Pattern.subscribe()` 时，新创建的 Topic 匹配正则会触发 Rebalance。

**解决方案**：
- 避免在正则订阅场景下频繁创建 Topic
- 新协议下由 Broker 端统一处理正则匹配，减少客户端不一致

### 最佳实践

| 场景 | 推荐配置 | 说明 |
|------|----------|------|
| 无状态 Consumer | 默认配置 + `CooperativeStickyAssignor` | 减少 Rebalance 期间的消费中断 |
| 有状态 Consumer（Kafka Streams 等） | `group.instance.id` + 大 `session.timeout.ms` | 滚动重启不触发 Rebalance |
| Kafka 4.0+ 新部署 | `group.protocol=consumer` | 使用新协议，享受增量 Rebalance |
| 快速扩缩容 | Admin API 手动移除成员 + 静态成员 | 避免等待 Session Timeout |
| 需要精确 Offset 控制 | `enable.auto.commit=false` + 手动提交 | 在 `onPartitionsRevoked` 中同步提交 |

## 版本演进时间线

| 版本 | 关键变化 | 相关 KIP |
|------|----------|----------|
| 0.9 | 引入 Consumer Group 和经典 Rebalance 协议 | - |
| 2.3 | 静态成员协议（减少滚动重启 Rebalance） | KIP-345 |
| 2.4 | Cooperative 增量 Rebalance 协议 | KIP-429 |
| 2.5 | Cooperative 模式下允许在 Rebalance 期间返回记录 | KAFKA-8421 |
| 3.0 | `CooperativeStickyAssignor` 成为默认 Assignor | - |
| 3.7 | 新 Consumer 协议进入 Beta | KIP-848 |
| 4.0 | 新 Consumer 协议 GA（`group.protocol=consumer`） | KIP-848 |
| 4.2 | Topic ID 支持 Offset Fetch/Commit | KIP-848 后续 |

## 相关概念

- [[Kafka 如何保证消息的顺序消费]]：Rebalance 对消息顺序的影响
- [[Kafka 如何实现 Auto Scaling 应对消息积压]]：Consumer 扩缩容与 Rebalance 的关系
- [[Kafka 核心概念]]：Consumer Group、Partition、Offset 等基础概念

## 参考资料

[^kip-848]: [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A+The+Next+Generation+of+the+Consumer+Rebalance+Protocol)
[^kip-429]: [KIP-429: Kafka Consumer Incremental Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol)
[^kip-345]: [KIP-345: Introduce static membership protocol to reduce consumer rebalances](https://cwiki.apache.org/confluence/display/KAFKA/KIP-345%3A+Introduce+static+membership+protocol+to+reduce+consumer+rebalances)
[^confluent-consumer]: [Kafka Consumer Design - Confluent Documentation](https://docs.confluent.io/kafka/design/consumer-design.html)
