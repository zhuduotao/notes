---
created: 2026-04-17
updated: 2026-04-17
tags:
  - Middleware
  - ServiceDiscovery
  - ConfigCenter
  - CloudNative
  - Microservices
aliases:
  - Nacos
  - Dynamic Naming and Configuration Service
  - 服务注册中心
  - 配置中心
source_type: official-doc
source_urls:
  - https://nacos.io/docs/latest/overview/
  - https://nacos.io/docs/latest/architecture/
  - https://github.com/alibaba/nacos
status: verified
---

## 是什么

Nacos（`/nɑ:kəʊs/`）是 **Dynamic Naming and Configuration Service** 的首字母缩写，由阿里巴巴开源，是一个易于构建云原生应用的**动态服务发现、配置管理和服务管理平台**[^what-is-nacos]。

Nacos 是构建以"服务"为中心的现代应用架构（微服务范式、云原生范式）的服务基础设施，服务（Service）是 Nacos 世界的一等公民。

[^what-is-nacos]: 来源：[Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)

## 为什么重要

在微服务架构中，服务数量多、动态变化频繁，传统硬编码服务地址或静态配置文件的方式无法满足需求。Nacos 解决了以下核心问题：

- **服务地址动态变化**：服务实例扩缩容、故障迁移时，消费者能自动感知
- **配置分散管理**：多环境、多服务配置分散在各处，变更需重新部署
- **服务治理缺失**：缺乏统一的服务元数据、健康状态、流量管理能力

## 核心功能

### 服务发现和服务健康监测

Nacos 支持基于 **DNS** 和基于 **RPC** 的服务发现[^naming]：

- 服务提供者通过 SDK、OpenAPI 或独立 Agent 注册 Service
- 服务消费者通过 DNS 或 HTTP&API 查找和发现服务
- 提供实时健康检查，阻止向不健康实例发送请求
- 支持传输层（PING/TCP）和应用层（HTTP、MySQL、自定义）健康检查
- 两种健康检查模式：agent 上报模式、服务端主动检测

[^naming]: 来源：[Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)

### 动态配置服务

以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置[^config]：

- 消除配置变更时重新部署应用和服务的需要
- 配置中心化管理让无状态服务更简单，弹性扩展更容易
- 提供简洁易用的 Web 控制台管理配置
- 开箱即用的配置管理特性：
  - 配置版本跟踪
  - 金丝雀发布（灰度发布）
  - 一键回滚配置
  - 客户端配置更新状态跟踪

[^config]: 来源：[Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)

### 动态 DNS 服务

- 支持权重路由，实现中间层负载均衡
- 灵活的路由策略、流量控制
- 数据中心内网的简单 DNS 解析服务
- 消除耦合到厂商私有服务发现 API 的风险

### 服务及其元数据管理

从微服务平台建设视角管理数据中心的所有服务及元数据[^metadata]：

- 服务的描述、生命周期
- 服务的静态依赖分析
- 服务的健康状态
- 服务的流量管理、路由及安全策略
- 服务的 SLA 以及 metrics 统计数据

[^metadata]: 来源：[Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)

## 架构设计

### 逻辑架构

Nacos 的逻辑架构包含以下核心模块[^architecture]：

| 模块 | 职责 |
|------|------|
| 服务管理 | 服务 CRUD、域名 CRUD、健康检查、权重管理 |
| 配置管理 | 配置 CRUD、版本管理、灰度管理、监听管理、推送轨迹 |
| 元数据管理 | 元数据 CRUD 和打标能力 |
| 插件机制 | 模块可分可合，扩展点 SPI 机制 |
| 一致性协议 | 不同数据、不同一致性要求下的不同一致性机制 |
| 存储模块 | 数据持久化、非持久化存储，数据分片 |
| 缓存机制 | 容灾目录、本地缓存、server 缓存 |
| 推送通道 | server 与存储、server 间、server 与 SDK 间推送 |
| 容量管理 | 管理租户、分组下的容量，防止存储写爆 |
| 流量管理 | 按租户、分组等维度对请求频率、长链接、报文大小进行控制 |

[^architecture]: 来源：[Nacos 官方文档 - 架构](https://nacos.io/docs/latest/architecture/)

### 数据模型

Nacos 数据模型由**三元组**唯一标识[^data-model]：

```
Namespace + Group + DataId
```

- **Namespace**：命名空间，用于环境隔离（如 dev、test、prod），默认为空串（public）
- **Group**：分组，用于服务或配置的分组，默认为 `DEFAULT_GROUP`
- **DataId**：配置 ID 或服务名，通常格式为 `${prefix}-${spring.profiles.active}.${file-extension}`

[^data-model]: 来源：[Nacos 官方文档 - 架构](https://nacos.io/docs/latest/architecture/)

### 服务领域模型

```
Service（服务）
  └── Cluster（集群）
        └── Instance（实例）
              ├── IP + Port
              ├── Weight（权重）
              ├── Healthy（健康状态）
              ├── Metadata（元数据）
              └── Ephemeral（临时/持久实例标记）
```

- **Service**：服务名称，如 `com.example.UserService`
- **Cluster**：集群，同一服务下按机房或区域划分的子集
- **Instance**：服务实例，包含 IP、端口、权重、健康状态、元数据等

### 配置领域模型

围绕配置主要有两个关联实体[^config-model]：

- **配置**：由 Namespace + Group + DataId 唯一标识
- **配置变更历史**：记录每次配置变更的内容、时间、操作人
- **服务标签**：用于打标分类，方便索引，由 ID 与配置关联

[^config-model]: 来源：[Nacos 官方文档 - 架构](https://nacos.io/docs/latest/architecture/)

## 部署模式

Nacos 支持多种部署模式[^deployment]：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| 单机模式（Standalone） | 单进程运行，使用嵌入式存储 | 开发、测试环境 |
| 集群模式（Cluster） | 多节点集群，使用外部数据库（MySQL） | 生产环境 |
| 多集群模式 | 多个集群间数据同步 | 跨机房、跨地域容灾 |

Nacos 支持两种启动模式：

- **合并部署**：注册中心（Service Registry）与配置中心（Config Center）在同一进程
- **分离部署**：注册中心与配置中心分别独立部署

[^deployment]: 来源：[Nacos 官方文档 - 部署手册概览](https://nacos.io/docs/latest/manual/admin/deployment/deployment-overview/)

## 与同类工具对比

| 特性 | Nacos | Eureka | Consul | ZooKeeper |
|------|-------|--------|--------|-----------|
| 服务发现 | ✅ | ✅ | ✅ | ✅ |
| 配置管理 | ✅ | ❌ | ✅（KV 存储） | ✅（KV 存储） |
| 健康检查 | TCP/HTTP/MySQL/自定义 | 客户端心跳 | TCP/HTTP/gRPC | 会话心跳 |
| 一致性协议 | Raft + Distro（AP/CP 可切换） | AP | CP（Raft） | CP（ZAB） |
| 多语言支持 | Java/Go/Python/... | Java 为主 | 多语言 | 多语言 |
| 控制台 | ✅ | ❌ | ✅ | ❌ |
| 灰度配置 | ✅ | ❌ | ❌ | ❌ |
| 权重路由 | ✅ | ❌ | ✅ | ❌ |
| 开源协议 | Apache 2.0 | Apache 2.0 | MPL 2.0 | Apache 2.0 |
| 维护方 | 阿里巴巴/社区 | Netflix（已停止更新） | HashiCorp | Apache |

**关键差异说明**：

- **Nacos vs Eureka**：Eureka 2.x 已开源但停止维护，Nacos 是更活跃的替代方案，且同时提供服务发现和配置管理
- **Nacos vs Consul**：Nacos 对 Spring Cloud/Dubbo 生态集成更友好，Consul 在多云/混合云场景更成熟
- **Nacos vs ZooKeeper**：ZooKeeper 是 CP 系统，分区时不可用；Nacos 支持 AP/CP 切换，可用性更高

## 支持的生态

Nacos 无缝支持以下主流开源生态[^ecosystem]：

- **Spring Cloud**：[Nacos 融合 Spring Cloud](https://nacos.io/docs/latest/ecology/use-nacos-with-spring-cloud/)
- **Apache Dubbo**：[Nacos 融合 Dubbo](https://nacos.io/docs/latest/ecology/use-nacos-with-dubbo/)
- **Kubernetes**：[Nacos Kubernetes 部署](https://nacos.io/docs/latest/quickstart/quick-start-kubernetes/)
- **CoreDNS**：[Nacos 融合 CoreDNS 下发 DNS 域名](https://nacos.io/docs/latest/ecology/use-nacos-with-coredns/)
- **Istio**：[Nacos 融合 Istio 下发 xDS 协议](https://nacos.io/docs/latest/ecology/use-nacos-with-istio/)
- **Spring / Spring Boot 3**：[Nacos 融合 Spring](https://nacos.io/docs/latest/ecology/use-nacos-with-spring/)

[^ecosystem]: 来源：[Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)

## 核心概念

### 基本概念

| 概念 | 说明 |
|------|------|
| 服务（Service） | 一个或一组软件功能，可被不同客户端重用 |
| 服务注册中心（Service Registry） | 服务、实例及元数据的数据库 |
| 服务元数据（Service Metadata） | 服务端点、标签、版本号、权重、路由规则、安全策略等 |
| 服务提供方（Service Provider） | 提供可复用和可调用服务的应用 |
| 服务消费方（Service Consumer） | 发起对某个服务调用的应用 |
| 配置（Configuration） | 从代码中分离出来的参数、变量，以独立配置文件形式存在 |
| 名字服务（Naming Service） | 提供名字到关联元数据的映射管理，如 ServiceName → Endpoints |
| 配置服务（Configuration Service） | 提供动态配置或元数据以及配置管理的服务提供者 |

### 临时实例 vs 持久实例

Nacos 支持两种实例注册模式：

- **临时实例（Ephemeral）**：实例通过心跳保持注册状态，心跳停止后实例自动删除。适用于短期存在的实例，如容器化部署
- **持久实例（Persistent）**：实例注册后永久存在，除非主动注销。适用于长期稳定运行的实例

## 快速开始

### 单机模式部署

```bash
# 下载 Nacos Server
wget https://github.com/alibaba/nacos/releases/download/${version}/nacos-server-${version}.tar.gz

# 解压
tar -xzf nacos-server-${version}.tar.gz
cd nacos/bin

# 单机模式启动
# Linux/Unix/Mac
sh startup.sh -m standalone

# Windows
cmd startup.cmd -m standalone
```

### Docker 部署

```bash
docker run -d \
  --name nacos-standalone \
  -e MODE=standalone \
  -p 8848:8848 \
  -p 9848:9848 \
  nacos/nacos-server:latest
```

### 访问控制台

- 地址：`http://localhost:8848/nacos`
- 默认用户名/密码：`nacos/nacos`

## 常见误区

### 误区 1：Nacos 只是服务注册中心

Nacos 同时提供**服务发现**和**配置管理**两大核心能力，是一个综合性的微服务基础设施平台。

### 误区 2：Nacos 只能用于 Java 生态

虽然 Nacos 由阿里巴巴用 Java 开发，但提供了多语言 SDK（Java、Go、Python 等）和 OpenAPI，可与任何语言集成。

### 误区 3：单机模式可用于生产

单机模式仅适用于开发测试，生产环境必须使用**集群模式 + 外部 MySQL 数据库**以保证高可用。

### 误区 4：Nacos 的 AP 和 CP 模式可以随意切换

Nacos 支持 AP（Distro 协议）和 CP（Raft 协议）两种一致性模式，但**针对不同类型的服务**：

- 临时实例使用 AP 模式（高可用，最终一致性）
- 持久实例使用 CP 模式（强一致性）

这不是运行时可切换的选项，而是根据实例类型自动选择的。

## 最佳实践

### 命名空间隔离

使用 Namespace 隔离不同环境：

```yaml
# application.yml
spring:
  cloud:
    nacos:
      server-addr: nacos-server:8848
      config:
        namespace: dev-namespace-id  # 开发环境
      discovery:
        namespace: dev-namespace-id
```

### 配置分组管理

按业务模块使用 Group 分组：

- `DEFAULT_GROUP`：默认分组
- `ORDER_GROUP`：订单相关配置
- `USER_GROUP`：用户相关配置

### 灰度发布配置

Nacos 支持配置的灰度发布，可针对特定 IP 或环境发布不同版本的配置，验证无误后再全量推送。

### 健康检查策略选择

- **客户端上报模式**：适用于 VPC、边缘网络等复杂网络环境
- **服务端主动检测**：适用于网络环境简单、服务端可直接访问实例的场景

## 相关概念

- **[服务网格（Service Mesh）](../服务网格)**：Nacos 可与 Istio 集成，通过 xDS 协议下发服务发现数据
- **[Dubbo](../Dubbo)**：Apache Dubbo 的默认注册中心之一
- **[Spring Cloud Alibaba](../Spring%20Cloud%20Alibaba)**：Spring Cloud 的阿里巴巴生态扩展，内置 Nacos 集成
- **[Consul](./Consul)**：HashiCorp 出品的服务网格和配置管理平台，Nacos 的同类竞品
- **[Eureka](./Eureka)**：Netflix 开源的服务注册中心，已停止更新
- **[ZooKeeper](./ZooKeeper)**：Apache 分布式协调服务，常用于服务发现和配置管理

## 参考资料

- [Nacos 官方文档 - 概览](https://nacos.io/docs/latest/overview/)
- [Nacos 官方文档 - 架构](https://nacos.io/docs/latest/architecture/)
- [Nacos 官方文档 - 快速开始](https://nacos.io/docs/latest/quickstart/quick-start/)
- [Nacos GitHub 仓库](https://github.com/alibaba/nacos)
- [Nacos 控制台样例](http://console.nacos.io/index.html)（默认用户名/密码：nacos/nacos）
- [Nacos 3.2 发布博客](https://nacos.io/blog/nacos-gvr7dx_awbbpb_eerxlks19kgclceq/)
