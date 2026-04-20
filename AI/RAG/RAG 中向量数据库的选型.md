---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - RAG
  - 向量数据库
  - AI
  - 检索增强生成
  - Vector Database
  - Embedding
aliases:
  - 向量数据库选型
  - Vector DB Selection
  - RAG Vector Store
source_type: official-doc
source_urls:
  - 'https://weaviate.io/developers/weaviate/introduction'
  - 'https://qdrant.tech/documentation/'
  - 'https://docs.trychroma.com/'
  - 'https://github.com/pgvector/pgvector'
  - 'https://docs.pinecone.io/guides/get-started/overview'
  - 'https://redis.io/docs/latest/develop/data-types/vector-sets/'
status: verified
---
向量数据库是 RAG（Retrieval-Augmented Generation）系统的核心组件，负责存储文本的向量嵌入并支持语义相似度检索。选型需要综合考虑性能、规模、运维成本、生态集成等因素。

## 一、向量数据库的核心功能

向量数据库主要提供以下能力：

- **向量存储**：存储高维向量（通常 128-1536 维）及其关联的原始数据和元数据
- **相似度检索**：基于距离度量（L2、余弦、内积）查找最相似的向量
- **近似最近邻搜索（ANN）**：使用 HNSW、IVF 等索引算法加速大规模检索
- **元数据过滤**：结合向量检索与传统条件过滤
- **混合检索**：向量语义检索 + 关键词检索的组合

## 二、主流向量数据库对比

### 专用向量数据库

| 数据库 | 开源 | 部署模式 | 特点 | 适用场景 |
|--------|------|----------|------|----------|
| **Milvus** | ✅ | 云托管/自托管 | 高性能、支持十亿级规模、丰富的索引类型 | 大规模生产系统 |
| **Pinecone** | ❌ | 仅云托管 | 全托管、零运维、内置嵌入和重排序 | 快速落地、无运维需求 |
| **Weaviate** | ✅ | 云托管/自托管 | 内置向量化、GraphQL API、模块化架构 | 需要内置嵌入生成的场景 |
| **Qdrant** | ✅ | 云托管/自托管 | Rust 实现、高性能、支持分布式部署 | 性能敏感、资源效率要求高 |
| **Chroma** | ✅ | 云托管/自托管 | 极简 API、Python 优先、开箱即用 | 快速原型、小型项目 |

### 扩展型向量数据库

| 数据库 | 开源 | 特点 | 适用场景 |
|--------|------|------|----------|
| **pgvector** | ✅ | PostgreSQL 扩展、HNSW/IVFFlat 索引、支持 ACID | 已有 Postgres 基础设施、中小规模 |
| **Redis Vector Sets** | ✅ | Redis 8.0+ 新数据类型、HNSW 索引、支持过滤 | 低延迟场景、已有 Redis 部署 |

### 索引算法对比

| 算法 | 特点 | 适用场景 |
|------|------|----------|
| **HNSW** | 多层图结构、查询速度快、构建慢、内存占用较高 | 实时查询、中等规模数据集 |
| **IVF** | 聚类分区、构建快、内存占用低、查询需设置 probes | 大规模批量导入、内存受限场景 |
| **Flat** | 精确搜索、无索引开销、慢 | 小数据集、需要精确结果 |

## 三、选型关键维度

### 1. 数据规模

- **百万级以下**：Chroma、pgvector、Redis Vector Sets 可满足需求
- **百万到千万级**：Milvus、Qdrant、Weaviate
- **十亿级以上**：Milvus（分布式部署）、Pinecone（云托管）

### 2. 性能需求

- **毫秒级延迟**：Redis Vector Sets、Qdrant
- **秒级延迟可接受**：大多数方案均可
- **高吞吐写入**：Milvus、Qdrant

### 3. 运维模式

- **无运维/全托管**：Pinecone、Chroma Cloud、Weaviate Cloud、Qdrant Cloud、Milvus Cloud (Zilliz)
- **自托管/可控基础设施**：开源版本 + Kubernetes/Docker 部署

### 4. 生态集成

主流向量数据库均有 LangChain、LlamaIndex 集成：

- **LangChain 支持**：Milvus、Pinecone、Weaviate、Qdrant、Chroma、pgvector
- **LlamaIndex 支持**：全部主流方案均有集成

### 5. 成本考量

- **云托管**：按向量数量/查询量计费，适合快速迭代阶段
- **自托管**：硬件成本 +运维人力，适合长期稳定运行的大规模系统

## 四、各方案详细说明

### Milvus

开源高性能向量数据库，支持十亿级规模 [^1]。

**核心特性**：
- 支持多种索引：HNSW、IVF_FLAT、IVF_PQ、GPU 索引
- 分布式架构，支持水平扩展
- 支持多向量检索、稀疏向量
- 云托管版本 Zilliz Cloud

**适用场景**：大规模企业级 RAG 系统、需要 GPU 加速的场景

### Pinecone

全托管向量数据库，强调易用性 [^2]。

**核心特性**：
- 零运维，自动扩缩容
- 内置嵌入模型集成（Integrated Embedding）
- 内置 Reranking 功能
- Namespace 支持多租户隔离

**适用场景**：快速迭代、无运维团队、中小规模生产应用

### Weaviate

开源 AI 原生向量数据库 [^3]。

**核心特性**：
- 内置向量化模块（支持多种嵌入模型）
- GraphQL/REST API
- 模块化架构，支持自定义模块
- 支持混合检索（BM25 +向量）

**适用场景**：需要端到端嵌入生成、GraphQL API 偏好

### Qdrant

Rust 实现的高性能向量数据库 [^4]。

**核心特性**：
- Rust 实现，高性能低资源占用
- HNSW 索引，支持量化（Scalar/Binary/Product）
- 分布式部署支持
- FastEmbed 内置嵌入支持

**适用场景**：性能敏感、资源受限、需要边缘部署

### Chroma

极简开源向量数据库 [^5]。

**核心特性**：
- Python/JavaScript API
- 内置嵌入模型支持（OpenAI、Hugging Face 等）
- 支持稠密/稀疏/混合检索
- Chroma Cloud 云托管版本

**适用场景**：快速原型开发、小型 RAG 应用、Python 项目

### pgvector

PostgreSQL 向量扩展 [^6]。

**核心特性**：
- 支持 HNSW 和 IVFFlat 索引
- vector 类型最多 2000 维，halfvec 类型最多 4000 维
- 支持 L2、内积、余弦、L1、Hamming、Jaccard 距离
- 继承 Postgres ACID、事务、JOIN 等能力

**适用场景**：已有 Postgres 基础设施、中小规模（<10M）、需要事务一致性

**索引参数建议** [^6]：
- HNSW `m` 默认 16，`ef_construction` 默认 64
- IVFFlat `lists` 建议：百万级数据 `rows/1000`，超百万级 `sqrt(rows)`
- 查询时 HNSW 设置 `hnsw.ef_search`（默认 40），IVFFlat 设置 `ivfflat.probes`

### Redis Vector Sets

Redis 8.0 引入的原生向量数据类型 [^7]。

**核心特性**：
- HNSW 索引实现
- 支持 FP32、量化（Q8/BIN）
- 支持 JSON 属性和过滤检索（FILTER 表达式）
- 低延迟，适合实时场景

**适用场景**：已有 Redis 部署、低延迟要求、中小规模

## 五、选型决策流程

```
是否已有 Postgres/Redis 基础设施？
├─ 是 → 优先考虑 pgvector / Redis Vector Sets
│        （评估规模和性能是否满足）
└─ 否 → 数据规模多大？
        ├─ <1M → Chroma（原型）/Qdrant（生产）
        ├─ 1M-100M → Qdrant / Weaviate / Milvus
        ├─ >100M → Milvus（分布式）/ Pinecone（云托管）
        
运维偏好？
├─ 无运维 → Pinecone / Chroma Cloud / Zilliz Cloud
└─ 自托管 → 开源版本 + K8s
```

## 六、常见误区

1. **混淆向量数据库与嵌入模型**
   - 向量数据库只存储和检索向量，嵌入生成需使用单独的嵌入模型（如 OpenAI Embeddings、sentence-transformers）

2. **忽视索引参数调优**
   - HNSW 的 `ef_search`、IVF 的 `probes` 直接影响召回率和查询速度

3. **低估内存需求**
   - HNSW 索引内存占用 ≈ 向量大小 × 1.5~2倍

4. **忽略元数据过滤的影响**
   - 过滤条件过于严格可能导致 ANN 索引返回结果不足，需启用迭代扫描或提高 `ef_search`

## 七、参考资料

[^1]: Milvus 官方文档 https://milvus.io/docs/
[^2]: Pinecone 官方文档 https://docs.pinecone.io/
[^3]: Weaviate 官方文档 https://weaviate.io/developers/weaviate/introduction
[^4]: Qdrant 官方文档 https://qdrant.tech/documentation/
[^5]: Chroma 官方文档 https://docs.trychroma.com/
[^6]: pgvector GitHub https://github.com/pgvector/pgvector
[^7]: Redis Vector Sets 文档 https://redis.io/docs/latest/develop/data-types/vector-sets/
