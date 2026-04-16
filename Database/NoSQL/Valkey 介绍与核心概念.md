---
created: 2026-04-16
updated: 2026-04-16
tags:
  - database
  - valkey
  - redis
  - nosql
  - in-memory
  - key-value
  - cache
  - linux-foundation
  - open-source
  - bsd
aliases:
  - Valkey 介绍
  - Valkey 是什么
  - Valkey vs Redis
  - Valkey introduction
  - What is Valkey
source_type: official-doc
source_urls:
  - https://valkey.io/
  - https://valkey.io/topics/history
  - https://valkey.io/topics/quickstart
  - https://valkey.io/topics/faq
  - https://valkey.io/blog/introducing-valkey-9/
  - https://github.com/valkey-io/valkey
  - https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community
status: verified
---

## 是什么

Valkey 是一个**高性能、开源的内存键值数据存储**，由 Linux Foundation 管理，采用 **BSD 3-Clause** 许可，确保项目永久开源。它支持丰富的原生数据结构（字符串、列表、集合、哈希、有序集合、位图、HyperLogLog 等），并提供 Lua 脚本和模块扩展能力，可用于缓存、消息队列、会话管理、实时分析等多种场景。

Valkey 是 Redis 7.2.4（最后一个 BSD 许可版本）的 fork，诞生于 2024 年 3 月 Redis Ltd. 将许可证变更为双 source-available 许可（RSALv2/SSPLv1）之后[^1]。

## 为什么重要

| 维度 | 说明 |
|------|------|
| **永久开源** | 由 Linux Foundation 管理，避免单一厂商控制，BSD 许可确保任何用途（包括商业托管）都不受限制 |
| **Redis 兼容** | 与 Redis 协议完全兼容，现有 Redis 客户端库、工具、运维经验可无缝迁移 |
| **性能保证** | 亚微秒级延迟，单节点可处理数十万至数百万 ops/sec，持续的性能优化迭代 |
| **生态健康** | AWS、Google Cloud、Oracle、Alibaba、ByteDance、Ericsson、Percona 等数十家企业参与贡献和托管支持 |
| **独立演进** | 不再受 Redis Ltd. 商业决策约束，按社区需求独立发布新功能和版本 |

## 诞生背景

### 时间线

| 时间 | 事件 |
|------|------|
| 2009-02 | Salvatore "antirez" Sanfilippo 创建 Redis 项目 |
| 2010-03 | antirez 被 VMware 雇佣全职开发 Redis |
| 2015 | antirez 转投 Redis Labs（原 Garantia Data），继续开源开发 |
| 2018 | Redis Labs 将部分模块许可证从 AGPL 变更为 source-available，引发社区争议 |
| 2020 | antirez 宣布退出 Redis 维护，交接给 Redis Labs 核心团队 |
| **2024-03-20** | **Redis Ltd. 宣布 7.4+ 版本采用双许可（RSALv2/SSPLv1），不再符合 OSI 开源定义**[^2] |
| **2024-03-28** | **Linux Foundation 联合 6 家创始公司（Alibaba、Amazon、Ericsson、Google、Huawei、Tencent）宣布 Valkey 项目**[^3] |
| 2024-04 | Valkey 7.2.5 RC 发布，随后 GA |
| 2025-10 | Valkey 9.0 发布，引入原子槽位迁移、Hash 字段过期、集群模式编号数据库等重大特性 |
| 2026-02 | Valkey 9.0.3 发布（当前最新稳定版） |

### 许可证变更详情

Redis Ltd. 从 7.4 开始采用的双许可：

- **RSALv2**（Redis Source Available License v2）：禁止将软件作为数据库产品或托管服务提供给第三方
- **SSPLv1**（Server Side Public License v1）：如果将产品作为服务提供，必须公开整个服务的源代码

这两种许可均**不满足 OSI 开源定义**（限制特定领域的使用），因此 Redis 7.4+ 不再是严格意义上的"开源软件"，改称"Community Edition"[^2]。

Valkey 则继续保持 **BSD 3-Clause** 许可，完全符合 OSI 开源定义。

## 核心架构

### 运行模式

| 模式 | 说明 |
|------|------|
| **Standalone** | 单实例运行，适合开发测试或小型部署 |
| **主从复制** | 一主多从，从节点提供读扩展和故障恢复 |
| **Sentinel** | 自动故障转移和监控，保障高可用 |
| **Cluster** | 分布式分片，16384 个哈希槽自动分配到多个节点，支持水平扩展 |

### 持久化机制

| 方式 | 说明 | 特点 |
|------|------|------|
| **RDB**（Redis Database） | 定期生成内存快照 | 紧凑、恢复快，但可能丢失最后一次快照后的数据 |
| **AOF**（Append Only File） | 记录每个写操作 | 数据安全性更高，文件体积较大 |
| **RDB + AOF** | 同时启用 | 兼顾恢复速度和数据安全，推荐生产使用 |

### 内存管理

- **默认分配器**：Linux 使用 jemalloc（碎片更少），其他平台使用 libc malloc
- **内存上限**：通过 `maxmemory` 配置，达到上限后可选择拒绝写入或启用驱逐策略
- **驱逐策略**：`volatile-lru`、`allkeys-lru`、`volatile-ttl`、`noeviction` 等
- **单实例上限**：理论上可处理 2^32 个键，实测至少 2.5 亿键/实例[^4]

## 数据结构

| 数据类型 | 说明 | 典型用途 |
|----------|------|----------|
| **String** | 二进制安全的字符串（最大 512 MB） | 缓存、计数器、会话存储 |
| **Hash** | 字段-值对的集合 | 存储对象/记录 |
| **List** | 有序字符串列表 | 消息队列、最近活动列表 |
| **Set** | 无序唯一元素集合 | 标签、好友关系、去重 |
| **Sorted Set** | 带分数的有序唯一集合 | 排行榜、时间序列 |
| **Bitmap** | 位级操作 | 用户签到、布隆过滤器预计算 |
| **HyperLogLog** | 基数估算 | UV 统计、去重计数 |
| **Stream** | 持久化消息流 | 消息队列、事件溯源 |
| **Geospatial** | 地理位置索引 | 附近的人、地理围栏 |

## 版本演进

### 当前维护的分支

| 分支 | 最新版本 | 发布日期 | 说明 |
|------|----------|----------|------|
| **9.x** | 9.0.3 | 2026-02-24 | 最新稳定版，引入原子槽位迁移、Hash 字段过期等 |
| **8.x** | 8.1.6 | 2026-02-24 | 持续维护分支 |
| **7.x** | 7.2.12 | 2026-02-24 | 基于 Redis 7.2.4 的初始分支，持续安全修复 |

### Valkey 9.0 关键特性

| 特性 | 说明 |
|------|------|
| **原子槽位迁移** | 整个槽位一次性原子迁移，替代逐键迁移，避免迁移期间的客户端重定向和数据不一致 |
| **Hash 字段过期** | 新增 `HEXPIRE`、`HPERSIST` 等命令，支持 Hash 内单个字段的独立过期时间 |
| **集群模式编号数据库** | 打破此前集群只能使用 db 0 的限制，支持完整的编号数据库 |
| **大规模集群** | 支持扩展至 2000 节点，实现超过 10 亿 requests/秒 |
| **Pipeline 内存预取** | 吞吐量提升最高 40% |
| **零拷贝响应** | 大请求避免内部内存复制，吞吐量提升最高 20% |
| **Multipath TCP** | 支持 MPTC，延迟降低 25% |
| **SIMD 优化** | BITCOUNT 和 HyperLogLog 吞吐量提升最高 200% |
| **条件删除** | 新增 `DELIFEQ` 命令 |
| **CLIENT LIST 过滤** | 支持按 flags、name、idle、database、IP 等过滤 |

### 与 Redis 的关系

| 维度 | Valkey | Redis 7.2 及之前 | Redis 7.4 及之后 |
|------|--------|-----------------|-----------------|
| 许可证 | **BSD 3-Clause**（OSI 认证开源） | BSD 3-Clause | RSALv2 / SSPLv1（source-available） |
| 管理方 | Linux Foundation | 社区 + Redis Ltd. | Redis Ltd. |
| 云托管限制 | 无 | 无 | 受限（需商业许可） |
| 协议兼容 | 完全兼容 Redis 协议 | — | — |
| 客户端兼容 | 现有 Redis 客户端可直接使用 | — | — |
| 版本起点 | 基于 Redis 7.2.4 fork | — | — |
| 独立演进 | 是（9.0 已超越 Redis 7.2 的能力） | — | — |

## 快速开始

### Docker 运行

```bash
docker run --rm valkey/valkey:9.0.3
```

### 源码编译

```bash
git clone https://github.com/valkey-io/valkey.git
cd valkey
make
make test
```

### 基本操作

```bash
cd src
./valkey-server          # 启动服务
./valkey-cli             # 启动客户端
```

```
valkey> SET user:1000 "Alice"
OK
valkey> GET user:1000
"Alice"
valkey> HSET user:1000 name "Alice" email "alice@example.com"
(integer) 2
valkey> HGETALL user:1000
1) "name"
2) "Alice"
3) "email"
4) "alice@example.com"
```

### 安装兼容性

`make install` 会创建 Redis 兼容的符号链接（`redis-server` → `valkey-server`、`redis-cli` → `valkey-cli` 等），确保现有脚本和工具无需修改即可工作[^5]。

## 适用场景

| 场景 | 说明 |
|------|------|
| **缓存层** | 数据库查询结果、API 响应、渲染页面的缓存，TTL 自动过期 |
| **会话存储** | Web 应用的用户会话管理，支持分布式部署 |
| **消息队列** | 使用 List（`LPUSH`/`BRPOP`）或 Stream 实现消息传递 |
| **排行榜** | Sorted Set 的分数排序能力天然适合排行榜场景 |
| **实时分析** | HyperLogLog 基数估算、Bitmap 位操作、Pub/Sub 实时推送 |
| **地理服务** | Geospatial 索引支持附近搜索、地理围栏 |
| **AI/ML 加速** | 缓存模型输出、预计算嵌入向量、管理访问令牌 |
| **主数据库** | 对持久化有要求的场景可启用 RDB + AOF 作为主存储 |

## 限制与注意事项

| 限制 | 说明 |
|------|------|
| **内存容量** | 数据集必须能装入内存，不支持虚拟内存或磁盘后端存储[^4] |
| **大键问题** | 单个大集合（如百万元素的 Sorted Set）操作可能阻塞主线程，建议拆分或使用模块 |
| **单线程模型** | 命令执行是单线程的（I/O 可多线程卸载），CPU 密集型操作会阻塞其他命令 |
| **事务非回滚** | `MULTI`/`EXEC` 事务中语法错误不会回滚已执行命令 |
| **编号数据库不推荐** | 官方建议不使用 `SELECT` 切换数据库，应使用命名空间前缀（如 `user:1000:settings`）[^4] |
| **集群模式限制** | 多键操作要求所有键在同一哈希槽（使用 hash tag `{user:1000}` 可强制同槽） |

## 常见误区

| 误区 | 事实 |
|------|------|
| "Valkey 只是 Redis 的改名" | Valkey 是独立 fork，已有独立版本演进（9.0 引入大量 Redis 7.2 不具备的特性） |
| "Valkey 不兼容 Redis 客户端" | 完全兼容 Redis 协议，现有客户端、工具、脚本可直接使用 |
| "Valkey 功能落后于 Redis" | Valkey 9.0 已引入原子槽位迁移、Hash 字段过期等特性，在部分领域超越 Redis 7.2 |
| "开源 = 没有商业支持" | Percona、AWS、Google Cloud、Oracle、Aiven 等提供托管和支持服务 |
| "Valkey 会分裂 Redis 生态" | Valkey 保留了 BSD 许可的 Redis 7.2 分支，是许可证变更后的延续而非分裂 |

## 生态参与者

Valkey 的参与者不仅是供应商，而是致力于维护项目长期健康的贡献者[^6]：

- **云服务**：AWS（ElastiCache）、Google Cloud（Memorystore）、Oracle Cloud（OCI Cache）、Alibaba Cloud
- **托管服务**：Percona、Aiven、UpCloud、DigitalOcean、Heroku、Momento、NetApp Instaclustr
- **技术公司**：ByteDance、Ericsson、Huawei、Tencent
- **工具平台**：BetterDB（Valkey 专属监控）、k0rdent（Kubernetes 服务模板）

## 相关概念

- **Redis**：Valkey 的前身项目，7.2 及之前版本为 BSD 许可，7.4+ 变更为双 source-available 许可
- **RESP**（Redis Serialization Protocol）：Valkey 使用的客户端-服务器通信协议
- **RDB/AOF**：Valkey 的两种持久化格式
- **Hash Slot**：Valkey Cluster 使用 16384 个哈希槽进行数据分片
- **Linux Foundation**：Valkey 的托管组织，确保项目治理的透明和中立

## 参考资料

- [Valkey 官网](https://valkey.io/)
- [Valkey 历史](https://valkey.io/topics/history)（官方文档）
- [Valkey 快速开始](https://valkey.io/topics/quickstart)（官方文档）
- [Valkey FAQ](https://valkey.io/topics/faq)（官方文档）
- [Valkey 9.0 发布公告](https://valkey.io/blog/introducing-valkey-9/)（2025-10-21）
- [Valkey GitHub 仓库](https://github.com/valkey-io/valkey)
- [Linux Foundation Valkey 发布新闻稿](https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community)（2024-03-28）
- [Redis 许可证变更公告](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)（2024-03-20）[^2]

[^1]: Valkey 基于 Redis 7.2.4 fork，这是 Redis 采用 BSD 3-Clause 许可的最后一个版本。详见 [Valkey 历史](https://valkey.io/topics/history)。
[^2]: Redis Ltd. 于 2024-03-20 宣布从 7.4 开始采用 RSALv2/SSPLv1 双许可。详见 [Redis 许可证变更公告](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)。
[^3]: Linux Foundation 于 2024-03-28 发布 Valkey 项目，距 Redis 许可证变更仅 8 天。详见 [Linux Foundation 新闻稿](https://www.linuxfoundation.org/press/linux-foundation-launches-open-source-valkey-community)。
[^4]: 详见 [Valkey FAQ](https://valkey.io/topics/faq)。
[^5]: 详见 [Valkey GitHub README](https://github.com/valkey-io/valkey)。
[^6]: 详见 [Valkey Participants 页面](https://valkey.io/participants/)。
