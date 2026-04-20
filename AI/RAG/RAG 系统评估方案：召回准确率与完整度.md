---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - AI
  - RAG
  - evaluation
  - metrics
  - observability
  - production
aliases:
  - RAG评估
  - RAG evaluation
  - RAG召回率
  - RAG完整度
source_type: mixed
source_urls:
  - 'https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/'
  - 'https://docs.llamaindex.ai/en/stable/module_guides/evaluating/'
  - 'https://arxiv.org/abs/2309.15217'
  - 'https://trulens.org/'
  - 'https://docs.arize.com/phoenix'
status: verified
---
## 概述

RAG（Retrieval-Augmented Generation）系统的评估涉及多个维度，核心包括：**召回准确率**（是否检索到正确内容）、**完整度**（检索内容是否充分）、**忠实度**（生成是否忠实于检索内容）、**答案相关性**（生成是否回答了问题）。

生产环境评估的挑战在于：
- 缺乏人工标注的 ground truth
- 评估维度多元（检索 + 生成）
- 需要可规模化、自动化的评估方案
- 需要追踪和监控能力

---

## 核心评估指标

### 检索侧指标

#### Context Recall（上下文召回率）

衡量检索到的上下文是否覆盖了回答问题所需的所有关键信息[^1]。

**计算方式**：
- 使用 LLM 从 ground truth answer 中提取关键陈述
- 判断每个陈述是否可以从检索到的上下文中推断出来
- Recall = 可推断陈述数 / 总陈述数

**适用场景**：有 ground truth answer 时使用。

#### Context Precision（上下文精确度）

衡量检索到的上下文是否与问题相关，是否存在无关噪声[^1]。

**计算方式**：
- 对每个检索到的 chunk，使用 LLM 判断其是否与问题相关
- Precision = 相关 chunk 数 / 检索到的 chunk 总数

**适用场景**：评估检索系统是否返回过多无关内容。

#### Hit Rate（命中率）

传统检索评估指标[^2]。

**计算方式**：
- 对于每个查询，检查检索结果中是否包含正确的文档
- Hit Rate = 包含正确文档的查询数 / 总查询数

**适用场景**：有 ground truth documents 时使用。

#### MRR（Mean Reciprocal Rank）

衡量正确文档在检索结果中的排名[^2]。

**计算方式**：
- 对于每个查询，取正确文档排名的倒数（如排名第 2，则 reciprocal rank = 1/2）
- MRR = 所有查询 reciprocal rank 的平均值

**适用场景**：有 ground truth documents 时使用。

### 生成侧指标

#### Faithfulness（忠实度）

衡量生成的答案是否忠实于检索到的上下文，是否存在幻觉[^1][^3]。

**计算方式**：
- 从生成答案中提取所有陈述
- 使用 LLM 判断每个陈述是否可以从检索上下文中推断
- Faithfulness = 可推断陈述数 / 总陈述数

**适用场景**：无需 ground truth，仅依赖检索上下文。

#### Answer Relevancy（答案相关性）

衡量生成的答案是否直接回答了用户问题[^1]。

**计算方式**：
- 使用 LLM 根据生成答案生成可能的问题
- 计算生成问题与原始问题的语义相似度
- Relevancy = 问题相似度的平均值

**适用场景**：评估答案是否偏离问题主题。

#### Context Relevancy（上下文相关性）

衡量检索到的上下文是否与用户问题相关[^3]。

**计算方式**：
- 使用 LLM 从检索上下文中提取与问题相关的句子
- Relevancy = 相关句子数 / 上下文总句子数

**适用场景**：评估检索内容是否聚焦问题。

### 指标关系总结

| 维度 | 指标 | 评估对象 | 需要 Ground Truth | 核心问题 |
|------|------|---------|-------------------|---------|
| 召召 | Context Recall | 检索结果 | 需要 answer | 是否覆盖所有关键信息？ |
| 精度 | Context Precision | 检索结果 | 不需要 | 是否包含过多噪声？ |
| 召回 | Hit Rate / MRR | 检索结果 | 需要 documents | 是否检索到正确文档？ |
| 完整度 | Faithfulness | 生成答案 | 不需要 | 是否忠实于上下文？ |
| 相关性 | Answer Relevancy | 生成答案 | 不需要 | 是否回答了问题？ |

---

## 主流评估框架

### Ragas

**定位**：参考无关的 RAG 评估框架[^1]。

**核心特性**：
- 无需人工标注 ground truth（除 Context Recall）
- 使用 LLM-as-a-Judge 方法
- 支持合成测试数据生成

**主要指标**：
- Context Precision / Recall
- Faithfulness
- Answer Relevancy
- Noise Sensitivity

**适用场景**：
- 快速迭代评估
- 无标注数据时的自动化评估

```python
from ragas import evaluate
from ragas.metrics import context_precision, faithfulness, answer_relevancy

result = evaluate(
    dataset,
    metrics=[context_precision, faithfulness, answer_relevancy]
)
```

### TruLens

**定位**：AI 应用的追踪和评估平台[^3]。

**核心特性**：
- 支持 OpenTelemetry traces
- 提供 Groundedness、Context Relevance、Answer Relevance
- 集成 LlamaIndex、LangChain

**RAG Triad**[^3]：
- Groundedness（答案是否有依据）
- Context Relevance（上下文是否相关）
- Answer Relevance（答案是否回答问题）

**适用场景**：
- 生产环境追踪和监控
- 实时评估

```python
from trulens.core import Feedback
from trulens.providers.openai import OpenAI

provider = OpenAI()
f_context_relevance = Feedback(provider.context_relevance)
f_groundedness = Feedback(provider.groundedness)
```

### Arize Phoenix

**定位**：AI Observability 和评估平台[^4]。

**核心特性**：
- 分布式追踪（基于 OpenTelemetry）
- 支持 Ragas、DeepEval、Cleanlab 集成
- Dataset 和 Experiment 功能
- Prompt Playground

**适用场景**：
- 生产环境全链路监控
- 实验对比和版本管理
- Prompt 工程

### LlamaIndex Evaluation

**定位**：LlamaIndex 内置评估模块[^2]。

**核心特性**：
- Response Evaluation：Correctness, Faithfulness, Context Relevancy
- Retrieval Evaluation：MRR, Hit Rate
- 合成数据集生成（generate_question_context_pairs）

**适用场景**：
- LlamaIndex 生态内评估
- 快速原型验证

```python
from llama_index.core.evaluation import RetrieverEvaluator

retriever_evaluator = RetrieverEvaluator.from_metric_names(
    ["mrr", "hit_rate"], retriever=retriever
)
eval_results = await retriever_evaluator.aevaluate_dataset(qa_dataset)
```

---

## 生产环境评估方案

### 离线评估流程

#### 1. 构建评估数据集

**方式 A：人工标注**
- 收集典型用户问题
- 标注正确答案和相关文档
- 成本高，质量可控

**方式 B：合成生成**[^2]
- 使用 LLM 从文档生成问答对
- 自动化程度高
- 需验证生成质量

```python
from llama_index.core.evaluation import generate_question_context_pairs

qa_dataset = generate_question_context_pairs(
    nodes, llm=llm, num_questions_per_chunk=2
)
```

**方式 C：采样生产数据**
- 从生产环境收集用户查询
- 对高价值或争议查询进行标注
- 覆盖真实场景

#### 2. 选择评估指标

**召回相关**：
- Context Recall（有 ground truth）
- Hit Rate / MRR（有标注文档）
- Context Precision（无标注）

**完整度相关**：
- Faithfulness（无需标注）

**相关性相关**：
- Answer Relevancy
- Context Relevancy

#### 3. 执行评估

```python
from ragas import evaluate
from ragas.metrics import context_precision, context_recall, faithfulness, answer_relevancy

metrics = [context_precision, context_recall, faithfulness, answer_relevancy]
result = evaluate(dataset, metrics=metrics)
```

#### 4. 分析结果

- 识别低分样本，分析失败原因
- 区分检索问题和生成问题
- 针对性优化（分块策略、检索策略、Prompt）

### 在线评估流程

#### 1. 实时追踪

使用 TruLens 或 Arize Phoenix：
- 采集每次查询的 trace
- 记录检索内容、生成答案、耗时

#### 2. 自动评估

- 对实时查询执行轻量评估（如 Faithfulness）
- 标记异常样本供人工审核

#### 3. 持续监控

- 建立指标 dashboard
- 设置阈值告警
- 追踪版本变化

### 评估工具对比

| 工具 | 离线评估 | 实时追踪 | 合成数据 | Ground Truth 需要 | 集成生态 |
|------|---------|---------|---------|------------------|---------|
| Ragas | ✅ | ❌ | ✅ | Context Recall | LlamaIndex, LangChain |
| TruLens | ✅ | ✅ | ❌ | 不强制 | LlamaIndex, LangChain |
| Phoenix | ✅ | ✅ | ❌ | 不强制 | 多框架 |
| LlamaIndex | ✅ | ❌ | ✅ | 强制 | LlamaIndex |

---

## 常见误区

### 误区 1：只评估生成质量

RAG 是检索 + 生成链路，仅评估答案质量无法定位问题根源。需区分检索问题和生成问题。

### 误区 2：过度依赖单一指标

不同指标评估不同维度，需组合使用：
- Context Precision 评估噪声
- Context Recall 评估覆盖
- Faithfulness 评估幻觉

### 误区 3：忽视数据集质量

评估结果依赖于测试数据集的代表性：
- 覆盖业务场景多样性
- 包含边界情况
- 难度分布合理

### 误区 4：忽略 LLM-as-a-Judge 的局限性

LLM 评估本身存在偏差：
- 不同 LLM 评分不一致
- 需要交叉验证
- 高成本场景需优化评估策略

---

## 实践建议

### 指标选择优先级

1. **Faithfulness**：必选，评估幻觉
2. **Answer Relevancy**：必选，评估答案质量
3. **Context Precision**：建议，评估检索噪声
4. **Context Recall**：有 ground truth 时选用

### 评估频率建议

- **离线评估**：每次重要变更前（分块策略、Embedding 模型、Prompt）
- **在线评估**：持续监控，每日/每周汇总

### 成本控制

- 评估样本抽样而非全量
- 使用较小 LLM 作为 Judge（如 GPT-3.5）
- 批量评估而非逐条调用

---

## 参考资料

[^1]: Es, S., et al. (2023). "Ragas: Automated Evaluation of Retrieval Augmented Generation". https://arxiv.org/abs/2309.15217 | Ragas Documentation: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/

[^2]: LlamaIndex Evaluation Guide. https://docs.llamaindex.ai/en/stable/module_guides/evaluating/

[^3]: TruLens Documentation. https://trulens.org/ | TruLens RAG Triad Benchmarks: https://www.snowflake.com/en/engineering-blog/benchmarking-LLM-as-a-judge-RAG-triad-metrics/

[^4]: Arize Phoenix Documentation. https://docs.arize.com/phoenix
