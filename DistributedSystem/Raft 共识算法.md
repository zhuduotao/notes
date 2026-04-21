---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - 分布式系统
  - 共识算法
  - Raft
  - 复制状态机
aliases:
  - Raft协议
  - Raft一致性算法
source_type: official-doc
source_urls:
  - 'https://raft.github.io/'
  - 'https://github.com/ongardie/dissertation'
status: verified
---
Raft 是一种共识算法，设计目标是易于理解。它在容错性和性能上与 Paxos 等价，但将共识问题分解为相对独立的子问题，并提供清晰的实现指南。

## 是什么

Raft（Reliable, Replicated, Redundant, And Fault-Tolerant）是一种用于管理复制日志的共识算法。由 Diego Ongaro 和 John Ousterhout 于 2014 年在论文 "In Search of an Understandable Consensus Algorithm" 中提出。

核心目标：
- 提供与 Paxos 等价的容错能力
- 显著降低理解难度
- 提供完整的实现细节

## 为什么重要

传统 Paxos 算法难以理解和正确实现：
- 原始论文使用隐喻描述，概念抽象
- 缺少清晰的实现指南
- 业界多次实现都存在设计缺陷

Raft 的改进：
- 明确分解为三个子问题：Leader Election、Log Replication、Safety
- 减少状态空间（每个节点只有三种状态）
- 提供完整的功能规范和实现细节

## 核心机制

### 节点状态

Raft 节点只有三种状态：

| 状态 | 职责 |
|------|------|
| Leader | 处理所有客户端请求，管理日志复制 |
| Follower | 被动接收 Leader 的日志条目，响应投票请求 |
| Candidate | Leader 选举过程中的过渡状态 |

状态转换：
```
Follower → Candidate（选举超时触发）
Candidate → Leader（获得多数投票）
Candidate → Follower（收到更高 term 的心跳或其他节点成为 Leader）
Leader → Follower（收到更高 term 的消息）
```

### Term（任期）

Term 是 Raft 的逻辑时钟概念：
- 每个 Term 有唯一递增编号
- 新 Term 开始于 Leader 选举
- 每个 Term 至多有一个 Leader（可能没有）
- 每个节点维护当前 Term，用于检测过期信息

### Leader Election

选举流程：

1. **触发条件**：Follower 在 electionTimeout 期间未收到 Leader 心跳
2. **发起选举**：
   - Follower 转为 Candidate
   - Term 自增
   - 投票给自己
   - 向其他节点发送 RequestVote RPC
3. **投票规则**：
   - 每个 Term 每个节点最多投一票
   - 先到先得（FIFO）
   - 只投票给日志至少和自己一样新的 Candidate
4. **选举结果**：
   - 获得多数票 → 成为 Leader，立即发送心跳
   - 收到更高 Term 消息 → 转为 Follower
   - 选举超时 → 开始新 Term 重新选举

**随机超时**：选举超时随机化（如 150-300ms），避免同时发起选举导致的投票分裂。

### Log Replication

日志结构：
- 每个日志条目包含：index（位置）、term（产生时的 Term）、command（状态机命令）
- Leader 维护每个 Follower 的 nextIndex 和 matchIndex

复制流程：

1. 客户端请求发送到 Leader
2. Leader 将命令追加到本地日志（uncommitted 状态）
3. Leader 向所有 Follower 发送 AppendEntries RPC
4. Follower 检查一致性（前一条日志的 index 和 term 匹配），成功则追加
5. Leader 收到多数 Follower 确认后，条目变为 committed
6. Leader 通知 Follower commit，所有节点应用条目到状态机

**一致性检查**：
- AppendEntries RPC 包含前一条日志的 index 和 term
- Follower 检查是否匹配
- 不匹配则拒绝，Leader 回退 nextIndex 重试

**日志匹配性质**：
1. 如果两个日志在相同 index 有相同 term，则该位置命令相同
2. 如果两个日志在相同 index 有相同 term，则之前所有日志都相同

### Safety（安全性保证）

**选举限制**：
- RequestVote RPC 包含 Candidate 的 lastLogIndex 和 lastLogTerm
- Follower 只投票给日志至少和自己一样新的 Candidate
- 保证新 Leader 必须包含所有已 committed 的日志

**提交规则**：
- Leader 只能提交当前 Term 的日志
- 通过当前 Term 的日志提交来间接提交之前 Term 的日志
- 避免图 8 场景：旧 Term 日志被新 Leader 覆盖

### 成员变更

单步成员变更：
- 从旧配置 C_old 到新配置 C_new 必须经过中间配置 C_old,new
- C_old,new 要求新旧配置各有独立多数
- 变更期间系统可用

流程：
1. Leader 收到配置变更请求
2. Leader 追加 C_old,new 到日志
3. C_old,new committed 后生效
4. Leader 追加 C_new 到日志
5. C_new committed 后完成变更

## 限制与注意事项

1. **最小节点数**：需要至少 2f+1 个节点容忍 f 个故障（多数原则）
2. **Leader 依赖**：Leader 崩溃时系统短暂不可用（选举期间）
3. **读一致性**：
   - Leader 本地读可能返回过期数据（网络分区场景）
   - 解决方案：ReadIndex、Lease Read
4. **日志压缩**：日志无限增长，需要 Snapshot 机制
5. **网络分区**：旧 Leader 在分区中可能短暂响应客户端，但无法提交新条目

## 相关概念

- [[Paxos 共识算法]] - Raft 的理论基础来源之一
- [[ZAB 协议]] - 类似 Raft 的 Leader-based 协议
- [[Viewstamped Replication 协议]] - Raft 的设计前身
- [[分布式共识算法对比]] - 各算法全面对比

## 实现与应用

主要实现：
- etcd（Go）- Kubernetes 核心存储
- Consul（Go）- 服务发现
- TiKV（Rust）- TiDB 存储引擎
- HashiCorp Vault - 密钥管理

可视化学习：
- [The Secret Lives of Data](http://thesecretlivesofdata.com/raft/)
- [RaftScope](https://raft.github.io/raftscope/index.html)

## 参考资料

- [Raft 官方网站](https://raft.github.io/)
- [Raft 论文（扩展版）](https://raft.github.io/raft.pdf)
- [Diego Ongaro 博士论文](https://github.com/ongardie/dissertation) - 包含 TLA+ 规范和更详细的成员变更算法
- [Raft 用户研究](https://ongardie.net/static/raft/userstudy/) - Raft 与 Paxos 理解难度对比实验
