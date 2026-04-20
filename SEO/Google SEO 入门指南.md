---
aliases:
  - Google 搜索引擎优化
  - 谷歌 SEO
  - 搜索引擎优化
created: '2026-04-20'
source_type: official-doc
source_urls:
  - 'https://developers.google.com/search/docs/essentials'
  - >-
    https://developers.google.com/search/docs/fundamentals/creating-helpful-content
  - 'https://developers.google.com/search/docs/appearance/core-web-vitals'
  - 'https://developers.google.com/search/docs/appearance/page-experience'
  - 'https://developers.google.com/search/docs/fundamentals/using-gen-ai-content'
status: verified
tags:
  - SEO
  - Google
  - 搜索引擎优化
  - Web开发
  - 数字营销
  - E-E-A-T
  - Core-Web-Vitals
updated: '2026-04-20'
---
本文整理自 Google Search Central 官方文档（截至 2025-12-10 最新更新）。

SEO（Search Engine Optimization）是帮助搜索引擎理解内容、帮助用户通过搜索引擎发现并决定访问网站的过程。针对 Google 的 SEO，核心在于遵循 **Google Search Essentials**（搜索要点），并在此基础上提升网站在搜索结果中的表现。

**重要声明**：Google 不接受付费以更频繁抓取或更高排名，排名完全由程序算法决定。出现在 Google 搜索结果中不需要付费。

## Google Search Essentials

> 来源：[Google Search Essentials](https://developers.google.com/search/docs/essentials)（2025-12-10 更新）

Google Search Essentials（原名 Webmaster Guidelines）构成了网页内容在 Google Search 中出现并表现良好的核心要素：

### 技术要求

技术要求涵盖 Google Search 显示网页所需的最基本条件。大多数网站无需刻意操作就能满足这些要求：

- Googlebot 能访问页面
- 页面能正常渲染
- 内容是可索引的文件类型

### 垃圾政策

垃圾政策详细说明了可能导致页面或整个网站排名降低或完全从 Google Search 中被剔除的行为和策略。

**专注于提供最佳内容和体验**、遵守原则精神的网站更有可能在 Google 搜索结果中表现良好。

### 关键最佳实践

虽然有很多可以改善 SEO 的方法，但几个核心实践对网页排名和外观影响最大：

1. **创建有帮助、可靠、以人为本的内容**
2. 使用人们会用来寻找内容的词语，放置在页面的显眼位置（如标题、主标题、alt 文本、链接文本）
3. **使链接可被抓取**，让 Google 通过页面链接发现网站其他页面
4. 告知人们关于你的网站，在相关社区中积极推广
5. 对于其他内容（图像、视频、结构化数据、JavaScript），遵循相应的最佳实践
6. 通过启用适合的功能增强网站在 Google Search 中的外观
7. 如果有不应在搜索结果中出现的内容，使用适当方法控制内容如何出现在 Google Search 中

## Google Search 工作机制

> 来源：[How Google Search Works](https://developers.google.com/search/docs/fundamentals/how-search-works)（2025-12-18 更新）

Google Search 是一个完全自动化的搜索引擎，使用名为 **Googlebot**（爬虫/机器人）的程序持续探索网络，寻找页面加入索引。

### 三阶段流程

并非所有页面都能完整通过以下三个阶段：

1. **抓取（Crawling）**
   - Google 从已发现的页面下载文本、图像和视频
   - URL 发现方式：已访问过的页面、从已知页面提取链接、提交的站点地图（sitemap）
   - Googlebot 使用算法决定抓取哪些站点、频率和页面数量
   - 爬虫会避免过快抓取以免过载服务器
   - 抓取时会渲染页面并执行 JavaScript（使用最新版 Chrome）

2. **索引（Indexing）**
   - Google 分析页面的文本、图像、视频，以及 `<title>` 元素、alt 属性等关键标签
   - 判断页面是否为重复内容或规范页面（canonical）
   - 收集页面信号：语言、国家/地区、可用性等
   - 信息存储在 Google 索引数据库中

3. **呈现搜索结果（Serving）**
   - 用户输入查询后，机器搜索索引，返回最相关、最高质量的结果
   - 相关性由数百个因素决定，包括用户位置、语言、设备

**重要**：页面满足所有技术要求和最佳实践，不保证 Google 会抓取、索引或呈现其内容。

## 创建有帮助、可靠、以人为本的内容

> 来源：[Creating helpful, reliable, people-first content](https://developers.google.com/search/docs/fundamentals/creating-helpful-content)（2025-12-10 更新）

Google 的自动排名系统优先考虑**有帮助、可靠、为人们利益而创建**的信息，而非为操纵搜索引擎排名而创建的内容。

### 内容和质量自检问题

- 内容是否提供原创信息、报道、研究或分析？
- 内容是否提供对主题的实质性、完整或全面描述？
- 内容是否提供超越显而易见的深入分析或有趣信息？
- 如果内容引用其他来源，是否避免简单复制或重写，而是提供实质性的额外价值和原创性？
- 主标题或页面标题是否提供描述性、有帮助的内容摘要？
- 主标题或页面标题是否避免夸大或令人震惊？
- 这是否是你会想收藏、分享给朋友或推荐的页面？
- 你是否期望此内容出现在印刷杂志、百科全书或书籍中？
- 内容是否比搜索结果中的其他页面提供实质性价值？
- 内容是否有拼写或风格问题？
- 内容是否制作良好，还是显得粗糙或匆忙制作？

### 专业性自检问题

- 内容是否以让你想信任的方式呈现信息，如清晰的来源、涉及的专业知识证据、作者背景？
- 如果有人研究发布内容的网站，他们会得出该网站在主题上是值得信任或被广泛认可的印象吗？
- 内容是否由明显了解主题的专家或爱好者撰写或审查？
- 内容是否有任何容易验证的事实错误？

### 以人为本 vs 搜索引擎优先内容

**以人为本内容**意味着主要为人创建的内容，而非为操纵搜索引擎排名。回答"是"以下问题表明你可能在正确的轨道上：

- 你是否有现有或预期的受众，他们直接来到你时会发现内容有用？
- 内容是否清楚地展示第一手经验和深度知识？
- 网站是否有主要目的或焦点？
- 阅读内容后，某人是否会觉得学到了足够关于主题的知识来帮助实现目标？
- 某人阅读内容后是否会感到有令人满意的体验？

**搜索引擎优先内容**警告信号（回答"是"表明应重新评估）：

- 内容主要是为了吸引搜索引擎访问吗？
- 是否在许多不同主题上制作大量内容，希望其中一些可能在搜索结果中表现良好？
- 是否使用大量自动化在许多主题上制作内容？
- 是否主要总结他人说的话而没有添加太多价值？
- 是否仅仅因为话题看起来流行而写，而不是因为你会为现有受众写？
- 内容是否让读者感到需要再次搜索以从其他来源获得更好信息？
- 是否因为听说 Google 有首选字数而写到特定字数？（没有，Google 没有）
- 是否决定进入某个利基主题领域而没有真正的专业知识，主要是因为认为会获得搜索流量？

### 提供 great page experience

Google 的核心排名系统奖励提供良好页面体验的内容。网站所有者不应仅关注页面体验的一两个方面，而应检查是否在许多方面提供整体良好的页面体验。

详见下文 "Page Experience 与 Core Web Vitals" 章节。

## E-E-A-T 评估框架

> 来源：[Creating helpful, reliable, people-first content](https://developers.google.com/search/docs/fundamentals/creating-helpful-content)（2025-12-10 更新）

Google 的自动系统使用**许多不同因素**来排名优质内容。在识别相关内容后，系统优先考虑似乎最有帮助的内容。为此，它们识别可以帮助确定哪些内容展示以下方面的因素组合：

- **Experience（经验）**
- **Expertise（专业知识）**
- **Authoritativeness（权威性）**
- **Trustworthiness（可信度）**

即 **E-E-A-T**。

**信任是最重要的**。其他方面贡献于信任，但内容不必展示所有方面。例如，某些内容可能因其展示的经验而有帮助，而其他内容可能因其分享的专业知识而有帮助。

**重要澄清**：E-E-A-T 本身**不是特定的排名因素**。使用可以识别具有良好 E-E-A-T 内容的因素组合是有用的。

### YMYL（Your Money or Your Life）主题

对于可能显著影响人们的健康、财务稳定性、安全或社会福祉的主题（称为 YMYL），系统会给与强 E-E-A-T 一致的内容更多权重。

### 搜索质量评估者指南

[搜索质量评估者](https://www.google.com/search/howsearchworks/how-search-works/rigorous-testing/)是为我们提供算法是否提供良好结果见解的人。评估者经过培训以了解内容是否具有强 E-E-A-T。他们使用的标准在我们的[搜索质量评估者指南](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf)中概述。

**评估者对页面排名没有控制权**。评估者数据不直接用于排名算法。相反，我们将其用作餐厅可能会从用餐者那里获得反馈卡。反馈帮助我们了解系统是否工作良好。

### 问 "Who, How, Why" 关于你的内容

**Who（谁创建了内容）**

- 对访问者来说谁撰写了内容是否显而易见？
- 页面是否带有署名（在可能预期的地方）？
- 署名是否指向有关作者的更多信息？

**How（内容是如何创建的）**

对于读者来说，了解内容是如何产生的很有帮助：

- 对于产品评论，读者理解测试的产品数量、测试结果、测试如何进行（伴随工作证据如照片）可以建立信任
- 如果自动化被用于大量生成内容，披露对访问者来说是否显而易见？
- 是否提供关于自动化或 AI 生成的背景信息？
- 是否解释为什么自动化或 AI 被视为有用？

**Why（为什么创建内容）**

"Why"是最重要的问题。"Why"应该是你主要为人创建有帮助的内容，而不是为吸引搜索引擎访问而创建。

如果 "Why" 是你主要制作内容以吸引搜索引擎访问，这与系统寻求奖励的不一致。如果你使用自动化（包括 AI 生成）以操纵搜索排名为主要目的制作内容，这是违反垃圾政策的行为。

## Page Experience 与 Core Web Vitals

> 来源：[Understanding page experience](https://developers.google.com/search/docs/appearance/page-experience) 和 [Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals)（2025-12-10 更新）

### Core Web Vitals 指标

Core Web Vitals 是衡量页面加载性能、交互性和视觉稳定性的用户体验指标集：

| 指标 | 测量内容 | 良好标准 |
|------|----------|----------|
| **Largest Contentful Paint (LCP)** | 加载性能 | 2.5 秒内发生 |
| **Interaction To Next Paint (INP)** | 响应性 | 小于 200 毫秒 |
| **Cumulative Layout Shift (CLS)** | 视觉稳定性 | 小于 0.1 |

**注意**：INP 已替代之前的 First Input Delay (FID) 指标。

### Page Experience 自检问题

回答"是"以下问题表明你可能提供了良好的页面体验：

- 页面是否有良好的 Core Web Vitals？
- 页面是否以安全方式提供？
- 内容是否在移动设备上显示良好？
- 内容是否避免使用过多分散或干扰主要内容的广告？
- 页面是否避免使用侵入性插页？
- 页面设计是否使访问者能轻松区分主要内容和其他内容？

### Page Experience FAQ

**是否有单一的"页面体验信号"用于排名？**

没有单一信号。核心排名系统查看与整体页面体验一致的各种信号。

**页面体验的哪些方面用于排名？**

Core Web Vitals 用于排名系统。但获得良好分数不保证页面会排名在 Google 搜索结果顶部；除了 Core Web Vitals 分数，还有更多因素构成良好的页面体验。

其他页面体验方面不直接帮助网站在搜索结果中排名更高，但可以使网站更令人满意使用，这通常与排名系统寻求奖励的一致。

**页面体验是在站点级还是页面级评估？**

核心排名系统通常在页面级评估内容，包括理解页面体验相关方面。但也有一些站点级评估。

**页面体验对排名成功有多重要？**

Google Search 始终寻求显示最相关内容，即使页面体验不佳。但对于许多查询，有很多有帮助的内容可用。在这些情况下，拥有良好的页面体验可以有助于在 Search 中成功。

## AI 生成内容指南

> 来源：[Guidance on using generative AI](https://developers.google.com/search/docs/fundamentals/using-gen-ai-content)（2025-12-10 更新）

生成式 AI 在研究主题和为原创内容添加结构时特别有用。但使用 AI 工具生成许多页面而不为用户添加价值可能违反 Google 的垃圾政策关于规模内容滥用的规定。

### 专注于准确性、质量和相关性

为网页创建内容时，专注于准确性、质量和相关性，特别是自动生成内容时。这包括元数据如 `<title>` 元素、meta description 元素、结构化数据和图像替代文本。

对于结构化数据，还需确保遵守通用指南、各个搜索功能的具体政策，并验证标记以确保搜索功能的资格。

### 为用户提供上下文

分享关于内容是如何创建的信息可以为读者提供更多上下文。如果自动生成内容，考虑以适合受众的方式添加关于内容如何创建的信息：

- 提供关于自动化如何使用的更多背景信息
- 添加图像元数据

**电商站点**：Google Merchant Center 有 AI 生成内容的政策。AI 生成的图像必须包含使用 IPTC `DigitalSourceType` `TrainedAlgorithmicMedia` 元数据。AI 生成的产品数据如标题和描述属性必须单独指定并标记为 AI 生成。

## 核心 SEO 实践

> 来源：[SEO Starter Guide](https://developers.google.com/search/docs/fundamentals/seo-starter-guide)

### 让 Google 发现内容

使用 `site:` 搜索运算符检查是否已被索引。Google 主要通过其他已抓取页面的链接发现新页面。也可提交站点地图。

### 让 Google 与用户看到相同页面

Google 需能访问与用户浏览器相同的资源（CSS、JavaScript）。使用 Search Console 的 URL Inspection Tool 检查 Google 如何看待页面。

### 组织网站结构

- 使用描述性 URL
- 按主题分组页面
- 减少重复内容

### 创建有帮助、可靠、以人为本的内容

这是影响搜索结果表现最重要的因素。参见上方详细指南。

### 链接到相关资源

链接帮助用户和搜索引擎发现网站其他部分或其他站点相关页面。编写好的锚文本。

### 影响搜索结果外观

- **标题链接**：独特、清晰简洁、准确描述内容
- **片段**：源自页面实际内容或 meta description 标签

### 添加并优化图像

- 在相关文本附近添加高质量图像
- 添加描述性 alt 文本

### 优化视频

创建高质量视频内容，嵌入在独立页面，靠近相关文本。

## 常见误区澄清

> 来源：Google Search Essentials 和 SEO Starter Guide

| 话题 | 官方立场 |
|------|----------|
| Meta keywords 标签 | Google Search **不使用** keywords meta 标签 |
| 关键词堆砌 | 过度重复词语违反垃圾政策 |
| 域名或 URL 路径中的关键词 | 域名中关键词对排名影响极小 |
| 内容长度最小/最大值 | 长度本身不影响排名，无神奇字数目标（Google 没有） |
| 子域名 vs 子目录 | 从业务角度决定，对 SEO 无显著差异 |
| PageRank | PageRank 是基础算法之一，但搜索有众多排名信号 |
| 重复内容"惩罚" | 内容在多个 URL 可访问不会导致手动行动 |
| 标题数量和顺序 | 标题语义顺序对屏幕阅读器有益，但搜索不依赖顺序 |
| E-E-A-T 是排名因素 | **不是**。E-E-A-T 是评估内容质量的理念，使用因素组合来识别具有良好 E-E-A-T 的内容 |

## 工具与监控

### Search Console

设置 Search Console 账户监控和优化网站表现：

- Core Web Vitals 报告
- HTTPS 报告
- 检查索引状态
- 查看搜索流量和查询
- 调试流量下降
- 提交站点地图
- URL Inspection Tool

### 其他工具

- **PageSpeed Insights**：分析页面性能和 Core Web Vitals
- **Rich Results Test**：测试结构化数据
- **Google Trends**：了解搜索趋势
- **Google Analytics**：结合 Search Console 数据进行 SEO 分析
- **Chrome Lighthouse**：识别页面体验改进范围

## 参考资料

### 最新官方文档（推荐优先查阅）

- [Google Search Essentials](https://developers.google.com/search/docs/essentials)（2025-12-10）
- [Creating helpful, reliable, people-first content](https://developers.google.com/search/docs/fundamentals/creating-helpful-content)（2025-12-10）
- [Understanding page experience](https://developers.google.com/search/docs/appearance/page-experience)（2025-12-10）
- [Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals)（2025-12-10）
- [Guidance on using generative AI](https://developers.google.com/search/docs/fundamentals/using-gen-ai-content)（2025-12-10）
- [A guide to Google Search ranking systems](https://developers.google.com/search/docs/appearance/ranking-systems-guide)
- [Search Quality Rater Guidelines](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf)（PDF）

### 基础文档

- [SEO Starter Guide](https://developers.google.com/search/docs/fundamentals/seo-starter-guide)
- [How Google Search Works](https://developers.google.com/search/docs/fundamentals/how-search-works)
- [Google Search Central Blog](https://developers.google.com/search/blog)
- [Documentation Updates](https://developers.google.com/search/updates)
- [Search Console Documentation](https://support.google.com/webmasters)
