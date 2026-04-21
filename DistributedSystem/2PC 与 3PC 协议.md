---
aliases:
  - 2PC
  - 3PC
  - 两阶段提交
  - 三阶段提交
  - 分布式事务提交
created: '2026-04-21'
source_type: mixed
source_urls:
  - 'https://en.wikipedia.org/wiki/Two-phase_commit_protocol'
  - 'https://en.wikipedia.org/wiki/Three-phase_commit_protocol'
  - 'https://martin.kleppmann.com/2015/05/14/consensus-vs-atomic-commit.html'
status: verified
tags:
  - 分布式系统
  - 分布式事务
  - 2PC
  - 3PC
  - 原子提交
updated: '2026-04-21'
---
两阶段提交（2PC）和三阶段提交（3PC）是分布式事务的提交协议。它们不是共识算法，而是用于保证分布式事务的原子性。理解它们有助于区分"共识"与"提交"问题。

## 是什么

2PC 和 3PC 解决分布式事务的原子提交问题：
- 多个节点执行事务的不同部分
- 需保证要么全部提交，要么全部回滚

与共识算法的区别：
- 共识算法解决"多个节点就某个值达成一致"
- 2PC/3PC 解决"事务是否提交的决策协调"
- 2PC/3PC 依赖协调者，无法容忍协调者崩溃（2PC）或需要恢复机制（3PC）

## 为什么重要

分布式事务场景需要：
- 跨数据库事务
- 跨服务事务（微服务）
- 分布式存储系统的写操作

理解 2PC/3PC 的限制有助于选择合适的一致性方案。

## 两阶段提交（2PC）

### 流程

**Phase 1: Prepare/Voting**

1. Coordinator 发送 `<PREPARE>` 给所有 Participant
2. Participant 执行事务（不提交），锁定资源
3. Participant 决策：
   - 成功：发送 `<PREPARED>` 给 Coordinator
   - 失败：发送 `<ABORT>` 给 Coordinator

**Phase 2: Commit/Abort**

1. Coordinator 收集所有响应：
   - 全部 PREPARED → 发送 `<COMMIT>` 给所有 Participant
   - 任一 ABORT 或超时 → 发送 `<ABORT>` 给所有 Participant
2. Participant 收到决策：
   - COMMIT → 提交事务，释放资源
   - ABORT → 回滚事务，释放资源

### 问题

**阻塞问题**：
- Participant 在 Prepare 后等待 Coordinator 决策，资源被锁定
- Coordinator 崩溃时，Participant 无法决策
- 可能导致长时间阻塞

**协调者崩溃**：
- Coordinator 在 Phase 2 崩溃，部分 Participant 收到决策，部分未收到
- 未收到的 Participant 无法决策，系统不一致

**网络分区**：
- 分区中 Participant 收不到 Coordinator 消息
- 无法判断其他 Participant 的决策

### 无法容忍崩溃

FLP 不可能性定理：
- 异步系统中，无法设计既能保证安全性又能保证活性的提交协议
- 2PC 保证安全性（不违反原子性），但不保证活性（可能阻塞）

## 三阶段提交（3PC）

### 设计目标

解决 2PC 的阻塞问题：
- Coordinator 崩溃时，新 Coordinator 可以恢复
- Participant 可以在超时后决策

### 流程

**Phase 1: CanCommit**

1. Coordinator 发送 `<CANCOMMIT>` 给所有 Participant
2. Participant 检查是否可以执行事务（不执行），回复 `<YES>` 或 `<NO>`

**Phase 2: PreCommit**

1. Coordinator 收集响应：
   - 全部 YES → 发送 `<PRECOMMIT>` 给所有 Participant
   - 任一 NO 或超时 → 发送 `<ABORT>` 给所有 Participant
2. Participant 收到 PRECOMMIT：
   - 执行事务（不提交），锁定资源
   - 回复 `<ACK>` 给 Coordinator

**Phase 3: DoCommit**

1. Coordinator 收到全部 ACK → 发送 `<DOCOMMIT>` 给所有 Participant
2. Participant 收到 DOCOMMIT → 提交事务

### 超时决策

关键设计：
- Participant 在 PreCommit 后超时可以 Abort
- Participant 在 DoCommit 后超时可以 Commit（因为已 PreCommit）

**非阻塞**：
- 如果所有 Participant 都进入 PreCommit，Coordinator 崩溃后新 Coordinator 可以恢复并 Commit
- 如果部分 Participant 未进入 PreCommit，新 Coordinator 可以 Abort

### 问题

**网络分区**：
- 分区中部分 Participant 收到 PreCommit，部分未收到
- 新 Coordinator 可能做出错误决策
- 无法完全避免不一致

**复杂性**：
- 三轮通信，延迟更高
- 实现复杂，实际很少使用

### 仍然无法完全解决问题

3PC 在异步网络中仍不能完全避免阻塞或不一致：
- 网络分区时可能违反原子性
- 只在网络"最终同步"假设下工作

## 与共识算法的关系

| 特性 | 2PC/3PC | Paxos/Raft |
|------|---------|------------|
| 问题类型 | 原子提交 | 共识 |
| 协调者 | 必须 | Leader（可选） |
| 容错 | 不容忍（2PC）/ 部分（3PC） | 容忍 f 崩溃 |
| 阻塞 | 可能阻塞 | 选举期间短暂阻塞 |
| 适用场景 | 分布式事务 | 状态机复制 |

关系：
- 可以用共识算法实现提交协议（如 Paxos Commit）
- 共识算法提供更强的一致性保证
- 实际系统常结合使用

## 实际应用

| 场景 | 使用协议 | 说明 |
|------|----------|------|
| 数据库跨节点事务 | 2PC | XA 标准 |
| Spanner | Paxos + 2PC | Paxos 保证副本一致性，2PC 保证跨分区事务 |
| 微服务事务 | Saga | 补偿事务，放弃强一致性 |

## 相关概念

- [[分布式共识算法对比]] - 共识算法对比
- [[Paxos 共识算法]] - 可用于实现原子提交
- CAP 定理：分布式系统的一致性、可用性、分区容错性权衡

## 参考资料

- [Two-Phase Commit - Wikipedia](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
- [Three-Phase Commit - Wikipedia](https://en.wikipedia.org/wiki/Three-phase_commit_protocol)
- [Consensus vs Atomic Commit - Martin Kleppmann](https://martin.kleppmann.com/2015/05/14/consensus-vs-atomic-commit.html)
