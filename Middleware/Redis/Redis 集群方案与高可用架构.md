---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - database
  - redis
  - cluster
  - sentinel
  - replication
  - high-availability
  - distributed-system
  - middleware
aliases:
  - Redis Cluster
  - Redis Sentinel
  - Redis HA
  - Redis 分布式
  - Redis 主从复制
source_type: official-doc
source_urls:
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/replication/'
status: verified
---
## 概述

Redis 提供四种部署架构以满足不同场景的可用性和扩展性需求：

| 方案 | 数据分片 | 高可用 | 适用场景 | 复杂度 |
|------|----------|--------|----------|--------|
| **单机** | 无 | 无 | 开发测试、小规模缓存 | 低 |
| **主从复制** | 无 | 手动故障转移 | 读多写少、需数据冗余 | 低 |
| **Sentinel** | 无 | 自动故障转移 | 中小规模生产、单机数据量 | 中 |
| **Cluster** | 有（16384 Slot） | 自动故障转移 + 自动分片 | 大规模数据、高并发、需水平扩展 | 高 |

---

## 一、Redis Cluster

Redis Cluster 是 Redis 官方的分布式实现，提供**数据自动分片**和**高可用**能力。

### 核心架构

**Hash Slot 分片机制**[^1]：
- 数据空间划分为 **16384 个 Hash Slot**
- 每个 Key 通过 `CRC16(key) mod 16384` 映射到唯一 Slot
- 每个 Master 节点负责一部分 Slot
- **不使用一致性哈希**，而是固定 Slot 数量

**主从高可用模型**[^1]：
- 每个 Slot 有 1 个 Master + N 个 Replica
- Master 故障时，Replica 自动提升为新 Master
- 若 Master 和所有 Replica 同时故障，对应 Slot 不可用

**Gossip 协议**[^2]：
- 节点间通过 Cluster Bus（端口 = 数据端口 + 10000）通信
- 使用二进制协议交换配置信息、故障检测、Failover 授权
- 全 mesh 拓扑，但通过 gossip 避免消息爆炸

### 一致性与可用性

**不保证强一致性**[^1]：
- 使用异步复制，Master 回复客户端后才传播到 Replica
- 网络分区时，少数派 Master 可能丢失写入数据
- 可通过 `WAIT` 命令实现同步写入（但仍非强一致）

**可用性保证**[^2]：
- 多数派 Master 可达 + 每个 Master 至少有 1 个可达 Replica 时，集群可用
- `NODE_TIMEOUT`（默认 15000ms）后少数派停止接受写入
- Replica Migration 可提升可用性（无 Replica 的 Master 会从其他 Master 获得多余 Replica）

### 关键配置参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `cluster-enabled yes` | 启用 Cluster 模式 | 必须 |
| `cluster-config-file` | 集群配置持久化文件 | 自动生成 |
| `cluster-node-timeout` | 节点不可达判定时间 | 5000-15000 ms |
| `cluster-require-full-coverage` | 部分 Slot 不可用时是否继续服务 | 生产环境建议 `no` |
| `cluster-allow-reads-when-down` | 集群 FAIL 时是否允许读 | 按需求配置 |
| `cluster-slave-validity-factor` | Replica 断连多久后不参与 Failover | 0（始终尝试）或 > 0 |

### Hash Tag（多键操作）

强制多个 Key 映射到同一 Slot[^2]：

```
{user:1000}:profile
{user:1000}:account
# 以上两个 Key 使用 user:1000 计算 Slot，可在同一节点操作
```

规则：Key 中 `{` 和 `}` 之间的内容用于计算 Slot（需满足 `{` 在 `}` 左侧且中间有内容）。

### 客户端重定向

- **MOVED**：Slot 已永久迁移到新节点，客户端应更新本地 Slot 映射表
- **ASK**：Slot 正在迁移中，仅当前请求转发到目标节点
- 客户端应缓存 Slot 映射表，定期通过 `CLUSTER SLOTS` 或 `CLUSTER SHARDS` 更新

### 部署要求

- **最少 3 个 Master**（官方强烈推荐 6 节点：3 Master + 3 Replica）[^1]
- 所有节点间需开放：数据端口 + Cluster Bus 端口（数据端口 + 10000）
- Docker 需使用 `--net=host` 模式（不支持 NAT/端口映射）[^1]

---

## 二、Redis Sentinel

Sentinel 为非 Cluster 的 Redis 提供**高可用**，适用于单机数据量、需自动故障转移的场景。

### 核心能力[^3]

| 功能 | 说明 |
|------|------|
| **监控** | 持续检查 Master/Replica 是否正常运行 |
| **通知** | 通过 API 或 Pub/Sub 通知系统管理员或其他程序 |
| **自动故障转移** | Master 不可用时，提升 Replica 为新 Master |
| **配置提供** | 为客户端提供当前 Master 地址（服务发现） |

### 部署要求[^3]

- **至少 3 个 Sentinel 进程**（奇数部署）
- Sentinel 应分布在不同物理机/可用区（独立故障）
- Sentinel + Redis 不保证零写入丢失（异步复制）
- 客户端需支持 Sentinel（主流客户端库均支持）

### 配置示例

```conf
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

参数说明[^3]：

| 参数 | 说明 |
|------|------|
| `quorum` | 同意 Master 下线的 Sentinel 数量（仅用于故障检测，实际 Failover 需多数派授权） |
| `down-after-milliseconds` | 实例无响应多久后标记为下线 |
| `failover-timeout` | Failover 超时时间 |
| `parallel-syncs` | 同时向新 Master 同步的 Replica 数量（越小越稳但恢复越慢） |

### 防止脑裂

配置 Master 在 Replica 数量不足时拒绝写入[^3]：

```conf
min-replicas-to-write 1
min-replicas-max-lag 10
```

当 Master 无法写入到至少 1 个 Replica 且延迟 > 10 秒时，停止接受写入。

### Sentinel 常用命令

| 命令 | 说明 |
|------|------|
| `SENTINEL masters` | 列出监控的所有 Master |
| `SENTINEL master <name>` | 查看指定 Master 状态 |
| `SENTINEL get-master-addr-by-name <name>` | 获取当前 Master 地址 |
| `SENTINEL replicas <name>` | 列出指定 Master 的所有 Replica |
| `SENTINEL reset <pattern>` | 重置指定 Master（清除状态，用于清理旧 Replica） |

---

## 三、主从复制

基础的复制方案，提供数据冗余和读扩展，无自动故障转移。

### 复制机制[^4]

- **异步复制**：Master 写入后立即返回，再传播到 Replica
- Replica 连接 Master 后发送 `PSYNC`，Master 执行部分重同步（基于 Replication Backlog）或完整重同步
- Replica 默认只读，可配置 `replica-read-only no` 允许写入（危险）

### 关键配置

```conf
replicaof 192.168.1.100 6379    # 指定 Master（Redis 5.0+ 用 REPLICAOF）
repl-backlog-size 64mb          # 复制缓冲区大小
repl-diskless-sync no           # 是否无盘同步（大数据量时推荐 yes）
repl-timeout 60                 # 复制超时
```

### 复制延迟问题

- Replica 数据落后于 Master（异步复制）
- 高负载、网络带宽不足、大 Key 传输会加剧延迟
- 监控 `INFO replication` 中 `master_repl_offset - slave_repl_offset` 差值

---

## 四、方案对比与选型建议

### 功能对比

| 特性 | 主从复制 | Sentinel | Cluster |
|------|----------|----------|---------|
| 数据分片 | ❌ | ❌ | ✅（16384 Slot） |
| 高可用 | ❌（手动） | ✅（自动） | ✅（自动） |
| 写入扩展 | ❌（单 Master） | ❌（单 Master） | ✅（多 Master） |
| 读扩展 | ✅（多 Replica） | ✅（多 Replica） | ✅（多 Replica） |
| 多键操作限制 | 无 | 无 | 需 Hash Tag |
| 客户端复杂度 | 低 | 中（需 Sentinel 支持） | 高（需重定向处理） |
| 运维复杂度 | 低 | 中 | 高 |

### 选型决策树

```
数据量 > 单机内存？
├─ 是 → 需要分片 → Redis Cluster
└─ 否 → 数据量可控
    └─ 需要自动故障转移？
        ├─ 是 → Redis Sentinel
        └─ 否 → 主从复制或单机
```

### 典型场景推荐

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 开发/测试环境 | 单机 | 简单，无需高可用 |
| 小规模缓存（< 10GB） | Sentinel | 数据量小，需 HA，运维简单 |
| 中等规模业务（10-50GB） | Sentinel + 多 Replica | 单机内存足够，读扩展 |
| 大规模数据（> 50GB） | Cluster | 需分片 + HA |
| 高写入吞吐（> 10万 QPS） | Cluster | 多 Master 分担写压力 |
| 需强多键事务 | Sentinel + 客户端事务 | Cluster 多键受限 |
| 已有 Redis 实例，逐步迁移 | Cluster + 迁移工具 | 可无缝迁移数据 |

---

## 五、最佳实践

### Cluster 部署建议

- 每个物理机/VM 上部署 1 个节点（避免单机故障影响多个节点）
- 跨可用区部署（提升灾难恢复能力）
- 使用智能客户端（缓存 Slot 映射，减少重定向）
- 禁用大 Key 操作（迁移阻塞）
- 配置 `cluster-require-full-coverage no`（部分故障时继续服务）

### Sentinel 部署建议[^3]

- 3 个 Sentinel 分布在 3 个独立物理机/VM
- `quorum` 设为 Sentinel 总数 / 2 + 1
- `down-after-milliseconds` 不宜过短（5000-30000 ms）
- 配置 `min-replicas-to-write` 防止脑裂
- 定期测试 Failover（避免配置错误）

### 监控关键指标

| 指标 | 来源 | 告警阈值建议 |
|------|------|--------------|
| Cluster 节点状态 | `CLUSTER NODES` | 任一节点 `FAIL` |
| Slot 覆盖完整性 | `CLUSTER INFO` | `cluster_slots_assigned < 16384` |
| 复制偏移差 | `INFO replication` | offset 差 > 10 MB 持续 1 分钟 |
| Sentinel Master 状态 | `SENTINEL master` | `flags` 含 `s_down` 或 `o_down` |
| Failover 历史 | Sentinel 日志 | 异常频繁 Failover |

---

## 六、迁移策略

### 从单机迁移到 Sentinel

1. 部署 Replica 并等待同步完成
2. 部署 3+ Sentinel 监控 Master
3. 客户端切换为 Sentinel 模式获取 Master 地址

### 从单机/Sentinel 迁移到 Cluster

1. 启动 Cluster 节点（空数据）
2. 使用 `redis-cli --cluster import` 导入数据
3. 或使用 `CLUSTER SETSLOT` + `MIGRATE` 手动迁移
4. 客户端切换为 Cluster 模式

---

## 参考资料

[^1]: Redis Cluster 官方文档 — [Scale with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
[^2]: Redis Cluster 规范 — [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
[^3]: Redis Sentinel 官方文档 — [High availability with Redis Sentinel](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
[^4]: Redis 复制文档 — [Replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
