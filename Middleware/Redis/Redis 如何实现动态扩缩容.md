---
created: "2026-04-16"
updated: "2026-04-16"
tags:
  - database
  - redis
  - clustering
  - scaling
  - resharding
  - slot-migration
  - high-availability
  - middleware
aliases:
  - Redis Cluster 动态扩缩容
  - Redis Cluster scaling
  - Redis resharding
  - Redis 槽位迁移
  - Redis 水平扩展
  - Redis 原子槽位迁移
  - Redis ASM
source_type: mixed
source_urls:
  - "https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/"
  - "https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/"
  - "https://redis.io/blog/atomic-slot-migration/"
  - "https://redis.io/blog/redis-8-4-open-source-ga/"
status: verified
---

## 核心概念

Redis 的动态扩缩容基于 **Redis Cluster** 架构实现，其核心机制是 **Hash Slot（哈希槽）的迁移**。Redis Cluster 将数据空间划分为 **16384 个 Hash Slot**，每个 Key 通过 `CRC16(key) mod 16384` 映射到唯一的 Slot，每个 Master 节点负责一部分 Slot。扩缩容的本质就是将这些 Slot 在节点之间重新分配，**整个过程无需停机**。

## 为什么需要动态扩缩容

| 场景 | 说明 |
|------|------|
| **容量扩展（Scale-out）** | 单节点内存达到上限，需要增加节点分担数据 |
| **性能扩展** | 单节点 CPU/网络带宽成为瓶颈，通过增加节点分散请求 |
| **缩容降本（Scale-in）** | 业务量下降，减少节点以节省资源成本 |
| **负载均衡** | 某些 Slot 因热点 Key 导致资源倾斜，需要重新分配 |
| **节点替换/升级** | 硬件升级、机房迁移时需要将数据迁移到新节点 |

## 前置概念

### Hash Slot 分片机制

- Redis Cluster **不使用一致性哈希**，而是固定 16384 个 Slot
- 每个 Key 的归属 Slot 计算方式：`HASH_SLOT = CRC16(key) mod 16384`
- CRC16 算法采用 XMODEM 规范（Poly: 1021, Init: 0000）
- 建议集群最大节点数约 **1000 个**（理论上限 16384 个 Master）

### Hash Tag（哈希标签）

强制多个 Key 映射到同一 Slot，以支持多 Key 操作：

```
# 以下两个 Key 会映射到同一个 Slot
{user:1000}.following
{user:1000}.followers
```

规则：Key 中 `{` 和 `}` 之间的子串被用于计算 Slot（需满足 `{` 在 `}` 左侧且中间有内容）。

### 主从模型与高可用

- 每个 Slot 有 1 个 Master 和 N 个 Replica
- Master 故障时，Replica 自动提升为新 Master
- 如果 Master 和其所有 Replica 同时故障，对应 Slot 不可用
- **Replica Migration**：当某个 Master 失去所有 Replica 时，其他有多余 Replica 的 Master 会自动迁移一个 Replica 过去

### 客户端重定向

| 错误类型 | 含义 | 客户端行为 |
|----------|------|-----------|
| `-MOVED <slot> <ip:port>` | Slot 已永久迁移到新节点 | 更新路由表，后续请求直接发往新节点 |
| `-ASK <slot> <ip:port>` | Slot 正在迁移中，Key 已在目标节点 | 仅下一次请求发往目标节点，需先发送 `ASKING` 命令 |
| `-TRYAGAIN` | 多 Key 操作期间 Slot 正在迁移 | 稍后重试 |

## Redis 8.4 之前：传统槽位迁移

### 迁移流程

```
1. 目标节点: CLUSTER SETSLOT <slot> IMPORTING <source-node-id>
2. 源节点:   CLUSTER SETSLOT <slot> MIGRATING <target-node-id>
3. 循环执行:
   a. CLUSTER GETKEYSINSLOT <slot> <count>    # 获取一批 Key
   b. MIGRATE <target-host> <target-port> "" 0 <timeout> KEYS k1 k2 ...  # 迁移 Key
4. 直到 CLUSTER COUNTKEYSINSLOT <slot> == 0
5. 源和目标节点: CLUSTER SETSLOT <slot> NODE <target-node-id>
6. 通知全集群更新配置
```

### 使用的 redis-cli 命令

```bash
# 交互式迁移指定数量 Slot
redis-cli --cluster reshard <host>:<port> \
  --cluster-from <node-id> \
  --cluster-to <node-id> \
  --cluster-slots <number> \
  --cluster-yes

# 自动均衡分配 Slot
redis-cli --cluster rebalance <host>:<port>

# 添加新节点（Master）
redis-cli --cluster add-node <new-host>:<new-port> <existing-host>:<existing-port>

# 添加新节点（Replica，随机分配给 Replica 最少的 Master）
redis-cli --cluster add-node <new-host>:<new-port> <existing-host>:<existing-port> \
  --cluster-slave

# 添加新节点（Replica，指定 Master）
redis-cli --cluster add-node <new-host>:<new-port> <existing-host>:<existing-port> \
  --cluster-slave \
  --cluster-master-id <master-node-id>

# 删除节点（Master 必须为空）
redis-cli --cluster del-node <host>:<port> <node-id>
```

### 传统迁移的问题

| 问题 | 具体表现 |
|------|---------|
| **非原子性** | Key 逐个迁移，中途失败可能导致数据不一致或丢失 |
| **大量客户端重定向** | 迁移期间产生大量 `-ASK`/`-MOVED` 重定向（最高 241.6 次/秒） |
| **多 Key 操作不可靠** | 部分 Key 已迁移时返回 `-TRYAGAIN` |
| **复制问题** | Replica 可能不知道 Slot 正在迁移，错误回复 Key 已删除 |
| **速度慢** | 逐 Key 迁移，约 21 Slot/秒 |
| **大 Key 延迟** | `MIGRATE` 大 Key 时可能超时或造成显著延迟峰值 |
| **集群消息开销大** | 持续更新集群状态，最高产生 5.4K 条消息 |

## Redis 8.4+：原子槽位迁移（ASM）

Redis 8.4 引入了 **Atomic Slot Migration（ASM）**，从根本上解决了传统迁移的所有问题。

### 核心改进

| 指标 | 传统迁移 | ASM | 改进 |
|------|---------|-----|------|
| 迁移速度 | ~21 Slot/秒 | ~640 Slot/秒 | **快 30 倍** |
| 客户端重定向 | 最高 241.6 次/秒 | 2.1 次/秒 | **减少 98%** |
| 延迟峰值 | 最高 127 ms | <70 ms | **降低 40%+** |
| 集群消息 | 最高 5.4K 条 | 212 条 | **减少 94%** |
| 数据一致性 | 中途失败可能不一致 | 原子手递手，要么全成功要么不切换 | **根本性改进** |

### ASM 工作原理

```
1. 从目标节点发起: CLUSTER MIGRATION IMPORT <start-slot> <end-slot>
   → 返回任务 ID

2. 目标节点向源节点请求 Slot 复制，建立两条并行连接:
   - 快照连接：发送 Slot 全量快照
   - 增量流连接：流式传输增量写入

3. 源节点 fork 后发送快照（通常以 per-key RESTORE 命令形式）
   大 Key 自动切换为 AOF-style 分块格式，降低内存峰值

4. 目标节点应用快照命令并累积增量更新

5. 源节点短暂暂停写入，转发剩余更新后
   → 目标节点原子接管 Slot 所有权（更新集群配置并通过 Cluster Bus 广播）

6. 源节点恢复写流量
   → 客户端收到 -MOVED 转向新所有者

7. 源节点后台线程异步删除已迁移的 Slot 数据
   （类似异步 FLUSHALL，不影响主线程延迟）
```

### ASM 使用命令

| 命令 | 说明 | 发送目标 |
|------|------|---------|
| `CLUSTER MIGRATION IMPORT <start-slot> <end-slot> [...]` | 发起迁移，返回任务 ID | **目标节点** |
| `CLUSTER MIGRATION STATUS <ID id \| ALL>` | 查看迁移状态 | **目标节点** |
| `CLUSTER MIGRATION CANCEL <ID id \| ALL>` | 取消迁移任务 | **目标节点** |

### ASM 性能实测数据

**扩容（3 → 4 分片）**：
- 迁移 1/3 Slot 耗时 **6.4 秒**
- 吞吐量持续上升，平均延迟增加 <5%（持续 2 秒）

**缩容（4 → 3 分片）**：
- 迁移耗时 **8.6 秒**
- 延迟从 2.3 ms 增至 2.8 ms（持续 3 秒）

### ASM 已知限制

在 ASM 迁移过程中执行以下命令时，结果可能**不完整或包含重复**：
- `FT.SEARCH`、`FT.AGGREGATE`、`FT.CURSOR`、`FT.HYBRID`
- `TS.MGET`、`TS.MRANGE`、`TS.MREVRANGE`、`TS.QUERYINDEX`

## 扩容操作完整流程

### 添加新 Master 节点并分配 Slot

```bash
# 1. 启动新节点（配置 cluster-enabled yes）
redis-server ./redis.conf --port 7006 --cluster-enabled yes

# 2. 将新节点加入集群
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# 3. 从现有节点迁移 Slot 到新节点
redis-cli --cluster reshard 127.0.0.1:7000 \
  --cluster-from all \
  --cluster-to <new-node-id> \
  --cluster-slots 1000 \
  --cluster-yes

# 4. 验证集群状态
redis-cli --cluster check 127.0.0.1:7000
```

### 添加新 Replica 节点

```bash
# 方式一：随机分配给 Replica 最少的 Master
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000 --cluster-slave

# 方式二：指定 Master
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000 \
  --cluster-slave \
  --cluster-master-id <master-node-id>

# 方式三：先作为空 Master 加入，再转为 Replica
redis-cli -c -p 7006
> CLUSTER REPLICATE <master-node-id>
```

## 缩容操作完整流程

### 移除节点

```bash
# 1. 如果是 Master，先迁移其所有 Slot 到其他节点
redis-cli --cluster reshard 127.0.0.1:7000 \
  --cluster-from <node-to-remove-id> \
  --cluster-to <target-node-id> \
  --cluster-slots <all-slots-count> \
  --cluster-yes

# 2. 确认节点 Slot 为空
redis-cli -p 7000 cluster nodes | grep <node-to-remove-id>

# 3. 删除节点
redis-cli --cluster del-node 127.0.0.1:7000 <node-to-remove-id>
```

### 优雅下线 Master（通过手动 Failover）

```bash
# 1. 在目标 Replica 上发起手动 Failover
redis-cli -p <replica-port> CLUSTER FAILOVER

# 2. 等待 Failover 完成，原 Master 变为 Replica
redis-cli -p <original-master-port> cluster nodes | grep myself

# 3. 迁移该节点（现为 Replica）的 Slot
# （其原 Slot 已随新 Master 转移，无需额外迁移）

# 4. 删除节点
redis-cli --cluster del-node 127.0.0.1:7000 <node-id>
```

**手动 Failover 的优势**：避免数据丢失，仅在 Replica 完全同步后才切换客户端流量。

## 检测负载不均与热点 Slot

### CLUSTER SHARDS

报告每个 Shard 负责的 Slot 集合：

```bash
redis-cli CLUSTER SHARDS
```

### CLUSTER SLOT-STATS（Redis 8.2+）

按指标排序发现热点 Slot：

```bash
# 按 CPU 使用量降序排列前 10 个 Slot
redis-cli CLUSTER SLOT-STATS ORDERBY CPU-USEC DESC LIMIT 10

# 按内存使用量降序排列
redis-cli CLUSTER SLOT-STATS ORDERBY MEMORY-BYTES DESC LIMIT 10
```

支持的指标：

| 指标 | 说明 | 版本 |
|------|------|------|
| `KEY-COUNT` | Slot 中的 Key 数量 | 8.2+ |
| `CPU-USEC` | 处理该 Slot 消耗的 CPU 时间（微秒） | 8.2+ |
| `NETWORK-BYTES-IN` | 入站网络流量（字节） | 8.2+ |
| `NETWORK-BYTES-OUT` | 出站网络流量（字节） | 8.2+ |
| `MEMORY-BYTES` | Slot 中 Key 占用的总内存 | **8.4+** |

## 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `cluster-enabled` | no | 是否启用 Cluster 模式 |
| `cluster-config-file` | nodes.conf | 集群状态持久化文件（自动生成，不要手动编辑） |
| `cluster-node-timeout` | 15000 | 节点不可达超时时间（毫秒），超过此时间视为 FAIL |
| `cluster-slave-validity-factor` | 10 | Replica 有效性因子，0 表示始终尝试 Failover |
| `cluster-migration-barrier` | 1 | Master 最少保留的 Replica 数，超过此数才允许 Replica 迁移 |
| `cluster-require-full-coverage` | yes | 是否要求全部 Slot 覆盖才接受写入 |
| `cluster-allow-reads-when-down` | no | 集群 FAIL 时是否允许读操作 |
| `cluster-port` | data_port + 10000 | Cluster Bus 通信端口 |

## 一致性保证与限制

### 弱一致性模型

Redis Cluster **不保证强一致性**，采用异步复制：

```
Client → Master (写入) → 回复 OK → 异步复制到 Replica
```

**可能丢失写入的场景**：
1. Master 回复客户端后崩溃，写入未传播到 Replica
2. 网络分区中，少数派一侧的 Master 接受的写入可能在 Failover 后丢失

**降低风险的方法**：
- 使用 `WAIT` 命令同步等待 Replica 确认
- 手动 Failover 比自动 Failover 更安全（等待完全同步后才切换）

### 多 Key 操作限制

- 所有 Key 必须在同一 Slot（使用 Hash Tag 强制）
- 迁移期间的多 Key 操作可能返回 `-TRYAGAIN`
- 仅支持数据库 0，`SELECT` 命令不可用

### 网络分区行为

- 少数派一侧的 Master 在 `cluster-node-timeout` 后停止接受写入
- 多数派一侧在 `NODE_TIMEOUT` + Failover 时间后恢复可用
- 典型 Failover 时间：1-2 秒

## 最佳实践

### 扩容时机

- 单节点内存使用率 > 70% 时考虑扩容
- 通过 `CLUSTER SLOT-STATS` 发现热点 Slot 时考虑重新分配
- 业务增长预期明确时提前扩容，避免紧急操作

### 缩容注意事项

- 缩容前确保目标节点有足够内存容纳迁移的 Slot
- 优先在业务低峰期执行缩容
- 使用 ASM（Redis 8.4+）可大幅降低缩容对业务的影响

### 客户端配置

- 使用支持 Cluster 的客户端库（如 Jedis Cluster、Lettuce、redis-py-cluster）
- 客户端应缓存 Slot → Node 映射，收到 `-MOVED` 时刷新
- 正确处理 `-ASK` 重定向（先发送 `ASKING` 命令）

### 监控建议

- 监控每个节点的内存、CPU、网络流量
- 定期检查 `CLUSTER INFO` 和 `CLUSTER NODES` 状态
- 关注 `-MOVED` 和 `-ASK` 重定向频率，异常升高说明可能有迁移在进行或配置问题

## 常见误区

| 误区 | 正确理解 |
|------|---------|
| "Redis Cluster 使用一致性哈希" | 使用固定 16384 Slot，不是一致性哈希 |
| "扩容需要停机" | Slot 迁移是在线操作，无需停机 |
| "Cluster 支持多数据库" | 仅支持数据库 0，不支持 `SELECT` |
| "Cluster 保证强一致性" | 异步复制，弱一致性模型，可能丢失写入 |
| "添加节点后自动分配 Slot" | 新节点加入后是空的，需要手动或通过 reshard 分配 Slot |
| "可以直接删除非空 Master" | Master 必须为空（所有 Slot 已迁移）才能删除 |

## 与相关概念的关系

| 概念 | 与扩缩容的关系 |
|------|--------------|
| **Sentinel** | 提供主从自动 Failover，但不支持数据分片；Cluster 内置了 Sentinel 的 Failover 能力并增加了分片 |
| **Proxy 方案（如 Twemproxy、Codis）** | 通过代理层实现分片，客户端无感知；Cluster 是去中心化方案，客户端直接连接节点 |
| **Replica Migration** | 自动将 Replica 从有多余副本的 Master 迁移到失去所有副本的 Master，提升集群可用性 |
| **手动 Failover** | 在升级或维护时将 Master 优雅降级为 Replica，是安全缩容的前置步骤 |

## 参考资料

- [Scale with Redis Cluster — 官方文档](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis Cluster Specification — 官方规范](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Atomic Slot Migration with Redis 8.4 — 官方博客](https://redis.io/blog/atomic-slot-migration/)（2026-04-02）
- [Redis 8.4 GA 官方公告](https://redis.io/blog/redis-8-4-open-source-ga/)（2025-11-18）
- [CLUSTER MIGRATION 命令文档](https://redis.io/docs/latest/commands/cluster-migration/)
- [CLUSTER SETSLOT 命令文档](https://redis.io/docs/latest/commands/cluster-setslot/)
- [CLUSTER SLOT-STATS 命令文档](https://redis.io/docs/latest/commands/cluster-slot-stats/)
- [CLUSTER FAILOVER 命令文档](https://redis.io/docs/latest/commands/cluster-failover/)
- [Atomic Slot Migration PR #14414](https://github.com/redis/redis/pull/14414)
