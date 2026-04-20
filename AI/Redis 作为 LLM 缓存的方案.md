---
created: "2026-04-20"
updated: "2026-04-20"
tags:
  - ai
  - llm
  - redis
  - semantic-cache
  - langcache
  - cost-optimization
aliases:
  - Redis LangCache
  - Redis LLM Cache
  - Semantic Cache
  - 语义缓存
source_type: official-doc
source_urls:
  - https://redis.io/docs/latest/develop/ai/langcache/
  - https://redis.io/langcache/
  - https://redis.io/redis-for-ai/
status: verified
---

## 概述

Redis 作为 LLM 缓存的核心方案是**语义缓存（Semantic Cache）**——通过存储和复用 LLM 响应，减少重复调用，降低成本和延迟。Redis 官方提供的 **LangCache** 是一个全托管语义缓存服务，基于 Redis 向量数据库实现语义相似度匹配。

---

## 一、语义缓存 vs 传统缓存

| 特性 | 传统键值缓存 | 语义缓存 |
|------|-------------|----------|
| 匹配方式 | 精确字符串匹配 | 向量语义相似度匹配 |
| 适用场景 | 固定查询、API 响应 | 自然语言查询、LLM 响应 |
| 变体处理 | 无法匹配相似表述 | 可匹配语义相近的问题 |
| 技术依赖 | Hash/String 数据结构 | 向量数据库 + Embedding 模型 |

**示例**：以下三个问题语义相同，传统缓存无法复用，语义缓存可返回同一响应[^1]：
- "What are the features of Product A?"
- "Can you list the main features of Product A?"
- "Tell me about Product A's features."

---

## 二、Redis LangCache（官方方案）

### 2.1 什么是 LangCache

LangCache 是 Redis 提供的**全托管语义缓存服务**，目前处于 Preview 状态[^2]：

- 通过 REST API 访问，无需数据库运维
- 自动生成 Embedding（可配置模型）
- 内置向量相似度搜索
- 支持高级缓存管理（TTL、驱逐策略、隐私控制）

### 2.2 核心优势

| 优势 | 说明 |
|------|------|
| 降低 LLM 成本 | 缓存命中时无需支付输出 Token 费用，可节省 90% API 成本[^3] |
| 加快响应速度 | 缓存命中时直接返回内存中的响应，响应时间缩短 4 倍[^4] |
| 简化部署 | REST API 接入，无需管理数据库或向量索引 |
| 精准匹配 | 向量搜索实现语义相似度匹配，避免精确匹配的局限 |

### 2.3 适用场景[^5]

- **AI 助手和聊天机器人**：缓存常见问答响应，降低高频问题处理成本
- **RAG 应用**：缓存相似查询的检索结果，避免重复向量检索和 LLM 调用
- **AI Agent**：缓存推理链中间结果和常见推理模式
- **AI Gateway**：集中管理多个应用的 LLM 调用成本

---

## 三、LangCache 工作流程

架构流程[^6]：

1. **用户发送 Prompt → AI 应用**
2. **应用调用 LangCache**：`POST /v1/caches/{cacheId}/entries/search`
3. **LangCache 生成 Embedding**：调用 Embedding 模型服务
4. **向量搜索匹配**：将 Prompt Embedding 与存储的 Embedding 比较
5. **缓存命中（Cache Hit）**：返回缓存的响应，跳过 LLM 调用
6. **缓存未命中（Cache Miss）**：应用调用 LLM 生成响应
7. **存储新响应**：`POST /v1/caches/{cacheId}/entries`，缓存 Prompt + Response

---

## 四、成本节省计算

缓存命中时无需支付**输出 Token** 费用，计算公式[^7]：

```
月节省成本 = 月输出 Token 成本 × 缓存命中率
```

**示例**：
- 月 LLM 支出：$200
- 输出 Token 占比：60% → 输出成本 $120
- 缓存命中率：50%
- 月节省：$120 × 50% = **$60**

Redis 提供 [LangCache 成本计算器](https://redis.io/calculator/langcache/) 估算年度节省。

---

## 五、接入方式

### 5.1 Redis Cloud 公开 Preview

步骤[^8]：
1. 在 Redis Cloud 创建数据库
2. 创建 LangCache 服务
3. 使用 LangCache REST API 接入应用

### 5.2 API 端点

| 端点 | 用途 |
|------|------|
| `POST /v1/caches/{cacheId}/entries/search` | 搜索语义相似的缓存条目 |
| `POST /v1/caches/{cacheId}/entries` | 存储新的 Prompt + Response |
| `GET /v1/caches/{cacheId}` | 获取缓存配置信息 |
| `DELETE /v1/caches/{cacheId}/entries/{entryId}` | 删除缓存条目 |

详细 API 参考：[LangCache API Reference](https://redis.io/docs/latest/develop/ai/langcache/api-reference/)

### 5.3 SDK 支持

LangCache 提供 Python、JavaScript、CLI 等客户端 SDK[^9]。

---

## 六、自建语义缓存方案

若不使用 LangCache，可基于 Redis 自建语义缓存：

### 6.1 技术栈

| 组件 | 说明 |
|------|------|
| Redis Search / RedisVL | 向量存储和相似度搜索 |
| Embedding 模型 | OpenAI text-embedding-ada-002、HuggingFace 模型等 |
| 应用层逻辑 | Embedding 生成、相似度阈值判断、缓存读写 |

### 6.2 实现步骤

```python
# 伪代码示例
import redis
from openai import OpenAI

# 1. 初始化 Redis 连接和 Embedding 客户端
r = redis.Redis(host='localhost', port=6379)
embedding_client = OpenAI()

# 2. 生成 Prompt Embedding
def get_embedding(text):
    response = embedding_client.embeddings.create(
        input=text, model="text-embedding-ada-002"
    )
    return response.data[0].embedding

# 3. 向量搜索缓存
def search_cache(prompt, threshold=0.95):
    embedding = get_embedding(prompt)
    # 使用 Redis Search 的 VSIM 或 FT.SEARCH 进行向量搜索
    results = r.ft("cache_index").search(
        f"*=>[KNN 1 @embedding $vec AS score]",
        query_params={"vec": embedding}
    )
    if results.total > 0 and results.docs[0].score > threshold:
        return results.docs[0].response  # 缓存命中
    return None  # 缓存未命中

# 4. 存储缓存
def store_cache(prompt, response):
    embedding = get_embedding(prompt)
    r.hset(f"cache:{prompt_hash}", mapping={
        "prompt": prompt,
        "response": response,
        "embedding": embedding
    })
```

### 6.3 与 LangCache 对比

| 维度 | LangCache | 自建方案 |
|------|-----------|----------|
| 运维成本 | 全托管，无需运维 | 需自行管理 Redis、Embedding 服务 |
| 接入复杂度 | REST API，简单 | 需实现完整逻辑 |
| 功能完整度 | 内置 TTL、驱逐、监控 | 需自行实现 |
| Embedding 管理 | 自动生成，可配置模型 | 需自行集成模型 |
| 适用场景 | 快速上线、企业级应用 | 高度定制需求、已有 Redis 基础设施 |

---

## 七、最佳实践

### 7.1 相似度阈值设置

- 过高阈值（如 0.99）：命中率低，节省有限
- 过低阈值（如 0.80）：可能返回语义不完全匹配的响应
- 推荐：根据业务容忍度调整，初始可设 **0.90-0.95**

### 7.2 缓存生命周期管理

- 设置合理的 TTL：根据内容时效性（如新闻类较短，产品信息类较长）
- 监控缓存命中率：低于预期时检查阈值或内容变化
- 定期清理过期或无效缓存

### 7.3 数据隐私与合规

- LangCache 数据存储在用户自己的 Redis 服务器[^10]
- Redis 不访问用户数据，不用于训练 AI 模型[^10]
- 符合企业级安全和隐私标准

---

## 八、限制与注意事项

- **LangCache 当前状态**：Preview 阶段，功能和行为可能变化[^2]
- **语义局限**：语义相似不一定意图相同（如"如何退款" vs"退款政策是什么"可能需不同响应）
- **冷启动问题**：新应用初期缓存命中率低
- **动态内容**：实时变化的内容不适合缓存（如股票价格、天气）
- **Embedding 成本**：虽低于 LLM 输出 Token，但 Embedding 生成仍有成本

---

## 九、相关概念

- **Vector Database**：Redis 向量数据库能力，支持 HNSW 和 Flat 索引
- **Embedding**：文本向量表示，用于语义相似度计算
- **RAG（Retrieval-Augmented Generation）**：检索增强生成，语义缓存可优化 RAG 查询阶段
- **RedisVL**：Redis 官方 Python 库，简化向量操作

---

## 十、参考资料

- [Redis LangCache 官方文档](https://redis.io/docs/latest/develop/ai/langcache/)
- [Redis LangCache 产品页](https://redis.io/langcache/)
- [Redis for AI](https://redis.io/redis-for-ai/)
- [LangCache API Examples](https://redis.io/docs/latest/develop/ai/langcache/api-examples/)
- [LangCache API Reference](https://redis.io/docs/latest/develop/ai/langcache/api-reference/)
- [LangCache 成本计算器](https://redis.io/calculator/langcache/)

[^1]: 语义相似问题示例来源于 [LangCache Overview](https://redis.io/docs/latest/develop/ai/langcache/#langcache-overview)
[^2]: Preview 状态说明来源于 [LangCache 文档](https://redis.io/docs/latest/develop/ai/langcache/)
[^3]: 90% 成本节省来源于 [LangCache 产品页](https://redis.io/langcache/)
[^4]: 4 倍响应速度来源于 [Mangoes.ai 客户案例](https://redis.io/langcache/)
[^5]: 适用场景来源于 [LangCache Overview](https://redis.io/docs/latest/develop/ai/langcache/#langcache-overview)
[^6]: 架构流程来源于 [LangCache Architecture](https://redis.io/docs/latest/develop/ai/langcache/#langcache-architecture)
[^7]: 成本计算公式来源于 [LLM Cost Reduction](https://redis.io/docs/latest/develop/ai/langcache/#llm-cost-reduction-with-langcache)
[^8]: Redis Cloud 接入步骤来源于 [LangCache Get Started](https://redis.io/docs/latest/develop/ai/langcache/#get-started)
[^9]: SDK 支持来源于 [LangCache 产品页](https://redis.io/langcache/)
[^10]: 数据隐私说明来源于 [LangCache Data Security](https://redis.io/docs/latest/develop/ai/langcache/#data-security-and-privacy)