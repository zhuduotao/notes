---
created: 2026-04-22
updated: 2026-04-22
tags:
  - ai
  - llm
  - prompt-caching
  - cost-optimization
  - openai
  - anthropic
  - google
  - gemini
  - claude
  - llm-pricing
aliases:
  - LLM Prompt Caching
  - LLM 缓存命中
  - Prompt Caching 折扣
  - Cached Tokens Discount
  - Context Caching
source_type: official-doc
source_urls:
  - https://platform.openai.com/docs/guides/prompt-caching
  - https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
  - https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview
  - https://openai.com/api/pricing/
status: verified
---

## 概述

LLM Provider 原生缓存（Prompt Caching / Context Caching）是指 OpenAI、Anthropic、Google 等 LLM 服务提供商在 API 层内置的缓存机制。当多次请求包含相同的 Prompt 前缀时，Provider 会复用已处理的缓存，**对缓存命中的输入 Token 给予大幅折扣**（通常为原价的 10%~50%），同时降低响应延迟。

这与应用层自建缓存（如 Redis 语义缓存）不同：Provider 原生缓存由模型服务方管理，无需应用层维护缓存存储，且缓存的是模型内部的前缀计算状态，而非简单的文本响应。

---

## 一、主流 Provider 缓存方案对比

### 1.1 核心特性对比

| 维度 | OpenAI Prompt Caching | Anthropic Prompt Caching | Google Context Caching |
|------|----------------------|-------------------------|----------------------|
| **折扣力度** | 缓存命中 Token 为原价的 **50%** | 缓存命中 Token 为原价的 **10%** | 显式缓存：2.5 模型 **90%** 折扣，2.0 模型 **75%** 折扣；隐式缓存 **90%** 折扣 |
| **启用方式** | 自动启用，无需配置 | 需在请求中设置 `cache_control` 标记断点 | 隐式：默认启用；显式：通过 API 创建 `cachedContent` |
| **缓存粒度** | 自动检测公共前缀 | 手动设置 breakpoint（最多 4 个） | 显式：创建独立的 `cachedContent` 资源 |
| **默认 TTL** | 5~10 分钟（动态） | 5 分钟（可选 1 小时，额外费用） | 显式：60 分钟（可延长）；隐式：动态 |
| **最小缓存长度** | 1024 tokens | 模型相关：1024~4096 tokens | 显式：32,768 tokens（32K） |
| **确认命中方式** | `usage.prompt_token_details.cached_tokens` | `usage.cache_read_input_tokens` | `usageMetadata.cachedContentTokenCount` |
| **缓存隔离** | 按 API Key 隔离 | 按 Workspace 隔离（2026-02-05 起） | 按 Project/Region 隔离 |

### 1.2 适用场景

| 场景 | 推荐 Provider | 原因 |
|------|-------------|------|
| 多轮对话 Agent | Anthropic | 自动缓存模式随对话增长自动移动断点，最适合 growing conversation |
| 长文档分析 | Google | 显式缓存支持 32K+ tokens，适合超长上下文复用 |
| 高频重复查询 | OpenAI | 自动检测前缀，无需手动配置，接入成本最低 |
| 工具调用频繁 | Anthropic | 支持 tools/system/messages 分层缓存，tools 可独立缓存 |
| 成本敏感型 | Google | 90% 折扣力度最大，但需满足 32K 最小长度门槛 |

---

## 二、各 Provider 详细方案

### 2.1 OpenAI Prompt Caching

**机制**：OpenAI 自动检测请求之间的公共前缀，当检测到重复前缀时自动应用缓存，无需任何配置。

**折扣**：缓存命中的输入 Token 按原价的 **50%** 计费。

**确认命中**：检查响应中的 `usage` 字段：

```json
{
  "usage": {
    "prompt_tokens": 1500,
    "prompt_token_details": {
      "cached_tokens": 1024
    },
    "completion_tokens": 200,
    "total_tokens": 1700
  }
}
```

- `prompt_token_details.cached_tokens` > 0 表示缓存命中
- 实际计费输入 Token = `prompt_tokens - cached_tokens + cached_tokens * 0.5`

**限制**：
- 最小缓存长度：1024 tokens
- 缓存 TTL：5~10 分钟（系统动态管理）
- 缓存隔离：按 API Key 隔离
- 不支持手动控制缓存断点

**最佳实践**：
- 将不变的 System Prompt、工具定义、长上下文放在消息列表的**最前面**
- 确保前缀内容在多次请求间保持一致
- 高频请求场景下，尽量在 TTL 窗口内发送请求

### 2.2 Anthropic Prompt Caching

**机制**：通过在请求中设置 `cache_control` 标记缓存断点（breakpoint），系统缓存从开头到断点的全部内容。支持两种模式：

1. **自动缓存（Automatic Caching）**：在请求顶层设置 `cache_control`，系统自动将断点应用到最后一个可缓存块，随对话增长自动前移
2. **显式断点（Explicit Cache Breakpoints）**：在具体的 content block 上设置 `cache_control`，精确控制缓存边界

**折扣**：

| 缓存操作 | 价格倍数 | 说明 |
|---------|---------|------|
| Cache Reads（命中） | **0.1x** 基础输入价格 | 即原价的 10% |
| Cache Writes（5m TTL） | 1.25x 基础输入价格 | 首次写入，比基础价贵 25% |
| Cache Writes（1h TTL） | 2.0x 基础输入价格 | 长时效写入，为基础价的 2 倍 |
| 普通输入 Token | 1.0x 基础输入价格 | 断点之后的内容 |

**确认命中**：检查响应中的 `usage` 字段：

```json
{
  "usage": {
    "input_tokens": 50,
    "cache_read_input_tokens": 100000,
    "cache_creation_input_tokens": 0,
    "output_tokens": 200,
    "cache_creation": {
      "ephemeral_5m_input_tokens": 0,
      "ephemeral_1h_input_tokens": 0
    }
  }
}
```

- `cache_read_input_tokens` > 0 表示缓存命中，值为命中的 Token 数
- `cache_creation_input_tokens` > 0 表示正在写入新缓存
- `input_tokens` 是断点之后、不参与缓存的 Token 数
- 总输入 Token = `cache_read_input_tokens + cache_creation_input_tokens + input_tokens`

**自动缓存在多轮对话中的行为**：

| 请求 | 内容 | 缓存行为 |
|------|------|---------|
| Request 1 | System + User(1) + Asst(1) + **User(2)** ◀ cache | 全部内容写入缓存 |
| Request 2 | System + User(1) + Asst(1) + User(2) + Asst(2) + **User(3)** ◀ cache | System 到 User(2) 从缓存读取；Asst(2) + User(3) 写入新缓存 |
| Request 3 | System + ... + User(3) + Asst(3) + **User(4)** ◀ cache | System 到 User(3) 从缓存读取；Asst(3) + User(4) 写入新缓存 |

**最小缓存长度**（按模型）：

| 模型 | 最小 Token 数 |
|------|-------------|
| Claude Opus 4.7/4.6/4.5 | 4096 |
| Claude Sonnet 4.6 | 2048 |
| Claude Sonnet 4.5/4.1/4, Haiku 4.5 | 1024 |
| Claude Haiku 3.5 | 2048 |
| Claude Haiku 3 | 1024 |

**重要限制**：
- 最多 4 个缓存断点
- Lookback window 为 20 个 blocks：系统最多回退 20 个 block 查找缓存匹配
- 缓存断点必须放在**跨请求保持不变的最后一个 block** 上
- 如果断点放在每次都会变化的 block（如时间戳、用户消息），则永远不会命中缓存
- 修改 tool definitions 会**使整个缓存失效**（tools → system → messages 层级连锁失效）
- 缓存写入在首次响应开始后才可用，并发请求需等待首个响应完成

**TTL 选项**：

```json
// 5 分钟 TTL（默认，免费刷新）
{ "cache_control": { "type": "ephemeral" } }

// 1 小时 TTL（2x 写入价格）
{ "cache_control": { "type": "ephemeral", "ttl": "1h" } }
```

**自动缓存示例**（Python）：

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # 顶层设置，自动缓存
    system="你是一个分析文学作品的 AI 助手。",
    messages=[
        {"role": "user", "content": "分析《傲慢与偏见》的主要主题。"},
    ],
)
print(response.usage)
```

**显式断点示例**（多段缓存）：

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {"type": "text", "text": "你是 AI 助手。"},
        {
            "type": "text",
            "text": "以下是工具定义：[大量工具描述文本]",
            "cache_control": {"type": "ephemeral"},  # 断点 1：缓存 system
        },
    ],
    messages=[
        {"role": "user", "content": "用户消息"},
    ],
)
```

### 2.3 Google Context Caching

Google 提供两种缓存模式：

#### 隐式缓存（Implicit Caching）

- **默认启用**，无需任何配置
- 折扣：缓存命中 Token 为原价的 **10%**（即 90% 折扣）
- 无存储成本
- 提升命中率：将大型公共内容放在 Prompt 开头，短时间内发送相似前缀请求

#### 显式缓存（Explicit Caching）

- 通过 Vertex AI API 创建 `cachedContent` 资源
- 折扣：Gemini 2.5+ 模型 **90%** 折扣，Gemini 2.0 模型 **75%** 折扣
- 有存储成本（按缓存时长计费）
- 默认 TTL 60 分钟，可延长
- 最小长度：32,768 tokens（32K）

**确认命中**：检查响应中的 `usageMetadata` 字段：

```json
{
  "usageMetadata": {
    "promptTokenCount": 50000,
    "candidatesTokenCount": 200,
    "totalTokenCount": 50200,
    "cachedContentTokenCount": 49000
  }
}
```

- `cachedContentTokenCount` > 0 表示缓存命中
- 该值表示从缓存中复用的 Token 数量

**创建显式缓存**（Python）：

```python
from google.cloud import aiplatform
from vertexai.generative_models import GenerativeModel, Content, Part

aiplatform.init(project="your-project", location="us-central1")

# 创建缓存内容
cached_content = aiplatform.VertexAIContextCaching.create_cached_content(
    display_name="my-context-cache",
    model="gemini-2.5-pro",
    contents=[
        Content(role="user", parts=[Part.from_text("长文档内容...")]),
    ],
    ttl_hours=2,  # 2 小时 TTL
)

# 使用缓存进行推理
model = GenerativeModel("gemini-2.5-pro")
response = model.generate_content(
    contents=[Content(role="user", parts=[Part.from_text("基于文档回答问题")])],
    cached_content=cached_content.resource_name,
)
```

---

## 三、如何确认缓存命中

### 3.1 各 Provider 响应字段对照

| Provider | 缓存命中字段 | 缓存写入字段 | 普通输入字段 |
|----------|------------|------------|------------|
| **OpenAI** | `usage.prompt_token_details.cached_tokens` | 无单独字段 | `usage.prompt_tokens` |
| **Anthropic** | `usage.cache_read_input_tokens` | `usage.cache_creation_input_tokens` | `usage.input_tokens` |
| **Google** | `usageMetadata.cachedContentTokenCount` | 创建 `cachedContent` 时返回 | `usageMetadata.promptTokenCount` |

### 3.2 通用检测方法

**方法一：检查响应字段**

每次 API 调用后检查对应的缓存字段，非零即表示命中：

```python
# OpenAI
if response.usage.prompt_token_details.cached_tokens > 0:
    print(f"缓存命中: {response.usage.prompt_token_details.cached_tokens} tokens")

# Anthropic
if response.usage.cache_read_input_tokens > 0:
    print(f"缓存命中: {response.usage.cache_read_input_tokens} tokens")

# Google
if response.usage_metadata.cached_content_token_count > 0:
    print(f"缓存命中: {response.usage_metadata.cached_content_token_count} tokens")
```

**方法二：成本对比验证**

通过对比实际扣费与预期费用，间接验证缓存命中：
- 记录请求的总输入 Token 数
- 检查账单中该请求的实际费用
- 若费用明显低于 `总Token × 单价`，则缓存已生效

**方法三：延迟对比**

缓存命中时，Time-To-First-Token (TTFT) 通常显著降低：
- Anthropic 官方数据：长文档场景下 TTFT 可改善
- 可通过记录响应时间分布来辅助判断

### 3.3 缓存未命中的常见原因排查

| 现象 | 可能原因 | 排查方法 |
|------|---------|---------|
| `cache_read_input_tokens` = 0 | 前缀内容发生变化 | 对比两次请求的 System Prompt 和前置消息 |
| `cache_read_input_tokens` = 0 | 超过 TTL | 检查两次请求间隔是否超过 5 分钟（Anthropic）或 60 分钟（Google） |
| `cache_read_input_tokens` = 0 | 未达最小长度 | 检查缓存前缀是否达到模型要求的最小 Token 数 |
| `cache_read_input_tokens` = 0 | 并发请求 | 首个请求还在写入缓存时，后续请求无法命中 |
| Anthropic 全缓存失效 | 修改了 tool definitions | tools 层变更会使整个缓存链失效 |
| Anthropic 全缓存失效 | `tool_choice` 变更 | 影响 message 层缓存 |
| Anthropic 全缓存失效 | 启用/禁用了 web search 或 citations | 影响 system 层缓存 |

---

## 四、如何提高缓存命中率

### 4.1 Prompt 结构设计

**核心原则：将不变的内容放在最前面，变化的内容放在最后面。**

```
推荐结构：
┌─────────────────────────────────┐
│  System Prompt（不变）            │  ← 缓存断点应设在这里或附近
│  Tool Definitions（很少变）       │
│  背景知识 / 长文档（不变）         │
├─────────────────────────────────┤
│  用户消息（每次变化）             │  ← 不参与缓存
└─────────────────────────────────┘
```

**错误示例**：将时间戳、session ID 等动态内容放在 System Prompt 中：

```json
// ❌ 错误：每次请求 system 都不同，缓存永远不会命中
{
  "system": "[2026-04-22 10:30:00] 你是一个助手。Session: abc123",
  "messages": [{"role": "user", "content": "你好"}]
}

// ✅ 正确：system 保持不变
{
  "system": "你是一个助手。",
  "messages": [{"role": "user", "content": "你好"}]
}
```

### 4.2 Anthropic 专属优化策略

**策略一：使用自动缓存模式处理多轮对话**

```python
# 自动缓存：只需在顶层设置 cache_control，断点自动跟随对话增长
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},  # 一行启用
    system="你是助手。",
    messages=conversation_history,  # 随对话增长，断点自动前移
)
```

**策略二：对变化频率不同的内容使用多个断点**

```python
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {"type": "text", "text": "核心指令（几乎不变）", "cache_control": {"type": "ephemeral"}},
        {"type": "text", "text": "每日更新的背景信息", "cache_control": {"type": "ephemeral"}},
    ],
    messages=[...],
)
```

**策略三：确保断点放在不变的 block 上**

```python
# ❌ 错误：断点在每次变化的用户消息上
messages=[
    {"role": "user", "content": f"当前时间: {now()}\n{user_question}",
     "cache_control": {"type": "ephemeral"}}  # 时间戳变化导致永不命中
]

# ✅ 正确：断点在不变的 system 上
system=[
    {"type": "text", "text": "你是助手。", "cache_control": {"type": "ephemeral"}}
],
messages=[
    {"role": "user", "content": f"当前时间: {now()}\n{user_question}"}
]
```

**策略四：利用 1 小时缓存应对低频场景**

```python
# 当请求间隔可能超过 5 分钟时，使用 1 小时 TTL
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    cache_control={"type": "ephemeral", "ttl": "1h"},  # 1 小时 TTL
    system="你是助手。",
    messages=[...],
)
```

### 4.3 Google 专属优化策略

**策略一：确保显式缓存达到 32K 最小长度**

显式缓存要求至少 32,768 tokens。如果内容不足：
- 组合多个文档/上下文一起缓存
- 或使用隐式缓存（无最小长度要求）

**策略二：合理使用 TTL**

```python
# 延长缓存 TTL 以覆盖更长的使用周期
cached_content = aiplatform.VertexAIContextCaching.create_cached_content(
    display_name="long-lived-cache",
    model="gemini-2.5-pro",
    contents=[...],
    ttl_hours=6,  # 最长可设置更长时间
)
```

**策略三：利用隐式缓存的零配置优势**

对于无法预测的重复查询，依赖隐式缓存：
- 将大型公共内容放在 Prompt 开头
- 在短时间内集中发送相似前缀的请求

### 4.4 OpenAI 专属优化策略

**策略一：确保前缀达到 1024 tokens 最小长度**

OpenAI 自动缓存要求前缀至少 1024 tokens。如果 System Prompt 较短：
- 在 System Prompt 中添加更多背景信息、示例
- 或将常用上下文前置到 messages 列表的前面

**策略二：在 5~10 分钟 TTL 窗口内发送请求**

OpenAI 缓存 TTL 较短，高频场景下效果最佳。

### 4.5 通用最佳实践

| 实践 | 说明 |
|------|------|
| **前置不变内容** | System Prompt、工具定义、长文档放最前 |
| **后置变化内容** | 用户消息、时间戳、动态参数放最后 |
| **避免动态 System** | 不要在 System Prompt 中嵌入时间、session ID 等 |
| **监控命中率** | 定期检查 `cache_read_input_tokens / total_input_tokens` 比率 |
| **批量预热** | 对已知高频查询，先发送一次请求写入缓存 |
| **工具定义稳定** | 避免频繁修改 tool definitions（Anthropic 中会使全缓存失效） |
| **JSON key 顺序稳定** | 某些语言（Swift、Go）随机化 JSON key 顺序，会破坏缓存匹配 |

### 4.6 命中率计算公式

```
缓存命中率 = cache_read_input_tokens / total_input_tokens × 100%

其中：
total_input_tokens = cache_read_input_tokens + cache_creation_input_tokens + input_tokens
```

**示例**：
- `cache_read_input_tokens`: 100,000
- `cache_creation_input_tokens`: 0
- `input_tokens`: 50
- 命中率 = 100,000 / 100,050 ≈ **99.95%**

---

## 五、成本计算示例

### 5.1 Anthropic 成本计算

假设使用 Claude Sonnet 4（基础输入 $3/MTok）：

| 场景 | 计算 | 费用 |
|------|------|------|
| 无缓存：100K 输入 Token | 100,000 × $3/1,000,000 | $0.30 |
| 缓存命中：100K 缓存读取 + 50 普通输入 | 100,000 × $0.30/1,000,000 + 50 × $3/1,000,000 | $0.03015 |
| **节省** | | **$0.26985（约 90%）** |

首次写入缓存时：
- Cache Write（5m）：100,000 × $3.75/1,000,000 = $0.375
- 后续每次命中：100,000 × $0.30/1,000,000 = $0.03
- 第 2 次请求开始即回本

### 5.2 OpenAI 成本计算

假设使用 GPT-4o（输入 $2.50/MTok）：

| 场景 | 计算 | 费用 |
|------|------|------|
| 无缓存：100K 输入 Token | 100,000 × $2.50/1,000,000 | $0.25 |
| 缓存命中：50K 缓存 + 50K 普通 | 50,000 × $1.25/1,000,000 + 50,000 × $2.50/1,000,000 | $0.1875 |
| **节省** | | **$0.0625（25%）** |

### 5.3 Google 成本计算

假设使用 Gemini 2.5 Pro（输入 $1.25/MTok）：

| 场景 | 计算 | 费用 |
|------|------|------|
| 无缓存：100K 输入 Token | 100,000 × $1.25/1,000,000 | $0.125 |
| 显式缓存命中：100K 缓存读取 | 100,000 × $0.125/1,000,000 | $0.0125 |
| **节省** | | **$0.1125（90%）** |

---

## 六、限制与注意事项

### 6.1 通用限制

| 限制 | 说明 |
|------|------|
| **前缀匹配** | 所有 Provider 的缓存都基于**前缀匹配**，不是全文匹配 |
| **精确匹配** | 缓存要求内容 100% 相同，包括空格、换行、标点 |
| **不影响输出** | 缓存仅影响输入 Token 的处理，输出 Token 的生成质量和费用不变 |
| **隔离性** | 缓存按组织/Workspace/Project 隔离，不同租户不共享 |

### 6.2 Anthropic 特有限制

- 最多 4 个缓存断点
- Lookback window 仅 20 blocks，对话过长时需添加额外断点
- 修改 tool definitions 使全缓存失效
- 修改 `tool_choice` 使 message 层缓存失效
- 启用/禁用 web search 或 citations 使 system 层缓存失效
- Thinking blocks 不能直接用 `cache_control` 标记，但可随其他内容一起被缓存
- 空 text block 无法被缓存
- 缓存写入在首次响应开始后才可用

### 6.3 Google 特有限制

- 显式缓存最小 32,768 tokens，门槛较高
- 显式缓存有存储成本
- 隐式缓存无法手动控制，命中率不确定

### 6.4 OpenAI 特有限制

- 无法手动控制缓存断点
- TTL 较短（5~10 分钟），低频场景效果有限
- 折扣力度（50%）低于 Anthropic（90%）和 Google（90%）

---

## 七、与自建语义缓存的关系

| 维度 | Provider 原生缓存 | 自建语义缓存（如 Redis LangCache） |
|------|------------------|--------------------------------|
| **缓存内容** | 模型前缀计算状态 | 完整的 Prompt + Response 对 |
| **匹配方式** | 精确前缀匹配 | 向量语义相似度匹配 |
| **折扣** | 输入 Token 10%~50% 价格 | 完全跳过 LLM 调用（100% 节省输出 Token） |
| **运维** | 零运维，Provider 管理 | 需自建和维护缓存服务 |
| **适用场景** | 相同前缀的重复请求 | 语义相似但表述不同的查询 |
| **可组合** | ✅ 两者可同时使用 | ✅ |

**推荐架构**：两者互补使用

```
用户请求
  → 自建语义缓存（Redis）：语义匹配，命中则直接返回 Response
  → 未命中 → LLM Provider API
    → Provider 原生缓存：前缀匹配，命中则输入 Token 打折
    → 生成 Response → 存入自建语义缓存
```

---

## 八、相关概念

- **Semantic Cache（语义缓存）**：应用层基于向量相似度的缓存，如 Redis LangCache
- **TTFT（Time-To-First-Token）**：首 Token 延迟，缓存命中时通常降低
- **Context Window**：模型的上下文窗口大小，影响可缓存内容的上限
- **Token**：LLM 处理文本的基本单位
- **RAG（Retrieval-Augmented Generation）**：检索增强生成，其检索结果可作为缓存前缀

---

## 九、参考资料

- [OpenAI Prompt Caching 文档](https://platform.openai.com/docs/guides/prompt-caching)
- [OpenAI Pricing](https://openai.com/api/pricing/)
- [Anthropic Prompt Caching 文档](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Anthropic Pricing](https://docs.anthropic.com/en/docs/build-with-claude/pricing)
- [Google Context Caching Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-overview)
- [Google Context Cache Create](https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/context-cache-create)
- [Google Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing#context-caching)
