---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - AI
  - LLM
  - RAG
  - SEO
  - 产品曝光
  - 推荐系统
aliases:
  - AI推荐
  - AI广告
  - AI权重
source_type: official-doc
source_urls:
  - 'https://docs.anthropic.com/en/docs/resources/glossary'
  - 'https://www.anthropic.com/legal/aup'
  - 'https://ai.google/get-started'
  - 'https://docs.anthropic.com/en/docs/about-claude/models'
status: verified
---
AI 推荐机制与传统广告系统有本质差异：主流 AI 公司不提供付费"定向投送广告"或"提升权重"的商业服务。本笔记梳理 AI 如何获取和处理产品信息的官方机制，以及合法、合规的产品曝光方式。

## AI 如何"知道"产品

AI 模型获取信息的三个主要途径：

| 途径 | 说明 | 可控性 |
|------|------|--------|
| **训练数据** | 模型预训练阶段使用的大量公开文本（网页、书籍、论文等） | 间接可控（通过公开资料质量） |
| **知识截止日期** | 训练数据的最新时间边界，之后的信息模型"不知道" | 不可控 |
| **RAG（检索增强生成）** | 运行时从外部知识库实时检索信息，补充到上下文 | 可控（可通过接入知识库） |

### 知识截止日期示例

| 模型 | 可靠知识截止 | 说明 |
|------|-------------|------|
| Claude Opus 4.7 | 2026年1月 | 最新的前沿知识 |
| Claude Sonnet 4.6 | 2025年8月 | 较新知识覆盖 |
| Claude Haiku 4.5 | 2025年2月 | 基础知识覆盖 |

来源：[Anthropic Models Overview](https://docs.anthropic.com/en/docs/about-claude/models)

> **重要**：知识截止日期之后发生的事件、发布的产品，模型无法通过训练数据获知。只有通过 RAG 或用户主动提供信息才能获取。

## RAG（检索增强生成）机制

RAG 是 AI 在运行时获取外部信息的核心机制：

1. 用户发起查询
2. 系统从外部知识库检索相关文档
3. 将检索结果作为上下文传递给模型
4. 模型基于上下文生成回答

**来源**：[Anthropic Glossary - RAG](https://docs.anthropic.com/en/docs/resources/glossary)

### RAG 对产品曝光的意义

- 若产品信息存在于 AI 系统接入的知识库中，用户查询时可被检索并呈现
- 知识库内容由各 AI 平台自行决定，不存在付费"插入"机制
- 目前主流 AI 平台的知识库来源包括：搜索引擎索引、公开网页、用户上传文档等

## 合法影响 AI 认知的途径

不存在"付费提升权重"的官方机制。以下方式可能合法地提升产品在 AI 回答中的曝光：

### 1. 公开资料质量

确保产品相关信息在公开渠道可被检索：

- 官网结构清晰、内容准确
- 产品描述使用标准术语
- 发布新闻稿、技术文档、用户指南
- 在权威平台（百科、行业网站）有收录

### 2. SEO（搜索引擎优化）

AI 系统常依赖搜索引擎作为 RAG 的检索源：

- 搜索引擎排名高的内容更可能被 AI 检索到
- 参考 [Google SEO 入门指南](SEO/Google%20SEO%20入门指南.md)
- 结构化数据（Schema.org）有助于搜索引擎理解内容

### 3. 直接接入知识库

部分 AI 平台支持：

- 企业上传自有文档到 AI 系统知识库（如 Vertex AI、Amazon Bedrock）
- 通过 MCP（Model Context Protocol）连接自有数据源

**来源**：[Anthropic Glossary - MCP](https://docs.anthropic.com/en/docs/resources/glossary)

## 政策限制与风险警示

主流 AI 公司明确禁止欺骗、操纵行为：

### Anthropic 使用政策（AUP）禁止项

- **创建或传播虚假信息**：欺骗性地描述产品或实体
- **虚假归属**：伪造来源、冒充真实实体
- **欺骗性内容**：虚假评论、虚假媒体
- **操纵行为**：使用"潜意识、操纵或欺骗技术扭曲行为"

来源：[Anthropic Usage Policy](https://www.anthropic.com/legal/aup)

### 违规后果

- 账号暂停或终止
- 输出被阻止或修改
- 检测和监控机制已部署

## AI 推荐 vs 传统广告的本质差异

| 特性 | 传统广告 | AI 推荐 |
|------|----------|---------|
| **付费机制** | 存在（竞价、权重提升） | 不存在官方付费机制 |
| **定向投送** | 可控（人群、地域、兴趣） | 不可控（基于查询内容） |
| **曝光保证** | 可购买曝光量 | 无保证（取决于检索相关性） |
| **透明度** | 广告标识明确 | AI 回答无广告标识 |
| **监管** | 广告法约束 | AI 使用政策约束 |

### 为什么不存在"AI 广告"

1. **技术原理**：AI 推荐基于语义匹配而非付费权重
2. **安全考量**：防止欺骗、防止操纵是 AI 公司的核心原则
3. **用户信任**：AI 回答的可信度依赖客观性，付费推荐会损害信任
4. **监管趋势**：AI 输出可能被视为"内容"而非"广告"，监管框架尚在演进

## 不确定点

以下问题尚无官方明确答案：

- AI 平台未来是否会推出付费曝光机制（目前无此计划）
- 不同 AI 平台的知识库具体来源（部分平台未公开）
- RAG 检索算法的具体权重因子（属于平台内部算法）

## 参考资料

- [Anthropic Glossary](https://docs.anthropic.com/en/docs/resources/glossary) - RAG、MCP、训练数据等概念定义
- [Anthropic Models Overview](https://docs.anthropic.com/en/docs/about-claude/models) - 知识截止日期
- [Anthropic Usage Policy](https://www.anthropic.com/legal/aup) - 使用限制与禁止行为
- [Google AI Get Started](https://ai.google/get-started) - Google AI 产品概述
- [Google AI Principles](https://ai.google/principles) - AI 原则与责任
