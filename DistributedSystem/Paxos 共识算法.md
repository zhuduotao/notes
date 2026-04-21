---
created: '2026-04-21'
source_type: official-doc
source_urls:
  - 'https://lamport.azurewebsites.net/pubs/paxos-simple.pdf'
  - 'https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf'
status: verified
tags:
  - 分布式系统
  - 共识算法
  - Paxos
  - Lamport
updated: '2026-04-21'
aliases:
  - Paxos协议
  - Paxos一致性算法
---
Paxos 是 Leslie Lamport 于 1998 年发表的分布式共识算法，被认为是共识算法的理论基础。尽管难以理解和实现，但它是第一个被证明正确且可用的共识算法。

## 是什么

Paxos 是一种在异步网络环境中容忍节点崩溃的共识算法。名称来自虚构的 Paxos 岛议会，Lamport 用此隐喻描述算法。

核心保证：
- **安全性**：所有正确节点最终同意同一个值
- **活性**：在足够长时间无新故障时，最终达成共识

原始论文 "The Part-Time Parliament"（1998）因晦涩难懂，Lamport 于 2001 年发表 "Paxos Made Simple" 简化解释。

## 为什么重要

- 第一个理论上完备的共识算法
- Google Chubby、Spanner 等系统的基础
- Raft、ZAB 等算法的设计灵感来源

Paxos 的难点：
- 原始论文使用隐喻，角色命名晦涩（Proposer、Acceptor、Learner）
- 算法细节分散在多个变体中
- 缺少清晰的实现指南

## 核心机制

### 基本角色

| 角色 | 职责 |
|------|------|
| Proposer | 提议值，推动共识过程 |
| Acceptor | 投票，存储已接受的值 |
| Learner | 学习最终达成共识的值 |

实际实现中，一个节点通常兼任多个角色。

### Basic Paxos（单值共识）

共识一个值的流程，分为两个阶段：

**Phase 1: Prepare**

1. Proposer 选择提议编号 n（全局唯一且递增）
2. Proposer 向多数 Acceptor 发送 Prepare(n) 请求
3. Acceptor 收到 Prepare(n)：
   - 如果 n > 已承诺的最大编号 promiseNum，回复 Promise(n, acceptedValue) 或 Promise(n, null)
   - 承诺不再接受编号小于 n 的提议
   - 如果之前接受过值，返回已接受的值和编号

**Phase 2: Accept**

1. Proposer 收到多数 Promise 后：
   - 如果有 Acceptor 返回已接受的值，Proposer 必须提议其中编号最大的值
   - 如果没有返回值，Proposer 可以提议自己想提议的值
2. Proposer 发送 Accept(n, value) 给多数 Acceptor
3. Acceptor 收到 Accept(n, value)：
   - 如果 n >= promiseNum，接受该值，回复 Accepted(n, value)
   - 否则拒绝

**共识达成**：当多数 Acceptor 接受了同一个值时，共识达成。

**提议编号生成**：
- 需保证全局唯一且递增
- 常见方法：节点 ID + 本地计数器

### Multi-Paxos（多值共识）

Basic Paxos 每次共识需要两轮通信。Multi-Paxos 通过选举稳定 Leader 优化：

- Leader 作为唯一的 Proposer，省略 Prepare 阶段
- Leader 可以连续提议多个值，只需 Accept 阶段
- Leader 崩溃时，新 Leader 通过 Prepare 阶段恢复

Multi-Paxos 类似 Raft 的 Leader + Log Replication 模式。

### Fast Paxos

Fast Paxos 允许 Proposer 直接跳过 Prepare 阶段：
- 需要更多 Acceptor（3f+1 vs 2f+1）
- 允许冲突（多个 Proposer 同时提议）
- 冲突时需要额外一轮解决

## 限制与注意事项

1. **提议编号必须递增**：Proposer 需要协调编号生成，避免冲突
2. **活锁风险**：多个 Proposer 交替提议更高编号，永远无法进入 Accept 阶段
   - 解决：选举 Leader 或随机等待
3. **实现复杂**：原始论文缺少实现细节，需要处理：
   - 重复消息处理
   - Proposal 编号协调
   - Leader 选举
   - 日志恢复
4. **无日志概念**：Basic Paxos 共识单值，Multi-Paxos 需额外设计日志机制
5. **最小节点数**：2f+1 个节点容忍 f 个崩溃故障

## 与 Raft 的对比

| 特性 | Paxos | Raft |
|------|-------|------|
| 设计理念 | 证明完备性 | 易于理解 |
| Leader | Multi-Paxos 可选 | 必须 |
| 日志 | 需额外设计 | 内置 |
| 成员变更 | 未详细说明 | 单步变更 |
| 可视化 | 少 | 多（动画、交互） |

关键区别：
- Raft 强制 Leader，简化共识流程
- Raft 明确日志结构和复制机制
- Raft 提供完整实现指南

## 相关概念

- [[Raft 共识算法]] - Paxos 的简化变体，更易理解
- [[ZAB 协议]] - 类 Paxos 的 ZooKeeper 协议
- [[分布式共识算法对比]] - 各算法全面对比

## 实现与应用

- Google Chubby：分布式锁服务
- Google Spanner：全球分布式数据库
- Microsoft Azure Cosmos DB：部分 Paxos 实现
- Amazon DynamoDB：部分 Paxos 思想

## 参考资料

- [The Part-Time Parliament - Leslie Lamport (1998)](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)
- [Paxos Made Simple - Leslie Lamport (2001)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) - 推荐首先阅读
- [Paxos for System Builders](https://jhu-dsn.github.io/Paxos-SB.html) - 实现者视角
- [Martin Fowler: Paxos](https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html)
