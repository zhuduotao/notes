---
aliases:
  - ZAB协议
  - ZooKeeper Atomic Broadcast
created: '2026-04-21'
source_type: official-doc
source_urls:
  - 'https://zookeeper.apache.org/doc/current/zookeeperInternals.html'
  - 'https://zdq0394.github.io/papers/zab.pdf'
status: verified
tags:
  - 分布式系统
  - 共识算法
  - ZAB
  - ZooKeeper
updated: '2026-04-21'
---
ZAB（ZooKeeper Atomic Broadcast）是 Apache ZooKeeper 使用的原子广播协议，专为 ZooKeeper 的协调服务需求设计。它保证消息的全局有序性和原子性交付。

## 是什么

ZAB 是一种 Leader-based 的原子广播协议，核心功能：
- **Leader 选举**：选举单一 Leader 处理所有写请求
- **原子广播**：保证所有服务器按相同顺序接收相同消息
- **崩溃恢复**：Leader 崩溃后快速恢复，保证一致性

ZAB 借鉴 Paxos 思想，但专为 ZooKeeper 设计，与 Paxos 和 Raft 有显著区别。

## 为什么重要

ZooKeeper 是分布式协调服务，被广泛用于：
- 配置管理
- 分布式锁
- 服务注册与发现
- Leader 选举

ZAB 是 ZooKeeper 的核心，保证其一致性。

## 核心机制

### 两阶段架构

ZAB 协议分为两个阶段：

1. **Leader Activation（Leader 激活/恢复）**
   - Leader 选举后，同步所有 Follower 状态
   - 确保新 Leader 包含所有已提交的事务
   - 提交 NEW_LEADER 提议

2. **Active Messaging（活跃消息广播）**
   - Leader 处理客户端请求
   - 广播提议给所有 Follower
   - 收到多数确认后提交

### zxid（事务 ID）

zxid 是 ZooKeeper 的全局事务标识符，结构：
- 高 32 位：epoch（Leader 任期编号）
- 低 32 位：counter（事务计数器）

epoch 表示 Leader 变更：
- 每次 Leader 变更，epoch 递增
- 确保唯一性：每个 Leader 使用独立 epoch
- zxid 递增保证全局有序

### Leader Election

Leader 选举要求：
- Leader 必须拥有所有 Follower 中最高的 zxid
- 多数 Follower 忆承诺跟随该 Leader

选举算法：FastLeaderElection（默认）
- 每个服务器投票给 zxid 最高的服务器
- 收到多数投票的服务器成为 Leader
- 如果平票，投票给服务器 ID 最大的

### Leader Activation

新 Leader 激活流程：

1. Leader 等待 Follower 连接
2. Leader 与每个 Follower 同步：
   - 发送 Follower 缺失的提议
   - 或发送完整快照（差异过大时）
3. 特殊情况处理：
   - 如果 Follower 有 Leader 未见的提议 U，Leader 通知 Follower 丢弃 U
   - 原因：U 未被多数确认（否则 Leader 应已见过）
4. Leader 提议 NEW_LEADER
5. 多数 Follower 确认后，Leader 提交 NEW_LEADER
6. Leader 开始接受客户端请求

### Active Messaging

消息广播流程（类似两阶段提交）：

1. 客户端请求发送到 Leader
2. Leader 生成提议（Proposal），分配 zxid
3. Leader 发送提议给所有 Follower
4. Follower 将提议写入日志，发送 ACK
5. Leader 收到多数 ACK 后，发送 COMMIT
6. Follower 收到 COMMIT 后，应用事务到状态机

关键约束：
- Leader 按顺序发送提议（FIFO）
- Follower 按顺序处理消息
- COMMIT 按顺序发送和接收

### 恢复保证

ZAB 保证：
- 已提交的事务不会丢失
- 未提交的事务不会被错误提交

关键设计：
- 新 Leader 必须见过最高 zxid
- NEW_LEADER 提议确保多数同步
- epoch 机制防止旧 Leader 干扰

## 与 Paxos/Raft 的区别

| 特性 | ZAB | Paxos | Raft |
|------|-----|-------|------|
| 设计目标 | ZooKeeper 专用 | 通用共识 | 易于理解 |
| Leader 激活 | 独立阶段，显式恢复 | Prepare 阶段隐含 | 隐式恢复 |
| 全局顺序 | zxid 跨任期连续 | 每次提议独立编号 | Term + Index 组合 |
| 恢复策略 | 同步完整历史 | 可跳过未提交提议 | 类似 ZAB |

关键区别：
- ZAB 恢复完整历史，确保 zxid 连续
- Paxos 可跳过未提交提议，使用新编号
- Raft 类似 ZAB，但使用 Term 区分

## 限制与注意事项

1. **Leader 依赖**：Leader 崩溃时短暂不可用
2. **写性能限制**：所有写请求经 Leader，Leader 成为瓶颈
3. **最小节点数**：2f+1 个节点容忍 f 个故障
4. **读一致性**：
   - Follower 读可能返回过期数据
   - 需要 sync() 操作保证强一致性
5. **网络分区**：分区 minority 的 Leader 无法提交新事务

## 相关概念

- [[Raft 共识算法]] - 类似 ZAB 的现代共识算法
- [[Paxos 共识算法]] - ZAB 的理论基础
- [[分布式共识算法对比]] - 各算法全面对比

## 参考资料

- [ZooKeeper Internals - Apache](https://zookeeper.apache.org/doc/current/zookeeperInternals.html)
- [Zab: High-performance broadcast for primary-backup systems (2011)](https://zdq0394.github.io/papers/zab.pdf)
- [ZooKeeper Protocol Specification (TLA+)](https://github.com/apache/zookeeper/blob/master/zookeeper-specifications/protocol-spec/doc.md)
- [Zab 1.0 - Apache Wiki](https://cwiki.apache.org/confluence/display/zookeeper/zab1.0)
