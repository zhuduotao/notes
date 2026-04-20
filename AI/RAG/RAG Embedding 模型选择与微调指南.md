---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - AI
  - RAG
  - embedding
  - vector-search
  - sentence-transformers
  - LLM
  - semantic-search
aliases:
  - Embedding Models
  - 向量编码模型
  - 语义编码
  - Text Embedding
source_type: mixed
source_urls:
  - 'https://www.sbert.net/docs/training/overview.html'
  - 'https://jina.ai/embeddings/'
  - 'https://docs.cohere.com/docs/embeddings'
  - 'https://huggingface.co/spaces/mteb/leaderboard'
  - 'https://huggingface.co/blog/mteb'
status: verified
---
## Embedding 模型概述

Embedding 模型将文本、图像或其他信息转换为固定维度的向量表示，向量编码了语义信息。通过向量间的相似度计算（如余弦相似度），可以判断两个内容的语义关联程度。

在 RAG（Retrieval-Augmented Generation）中，Embedding 模型承担两项核心任务：
- **索引阶段**：将知识库文档转换为向量存入向量数据库
- **检索阶段**：将用户查询转换为向量，与文档向量匹配找出相关内容

---

## 模型分类

按部署方式分为两类：

| 类型 | 特点 | 代表模型 |
|------|------|---------|
| 商业 API 模型 | 无需本地部署，按量付费，维护成本低 | OpenAI、Cohere、Jina AI |
| 开源本地模型 | 完全控制，无调用成本，可微调定制 | Sentence Transformers、BGE、MTEB 模型 |

---

## 商业 API Embedding 模型

### OpenAI Embedding

OpenAI 提供两个主力 Embedding 模型：

| 模型 | 向量维度 | 最大 Token | 适用场景 |
|------|---------|-----------|---------|
| `text-embedding-3-small` | 1536（可缩减） | 8191 | 成本敏感场景，大规模索引 |
| `text-embedding-3-large` | 3072（可缩减） | 8191 | 高精度检索，质量优先场景 |

**Matryoshka 嵌套维度**：两个模型支持维度缩减（如 256、512、1024），可在存储和精度间平衡。

**优势**：
- 与 OpenAI LLM 生态集成度高
- 稳定性和一致性有保障
- 支持维度缩减降低存储成本

**限制**：
- 仅支持英文和多语言，中文效果一般
- 无法微调定制
- 按量计费，大规模使用成本累积

### Cohere Embedding

Cohere 最新模型 `embed-v4.0` 特性：

| 特性 | 说明 |
|------|------|
| 向量维度 | 256/512/1024/1536（Matryoshka） |
| 多语言 | 支持 100+ 语言 |
| 多模态 | 支持文本+图像混合嵌入 |
| 压缩格式 | 支持 float/int8/binary 等压缩 |

**`input_type` 参数**：
- `search_query`：嵌入用户查询
- `search_document`：嵌入检索文档
- `classification`：分类任务
- `clustering`：聚类任务
- `image`：图像嵌入

**优势**：
- 多模态支持（文本+图像混合）
- 高质量多语言支持
- 多种压缩格式降低存储

### Jina AI Embedding

Jina AI 最新版本 `jina-embeddings-v4` 和 `v5-text`：

| 模型 | 参数量 | 向量维度 | 特点 |
|------|--------|---------|------|
| v5-text | 677M（small）/239M（nano） | 可变 | 小型高效，适合边缘部署 |
| v4 | 3.8B | 2048 | 多模态（文本+图像），长上下文 32K |

**优势**：
- 长文本支持（32K context）
- 多模态检索能力强
- 支持本地部署（AWS/Azure/GCP）
- 中文效果较好

---

## 开源本地 Embedding 模型

### Sentence Transformers 系列

Sentence Transformers 是最成熟的开源 Embedding 框架，基于 PyTorch 和 Transformers。

**常用模型**：

| 模型 | 参数量 | 向量维度 | MTEB 排名 | 适用场景 |
|------|--------|---------|----------|---------|
| `all-MiniLM-L6-v2` | 22M | 384 | 中等 | 高速检索，资源受限 |
| `all-mpnet-base-v2` | 109M | 768 | 较高 | 通用场景，平衡方案 |
| `paraphrase-multilingual-MiniLM-L12-v2` | 118M | 384 | - | 多语言，轻量 |
| `sentence-t5-xxl` | 4.7B | 1024 | 高 | 最高精度，资源充足 |

### BGE 系列（中文优化）

北京智源人工智能研究院发布的 BGE 系列是中文检索首选：

| 模型 | 参数量 | 向量维度 | 说明 |
|------|--------|---------|------|
| `bge-small-zh-v1.5` | 33M | 512 | 轻量中文模型 |
| `bge-base-zh-v1.5` | 109M | 768 | 通用中文场景 |
| `bge-large-zh-v1.5` | 326M | 1024 | 高精度中文 |
| `bge-m3` | 568M | 1024 | 多语言+长文本 |

### MTEB Leaderboard 模型选择

MTEB（Massive Text Embedding Benchmark）是 Embedding 模型权威评测榜单[^1]，覆盖 8 类任务：
- Bitext Mining（双文本挖掘）
- Classification（分类）
- Clustering（聚类）
- Pair Classification（成对分类）
- Reranking（重排序）
- Retrieval（检索）
- STS（语义文本相似度）
- Summarization（摘要）

**选择策略**[^1]：
- **追求速度**：选小型模型如 MiniLM 系列
- **速度与性能平衡**：选 `all-mpnet-base-v2`
- **追求最高性能**：选大型模型如 ST5-XXL、GTR-XXL

---

## 适用场景对比

| 场景 | 推荐模型 | 原因 |
|------|---------|------|
| 中文检索 | BGE-large-zh / Jina v4 | 中文语义理解最优 |
| 多语言检索 | Cohere embed-v4 / BGE-m3 | 覆盖广，质量高 |
| 大规模低成本 | OpenAI small / MiniLM | 存储和计算成本低 |
| 高精度检索 | OpenAI large / mpnet-base | 语义匹配质量高 |
| 长文档检索 | Jina v4（32K） | 支持超长文本 |
| 多模态检索 | Cohere v4 / Jina v4 | 支持文本+图像 |
| 私有化部署 | BGE / Sentence Transformers | 完全本地控制 |
| 需要微调定制 | Sentence Transformers | 完整微调支持 |

---

## Embedding 模型微调

### 为什么需要微调

预训练 Embedding 模型在特定领域可能存在以下问题[^2]：
- **领域术语理解不足**：医疗、法律、金融等专业术语
- **任务不匹配**：通用模型未针对检索任务优化
- **语言风格差异**：技术文档、对话、学术论文语义差异
- **检索精度不足**：特定数据集上的检索效果不佳

### 微调核心组件[^2]

Sentence Transformers 微调需要以下组件：

1. **基础模型**：选择预训练模型作为起点
2. **训练数据集**：
   - 成对数据（anchor + positive）
   - 三元组数据（anchor + positive + negative）
   - 标签数据（分类任务）
3. **损失函数**：根据数据类型选择
4. **评估器**：验证微调效果
5. **训练参数**：batch size、学习率、epoch 等

### 常用损失函数[^2]

| 损失函数 | 数据格式 | 适用场景 |
|---------|---------|---------|
| `MultipleNegativesRankingLoss` | 成对数据 | 检索任务，最常用 |
| `TripletLoss` | 三元组数据 | 精确对比学习 |
| `ContrastiveLoss` | 成对+标签 | 相似/不相似分类 |
| `CosineSimilarityLoss` | 连续相似度 | STS 任务 |
| `MatryoshkaLoss` | 成对数据 | 支持维度缩减 |
| `CachedMultipleNegativesRankingLoss` | 成对数据 | 高效批量训练 |

### 检索任务微调示例

```python
from sentence_transformers import SentenceTransformer, SentenceTransformerTrainer
from sentence_transformers.losses import MultipleNegativesRankingLoss
from sentence_transformers.training_args import SentenceTransformerTrainingArguments

model = SentenceTransformer("all-mpnet-base-v2")

train_dataset = [
    {"anchor": "查询文本", "positive": "相关文档"},
    # 更多训练数据...
]

loss = MultipleNegativesRankingLoss(model)

args = SentenceTransformerTrainingArguments(
    output_dir="fine_tuned_model",
    num_train_epochs=10,
    per_device_train_batch_size=32,
    learning_rate=2e-5,
)

trainer = SentenceTransformerTrainer(
    model=model,
    args=args,
    train_dataset=train_dataset,
    loss=loss,
)

trainer.train()
```

### 微调数据准备策略

**Hard Negatives Mining**：
- 从检索结果中选取与 anchor 相似但实际不相关的文档作为 negative
- 提高模型区分相似但不相关内容的能力

**数据增强（Augmented SBERT）[^2]**：
- 使用 LLM 生成 paraphrase 作为 positive
- 扩充有限标注数据

**无监督训练**：
- `TSDAE`：基于去噪自编码器
- `SimCSE`：对比学习，无需标注数据
- `GPL`：生成伪标签

### Matryoshka 嵌套维度训练[^2]

训练时同时优化多个维度（如 256、512、1024），推理时可截断使用任意维度：

```python
from sentence_transformers.losses import MatryoshkaLoss, MultipleNegativesRankingLoss

inner_loss = MultipleNegativesRankingLoss(model)
loss = MatryoshkaLoss(
    model,
    inner_loss,
    matryoshka_dims=[256, 512, 1024],
)
```

**优势**：
- 存储成本可控（小向量占用少）
- 检索精度可调（大向量精度高）
- 单次训练支持多种维度

### 微调效果评估

使用 MTEB 在目标任务数据集评估：

```python
from mteb import MTEB

evaluation = MTEB(tasks=["Banking77Classification"])
results = evaluation.run(model)
```

**关键指标**：
- Retrieval 任务：NDCG@10、Recall@10
- Classification 任务：Accuracy、F1
- STS 任务：Spearman 相关系数

---

## 微调 vs 不微调决策

| 条件 | 建议 |
|------|------|
| 数据量 < 1000 条 | 不微调，使用预训练模型 |
| 通用领域、标准语言 | 不微调 |
| 专业领域术语密集 | 微调 |
| 检索精度不达标 | 微调并评估 |
| 需要特殊语义理解 | 微调 |
| 无标注数据 | 使用无监督方法或数据增强 |

---

## 生产实践建议

### 向量维度选择

| 维度 | 存储成本 | 检索精度 | 适用场景 |
|------|---------|---------|---------|
| 256 | 低 | 中 | 大规模索引，存储受限 |
| 512 | 中 | 较高 | 平衡方案 |
| 1024 | 高 | 高 | 高精度检索 |
| 1536+ | 很高 | 最高 | 极高精度需求 |

### 向量压缩

- **int8 量化**：存储减少 4x，精度损失小
- **binary 量化**：存储减少 32x，精度损失明显，适合海量数据

### 索引与检索流程

1. **数据预处理**：清洗、去噪、分块
2. **Embedding 生成**：批量处理，控制并发
3. **向量存储**：选择向量数据库（Pinecone、Milvus、Qdrant 等）
4. **检索优化**：结合 BM25 混合检索、Reranker 精排

---

## 常见误区

### 误区 1：维度越高越好

高维度增加存储和计算成本，且可能引入噪声。应根据实际需求选择。

### 误区 2：所有任务用同一模型

检索、分类、聚类任务最优模型不同，应根据 MTEB 任务分项排名选择。

### 误区 3：忽略 input_type 参数

Cohere 等模型需要区分 `search_query` 和 `search_document`，错误设置会降低效果。

### 误区 4：直接使用开源模型不做评估

应使用目标领域的测试集评估检索效果，必要时微调。

---

## 参考资料

[^1]: Muennighoff, N. et al. (2022). "MTEB: Massive Text Embedding Benchmark". https://huggingface.co/blog/mteb

[^2]: Sentence Transformers Documentation. "Training Overview". https://www.sbert.net/docs/training/overview.html

[^3]: Cohere. "Introduction to Embeddings". https://docs.cohere.com/docs/embeddings

[^4]: Jina AI. "Embedding API". https://jina.ai/embeddings/

[^5]: MTEB Leaderboard. https://huggingface.co/spaces/mteb/leaderboard
