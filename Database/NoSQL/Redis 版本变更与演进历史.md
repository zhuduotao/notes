---
created: 2026-04-16
updated: 2026-04-16
tags:
  - database
  - redis
  - version-history
  - licensing
  - changelog
aliases:
  - Redis 版本历史
  - Redis 演进
  - Redis 许可证变更
  - Redis version history
  - Redis changelog
source_type: mixed
source_urls:
  - https://github.com/redis/redis/releases
  - https://redis.io/blog/redis-adopts-dual-source-available-licensing/
  - https://redis.io/blog/announcing-redis-86-performance-improvements-streams/
  - https://redis.io/release/
status: verified
---

## 概述

Redis（Remote Dictionary Server）是一个开源的内存数据结构存储，广泛用作数据库、缓存、消息代理和流处理引擎。其版本演进可分为两个关键阶段：**BSD 许可时期**（7.2 及之前）和**双许可时期**（7.4 及之后），许可证变更是 Redis 历史上最重要的分水岭。

## 许可证变更：从 BSD 到双许可

### 变更时间

**2024 年 3 月 20 日**，Redis Inc. 宣布从 Redis 7.4 开始，将许可证从 **BSD 3-Clause** 变更为双许可：

- **RSALv2**（Redis Source Available License v2）— 非 copyleft，允许使用、复制、分发和制作衍生作品，但限制两点：
  1. 不得将软件商业化或作为托管服务提供给他人
  2. 不得移除或模糊任何许可、版权或其他声明
- **SSPLv1**（Server Side Public License v1）— 基于 AGPL 的 copyleft 许可，如果将产品作为服务提供给第三方，必须公开整个服务的源代码（包括管理层、UI、API、自动化、监控等）

### 变更原因

- Redis Inc. 指出大部分 Redis 商业销售通过大型云服务提供商进行，这些提供商将 Redis 的投资和开源社区成果商品化
- 同时维护开源、source-available 和商业多个发行版与推动 Redis 未来发展存在矛盾
- 新许可证下，云服务提供商不能再免费使用 Redis 源代码提供托管服务

### 影响范围

| 群体 | 影响 |
|------|------|
| 个人/内部使用开发者 | **无影响**，可继续免费使用 |
| 客户端库（client libraries） | **无影响**，继续保持开源许可 |
| Redis 商业客户 | **无影响**，按协商条款获取 |
| 云服务提供商 | **受影响**，需与 Redis 达成许可协议才能提供 7.4+ 版本 |
| 竞争对手 | **受影响**，不得免费使用新源代码构建竞争产品 |

### 重要说明

- **不追溯**：7.2 及之前版本仍保持 BSD 3-Clause 许可，可永久使用
- Redis Inc. 承诺在 Redis Community Edition 9.0 发布前继续为旧版本回补关键安全补丁
- Redis 明确承认变更后**不再符合 OSI 定义的开源标准**，改称"Community Edition"
- Redis Stack 在 Redis 8 发布后终止（EOL），其模块能力并入 Redis 核心

## Valkey 分支项目

作为对许可证变更的响应，Linux Foundation 联合 AWS、Google Cloud、Oracle、Ericsson 等公司于 2024 年 3 月发起 **Valkey** 项目：

- 基于 Redis 7.2.4（最后一个 BSD 许可版本）的 fork
- 继续保持 **BSD 3-Clause** 许可
- 由 Linux Foundation 管理，避免单一厂商控制
- 目标：提供真正的开源替代方案，保持与 Redis 协议的兼容性
- 官网：https://valkey.io/

## 主要版本特性演进

### Redis 6.x 系列

| 版本 | 关键特性 |
|------|----------|
| 6.0 | 多线程 I/O（可选）、ACL 访问控制、RESP3 协议、客户端缓存（Client-side caching） |
| 6.2 | Stream 增强（`XAUTOCLAIM`）、函数式编程能力预研、`ZINTER`/`ZUNION` 命令 |

### Redis 7.x 系列（BSD 许可最后阶段）

| 版本 | 发布时间 | 关键特性 |
|------|----------|----------|
| 7.0 | 2022 年 4 月 | Functions（Lua 脚本替代）、Multi-part keys、ACL 增强、新的 RDB 格式 |
| 7.2 | 2023 年 8 月 | 最后一个 BSD 许可版本；JSON 模块预集成、性能优化 |
| **7.4** | **2024 年 3 月** | **首个双许可版本**（RSALv2 / SSPLv1）；Redis Stack 模块开始并入核心 |

### Redis 8.x 系列（双许可 + 模块整合）

Redis 8 标志着 Redis Stack 的终结——搜索、JSON、向量、概率和时间序列等数据模型统一并入 Redis 核心。

#### Redis 8.0

- JSON 数据类型的原生支持（`JSON.GET`、`JSON.SET` 等）
- 向量数据库能力增强（Vector Set 数据类型）
- 搜索与查询引擎整合
- 时间序列模块并入核心
- 概率数据结构（Bloom filter、Cuckoo filter、HyperLogLog 增强）

#### Redis 8.2

- Streams 增强：跨多个消费组的消息确认和删除简化
- Bitmap 能力改进
- 集群槽位统计：`CLUSTER SLOT-STATS` 命令，用于检测热点槽位

#### Redis 8.4

- 原子槽位迁移（Atomic Slot Migration）：安全高效地迁移集群槽位以重新平衡负载
- Streams 消费者改进：更容易读取新消息和空闲待处理消息
- 大量 RediSearch 模块修复和性能优化

#### Redis 8.6（2026 年 2 月 10 日 GA）

当前最新 GA 版本，重点在性能和资源利用率提升：

**性能提升（对比 Redis 7.2）**
- 吞吐量提升超过 **5 倍**（单节点 16 核 ARM Graviton4，1:10 SET:GET 比例）
- Pipeline size 为 16 时可达 **3.5M ops/sec**

**延迟降低（对比 Redis 8.4）**
- 有序集合命令延迟降低最高 **35%**
- 短字符串 GET 命令延迟降低最高 **15%**
- 列表命令延迟降低最高 **11%**
- 哈希命令延迟降低最高 **7%**

**内存占用降低（对比 Redis 8.4）**
- 哈希（hashtable 编码）内存占用降低最高 **16.7%**
- 有序集合（skiplist 编码）内存占用降低最高 **30.5%**

**新功能**

| 功能 | 说明 |
|------|------|
| **Streams 幂等生产** | `XADD` 支持 `IDMP pid iid` 和 `IDMPAUTO pid` 参数，提供 at-most-once 投递保证，防止网络重试导致消息重复 |
| **新驱逐策略** | `volatile-lrm` 和 `allkeys-lrm`（Least Recently Modified），仅写操作刷新键的活跃度，适合缓存响应和聚合数据场景 |
| **热点键检测** | `HOTKEYS` 命令，支持按 CPU 和网络带宽维度检测 top-k 热点键，可限制到特定槽位 |
| **TLS 证书自动认证** | `tls-auth-clients-user CN` 配置，基于 mTLS 证书 Common Name 自动映射到 ACL 用户，无需再执行 `AUTH` |
| **时间序列 NaN 支持** | `TS.ADD` 和 `TS.MADD` 支持 NaN 值标记不可用数据；新增 `countNaN` 和 `countAll` 聚合器 |

**已知限制**
- Streams `IDMP`/`IDMPAUTO` 在使用 `appendonly yes` + `aof-use-rdb-preamble no`（非默认配置）时存在已知 bug，将在后续补丁修复

### 最新补丁版本

| 版本 | 发布日期 | 类型 | 关键修复 |
|------|----------|------|----------|
| 8.6.2 | 2026-03-24 | Bug 修复 | Streams IDMP 相关 bug、潜在 UAF 修复、内存泄漏修复 |
| 8.4.2 | 2026-02-23 | 安全修复 | 错误回复中 `\r\n` 序列注入漏洞 |
| 8.2.5 | 2026-02-23 | 安全修复 | 同上 |
| 8.0.6 | 2026-02-23 | 安全修复 | 同上 |
| 7.4.8 | 2026-02-23 | 安全修复 | 同上 |
| 7.2.13 | 2026-02-23 | 安全修复 | 同上 |

## 版本选择建议

| 场景 | 推荐版本 | 理由 |
|------|----------|------|
| 需要真正开源（BSD） | **Valkey**（基于 Redis 7.2.4 fork）或 **Redis 7.2.x** | 7.4+ 不再是 OSI 定义的开源许可 |
| 云托管服务 | 需与 Redis Inc. 达成商业许可，或使用 Valkey | RSALv2/SSPLv1 限制云托管竞争产品 |
| 内部使用/个人项目 | **Redis 8.6+**（最新） | 性能最佳、功能最全，内部使用不受许可证限制 |
| 需要 JSON/向量/搜索 | **Redis 8.0+** | 模块已并入核心，无需单独安装 Redis Stack |
| 生产环境稳定性优先 | **8.4.x 或 8.6.x**（最新补丁） | 经过充分验证的 GA 版本，及时修复安全问题 |

## 版本命名规则

Redis 采用语义化版本控制（SemVer）：

- **主版本号**（如 7 → 8）：不兼容的 API 变更或重大架构调整
- **次版本号**（如 8.4 → 8.6）：向后兼容的功能新增
- **补丁版本号**（如 8.6.0 → 8.6.2）：向后兼容的 bug 修复和安全补丁

## 相关概念

- **Redis Stack**：曾将 Redis 核心与 Search、JSON、TimeSeries、Bloom、Graph 等模块打包分发，Redis 8 发布后 EOL
- **Redis Community Edition**：Redis 7.4+ 的官方称呼，替代"Redis OSS"说法
- **Valkey**：Linux Foundation 管理的 BSD 许可 fork，Redis 许可证变更后的开源替代
- **RESP3**：Redis Serialization Protocol v3，Redis 6.0 引入的新协议版本

## 参考资料

- [Redis 采用双 source-available 许可公告](https://redis.io/blog/redis-adopts-dual-source-available-licensing/)（2024-03-20）
- [Redis 8.6 发布公告](https://redis.io/blog/announcing-redis-86-performance-improvements-streams/)（2026-02-10）
- [Redis GitHub Releases](https://github.com/redis/redis/releases)
- [Redis 官方 Release 页面](https://redis.io/release/)
- [RSALv2 许可全文](https://redis.io/legal/rsalv2-agreement/)
- [SSPLv1 许可全文](https://redis.io/legal/server-side-public-license-sspl/)
- [Valkey 官网](https://valkey.io/)
