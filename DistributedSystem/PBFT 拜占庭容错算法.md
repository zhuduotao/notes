---
aliases:
  - PBFT协议
  - Practical Byzantine Fault Tolerance
  - 拜占庭容错
created: '2026-04-21'
source_type: official-doc
source_urls:
  - >-
    https://www.usenix.org/legacy/publications/library/proceedings/osdi99/full_papers/castro/castro_html/castro.html
  - 'https://www.scs.stanford.edu/nyu/03sp/sched/bfs.pdf'
status: verified
tags:
  - 分布式系统
  - 共识算法
  - PBFT
  - 拜占庭容错
  - BFT
updated: '2026-04-21'
---
PBFT（Practical Byzantine Fault Tolerance）是 Miguel Castro 和 Barbara Liskov 于 1999 年发表的拜占庭容错共识算法。它是第一个实用且高效的 BFT 算法，能容忍恶意节点的任意行为。

## 是什么

PBFT 是一种在异步网络环境中容忍拜占庭故障的共识算法。拜占庭故障指节点可能：
- 发送错误消息
- 不发送消息
- 发送不一致消息给不同节点
- 恶意协作攻击系统

PBFT 保证：
- **安全性**：最多容忍 f 个拜占庭节点，总节点数 n ≥ 3f+1
- **活性**：在网络最终同步时达成共识

## 为什么重要

传统共识算法（Paxos、Raft）仅容忍崩溃故障（CFT）。在以下场景需要 BFT：
- 区块链联盟链：节点可能恶意行为
- 金融系统：安全要求极高
- 跨信任域系统：无法完全信任所有节点

PBFT 是 BFT 的经典算法，后续许多 BFT 变体基于它设计。

## 核心机制

### 节点角色

| 角色 | 职责 |
|------|------|
| Primary（主节点） | 协调共识过程，相当于 Leader |
| Backup（备份节点） | 验证并参与共识 |
| Client | 发送请求 |

所有节点（除 Client）都参与共识。Primary 可能是拜占庭节点。

### 最小节点数

n ≥ 3f+1 的原因：
- 需要 2f+1 个正确节点达成共识
- 其中最多 f 个可能是拜占庭节点假装正确
- 所以需要 n ≥ 3f+1 才能保证 2f+1 个真实正确节点

### 三阶段共识

PBFT 共识分为三个阶段：

**Phase 1: Pre-Prepare**

1. Client 发送请求 `<REQUEST, o, t, c>` 给 Primary
   - o: 操作
   - t: 时间戳
   - c: Client ID
2. Primary 分配序列号 n，广播 `<PRE-PREPARE, v, n, d, m>` 给所有 Backup
   - v: View 编号
   - n: 序列号
   - d: 消息摘要
   - m: 原始请求（可选）

**Phase 2: Prepare**

1. Backup 收到 Pre-Prepare，验证：
   - 消息签名正确
   - View 和序列号有效
   - 消息摘要匹配
2. Backup 广播 `<PREPARE, v, n, d, i>` 给所有节点
   - i: 节点 ID

**Phase 3: Commit**

1. 节点收到 2f 个匹配的 Prepare 消息（来自不同节点，包括自己），进入 Prepare 状态
2. 节点广播 `<COMMIT, v, n, d, i>` 给所有节点
3. 节点收到 2f+1 个匹配的 Commit 消息（来自不同节点，包括自己），执行请求
4. 节点回复 Client `<REPLY, v, t, c, i, r>`

### View Change

Primary 可能是拜占庭节点，需要机制切换：

触发条件：
- Backup 等待 Pre-Prepare 超时
- Backup 等待 Prepare 超时

View Change 流程：
1. Backup 广播 `<VIEW-CHANGE, v+1, n, C, P, i>`
   - n: 最后执行的序列号
   - C: 已执行请求的证明
   - P: 未执行请求的 Prepare 证明
2. 新 Primary 收到 2f 个 View-Change 消息
3. 新 Primary 选择最高序列号 n，发送 `<NEW-VIEW, v+1, V, O>`
   - V: View-Change 消息集合
   - O: Pre-Prepare 消息集合（恢复未完成请求）
4. Backup 验证 NEW-VIEW，进入新 View

### 消息验证

PBFT 使用数字签名验证：
- 所有消息包含签名
- 防止拜占庭节点伪造消息
- 客户端验证回复签名

## 消息复杂度

PBFT 消息复杂度 O(n²)：
- Pre-Prepare: Primary → 所有节点（O(n)）
- Prepare: 所有节点 → 所有节点（O(n²)）
- Commit: 所有节点 → 所有节点（O(n²)）

这限制了 PBFT 的节点数量（通常 < 20）。

## 限制与注意事项

1. **节点数限制**：n ≥ 3f+1，消息复杂度 O(n²)，不适合大规模节点
2. **Primary 依赖**：Primary 可能是拜占庭节点，View Change 成本高
3. **网络假设**：需要网络最终同步（消息延迟有上限）
4. **性能开销**：数字签名验证开销大
5. **状态管理**：需要维护大量消息日志

## BFT 变体

针对 PBFT 的优化：

| 变体 | 优化方向 |
|------|----------|
| RBFT | 减少 Primary 依赖 |
| SBFT | 简化三阶段为两阶段 |
| Tendermint | 结合 BFT 与 PoS |
| HotStuff | pipelining，降低复杂度 |

## 相关概念

- [[Paxos 共识算法]] - CFT 算法对比
- [[Raft 共识算法]] - CFT 算法对比
- [[分布式共识算法对比]] - 各算法全面对比

## 参考资料

- [Practical Byzantine Fault Tolerance - Castro & Liskov (1999)](https://www.usenix.org/legacy/publications/library/proceedings/osdi99/full_papers/castro/castro_html/castro.html)
- [PBFT 论文 PDF](https://www.scs.stanford.edu/nyu/03sp/sched/bfs.pdf)
- [MIT LCS TR-817](https://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-817.pdf) - 详细技术报告
