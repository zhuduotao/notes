---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - AI
  - RAG
  - query-transformation
  - HyDE
  - query-rewriting
  - retrieval-optimization
  - LLM
aliases:
  - Query Transformation
  - Query Rewriting
  - 查询改写
  - HyDE
source_type: mixed
source_urls:
  - 'https://arxiv.org/abs/2305.14283'
  - 'https://arxiv.org/abs/2212.10496'
  - 'https://arxiv.org/abs/2312.10997'
  - >-
    https://docs.llamaindex.ai/en/stable/optimizing/advanced_retrieval/query_transformations/
status: verified
---
## 概述

Query 改写（Query Transformation / Query Rewriting）是 RAG 检索增强策略的核心技术之一。用户原始查询与知识库中的文档之间存在语义鸿沟——原始查询可能表述模糊、术语不匹配、或包含多意图，导致向量检索效果不佳。Query 改写通过 LLM 对原始查询进行变换，生成更适合检索的形式，从而提升召回质量。

**核心问题**：用户查询 q 与目标文档 d 之间的语义相似度 ≠ q 与 d 在向量空间中的距离[^1]。改写旨在缩小这一差距。

---

## 主要改写策略

### HyDE（Hypothetical Document Embeddings）

**原理**：不直接对查询向量检索，而是先用 LLM 生成一个"假设文档"，再对该假设文档进行 Embedding 检索[^2]。

**流程**：
```
用户查询 → LLM 生成假设文档 → Embedding 假设文档 → 向量检索 → 返回真实文档
```

**依据**：假设文档虽然可能包含错误细节，但其语义模式（relevance patterns）与真实相关文档相近。Embedding 模型的瓶颈特性会过滤掉不正确的细节[^2]。

**优势**：
- 无需 relevance labels，zero-shot 适用
- 对抽象查询、概念性问题效果好
- 论文实验显示显著优于无监督 baseline Contriever[^2]

**局限**：
- 增加一次 LLM 调用，延迟和成本上升
- 假设文档质量依赖 LLM 能力
- 对事实性查询可能引入幻觉误导

**适用场景**：
- 概念性、抽象性问题（如"什么是分布式系统的核心设计原则"）
- 用户表述与文档术语差异大的场景
- Zero-shot 冷启动阶段

**LlamaIndex 实现**[^3]：
```python
from llama_index.core.indices.query.query_transform.base import HyDEQueryTransform
from llama_index.core.query_engine import TransformQueryEngine

hyde = HyDEQueryTransform(include_original=True)
query_engine = TransformQueryEngine(base_engine, query_transform=hyde)
```

---

### Rewrite-Retrieve-Read

**原理**：将传统 Retrieve-Then-Read 流程改为 Rewrite-Retrieve-Read，先改写查询再检索[^1]。

**流程对比**：
```
传统：Query → Retrieve → Read
改写：Query → Rewrite → Retrieve → Read
```

**核心洞察**：用户输入文本与所需知识之间存在 inevitable gap[^1]。改写可以弥合这一差距。

**训练方案**：论文提出 trainable rewriter[^1]：
- 用小型 LM 作为可训练改写器
- 通过 RL（Reinforcement Learning）利用 LLM reader 的反馈训练
- 适应 frozen modules（冻结的检索器和 LLM）

**实验结果**[^1]：
- 在 open-domain QA 和 multiple-choice QA 上持续提升
- EMNLP 2023 收录

**适用场景**：
- 需要针对特定领域优化改写效果
- 有足够数据训练小型 rewriter
- 追求长期性能优化的生产系统

---

### Multi-Query

**原理**：从原始查询生成多个相关变体查询，分别检索后合并结果[^4]。

**流程**：
```
原始查询 → LLM 生成多个变体查询 → 每个查询独立检索 → 结果融合
```

**目标**：覆盖用户意图的不同表述方式，提高召回率。

**LangChain 实现**[^4]：
```python
from langchain.retrievers.multi_query import MultiQueryRetriever

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm
)
# 自动生成多个查询变体并检索
docs = retriever.get_relevant_documents(query)
```

**结果融合策略**：
- **Reciprocal Rank Fusion (RRF)**：按排名倒数加权合并
- **Simple Concat + Dedup**：简单拼接去重
- **Score Fusion**：按相似度分数加权

**优势**：
- 提高召回覆盖度
- 减少单一查询表述偏差的影响
- 无需额外训练

**局限**：
- 多次检索，延迟倍增
- 可能引入噪声（变体查询偏离原意图）
- LLM 生成变体的质量不稳定

**适用场景**：
- 用户表述多样化场景
- 对召回率要求高，延迟容忍度较高
- 复杂查询难以用单一表述覆盖

---

### Query Decomposition（查询分解 / Multi-Step）

**原理**：将复杂查询分解为多个子问题，逐一检索回答[^3]。

**流程**：
```
复杂查询 → LLM 分解为子问题 → 每个子问题检索 → 逐步回答 → 最终整合
```

**适用情况**：
- 查询包含多个子意图（如"比较 A 和 B 的优缺点并给出选择建议"）
- 需要跨多个文档或主题综合回答
- 单次检索无法获取完整信息

**LlamaIndex 实现**[^3]：
```python
from llama_index.core.indices.query.query_transform.base import StepDecomposeQueryTransform
from llama_index.core.query_engine import MultiStepQueryEngine

step_decompose = StepDecomposeQueryTransform(llm, verbose=True)
query_engine = MultiStepQueryEngine(base_engine, query_transform=step_decompose)
```

**优势**：
- 处理复杂查询能力强
- 可追溯推理链路
- 提高答案完整度

**局限**：
- 多轮 LLM + 检索调用，延迟累积
- 子问题分解质量依赖 LLM
- 可能过度分解导致信息碎片化

**适用场景**：
- 多意图复杂查询
- 需要综合多文档信息的问答
- 需要推理链路的场景

---

### Query Expansion

**原理**：扩展查询词，增加同义词、相关术语或上下文信息[^5]。

**方法**：
- **同义词扩展**：添加领域术语的同义词或别名
- **关键词提取**：从查询中提取关键词并扩展
- **上下文注入**：结合用户历史或文档上下文补充查询

**优势**：
- 弥合术语鸿沟
- 成本较低（无需多次 LLM 调用）
- 可与规则或小型模型结合

**局限**：
- 扩展词可能引入噪声
- 需要领域词汇库支持
- 效果依赖扩展策略质量

---

## 策略对比

| 策略 | 核心思路 | LLM 调用次数 | 检索次数 | 延迟 | 适用场景 |
|------|---------|-------------|---------|------|---------|
| HyDE | 生成假设文档检索 | 1 | 1 | 中 | 抽象查询、术语鸿沟 |
| Rewrite | 改写后检索 | 1 | 1 | 中 | 通用改写优化 |
| Multi-Query | 多变体并行检索 | 1 | N | 高 | 多意图覆盖 |
| Decomposition | 分解子问题逐步检索 | N | N | 最高 | 复杂多意图查询 |
| Expansion | 扩展关键词 | 0-1 | 1 | 低 | 术语匹配优化 |

---

## 实现框架支持

### LlamaIndex

| 模块 | 功能 |
|------|------|
| `HyDEQueryTransform` | HyDE 改写 |
| `StepDecomposeQueryTransform` | 多步分解 |
| `TransformQueryEngine` | 包装改写 + 检索 |
| `MultiStepQueryEngine` | 多步查询引擎 |

### LangChain

| 模块 | 功能 |
|------|------|
| `MultiQueryRetriever` | 多查询检索 |
| `ContextualCompressionRetriever` | 上下文压缩 |
| 自定义 Chain | Query rewrite + retrieval 组合 |

---

## 限制与注意事项

### 性能与成本权衡

改写策略增加 LLM 调用和检索次数，需权衡：
- **延迟敏感场景**：优先 Expansion 或简单 Rewrite
- **质量敏感场景**：HyDE 或 Decomposition
- **成本敏感场景**：减少调用次数，使用小型 LLM

### 改写质量风险

- **幻觉引入**：HyDE 生成的假设文档可能包含错误信息
- **意图偏离**：Multi-Query 变体可能偏离原意图
- **过度分解**：Decomposition 可能拆分过细

### 适用边界

并非所有查询都需要改写：
- **简单精确查询**（如"Python 3.11 新特性"）：直接检索效果可能更好
- **术语高度匹配场景**：改写收益有限
- **实时性要求场景**：改写延迟不可接受

---

## 最佳实践建议

1. **按查询类型选择策略**：
   - 抽象概念 → HyDE
   - 多意图复杂 → Decomposition
   - 术语不匹配 → Expansion
   - 表述多样性高 → Multi-Query

2. **组合使用**：
   - Rewrite + HyDE 组合
   - Multi-Query + Rerank 组合
   - Expansion + Hybrid Search 组合

3. **评估验证**：
   - 使用 RAG 评估框架（Ragas、LlamaIndex Evaluation）验证改写效果
   - 对比改写前后 Recall、Precision、Answer Relevancy

4. **成本控制**：
   - 改写查询缓存（相同或相似查询复用改写结果）
   - 使用小型 LLM（GPT-3.5 / Claude Haiku）执行改写
   - 条件触发（仅对复杂或低召回查询触发改写）

---

## 参考资料

[^1]: Ma, X., et al. (2023). "Query Rewriting for Retrieval-Augmented Large Language Models". EMNLP 2023. https://arxiv.org/abs/2305.14283

[^2]: Gao, L., et al. (2022). "Precise Zero-Shot Dense Retrieval without Relevance Labels". https://arxiv.org/abs/2212.10496

[^3]: LlamaIndex Documentation. "Query Transformations". https://docs.llamaindex.ai/en/stable/optimizing/advanced_retrieval/query_transformations/

[^4]: LangChain MultiQueryRetriever. https://api.python.langchain.com/en/latest/retrievers/langchain.retrievers.multi_query.MultiQueryRetriever.html

[^5]: Gao, Y., et al. (2024). "Retrieval-Augmented Generation for Large Language Models: A Survey". https://arxiv.org/abs/2312.10997
