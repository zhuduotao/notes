---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - AI
  - RAG
  - chunking
  - embedding
  - vector-database
  - LLM
aliases:
  - 分块策略
  - 文本分割
  - Text Splitter
source_type: mixed
source_urls:
  - 'https://www.pinecone.io/learn/chunking-strategies/'
  - >-
    https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/modules/
  - 'https://python.langchain.com/docs/concepts/text_splitters/'
status: verified
---
## 什么是 Chunking

Chunking（分块）是将大文本拆分成较小片段的过程。在 RAG（Retrieval-Augmented Generation）应用中，这是数据预处理的关键步骤，直接影响向量数据库中的内容质量和检索效果。

核心目标是找到**足够大以包含有意义信息，足够小以保证检索精度和响应延迟**的平衡点。

---

## Chunking 的作用

### 适配 Embedding 模型的上下文窗口

所有 Embedding 模型都有固定的上下文窗口限制（如 `text-embedding-3-small` 为 8196 tokens，`llama-text-embed-v2` 为 1024 tokens）。超出限制的内容会被截断，导致关键信息丢失，影响向量表示的准确性。

### 提升语义检索精度

语义检索通过比较查询向量与文档向量相似度来匹配内容。如果 chunk 过大：
- 向量表示会"稀释"关键信息
- 难以精确匹配特定问题

如果 chunk 过小：
- 缺乏上下文，检索结果难以理解
- 可能遗漏关联信息

### 优化 LLM 上下文利用

即使使用长上下文模型（如 Claude 4 Sonnet 200k），直接传入大文档仍存在问题：
- **Lost-in-the-middle 问题**[^1]：长文档中间的信息容易被忽略
- 延迟和成本增加

合理的 chunking 能确保传入 LLM 的信息量最优。

---

## Chunking 方法

### 固定大小分块（Fixed-size Chunking）

**原理**：按固定 token 数或字符数分割文本。

**优点**：
- 实现简单，易于控制
- 适合大多数场景
- 可直接利用 Embedding 模型的最大上下文窗口

**缺点**：
- 可能切断语义完整的内容
- 不考虑文档结构

**适用场景**：无明确结构的文档、首次实验阶段。

**建议**：作为默认方案，仅在效果不佳时才尝试其他策略[^2]。

```python
# LlamaIndex 示例
from llama_index.core.node_parser import SentenceSplitter

splitter = SentenceSplitter(
    chunk_size=1024,
    chunk_overlap=20,
)
nodes = splitter.get_nodes_from_documents(documents)
```

---

### 递归字符分块（Recursive Character Chunking）

**原理**：按优先级顺序尝试多种分隔符（如 `\n\n` → `\n` → ` ` → 空字符），优先保持段落完整性，再尝试句子，最后按字符分割。

**优点**：
- 平衡语义完整性和固定大小
- 尽可能保持段落、句子边界

**缺点**：
- 仍可能在语义边界处截断
- 不完全理解文档结构

**适用场景**：通用文档、需要兼顾结构与固定大小的场景。

```python
# LangChain 示例
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
)
chunks = splitter.split_text(text)
```

---

### 文档结构分块（Structure-based Chunking）

根据文档类型特性，利用结构标记进行分块。

#### Markdown 分块

按标题层级（`#`、`##` 等）分割，保持章节完整性。

```python
from llama_index.core.node_parser import MarkdownNodeParser

parser = MarkdownNodeParser()
nodes = parser.get_nodes_from_documents(markdown_docs)
```

**优点**：语义边界清晰，适合技术文档、博客。

#### HTML 分块

按 HTML 标签（`<p>`、`<h1>` 等）分割。

```python
from llama_index.core.node_parser import HTMLNodeParser

parser = HTMLNodeParser(tags=["p", "h1", "h2"])
nodes = parser.get_nodes_from_documents(html_docs)
```

**优点**：适合网页抓取内容。

#### JSON 分块

按 JSON 结构层级分割。

```python
from llama_index.core.node_parser import JSONNodeParser

parser = JSONNodeParser()
nodes = parser.get_nodes_from_documents(json_docs)
```

#### 代码分块

按语法结构（函数、类）分割，而非简单按行数。

```python
from llama_index.core.node_parser import CodeSplitter

splitter = CodeSplitter(
    language="python",
    chunk_lines=40,
    chunk_lines_overlap=15,
    max_chars=1500,
)
nodes = splitter.get_nodes_from_documents(documents)
```

**优点**：保持代码逻辑完整。

---

### 语义分块（Semantic Chunking）

**原理**：基于语义相似度动态确定 chunk 边界[^3]。

步骤：
1. 将文档拆分为句子
2. 计算句子间的语义距离（通过 Embedding）
3. 当语义距离超过阈值时，标记为新 chunk 起点

**优点**：
- 基于内容语义，而非机械分割
- 同一主题内容保持在同一 chunk

**缺点**：
- 计算开销大（需多次 Embedding）
- 需调优距离阈值

**适用场景**：主题频繁转换的长文档。

```python
from llama_index.core.node_parser import SemanticSplitterNodeParser

parser = SemanticSplitterNodeParser(
    buffer_size=1,
    breakpoint_percentile_threshold=95,
    embed_model=embed_model,
)
nodes = parser.get_nodes_from_documents(documents)
```

---

### 句子窗口分块（Sentence Window Chunking）

**原理**：将每个句子作为独立 chunk，同时在 metadata 中存储前后若干句子的"窗口"。

检索时：
- Embedding 基于**单句子**生成（语义精确）
- 返回给 LLM 时替换为**完整窗口内容**（上下文完整）

**优点**：
- 检索粒度细，精确匹配
- 生成时上下文完整

**适用场景**：需要精确定位但需完整上下文理解的场景。

```python
from llama_index.core.node_parser import SentenceWindowNodeParser

node_parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,
    window_metadata_key="window",
    original_text_metadata_key="original_sentence",
)
```

---

### 上下文分块（Contextual Chunking）

**原理**：Anthropic 提出的 Contextual Retrieval 方法[^4]。

每个 chunk 生成时，通过 LLM 添加一个**上下文说明前缀**：
- LLM 接收完整文档 + 当前 chunk
- 生成简短描述，说明 chunk 在文档中的位置和主题
- 描述前缀与 chunk 合并后再 Embedding

**优点**：
- 解决长文档中 chunk 失去全局上下文的问题
- 检索时能匹配到文档高层信息

**缺点**：
- 需额外 LLM 处理成本
- Prompt caching 可降低重复处理开销

**适用场景**：数百页的长文档、频繁切换主题的文档。

---

## 各方法优缺点对比

| 方法 | 语义完整性 | 实现复杂度 | 计算开销 | 适用场景 |
|------|-----------|-----------|---------|---------|
| 固定大小 | 低 | 低 | 低 | 通用文档，首选方案 |
| 递归字符 | 中 | 低 | 低 | 通用文档，兼顾结构 |
| 文档结构 | 高 | 中 | 低 | Markdown、HTML、代码 |
| 语义分块 | 高 | 高 | 高 | 主题频繁变化的长文档 |
| 句子窗口 | 高 | 中 | 中 | 精检索 + 全上下文 |
| 上下文分块 | 最高 | 高 | 高（需 LLM） | 超长复杂文档 |

---

## 生产环境实践方案

### 分块参数选择建议

**Chunk 大小范围测试**[^2]：
- 小 chunk（128-256 tokens）：语义信息更细粒度
- 中 chunk（512 tokens）：平衡点
- 大 chunk（1024+ tokens）：更多上下文

**重叠大小（Overlap）**：
- 通常设置 10-20% 的 chunk 大小
- 确保跨边界的内容不会完全丢失

### 分步实践流程

1. **数据预处理**：清洗、去除无关内容
2. **选择基础策略**：从固定大小分块开始
3. **评估效果**：
   - 准备测试查询集
   - 对比不同 chunk 大小的检索质量
   - 使用 Ragas、TruLens 等评估工具[^5]
4. **迭代优化**：根据效果调整策略
5. **结合 Chunk Expansion**：检索后扩展返回相邻 chunks

### Chunk Expansion（后处理扩展）

检索时返回的 chunk 可扩展相邻内容：
- 检索基于小 chunk（高精度）
- 返回给 LLM 时扩展为段落/页面/完整文档（上下文完整）

```python
from llama_index.core.postprocessor import AutoMergingRetriever

# 自动合并检索相邻 chunks
retriever = AutoMergingRetriever(
    vector_retriever,
    storage_context,
)
```

### 多策略组合

生产中常组合多种策略：
- **主检索**：固定大小 + Overlap
- **辅助检索**：BM25 关键词检索
- **后处理**：Chunk Expansion + Reranking

---

## 常见误区

### 误区 1：Chunk 越大越好

大 chunk 会稀释向量表示的关键信息，降低检索精度。

### 误区 2：直接使用 Embedding 模型最大窗口

虽然技术上可行，但：
- `lost-in-the-middle` 问题[^1]
- 延迟和成本增加
- 噪声信息稀释关键内容

### 误区 3：忽略重叠设置

无重叠可能导致跨边界的关键信息被分割，检索时两部分都匹配度不足。

---

## 参考资料

[^1]: Liu, N. F., et al. (2023). "Lost in the Middle: How Language Models Use Long Contexts". https://arxiv.org/abs/2307.03172

[^2]: Pinecone. "Chunking Strategies for LLM Applications". https://www.pinecone.io/learn/chunking-strategies/

[^3]: Kamradt, G. "Levels of Text Splitting". https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/tutorials/LevelsOfTextSplitting/5_Levels_Of_Text_Splitting.ipynb

[^4]: Anthropic. "Contextual Retrieval". https://www.anthropic.com/news/contextual-retrieval

[^5]: LlamaIndex Evaluation Guide. https://docs.llamaindex.ai/en/stable/module_guides/evaluating/
