---
created: 2026-04-16
updated: 2026-04-16
tags:
  - observability
  - tracing
  - skywalking
  - kafka
  - clickhouse
  - grafana
  - apm
  - distributed-systems
aliases:
  - 全链路追踪方案
  - SkyWalking tracing architecture
  - 分布式链路追踪
  - APM tracing pipeline
source_type: mixed
source_urls:
  - https://skywalking.apache.org/docs/main/latest/en/concepts-and-designs/overview/
  - https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-storage/
  - https://skywalking.apache.org/docs/main/latest/en/setup/backend/kafka-fetcher/
  - https://skywalking.apache.org/docs/main/latest/en/setup/backend/ui-grafana/
  - https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-clickhouse-monitoring/
  - https://skywalking.apache.org/docs/main/latest/en/setup/backend/otlp-trace/
status: verified
---

## 是什么

全链路追踪（Distributed Tracing）是一种用于监控和诊断分布式系统请求流转路径的技术。通过记录请求在各个服务间的调用关系、耗时和状态，帮助开发者快速定位性能瓶颈和故障根因。

本方案以 **Apache SkyWalking** 为核心 APM 平台，结合 **Kafka**（数据采集与缓冲传输）、**ClickHouse**（高性能列式存储与分析）、**Grafana**（可视化展示），构建一套完整的全链路可观测性解决方案。

## 为什么重要

在微服务和云原生架构中，一个用户请求通常会经过多个服务的协同处理。传统单体监控无法回答以下问题：

- 请求经过了哪些服务？各服务的耗时分布如何？
- 哪个服务是性能瓶颈？
- 服务间的调用依赖关系是什么？
- 故障发生时，如何快速定位根因？

全链路追踪通过 **Trace → Span → Segment** 的数据模型，将请求的完整生命周期串联起来，提供端到端的可观测能力。

## 方案架构

SkyWalking 逻辑上分为四个部分：Probes（探针）、Platform Backend（平台后端）、Storage（存储）和 UI（界面）[^1]。

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌─────────────┐
│   Probes    │────▶│   Backend    │────▶│  Storage    │────▶│     UI      │
│  (Agents)   │     │   (OAP)      │     │(ClickHouse) │     │  (Grafana)  │
└─────────────┘     └──────────────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    │   Kafka     │
                    │ (Fetcher)   │
                    └─────────────┘
```

### 各组件职责

| 组件 | 职责 | 说明 |
|------|------|------|
| **SkyWalking Agent** | 数据采集 | 自动插桩探针，支持 Java、Go、Python、Node.js 等语言，无侵入式采集 Trace、Metrics、Log |
| **Kafka** | 数据传输与缓冲 | 作为消息队列解耦 Agent 与 OAP，提供高吞吐、持久化、可回溯的数据管道 |
| **SkyWalking OAP** | 数据处理与分析 | 接收、聚合、分析追踪数据，执行 OAL（观测分析语言）脚本生成聚合指标 |
| **ClickHouse** | 数据存储 | 列式存储引擎，适合大规模 Trace/Log 数据的高效写入和查询 |
| **Grafana** | 数据可视化 | 通过 PromQL/LogQL/SkyWalking 数据源插件展示指标、日志和拓扑图 |

## 核心概念

### SkyWalking 数据模型

- **Trace（追踪）**：一次完整的请求链路，由多个 Span 组成
- **Span（跨度）**：链路中的一个操作单元，包含开始时间、结束时间、操作名称、标签等
- **Segment（段）**：SkyWalking 特有的概念，表示单个服务实例内的一组 Span
- **Service（服务）**：提供相同行为的一组工作负载
- **Service Instance（服务实例）**：服务组中的单个工作负载（如一个 Pod 或进程）
- **Endpoint（端点）**：服务中处理请求的路径（如 HTTP URI 或 gRPC 方法）
- **Layer（层）**：抽象的计算框架分类，如 GENERAL、MESH、K8S 等[^1]

### 数据流转过程

1. **采集阶段**：SkyWalking Agent 通过字节码增强（Java）或 SDK 集成自动采集 Trace 数据
2. **传输阶段**：Agent 将数据发送到 Kafka Topic（或直接通过 gRPC/HTTP 发送到 OAP）
3. **处理阶段**：OAP 通过 Kafka Fetcher 消费数据，执行聚合分析
4. **存储阶段**：处理后的数据写入 ClickHouse（或其他存储后端）
5. **展示阶段**：Grafana 通过数据源插件查询 SkyWalking OAP 的 PromQL/GraphQL 接口进行可视化

## 关键配置

### Kafka Fetcher 配置

Kafka Fetcher 默认禁用，需在 `application.yml` 中启用[^2]：

```yaml
kafka-fetcher:
  selector: ${SW_KAFKA_FETCHER:default}
  default:
    bootstrapServers: ${SW_KAFKA_FETCHER_SERVERS:localhost:9092}
    namespace: ${SW_NAMESPACE:""}
    partitions: ${SW_KAFKA_FETCHER_PARTITIONS:3}
    replicationFactor: ${SW_KAFKA_FETCHER_PARTITIONS_FACTOR:2}
    consumers: ${SW_KAFKA_FETCHER_CONSUMERS:1}
```

Kafka Fetcher 需要以下 Topic（OAP 可自动创建）[^2]：
- `skywalking-segments`（追踪段数据）
- `skywalking-metrics`（指标数据）
- `skywalking-profilings`（性能分析数据）
- `skywalking-managements`（管理数据）
- `skywalking-meters`（仪表数据）
- `skywalking-logs`（日志数据）
- `skywalking-logs-json`（JSON 格式日志）

### 存储后端配置

SkyWalking 支持多种存储后端[^3]：

| 存储方案 | 适用场景 | 特点 |
|----------|----------|------|
| **BanyanDB** | 推荐，专为 APM 设计 | SkyWalking 团队自研，性能最优，支持冷热数据分层、100% 全量采样 |
| **Elasticsearch/OpenSearch** | 大规模生产环境 | 适合大规模部署，但资源消耗高，需要较多内存和副本 |
| **MySQL/PostgreSQL** | 中小规模、低采样率 | 适合低 Trace/Log 采样率场景，扩展性有限 |
| **ClickHouse** | 监控目标（非后端存储） | SkyWalking 通过 OTel Collector 采集 ClickHouse 自身指标，**不是**作为 OAP 的后端存储 |

> **重要说明**：截至 SkyWalking v10.x，ClickHouse **不是** SkyWalking OAP 的官方支持存储后端。ClickHouse 在 SkyWalking 生态中的角色是**被监控的目标**——SkyWalking 通过 OpenTelemetry Collector 采集 ClickHouse 的内置指标（Prometheus endpoint），用于监控 ClickHouse 服务器性能[^5]。

### Grafana 集成配置

SkyWalking 提供 PromQL 和 LogQL 接口，可对接 Grafana[^4]：

**数据源配置**：
- **Prometheus 数据源**：URL 指向 OAP 服务器地址（默认端口 `9090`）
- **SkyWalking 数据源**：需安装 [skywalking-grafana-plugins](https://github.com/apache/skywalking-grafana-plugins)，URL 指向 OAP GraphQL 服务（默认端口 `12800`）
- **Loki 数据源**：URL 指向 OAP 地址（默认端口 `3100`）

**注意事项**：
- Grafana 使用 [AGPL-3.0 许可证](https://github.com/grafana/grafana/blob/main/LICENSE)，与 SkyWalking 的 Apache 2.0 许可证不同，需遵守 AGPL 3.0 的要求[^4]
- SkyWalking 优先使用原生 UI，Grafana UI 是 PromQL API 的扩展，官方不维护完整的 Grafana Dashboard 配置[^4]
- 指标最小时间桶为 1 分钟，Grafana 查询需设置 `Min interval = 1m`[^4]

## 存储方案对比

| 维度 | BanyanDB | Elasticsearch | MySQL/PostgreSQL |
|------|----------|---------------|------------------|
| 写入性能 | 最优 | 高 | 中 |
| 查询性能 | 优 | 优 | 中 |
| 资源消耗 | 低 | 高（内存密集） | 中 |
| 扩展性 | 支持集群 | 支持集群 | 有限 |
| 全量采样 | 支持 | 支持 | 不推荐 |
| 运维复杂度 | 中 | 高 | 低 |
| 适用规模 | 全规模 | 大规模 | 中小规模 |

## 限制与注意事项

### SkyWalking 架构限制

- **默认不引入 MQ**：SkyWalking 官方 FAQ 明确说明，默认架构中不引入消息队列，因为会增加系统复杂性和运维成本。Kafka Fetcher 是可选组件，适用于需要解耦采集与处理的场景[^1]
- **Agent 兼容性**：不同语言 Agent 的功能支持程度不同，Java Agent 功能最完善
- **存储扩展性**：MySQL/PostgreSQL 的性能无法通过线性扩展节点来显著提升[^3]

### Kafka 相关

- Kafka Fetcher 需要预先创建或允许 OAP 自动创建 7 个 Topic
- Namespace 用于隔离多个 OAP 集群共用同一 Kafka 集群的场景
- 使用 Kafka MirrorMaker 2.0 跨集群复制时，需配置 `mm2SourceAlias` 和 `mm2SourceSeparator`[^2]

### ClickHouse 角色澄清

- ClickHouse 在本方案中是**被监控对象**，而非 SkyWalking 的存储后端
- SkyWalking 通过 OpenTelemetry Collector + Prometheus Receiver 采集 ClickHouse 内置指标[^5]
- 如需将 ClickHouse 作为 SkyWalking 存储后端，需自行开发存储插件（官方未提供）

### Grafana 相关

- SkyWalking 原生 UI 提供完整功能，Grafana 是补充方案
- 部分可视化功能仅在原生 UI 中可用
- 需自行维护 Grafana Dashboard 配置

## 最佳实践

### 部署建议

1. **中小规模**（< 200 服务，< 200 CPS）：BanyanDB（2 liaison + 2 data 节点，4C8G 配置）即可满足[^3]
2. **大规模**：Elasticsearch/OpenSearch 集群，注意资源规划
3. **Kafka 部署**：3 节点集群，根据数据量调整 partitions 和 retention 策略

### 采样策略

- **全量采样**：BanyanDB 支持 100% 全量 Trace 采样
- **比例采样**：通过 Agent 配置 `agent.sample_n_per_3_secs` 控制采样率
- **慢请求采样**：配置慢数据库语句/缓存命令检测阈值

### 监控指标

SkyWalking 采集的 ClickHouse 指标包括[^5]：
- **实例指标**：CPU 使用率、内存使用率、运行时间
- **网络指标**：TCP/MySQL/HTTP/PostgreSQL 连接数、收发字节数
- **查询指标**：查询计数、SELECT/INSERT 查询率、查询耗时
- **写入指标**：插入行数、插入字节数、延迟插入计数
- **复制指标**：复制检查、拉取、发送计数
- **MergeTree 指标**：后台合并计数、合并行数、活跃数据部分数

## 常见问题

### Q: SkyWalking 是否必须使用 Kafka？

**否**。Kafka 是可选组件。Agent 可以直接通过 gRPC/HTTP 将数据发送到 OAP。Kafka 主要用于以下场景[^2]：
- 需要解耦 Agent 和 OAP，提高系统弹性
- 需要数据持久化和回溯能力
- 需要多 OAP 集群消费同一数据源

### Q: 能否用 ClickHouse 替代 Elasticsearch 作为 SkyWalking 存储？

**官方不支持**。截至 v10.x，SkyWalking 官方支持的存储后端为 BanyanDB、Elasticsearch/OpenSearch、MySQL、PostgreSQL[^3]。ClickHouse 在 SkyWalking 生态中是作为被监控的目标，而非存储后端[^5]。

### Q: Grafana 能否完全替代 SkyWalking 原生 UI？

**不能**。SkyWalking 官方明确表示原生 UI 是第一选择，所有可视化功能仅在原生 UI 中完整可用。Grafana UI 是 PromQL API 的扩展，官方不维护完整的 Dashboard 配置[^4]。

### Q: 如何降低 SkyWalking 对应用性能的影响？

- 调整采样率（默认全量采样，可降低比例）
- 使用 Metrics 模式而非 Trace 模式（性能更好）
- 配置慢请求阈值，仅记录超过阈值的请求
- 使用 Kafka 缓冲，避免 OAP 压力直接影响 Agent

## 相关概念

- **OpenTelemetry**：云原生计算基金会（CNCF）的可观测性标准，SkyWalking 支持接收 OTLP 格式的 Trace 和 Metrics[^6]
- **Zipkin**：Twitter 开源的分布式追踪系统，SkyWalking 兼容 Zipkin v1/v2 格式[^1]
- **Prometheus**：云原生监控事实标准，SkyWalking 提供 PromQL 兼容接口[^4]
- **eBPF**：Linux 内核技术，SkyWalking 通过 eBPF Agent 实现无侵入式网络和应用监控

## 参考资料

[^1]: [SkyWalking Overview and Core Concepts](https://skywalking.apache.org/docs/main/latest/en/concepts-and-designs/overview/)
[^2]: [SkyWalking Kafka Fetcher 配置](https://skywalking.apache.org/docs/main/latest/en/setup/backend/kafka-fetcher/)
[^3]: [SkyWalking Backend Storage 选择](https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-storage/)
[^4]: [SkyWalking Grafana UI 集成](https://skywalking.apache.org/docs/main/latest/en/setup/backend/ui-grafana/)
[^5]: [SkyWalking ClickHouse Monitoring](https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-clickhouse-monitoring/)
[^6]: [SkyWalking OpenTelemetry Trace 接收](https://skywalking.apache.org/docs/main/latest/en/setup/backend/otlp-trace/)
