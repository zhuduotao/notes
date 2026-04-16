---
created: 2026-04-16
updated: 2026-04-16
tags:
  - 全文搜索
  - BM25
  - TF-IDF
  - 信息检索
  - 排序算法
  - Elasticsearch
  - Lucene
  - 搜索引擎
aliases:
  - Okapi BM25
  - Best Matching 25
  - TF-IDF
  - Full-Text Search Algorithms
  - 全文检索算法
  - 信息检索排序
source_type: mixed
source_urls:
  - https://en.wikipedia.org/wiki/Okapi_BM25
  - https://lucene.apache.org/core/9_7_0/core/org/apache/lucene/search/similarities/TFIDFSimilarity.html
  - https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html
  - https://nlp.stanford.edu/IR-book/html/htmledition/queries-as-vectors-1.html
status: verified
---

## 概述

全文搜索（Full-Text Search）旨在从大规模文本集合中快速找出与用户查询最相关的文档。其核心问题是**如何量化文档与查询之间的相关性**。业界主流的全文搜索引擎（如 Elasticsearch、Lucene、Meilisearch 等）均采用基于词袋模型的排序函数，其中最具代表性的是 **BM25** 和 **TF-IDF**。

## 信息检索基础概念

### 倒排索引（Inverted Index）

全文搜索的底层数据结构。与正向索引（文档 → 词）相反，倒排索引建立 **词 → 文档列表** 的映射：

```
"搜索" → [doc1, doc3, doc7, ...]
"算法" → [doc2, doc3, doc5, ...]
"BM25" → [doc1, doc7, ...]
```

每个词项（term）关联一个**倒排列表**（posting list），记录包含该词的所有文档 ID，以及可选的词频、位置等元数据。查询时，引擎通过倒排索引快速定位候选文档集，再用排序函数计算相关度。

### 词袋模型（Bag-of-Words Model）

将文档视为词的无序集合，忽略词序和语法结构。BM25 和 TF-IDF 均基于此模型。虽然简化了语言结构，但在大多数搜索场景中已能取得良好效果。

### 相关性排序（Relevance Ranking）

给定查询 Q 和文档集合 D，排序函数为每个文档计算一个分数 `score(D, Q)`，分数越高表示相关性越强。排序函数的设计目标是让真正相关的文档排在前面。

## 布尔模型（Boolean Model）

### 原理

最基础的信息检索模型。将文档和查询都表示为布尔逻辑表达式，文档要么**匹配**要么**不匹配**，没有中间程度。

```
查询: (搜索 AND 算法) OR BM25
```

### 特点

| 特性 | 说明 |
|------|------|
| 结果 | 二元（匹配/不匹配） |
| 排序 | 无排序，所有匹配文档平等 |
| 优点 | 精确、可预测、实现简单 |
| 缺点 | 无法区分"高度相关"和"勉强匹配"的文档 |

### 在 Lucene 中的角色

Lucene 采用**混合模型**：先用布尔模型过滤出候选文档集（必须满足的硬性条件），再用向量空间模型对候选文档进行打分排序。

## TF-IDF（Term Frequency-Inverse Document Frequency）

### 是什么

TF-IDF 是一种统计方法，用于评估一个词对文档集合中某篇文档的重要程度。由两部分组成：

- **TF（Term Frequency，词频）**：词在文档中出现的频率
- **IDF（Inverse Document Frequency，逆文档频率）**：衡量词的稀缺程度

### 公式

```
TF(t, d) = 词 t 在文档 d 中出现的次数

IDF(t) = log(N / n(t))

TF-IDF(t, d) = TF(t, d) × IDF(t)
```

其中：
- `N` = 文档集合中的文档总数
- `n(t)` = 包含词 t 的文档数量

**多词查询的文档得分**：对查询中每个词的 TF-IDF 值求和或取向量余弦相似度。

### Lucene 的 TF-IDF 实现

Lucene 的 `TFIDFSimilarity` 类定义了实际的打分公式（`ClassicSimilarity`）：

```
score(q, d) = Σ [ tf(t in d) × idf(t)² × t.getBoost() × norm(t, d) ]
               t in q
```

各分量说明：

| 分量 | 说明 | 默认计算方式 |
|------|------|-------------|
| `tf(t in d)` | 词 t 在文档 d 中的频率贡献 | `sqrt(freq)` |
| `idf(t)` | 逆文档频率 | `1 + log((docCount + 1) / (docFreq + 1))` |
| `t.getBoost()` | 查询时指定的词权重 | 用户自定义，默认 1.0 |
| `norm(t, d)` | 字段长度归一化 × 索引时 boost | 索引时预计算并存储 |

**注意**：`idf(t)` 在公式中**平方**出现，因为查询向量和文档向量各贡献一次 IDF。

### 直观理解

- 一个词在**当前文档**中出现越频繁 → TF 越高 → 越可能相关
- 一个词在**整个集合**中越罕见 → IDF 越高 → 区分能力越强
- 例如："的"在所有文档中都出现，IDF 接近 0；"BM25"只在少数文档中出现，IDF 很高

### 局限

1. **TF 线性增长问题**：TF-IDF 中 TF 是线性的（或开根号），一个词出现 100 次的文档得分远高于出现 5 次的文档，但实际相关性并非线性增长
2. **无文档长度归一化的精细控制**：`norm(t, d)` 在索引时预计算，无法在搜索时动态调整
3. **IDF 对罕见词过度敏感**：仅在 1 篇文档中出现的词 IDF 极高，可能是噪声

## BM25（Okapi BM25）

### 是什么

BM25（Best Matching 25）是基于**概率检索框架**的排序函数，由 Stephen E. Robertson、Karen Spärck Jones 等人在 1970-1980 年代开发。名称中的 "25" 表示这是 Okapi 搜索系统中的第 25 次迭代。BM25 可视为 TF-IDF 的改进版本，是目前 Elasticsearch、Lucene（默认）、Meilisearch 等主流搜索引擎的**默认排序算法**。

### 公式

给定查询 Q = (q₁, ..., qₙ)，文档 D 的 BM25 得分为：

```
score(D, Q) = Σ IDF(qᵢ) × [ f(qᵢ, D) × (k₁ + 1) ] / [ f(qᵢ, D) + k₁ × (1 - b + b × |D| / avgdl) ]
              i=1 to n
```

其中：
- `f(qᵢ, D)` = 词 qᵢ 在文档 D 中的词频
- `|D|` = 文档 D 的长度（词数）
- `avgdl` = 文档集合的平均文档长度
- `k₁`、`b` = 自由参数（见下方说明）

**IDF 分量**（BM25 标准形式）：

```
IDF(qᵢ) = ln((N - n(qᵢ) + 0.5) / (n(qᵢ) + 0.5) + 1)
```

其中 `N` = 文档总数，`n(qᵢ)` = 包含 qᵢ 的文档数。

### 参数说明

| 参数 | 典型值 | 作用 |
|------|--------|------|
| `k₁` | 1.2 ~ 2.0 | 控制词频饱和速度。值越大，TF 增长越接近线性；值越小，TF 越快饱和 |
| `b` | 0.75 | 控制文档长度归一化的强度。`b=0` 表示不考虑文档长度；`b=1` 表示完全按长度归一化 |

### BM25 相比 TF-IDF 的关键改进

#### 1. 词频饱和（TF Saturation）

BM25 的 TF 分量是**饱和函数**：

```
TF_sat = f × (k₁ + 1) / (f + k₁ × length_norm)
```

当词频 `f` 趋向无穷大时，该值趋近于 `k₁ + 1`，即存在**上限**。这意味着一个词出现 100 次和出现 50 次的差距，远小于 TF-IDF 中的线性差距。这更符合直觉：一篇文档中某个词出现多次后，继续增加次数对相关性判断的边际贡献递减。

#### 2. 精细的文档长度归一化

BM25 通过 `(1 - b + b × |D| / avgdl)` 实现动态长度归一化：

- 短文档（`|D| < avgdl`）：归一化因子 < 1，TF 分量被放大
- 长文档（`|D| > avgdl`）：归一化因子 > 1，TF 分量被压缩
- `b=0.75` 是经验值，在多数场景下表现良好

#### 3. IDF 的平滑处理

BM25 的 IDF 公式使用 `+0.5` 平滑项，避免 `n(qᵢ) = 0` 或 `n(qᵢ) = N` 时的极端值。

### 直观示例

假设文档集合平均长度为 1000 词，`k₁=1.2`，`b=0.75`：

| 场景 | 词频 | 文档长度 | BM25 TF 分量 | 说明 |
|------|------|---------|-------------|------|
| A | 5 | 500（短文档） | ≈ 3.0 | 短文档 + 适中词频，得分较高 |
| B | 5 | 2000（长文档） | ≈ 1.6 | 同样词频，但长文档被惩罚 |
| C | 100 | 1000 | ≈ 2.1 | 词频很高但已饱和，不会无限增长 |
| D | 1 | 1000 | ≈ 0.55 | 仅出现一次，得分较低 |

### BM25 的变体

#### BM25F

将文档视为多个字段（标题、正文、锚文本等）的组合，每个字段有独立的权重、词频饱和参数和长度归一化。适用于结构化文档搜索（如网页的 title、body、meta 标签）。

```
score(D, Q) = Σ Σ w_s × IDF(qᵢ) × [ f_s(qᵢ, D) × (k₁ + 1) ] / [ f_s(qᵢ, D) + k₁ × (1 - b_s + b_s × |D_s| / avgdl_s) ]
              s   i
```

其中 `s` 遍历文档的各个字段（stream），`w_s` 是字段权重。

#### BM25+

针对标准 BM25 的一个缺陷：长文档即使匹配查询词，其得分可能被压得过低，甚至低于不匹配查询词的短文档。BM25+ 在 TF 分量后添加一个下限参数 `δ`（默认 1.0）：

```
score(D, Q) = Σ IDF(qᵢ) × [ TF_sat + δ ]
              i
```

#### BM11 / BM15

BM25 在极端 `b` 值下的特例：
- `b = 1` → BM11（完全按文档长度归一化）
- `b = 0` → BM15（不考虑文档长度）

## 算法对比

| 维度 | 布尔模型 | TF-IDF | BM25 |
|------|---------|--------|------|
| 排序能力 | 无（仅匹配/不匹配） | 有 | 有 |
| 词频处理 | 二元的 | 线性或开根号 | **饱和函数**（有上限） |
| 文档长度归一化 | 无 | 索引时预计算 | **搜索时动态计算** |
| 参数可调性 | 无 | 有限 | `k₁`、`b` 可调 |
| 适用场景 | 精确过滤 | 通用搜索 | **通用搜索（推荐）** |
| 工业采用 | 作为过滤层 | 早期搜索引擎 | Elasticsearch/Lucene 默认 |

## 在主流引擎中的使用

### Elasticsearch / OpenSearch

- **默认相似度算法**：BM25（`"type": "BM25"`）
- 可配置参数：`k1`（默认 1.2）、`b`（默认 0.75）
- 也支持 `classic`（TF-IDF）、`boolean`、`scripted` 等

```json
{
  "settings": {
    "index": {
      "similarity": {
        "my_bm25": {
          "type": "BM25",
          "b": 0.6,
          "k1": 1.5
        }
      }
    }
  }
}
```

### Lucene

- Lucene 9.x 默认使用 `BM25Similarity`
- 通过 `IndexWriterConfig.setSimilarity()` 或 `IndexSearcher.setSimilarity()` 切换
- `ClassicSimilarity` 提供 TF-IDF 实现

### MySQL Full-Text

- InnoDB 全文索引使用内置评分机制，不直接暴露 BM25 参数
- 支持 `IN NATURAL LANGUAGE MODE`、`IN BOOLEAN MODE` 等搜索模式

### Redis（RediSearch 模块）

- 默认使用 BM25 评分
- Redis 8.4 新增 `FT.HYBRID` 命令，支持全文搜索 + 向量相似度融合评分

## 调参建议

### k₁ 的选择

| 场景 | 推荐值 | 理由 |
|------|--------|------|
| 通用搜索 | 1.2 ~ 1.5 | 平衡饱和速度与区分度 |
| 短文本（标题、标签） | 1.5 ~ 2.0 | 词频上限较高，允许更多区分 |
| 长文本（文章、文档） | 1.0 ~ 1.2 | 更快饱和，避免长文档中高频词主导 |

### b 的选择

| 场景 | 推荐值 | 理由 |
|------|--------|------|
| 文档长度差异大 | 0.75 ~ 1.0 | 强归一化，防止长文档占优 |
| 文档长度均匀 | 0.3 ~ 0.5 | 弱归一化，保留长度信息 |
| 不确定 | 0.75 | 经验默认值，多数场景可用 |

### 最佳实践

1. **先使用默认值**：`k1=1.2, b=0.75` 在大多数场景下已经表现良好
2. **基于评估集调参**：使用 NDCG、MAP 等指标在标注数据上评估不同参数组合
3. **考虑字段差异**：标题字段和正文字段可能需要不同的 BM25 参数（使用 BM25F）
4. **注意语料变化**：当文档集合显著变化时，`avgdl` 会改变，可能需要重新评估参数

## 限制与注意事项

1. **词袋模型的固有局限**：BM25 不考虑词序、短语匹配、语义关系。"猫追狗"和"狗追猫"得分相同
2. **同义词问题**：BM25 无法自动识别同义词，需要额外的同义词词典或语义扩展
3. **跨语言搜索**：BM25 依赖精确词匹配，不直接支持跨语言检索
4. **短查询退化**：当查询只有 1-2 个词时，BM25 的区分能力有限
5. **新词/冷启动**：新加入文档集合的词，其 IDF 值需要随集合更新而重新计算
6. **与向量搜索的关系**：BM25 是**词法匹配**（lexical matching），与基于嵌入的**语义匹配**（semantic matching）互补。现代搜索系统常将两者结合（如 Elasticsearch 的 `knn` + `BM25` 混合搜索）

## 相关概念

- **倒排索引**：全文搜索的底层数据结构
- **向量空间模型（VSM）**：将文档和查询表示为向量，用余弦相似度衡量相关性
- **概率检索框架**：BM25 的理论基础，基于文档与查询相关的概率估计
- **NDCG（Normalized Discounted Cumulative Gain）**：评估排序质量的常用指标
- **语义搜索**：基于词嵌入（如 BERT、DPR）的搜索方法，与 BM25 互补
- **混合搜索（Hybrid Search）**：结合 BM25（词法）和向量搜索（语义）的搜索结果

## 参考资料

- Wikipedia - Okapi BM25: https://en.wikipedia.org/wiki/Okapi_BM25
- Robertson & Zaragoza (2009) - "The Probabilistic Relevance Framework: BM25 and Beyond": https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf
- Lucene 9.7 TFIDFSimilarity API: https://lucene.apache.org/core/9_7_0/core/org/apache/lucene/search/similarities/TFIDFSimilarity.html
- Elasticsearch Similarity Documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html
- Stanford IR Book - Vector Space Model: https://nlp.stanford.edu/IR-book/html/htmledition/queries-as-vectors-1.html
- Lv & Zhai (2011) - "Lower-bounding term frequency normalization" (BM25+): https://doi.org/10.1145/2063576.2063584
- Robertson et al. (2004) - "Simple BM25 extension to multiple weighted fields" (BM25F): https://doi.org/10.1145/1031171.1031181
