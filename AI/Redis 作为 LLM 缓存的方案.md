---
created: '2026-04-20'
updated: '2026-04-22'
tags:
  - ai
  - llm
  - redis
  - semantic-cache
  - langcache
  - cost-optimization
  - llm-cost
  - 实践
  - mem0
aliases:
  - Redis LangCache
  - Redis LLM Cache
  - Semantic Cache
  - 语义缓存
  - LangCache 费用优化
source_type: official-doc
source_urls:
  - 'https://redis.io/docs/latest/develop/ai/langcache/'
  - 'https://redis.io/langcache/'
  - 'https://redis.io/redis-for-ai/'
  - 'https://redis.io/calculator/langcache/'
  - 'https://redis.io/docs/latest/develop/ai/langcache/api-examples/'
  - 'https://redis.io/docs/latest/operate/rc/langcache/'
  - 'https://redis.io/customers/mangoes-ai/'
  - 'https://docs.mem0.ai/'
  - 'https://github.com/mem0ai/mem0'
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

## 四、成本节省模型与计算

### 4.1 核心节省公式

缓存命中时无需支付**输出 Token** 费用，计算公式[^7]：

```
月节省成本 = 月输出 Token 成本 × 缓存命中率
```

**示例**：
- 月 LLM 支出：$200
- 输出 Token 占比：60% → 输出成本 $120
- 缓存命中率：50%
- 月节省：$120 × 50% = **$60**

### 4.2 完整成本模型

使用 LangCache 后的总成本包含三部分[^calc]：

```
总成本 = (LLM 输出成本 × (1 - 缓存命中率)) + Embedding 成本 + 存储成本
```

| 成本项 | 说明 | 参考单价 |
|--------|------|----------|
| LLM 输入 Token | 每次查询仍需支付 | GPT-4o: $2.5/1M tokens |
| LLM 输出 Token | 缓存命中时**不产生** | GPT-4o: $10/1M tokens |
| Embedding | 每次查询都需生成向量 | LangCache: $1.5/1M tokens（Preview 免费） |
| 存储 | 缓存条目占用空间 | ~$100/月（视规模而定） |

### 4.3 大规模成本计算器示例

基于官方计算器的典型场景[^calc]：

| 参数 | 值 |
|------|-----|
| 日查询量 | 1,000,000 |
| 每查询输入 Token | 100 |
| 每查询输出 Token | 1,000 |
| 缓存命中率 | 85% |
| LLM 模型 | GPT-4o |

**年度成本对比**：

| 项目 | 无 LangCache | 使用 LangCache |
|------|-------------|----------------|
| LLM 输入成本 | $91,250 | $91,250 |
| LLM 输出成本 | $3,650,000 | $547,500（节省 85%） |
| Embedding 成本 | $0 | $54,750 |
| 存储成本 | $0 | $1,200 |
| **年度总成本** | **$3,741,250** | **$694,700** |
| **年度节省** | — | **$3,046,550（81.4%）** |

> 注：LangCache 服务成本在 Public Preview 期间免费，GA 后定价待定[^calc]。

### 4.4 影响节省的关键因素

1. **查询重复度**：客服、FAQ 等场景重复查询多，命中率可达 70%+
2. **输出 Token 占比**：长回答场景（如代码生成、摘要）节省更显著
3. **模型选择**：使用越昂贵的 LLM 模型，缓存节省的绝对值越大
4. **Embedding 成本**：LangCache 内置 Embedding 比自建方案更经济

Redis 提供 [LangCache 成本计算器](https://redis.io/calculator/langcache/) 可上传自有查询数据估算命中率。

---

## 五、接入方式与 SDK 实践

### 5.1 Redis Cloud 公开 Preview

步骤[^8]：
1. 在 Redis Cloud 创建数据库
2. 创建 LangCache 服务
3. 使用 LangCache REST API 接入应用

### 5.2 认证与前置条件

访问 LangCache API 需要三个要素[^sdk]：
- **API Base URL**（`$HOST`）
- **API Key**（作为 Bearer Token 传入）
- **Cache ID**（`$CACHE_ID`）

### 5.3 API 端点

| 端点 | 用途 |
|------|------|
| `POST /v1/caches/{cacheId}/entries/search` | 搜索语义相似的缓存条目 |
| `POST /v1/caches/{cacheId}/entries` | 存储新的 Prompt + Response |
| `GET /v1/caches/{cacheId}` | 获取缓存配置信息 |
| `DELETE /v1/caches/{cacheId}/entries/{entryId}` | 删除缓存条目 |
| `DELETE /v1/caches/{cacheId}/entries` | 按属性批量删除 |
| `POST /v1/caches/{cacheId}/flush` | 清空所有缓存 |

详细 API 参考：[LangCache API Reference](https://redis.io/docs/latest/develop/ai/langcache/api-reference/)

### 5.4 Python SDK 完整示例

安装：`pip install langcache`[^sdk]

```python
from langcache import LangCache
from langcache.models import SearchStrategy
import os

# 初始化客户端
lang_cache = LangCache(
    server_url=f"https://{os.getenv('HOST')}",
    cache_id=os.getenv("CACHE_ID"),
    api_key=os.getenv("API_KEY")
)

# 1. 基础搜索（语义匹配）
res = lang_cache.search(
    prompt="用户查询文本",
    similarity_threshold=0.9
)

# 2. 带属性过滤的搜索（限定上下文）
res = lang_cache.search(
    prompt="用户查询文本",
    attributes={"tenant_id": "user_123", "locale": "zh-CN"},
    similarity_threshold=0.9,
)

# 3. 混合搜索策略（精确 + 语义）
res = lang_cache.search(
    prompt="用户查询文本",
    search_strategies=[SearchStrategy.EXACT, SearchStrategy.SEMANTIC],
    similarity_threshold=0.9,
)

# 4. 存储缓存
res = lang_cache.set(
    prompt="用户查询文本",
    response="LLM 响应文本",
)

# 5. 带属性存储（支持后续过滤）
res = lang_cache.set(
    prompt="用户查询文本",
    response="LLM 响应文本",
    attributes={"tenant_id": "user_123", "version": "v2"},
)

# 6. 删除单条缓存
lang_cache.delete_by_id(entry_id="<entryId>")

# 7. 按属性批量删除
lang_cache.delete_query(
    attributes={"tenant_id": "user_123"},
)

# 8. 清空缓存（不可逆）
lang_cache.flush()
```

### 5.5 JavaScript SDK 完整示例

安装：`npm install @redis-ai/langcache`[^sdk]

```javascript
import { LangCache } from "@redis-ai/langcache";
import { SearchStrategy } from '@redis-ai/langcache/models/searchstrategy.js';

const langCache = new LangCache({
  serverURL: "https://" + process.env.HOST,
  cacheId: process.env.CACHE_ID,
  apiKey: process.env.API_KEY,
});

// 基础搜索
const result = await langCache.search({
  prompt: "用户查询文本",
  similarityThreshold: 0.9,
});

// 带属性过滤
const resultFiltered = await langCache.search({
  prompt: "用户查询文本",
  attributes: { tenant_id: "user_123" },
  similarityThreshold: 0.9,
});

// 混合搜索策略
const resultHybrid = await langCache.search({
  prompt: "用户查询文本",
  searchStrategies: [SearchStrategy.Exact, SearchStrategy.Semantic],
  similarityThreshold: 0.9,
});

// 存储缓存
await langCache.set({
  prompt: "用户查询文本",
  response: "LLM 响应文本",
  attributes: { tenant_id: "user_123" },
});
```

### 5.6 应用层集成模式

推荐的 LangCache 集成流程：

```python
def handle_user_prompt(prompt: str, context: dict) -> str:
    # 1. 先查缓存
    cache_result = lang_cache.search(
        prompt=prompt,
        attributes=context,  # 租户、版本等上下文
        similarity_threshold=0.9,
    )
    
    if cache_result:
        # 缓存命中，直接返回
        metrics.record("cache_hit")
        return cache_result.response
    
    # 2. 缓存未命中，调用 LLM
    metrics.record("cache_miss")
    llm_response = call_llm(prompt)
    
    # 3. 存储到缓存供后续复用
    lang_cache.set(
        prompt=prompt,
        response=llm_response,
        attributes=context,
    )
    
    return llm_response
```

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

## 九、与 Mem0 的对比

### 9.1 Mem0 是什么

Mem0（mem-zero）是一个**通用 AI 记忆层**，定位为 LLM 应用提供持久化、可自我改进的记忆能力[^mem0]。它通过提取、存储和检索用户/会话/Agent 级别的结构化事实（如偏好、历史交互），实现跨对话的个性化体验。

- GitHub: [mem0ai/mem0](https://github.com/mem0ai/mem0)（53.8k Stars，Apache 2.0）
- 最新版本：2026 年 4 月发布新记忆算法，LoCoMo 基准 91.6 分（+20 分）
- 提供托管平台（Mem0 Platform）和开源自托管两种模式

### 9.2 核心定位差异

| 维度 | Redis LangCache | Mem0 |
|------|-----------------|------|
| **核心定位** | 语义缓存（Semantic Cache） | AI 记忆层（Memory Layer） |
| **主要目标** | 降低 LLM 调用成本、减少延迟 | 实现跨会话持久化上下文、个性化 |
| **存储内容** | Prompt + LLM 响应（完整对话对） | 结构化事实/偏好/关系（如"用户喜欢暗色模式"） |
| **匹配方式** | 向量语义相似度 | 多信号融合：语义 + BM25 关键词 + 实体链接 |
| **更新机制** | 覆盖/追加式缓存条目 | 自改进：自动提取、更新、关联记忆 |
| **费用优化** | 直接减少 LLM 输出 Token 调用 | 间接优化：通过精准记忆减少冗余上下文 |
| **适用场景** | 高频重复查询、FAQ、RAG 缓存 | 个性化助手、长期用户关系、Agent 状态管理 |

### 9.3 技术架构对比

| 维度 | LangCache | Mem0 |
|------|-----------|------|
| **底层存储** | Redis 向量数据库（HNSW/Flat 索引） | 可插拔：Qdrant、Weaviate、Chroma、PostgreSQL 等 |
| **Embedding** | 内置自动生成，可配置模型 | 可配置：OpenAI、HuggingFace 等 |
| **搜索策略** | 语义搜索 / 精确匹配 / 混合 | 语义 + BM25 + 实体匹配，多路召回融合 |
| **API 形式** | REST API + Python/JS SDK | Python/JS SDK + CLI + REST API |
| **部署模式** | 全托管（Redis Cloud） | 托管平台 或 开源自托管 |
| **记忆生命周期** | TTL + 驱逐策略 | 自适应更新、实体关联、记忆衰减 |

### 9.4 费用优化视角对比

**LangCache 的费用优化路径**：
- 直接拦截 LLM 调用：缓存命中时完全跳过 LLM
- 节省公式明确：`月节省 = 月输出 Token 成本 × 缓存命中率`
- 官方数据：可节省 90% API 成本，响应速度提升 4 倍
- 适合：输出 Token 成本高、查询重复度高的场景

**Mem0 的费用优化路径**：
- 间接优化：通过精准记忆检索，减少注入 LLM 的上下文 Token 量
- 单 pass 提取算法：一次 LLM 调用完成记忆提取，无 UPDATE/DELETE 循环
- 适合：需要长期维护用户状态、避免重复询问的场景（如"我之前说过我的偏好吗"）

### 9.5 选型建议

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 客服 FAQ 缓存 | LangCache | 查询重复度高，直接拦截 LLM 调用 |
| RAG 查询缓存 | LangCache | 相似问题复用检索结果 |
| 个性化 AI 助手 | Mem0 | 需要跨会话记忆用户偏好 |
| Agent 长期状态 | Mem0 | 结构化事实存储 + 实体关联 |
| 两者结合 | LangCache + Mem0 | LangCache 缓存 LLM 响应，Mem0 管理用户记忆 |

### 9.6 组合使用模式

两者并非互斥，可在同一架构中协同：

```
用户请求 → Mem0（检索用户记忆）→ 构建个性化 Prompt
         → LangCache（搜索缓存响应）
         → 缓存命中：直接返回
         → 缓存未命中：调用 LLM → 存储到 LangCache
```

这种模式下：
- Mem0 负责**个性化上下文**（用户是谁、偏好什么）
- LangCache 负责**响应复用**（相同/相似问题不再调用 LLM）
- 两者叠加可实现更高的成本节省和更好的用户体验

---

## 十、相关概念

- **Vector Database**：Redis 向量数据库能力，支持 HNSW 和 Flat 索引
- **Embedding**：文本向量表示，用于语义相似度计算
- **RAG（Retrieval-Augmented Generation）**：检索增强生成，语义缓存可优化 RAG 查询阶段
- **RedisVL**：Redis 官方 Python 库，简化向量操作
- **Mem0**：通用 AI 记忆层，与 LangCache 定位不同但可互补使用

---

## 十一、参考资料

- [Redis LangCache 官方文档](https://redis.io/docs/latest/develop/ai/langcache/)
- [Redis LangCache 产品页](https://redis.io/langcache/)
- [Redis for AI](https://redis.io/redis-for-ai/)
- [LangCache API Examples](https://redis.io/docs/latest/develop/ai/langcache/api-examples/)
- [LangCache API Reference](https://redis.io/docs/latest/develop/ai/langcache/api-reference/)
- [LangCache 成本计算器](https://redis.io/calculator/langcache/)
- [LangCache on Redis Cloud](https://redis.io/docs/latest/operate/rc/langcache/)
- [Mem0 官方文档](https://docs.mem0.ai/)
- [Mem0 GitHub 仓库](https://github.com/mem0ai/mem0)

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
[^calc]: 成本计算器数据来源于 [LangCache Calculator](https://redis.io/calculator/langcache/)
[^sdk]: SDK 示例来源于 [LangCache API Examples](https://redis.io/docs/latest/develop/ai/langcache/api-examples/)
[^mem0]: Mem0 信息来源于 [Mem0 官方文档](https://docs.mem0.ai/) 和 [GitHub 仓库](https://github.com/mem0ai/mem0)
