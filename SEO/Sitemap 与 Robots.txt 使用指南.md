---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - SEO
  - Sitemap
  - Robots.txt
  - 搜索引擎
  - 网站抓取
  - Web开发
aliases:
  - 站点地图
  - 爬虫协议
  - Robots协议
  - Sitemap指南
  - Robots协议使用
source_type: official-doc
source_urls:
  - >-
    https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview
  - 'https://developers.google.com/search/docs/crawling-indexing/robots/intro'
  - >-
    https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap
  - 'https://developers.google.com/crawling/docs/robots-txt/create-robots-txt'
  - 'https://ziyuan.baidu.com/college/documentinfo?id=193'
status: verified
---
本文整理自 Google Search Central 和百度搜索资源平台官方文档（截至 2025-12-10 最新更新），介绍 Sitemap 和 robots.txt 在 SEO 中的作用、使用方法及最佳实践。

## Sitemap（站点地图）

### 是什么

**Sitemap** 是一个文件，用于提供网站页面、视频和其他文件的信息，以及它们之间的关系。搜索引擎（如 Google）读取此文件以更高效地抓取网站。

Sitemap 告知搜索引擎：
- 哪些页面和文件是重要的
- 页面最后更新时间
- 页面的其他语言版本

### 为什么重要

搜索引擎主要通过已抓取页面中的链接发现新页面。但对于某些网站，仅靠链接发现可能不够充分：

**建议使用 Sitemap 的场景：**

- **大型网站**：难以确保每个页面都被至少一个其他页面链接，爬虫可能遗漏部分新页面
- **新网站且外链少**：Googlebot 通过已抓取页面的链接发现新 URL，若无外链可能无法发现
- **富媒体内容多**：大量视频、图片内容，或内容出现在 Google News 中

**可能不需要 Sitemap 的场景：**

- **小型网站**：约 500 页或更少（仅计入希望出现在搜索结果中的页面）
- **内部链接完善**：所有重要页面都能从首页通过链接访问到
- **无大量媒体文件或新闻页面**：不需要 Sitemap 帮助发现和理解视频、图像、新闻文章

### 支持的格式

Google 支持以下格式（选择最适合网站的格式，Google 无偏好）：

| 格式 | 优点 | 缺点 |
|------|------|------|
| **XML Sitemap** | 最通用、可扩展，可提供图像、视频、新闻内容及页面本地化版本信息 | 大型站点维护复杂 |
| **RSS/mRSS/Atom 1.0** | CMS 自动生成，可提供视频信息 | 仅能提供视频信息（无图像、新闻），处理繁琐 |
| **文本 Sitemap** | 简单易维护 | 仅支持 HTML 和其他可索引文本内容 URL |

### 最佳实践

**大小限制：**
- 单个 Sitemap 文件：≤ 50MB（未压缩）或 ≤ 50,000 URLs
- 超过限制需拆分为多个 Sitemap，可选创建 Sitemap index 文件

**文件编码与位置：**
- 必须使用 UTF-8 编码
- 可放置在网站任何位置，但仅影响父目录下的后代文件
- **推荐放置在网站根目录**，可影响全站文件

**URL 规范：**
- 使用完整、绝对 URL（如 `https://www.example.com/mypage.html`，而非 `/mypage.html`）
- 包含希望在搜索结果中出现的 URL（通常为规范 URL）
- 若有移动版和桌面版，建议在 Sitemap 中仅指向一个版本

**XML Sitemap 特殊说明：**
- Google **忽略 `<priority>` 和 `<changefreq>` 值**
- 若 `<lastmod>` 值一致且可验证（如对比页面最后修改时间），Google 会使用该值
- `<lastmod>` 应反映页面重要更新的日期时间（如主要内容、结构化数据、链接更新，而非版权日期更新）

### 创建方法

根据网站规模选择：

**CMS 自动生成：**
- WordPress、Wix、Blogger 等通常自动生成 Sitemap
- 搜索 CMS 文档了解 Sitemap 生成方法

**手动创建（适合几十个 URL）：**
- 使用文本编辑器（如 Windows Notepad、Nano）按格式编写
- 适合小型网站，长期维护困难

**自动生成（适合大量 URL）：**
- 使用第三方工具生成
- 最佳方式：网站软件自动生成（从数据库提取 URL 并导出）

### XML Sitemap 示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://www.example.com/foo.html</loc>
    <lastmod>2022-06-04</lastmod>
  </url>
</urlset>
```

### 文本 Sitemap 示例

```
https://www.example.com/file1.html
https://www.example.com/file2.html
```

### 提交方式

**方式一：Search Console**
- 使用 [Sitemaps 报告](https://support.google.com/webmasters/answer/7451001)提交
- 可查看 Googlebot 访问时间和潜在处理错误

**方式二：robots.txt 引用**
- 在 robots.txt 文件任意位置添加：
  ```
  Sitemap: https://example.com/my_sitemap.xml
  ```
- Google 下次抓取 robots.txt 时将发现

**方式三：Search Console API**
- 使用 API [程序化提交](https://developers.google.com/webmaster-tools/v1/sitemaps/submit)

**方式四：WebSub（Atom/RSS）**
- 若使用 Atom 或 RSS，可使用 [WebSub](https://www.w3.org/TR/websub/) 向搜索引擎广播变更

**重要：**提交 Sitemap 仅是提示，不保证 Google 会下载或使用。

### 扩展类型

使用 Google 支持的 Sitemap 扩展可提供更多信息：

- **图像 Sitemap**：提供页面中图像位置信息
- **视频 Sitemap**：提供视频运行时长、评分、适龄评级等
- **新闻 Sitemap**：提供文章标题和发布日期

## Robots.txt（爬虫协议）

### 是什么

**robots.txt** 是一个文件，用于告知搜索引擎爬虫可以访问网站上的哪些 URL。

### 主要用途

robots.txt 主要用于管理爬虫流量，**不是隐藏网页的机制**。

| 文件类型 | robots.txt 作用 |
|----------|----------------|
| **网页（HTML、PDF等）** | 管理抓取流量，避免服务器过载；避免抓取不重要或相似页面 |
| **媒体文件** | 管理抓取流量；防止图像、视频、音频出现在搜索结果中 |
| **资源文件** | 阻止不重要脚本、样式文件（若不影响页面理解） |

**警告：**不要使用 robots.txt 隐藏网页（包括 PDF 等文本格式）！

若网页被 robots.txt 阻止：
- URL 可能仍出现在搜索结果中（如有外链指向）
- 搜索结果将无描述
- 嵌入的图像、视频、PDF 等可能不被抓取

**正确隐藏网页方法：**
- 使用 [`noindex` meta 标签](https://developers.google.com/search/docs/crawling-indexing/block-indexing)
- 密码保护页面
- 完全删除页面

### 文件位置规则

- 文件必须命名为 `robots.txt`
- 每个网站仅能有**一个** robots.txt 文件
- 必须放置在**网站根目录**（如 `https://www.example.com/robots.txt`）
- 不能放在子目录
- 可用于子域名或非标准端口
- 仅适用于发布位置的协议、主机和端口下的路径
- 必须是 UTF-8 编码文本文件（包含 ASCII）

### 规则语法

robots.txt 由一个或多个**规则组**组成，每组包含：

- `User-agent:` 指定规则适用的爬虫（必需，每组一个或多个）
- `Disallow:` 禁止抓取的路径（每组至少一个 Disallow 或 Allow）
- `Allow:` 允许抓取的路径（用于在禁止目录中允许子目录或页面）
- `sitemap:` Sitemap 位置（可选，可多个）

**处理规则：**
- 爬虫从上到下处理规则组
- 用户代理仅匹配第一个最具体的规则组
- 默认允许抓取未被 Disallow 阻止的任何页面或目录
- 规则区分大小写
- `#` 开启注释（被忽略）

**通配符支持：**
- `*`：匹配 0 或多个任意字符（可用于路径前缀、后缀或整个字符串）
- `$`：匹配行结束符

### 规则示例

**阻止特定爬虫访问特定目录：**

```txt
User-agent: Googlebot
Disallow: /nogooglebot/

User-agent: *
Allow: /

Sitemap: https://www.example.com/sitemap.xml
```

含义：
1. Googlebot 不能抓取 `https://example.com/nogooglebot/` 下的任何 URL
2. 所有其他爬虫可抓取整个网站
3. Sitemap 位于 `https://www.example.com/sitemap.xml`

**阻止所有爬虫访问整个网站：**

```txt
User-agent: *
Disallow: /
```

**阻止所有爬虫访问特定目录：**

```txt
User-agent: *
Disallow: /cgi-bin/
Disallow: /tmp/
Disallow: /private/
```

**允许特定爬虫访问，阻止其他：**

```txt
User-agent: Googlebot
Allow: /

User-agent: *
Disallow: /
```

**使用通配符：**

```txt
# 阻止所有以 .pdf 结尾的文件
User-agent: *
Disallow: /*.pdf$

# 阻止所有包含 private 的路径
User-agent: *
Disallow: /*private*
```

### 局限性

robots.txt 有重要限制：

1. **不保证所有搜索引擎支持**：指令无法强制执行，取决于爬虫遵守。Googlebot 等信誉良好的爬虫遵守规则，其他可能不遵守。机密信息应使用密码保护。

2. **不同爬虫解释可能不同**：应了解针对不同爬虫的正确语法，部分可能不理解某些指令。

3. **被禁止页面仍可能被索引**：若其他网站链接到被 robots.txt 禁止的页面，URL 和锚文本等信息仍可能出现在搜索结果中。正确阻止方法是密码保护、noindex 或删除页面。

### 创建流程

1. 使用文本编辑器创建 robots.txt（UTF-8 编码）
2. 添加规则
3. 上传至网站根目录
4. 测试文件是否公开可访问（浏览器打开 `https://example.com/robots.txt`）
5. 测试规则是否正确（[Search Console robots.txt 报告](https://support.google.com/webmasters/answer/6062598) 或 [Google robots.txt 开源库](https://github.com/google/robotstxt)）

### 百度 Robots 协议说明

> 来源：百度搜索资源平台

**百度严格遵守 robots 协议。**

**注意事项：**
- 路径大小写需精确匹配，否则协议无法生效
- 支持通配符 `*`（匹配 0 或多个任意字符）和 `$`（匹配行结束符）

**百度支持的 Robots Meta 标签：**
- `nofollow`：不追踪此网页链接且不传递链接权重
- `noarchive`：不显示网页快照

示例：
```html
<!-- 防止所有搜索引擎显示网站快照 -->
<meta name="robots" content="noarchive">

<!-- 不追踪链接且不传递权重 -->
<meta name="robots" content="nofollow">
```

**视频资源 Robots 协议（2018年9月升级）：**
- 未设置 robots：百度收录视频 URL 时包含播放页 URL、视频文件、周边文本等信息
- 已收录短视频：呈现为**视频极速体验页**
- 综艺影视类长视频：仅收录页面 URL

## 常见误区

### Sitemap 误区

| 误区 | 正确理解 |
|------|----------|
| 提交 Sitemap 保证收录 | 仅是提示，不保证 Google 下载或使用 |
| Sitemap 可隐藏页面 | 不能隐藏，应使用 noindex 或密码保护 |
| priority 和 changefreq 影响排名 | Google 忽略这两个值 |
| 必须使用 XML 格式 | RSS/Atom/文本格式均可，选择适合的即可 |

### Robots.txt 误区

| 误区 | 正确理解 |
|------|----------|
| robots.txt 可隐藏网页 | 不能隐藏，URL 可能仍被索引（如有外链） |
| robots.txt 规则强制执行 | 取决于爬虫遵守，机密信息应密码保护 |
| 所有爬虫都遵守 robots.txt | 信誉良好的爬虫遵守，其他可能不遵守 |
| robots.txt 阻止索引 | 仅阻止抓取，如有外链 URL 可能仍被索引 |

## 使用建议

### Sitemap 建议

- 定期更新 Sitemap，反映页面变更
- 仅包含希望出现在搜索结果中的 URL
- 大型网站拆分 Sitemap 或使用 Sitemap index
- 使用 Search Console 监控 Sitemap 状态
- CMS 自动生成的 Sitemap 通常无需干预

### Robots.txt 建议

- 仅阻止真正不需要抓取的内容
- 不要阻止重要资源（CSS、JS、图像），否则影响页面理解
- 测试规则是否正确生效
- 机密内容使用密码保护而非 robots.txt
- 在 robots.txt 中声明 Sitemap 位置

## 参考资料

### Google 官方文档（推荐优先查阅）

- [Learn about sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview)（2025-12-10）
- [Build and submit a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)（2025-12-10）
- [Introduction to robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/intro)（2025-12-10）
- [Create and submit a robots.txt file](https://developers.google.com/crawling/docs/robots-txt/create-robots-txt)（2025-11-21）
- [How Google interprets the robots.txt specification](https://developers.google.com/crawling/docs/robots-txt/robots-txt-spec)

### 百度官方文档

- [百度搜索引擎优化指南2.0](https://ziyuan.baidu.com/college/documentinfo?id=193)（2014-12-12）
- [Robots协议升级公告](https://ziyuan.baidu.com/wiki/2579)（2018-09-13）

### 规范文档

- [sitemaps.org 协议](https://www.sitemaps.org/protocol.html) - Sitemap 官方规范
- [Robots Exclusion Standard](https://en.wikipedia.org/wiki/Robots_exclusion_standard) - Robots 协议标准

### 工具

- [Search Console Sitemaps 报告](https://support.google.com/webmasters/answer/7451001)
- [Search Console robots.txt 报告](https://support.google.com/webmasters/answer/6062598)
- [Google robots.txt 开源库](https://github.com/google/robotstxt)
