---
aliases:
  - VR协议
  - Viewstamped Replication
  - 视图 stamped 复制
created: '2026-04-21'
source_type: official-doc
source_urls:
  - 'https://dl.acm.org/doi/10.1145/62546.62549'
  - >-
    https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/viewstamped.pdf
status: verified
tags:
  - 分布式系统
  - 共识算法
  - Viewstamped Replication
  - VR
updated: '2026-04-21'
---
Viewstamped Replication（VR）是 Brian Oki 和 Barbara Liskov 于 1988 年发表的复制状态机协议。它是最早的 Leader-based 共识算法之一，比 Paxos（1998）早 10 年，比 Raft（2014）早 26 年。Raft 的设计直接借鉴了 VR。

## 是什么

VR 是一种用于实现复制状态机的协议，保证：
- 所有副本按相同顺序执行相同命令
- 容忍 f 个崩溃故障（需要 2f+1 个副本）
- 提供高可用性

核心概念：
- **View**：相当于 Raft 的 Term，表示 Leader 任期
- **Viewstamp**：View 编号 + 操作序列号，全局唯一标识

## 为什么重要

- 第一个实用的 Leader-based 复制协议
- Raft 的设计前身
- 证明 Leader-based 共识的正确性

VR 与 Raft 关系：
- Raft 论文承认受 VR 启发
- VR 的 View 对应 Raft 的 Term
- VR 的 Primary 对应 Raft 的 Leader
- VR 的 Backup 对应 Raft 的 Follower

## 核心机制

### 节点角色

| 角色 | 职责 |
|------|------|
| Primary（主副本） | 处理客户端请求，协调复制 |
| Backup（备份副本） | 接收并执行 Primary 的命令 |

每个 View 只有一个 Primary。

### View 编号

View 是递增编号：
- 每个 View 有一个 Primary
- Primary 按节点 ID 顺序轮换（如节点 0 → 1 → 2 → ...）
- View 变更触发新 Primary 选举

### 正常操作

Primary 处理请求流程：

1. Client 发送请求给 Primary
2. Primary 分配序列号，发送 `<PREPARE, v, n, m>` 给所有 Backup
   - v: View 编号
   - n: 序列号
   - m: 客户端请求
3. Backup 收到 Prepare，记录到日志，发送 `<PREPAREOK, v, n, i>` 给 Primary
4. Primary 收到 2f 个 PrepareOK（多数），执行请求
5. Primary 发送 `<COMMIT, v, n>` 给所有 Backup
6. Backup 收到 Commit，执行请求
7. Primary 回复 Client

### View Change

Primary 崩溃时触发 View Change：

触发条件：
- Backup 筝待 Prepare 超时
- Backup 筝待 Commit 超时

流程：
1. Backup 广播 `<STARTVIEWCHANGE, v+1, i>` 给所有节点
2. 节点收到 2f 个 StartViewChange（多数），发送 `<DOVIEWCHANGE, v+1, l, state>` 给新 Primary
   - l: 最后执行的序列号
   - state: 日志状态
3. 新 Primary 收到 2f 个 DoViewChange：
   - 选择最新的状态（最高 View + 最高序列号）
   - 发送 `<STARTVIEW, v+1, log, n>` 给所有 Backup
4. Backup 收到 StartView，进入新 View

### 恢复

节点恢复后：
- 发送 `<RECOVERY, i>` 给所有节点
- 收到多数 `<RECOVERYRESPONSE, v, n, log>` 后，恢复状态

## 与 Paxos/Raft 的对比

| 特性 | VR (1988) | Paxos (1998) | Raft (2014) |
|------|-----------|---------------|-------------|
| 设计者 | Oki & Liskov | Lamport | Ongaro & Ousterhout |
| Leader | 必须（Primary） | 可选（Multi-Paxos） | 必须 |
| 选举 | View Change | Leader Election | Leader Election |
| 日志 | 内置 | 需额外设计 | 内置 |
| 可理解性 | 中 | 高（原始）/中（简化） | 低（最易理解） |

关键相似点：
- VR 和 Raft 都强制 Leader
- VR 的 View Change 类似 Raft 的 Leader Election
- VR 的 Prepare/PrepareOK/Commit 类似 Raft 的 AppendEntries

关键区别：
- VR 使用轮换 Primary（按节点 ID），Raft 使用投票选举
- VR 的 Viewstamp 是连续编号，Raft 的 Term + Index 组合

## 历史意义

VR 是共识算法的重要里程碑：
- 证明 Leader-based 复制的正确性
- 提供完整的实现指南
- 影响 Raft 的设计

VR 被忽视的原因：
- 早期关注点不同（高可用 vs 共识）
- Paxos 论文影响力更大
- Raft 的"易于理解"目标更吸引人

## 相关概念

- [[Raft 共识算法]] - VR 的现代变体，更易理解
- [[Paxos 共识算法]] - 理论完备但复杂
- [[分布式共识算法对比]] - 各算法全面对比

## 参考资料

- [Viewstamped Replication: A New Primary Copy Method (1988)](https://dl.acm.org/doi/10.1145/62546.62549)
- [Viewstamped Replication Revisited - Liskov & Cowling (2012)](https://classes.cs.uchicago.edu/archive/2026/spring/23380-1/papers/oki_viewstampedreplication.pdf) - 更新版本
- [原始论文 PDF](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/viewstamped.pdf)
