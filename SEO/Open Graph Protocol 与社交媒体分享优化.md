---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - SEO
  - Open-Graph-Protocol
  - OGP
  - 社交媒体
  - Facebook
  - Twitter-Card
  - LinkedIn
  - Meta-标签
  - Web开发
aliases:
  - OGP
  - Open Graph
  - 开放图谱协议
  - 社交分享优化
  - og meta
source_type: spec
source_urls:
  - 'https://ogp.me/'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name
status: verified
---
Open Graph Protocol（OGP，开放图谱协议）是一套元数据规范，使任何网页都能在社交图谱（Social Graph）中成为富媒体对象。最典型的应用场景是：当用户在 Facebook、Twitter/X、LinkedIn 等社交平台分享链接时，平台通过读取 OGP 标签来展示标题、图片、描述等信息，而非仅显示无吸引力的纯链接。

OGP 由 Facebook 于 2010 年创建，灵感源自 Dublin Core、link-rel canonical、Microformats 和 RDFa。规范在 Open Web Foundation Agreement 0.9 下开放，官网源码托管于 [GitHub](https://github.com/facebook/open-graph-protocol)。

> 来源：[Open Graph Protocol 官方规范](https://ogp.me/)

## 核心原理

### 为什么需要 OGP

社交平台在展示分享链接时，需要从网页中提取关键信息（标题、描述、图片）。传统方式是平台自行爬取页面并猜测内容，但存在以下问题：

- 平台爬虫能力有限，可能无法正确解析复杂页面结构
- 提取的信息不准确或不完整
- 图片可能缺失或尺寸不合适
- 缺乏对内容类型的语义理解

OGP 通过标准化的 `<meta>` 标签让网页**主动声明**分享时应展示的信息，解决上述问题。网页添加 OGP 标签后，在社交平台分享时将呈现**富媒体卡片**而非普通链接，提升点击率和传播效果。

### 技术基础

OGP 基于 **RDFa**（Resource Description Framework in Attributes），使用 `<meta>` 标签的 `property` 和 `content` 属性在 `<head>` 中声明元数据：

```html
<meta property="og:title" content="页面标题" />
```

**与传统 meta 标签的区别**：

| 类型 | 属性 | 示例 | 用途 |
|------|------|------|------|
| 标准 meta | `name` / `content` | `<meta name="description" content="...">` | SEO、浏览器 |
| OGP meta | `property` / `content` | `<meta property="og:title" content="...">` | 社交平台 |

> 来源：[MDN - meta name attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name)

### HTML 前缀声明

在 `<html>` 标签中声明 OGP 前缀，确保属性被正确识别：

```html
<html prefix="og: https://ogp.me/ns#">
```

对于自定义类型，需声明额外命名空间：

```html
<head prefix="my_namespace: https://example.com/ns#">
<meta property="og:type" content="my_namespace:my_type" />
```

## 必需属性

每个使用 OGP 的页面**必须**包含四个基础属性：

| 属性 | 说明 | 示例 |
|------|------|------|
| `og:title` | 对象标题，应出现在社交图谱中 | `"The Rock"` |
| `og:type` | 对象类型，见下文类型列表 | `"video.movie"` |
| `og:image` | 代表对象的图片 URL | `"https://example.com/rock.jpg"` |
| `og:url` | 规范 URL，作为对象在图谱中的永久 ID | `"https://www.imdb.com/title/tt0117500/"` |

**完整示例**：

```html
<html prefix="og: https://ogp.me/ns#">
<head>
<title>The Rock (1996)</title>
<meta property="og:title" content="The Rock" />
<meta property="og:type" content="video.movie" />
<meta property="og:url" content="https://www.imdb.com/title/tt0117500/" />
<meta property="og:image" content="https://ia.media-imdb.com/images/rock.jpg" />
</head>
</html>
```

> 来源：[OGP - Basic Metadata](https://ogp.me/#metadata)

## 可选属性

以下属性为可选但通常推荐：

| 属性 | 说明 |
|------|------|
| `og:audio` | 伴随对象的音频文件 URL |
| `og:description` | 一到两句描述 |
| `og:determiner` | 标题前的冠词，枚举值：`a`, `an`, `the`, `""`, `auto` |
| `og:locale` | 标签语言，格式 `language_TERRITORY`，默认 `en_US` |
| `og:locale:alternate` | 页面其他可用语言的数组 |
| `og:site_name` | 整体网站名称，如 `"IMDb"` |
| `og:video` | 伴随对象的视频文件 URL |

**示例**：

```html
<meta property="og:description" content="Sean Connery found fame and fortune as the suave British agent, James Bond." />
<meta property="og:locale" content="en_GB" />
<meta property="og:locale:alternate" content="fr_FR" />
<meta property="og:locale:alternate" content="es_ES" />
<meta property="og:site_name" content="IMDb" />
```

> 来源：[OGP - Optional Metadata](https://ogp.me/#optional)

## 结构化属性

部分属性可附加额外元数据，通过 `:` 扩展。

### og:image 结构化属性

| 属性 | 说明 |
|------|------|
| `og:image:url` | 等同于 `og:image` |
| `og:image:secure_url` | HTTPS 版本 URL（如网页需要 HTTPS） |
| `og:image:type` | 图片 MIME 类型 |
| `og:image:width` | 宽度像素值 |
| `og:image:height` | 高度像素值 |
| `og:image:alt` | 图片内容描述（非标题），应与 `og:image` 同时指定 |

**完整示例**：

```html
<meta property="og:image" content="http://example.com/ogp.jpg" />
<meta property="og:image:secure_url" content="https://secure.example.com/ogp.jpg" />
<meta property="og:image:type" content="image/jpeg" />
<meta property="og:image:width" content="400" />
<meta property="og:image:height" content="300" />
<meta property="og:image:alt" content="A shiny red apple with a bite taken out" />
```

### og:video 结构化属性

与 `og:image` 类似的属性：`secure_url`, `type`, `width`, `height`。

```html
<meta property="og:video" content="http://example.com/movie.swf" />
<meta property="og:video:secure_url" content="https://secure.example.com/movie.swf" />
<meta property="og:video:type" content="application/x-shockwave-flash" />
<meta property="og:video:width" content="400" />
<meta property="og:video:height" content="300" />
```

### og:audio 结构化属性

仅支持 `secure_url` 和 `type`（音频无尺寸概念）：

```html
<meta property="og:audio" content="http://example.com/sound.mp3" />
<meta property="og:audio:secure_url" content="https://secure.example.com/sound.mp3" />
<meta property="og:audio:type" content="audio/mpeg" />
```

> 来源：[OGP - Structured Properties](https://ogp.me/#structured)

## 数组属性

部分属性可声明多个值，使用多个 `<meta>` 标签。**第一个标签优先级最高**。

```html
<meta property="og:image" content="https://example.com/rock.jpg" />
<meta property="og:image" content="https://example.com/rock2.jpg" />
```

**结构化属性的数组规则**：结构化属性需紧跟其根标签，遇到新根标签则视为上一项结束。

```html
<meta property="og:image" content="https://example.com/rock.jpg" />
<meta property="og:image:width" content="300" />
<meta property="og:image:height" content="300" />
<meta property="og:image" content="https://example.com/rock2.jpg" />
<meta property="og:image" content="https://example.com/rock3.jpg" />
<meta property="og:image:height" content="1000" />
```

上述表示 3 张图片：第一张 300x300，第二张尺寸未指定，第三张高度 1000px。

> 来源：[OGP - Arrays](https://ogp.me/#array)

## 对象类型

通过 `og:type` 指定对象类型，不同类型可能需要额外属性。

### 常用全局类型

| 类型 | 说明 | 命名空间 |
|------|------|----------|
| `website` | 普通网页，无额外属性 | `https://ogp.me/ns/website#` |
| `article` | 文章 | `https://ogp.me/ns/article#` |
| `profile` | 个人/组织资料 | `http://ogp.me/ns/profile#` |
| `book` | 书籍 | `https://ogp.me/ns/book#` |

> 未标记 `og:type` 的网页默认视为 `website`。

### article 类型属性

```html
<meta property="og:type" content="article" />
<meta property="article:published_time" content="2025-01-01T00:00:00Z" />
<meta property="article:modified_time" content="2025-01-15T00:00:00Z" />
<meta property="article:author" content="https://example.com/author/profile" />
<meta property="article:section" content="Technology" />
<meta property="article:tag" content="web" />
<meta property="article:tag" content="SEO" />
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `article:published_time` | datetime | 发布时间 |
| `article:modified_time` | datetime | 最后修改时间 |
| `article:expiration_time` | datetime | 过期时间（可选） |
| `article:author` | profile 数组 | 作者 |
| `article:section` | string | 高级分类名 |
| `article:tag` | string 数组 | 标签 |

### profile 类型属性

```html
<meta property="og:type" content="profile" />
<meta property="profile:first_name" content="John" />
<meta property="profile:last_name" content="Doe" />
<meta property="profile:username" content="johndoe" />
```

### 音乐/视频类型

命名空间分别为 `https://ogp.me/ns/music#` 和 `https://ogp.me/ns/video#`：

- `music.song`, `music.album`, `music.playlist`, `music.radio_station`
- `video.movie`, `video.episode`, `video.tv_show`, `video.other`

详细属性见 [OGP 官方文档 Object Types 章节](https://ogp.me/#types)。

> 来源：[OGP - Object Types](https://ogp.me/#types)

## 数据类型规范

OGP 定义了以下数据类型：

| 类型 | 说明 | 格式示例 |
|------|------|----------|
| Boolean | 真假值 | `true`, `false`, `1`, `0` |
| DateTime | 时间值 | ISO 8601，如 `2025-01-01T00:00:00Z` |
| Enum | 有界字符串枚举 | 枚举成员值 |
| Float | 64 位浮点数 | `1.234`, `-1.2e3` |
| Integer | 32 位整数 | `1234`, `-123` |
| String | Unicode 字符序列 | Unicode 字符，无转义 |
| URL | HTTP/HTTPS URL | `https://example.com` |

> 来源：[OGP - Types](https://ogp.me/#data_types)

## 社交媒体平台支持

### Facebook

Facebook 是 OGP 的创建者，对 OGP 支持最完整：

- 读取所有标准 OGP 属性
- 提供官方调试工具：[Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/)
- 图片建议尺寸：至少 200x200，推荐 1200x630（1.91:1 比例）
- 支持 `article`、`video` 等类型的富媒体展示

**调试工具用途**：
- 验证 OGP 标签是否正确
- 预览分享卡片效果
- 强制刷新 Facebook 缓存（点击"Scrape Again"）

> Facebook 开发者文档需登录访问，核心调试工具公开可用。

### Twitter/X

Twitter/X 使用 **Twitter Cards** 系统，与 OGP 并存：

| Card 类型 | 说明 | 所需标签 |
|-----------|------|----------|
| `summary` | 标题+描述+小图 | `twitter:card`, `twitter:title`, `twitter:description`, `twitter:image` |
| `summary_large_image` | 大图卡片 | 同上，图片更大 |
| `player` | 视频/音频播放器 | 额外需 `twitter:player` 等 |

**Twitter 专用标签**：

```html
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@username" />
<meta name="twitter:title" content="标题" />
<meta name="twitter:description" content="描述" />
<meta name="twitter:image" content="https://example.com/image.jpg" />
```

**与 OGP 的关系**：Twitter 会优先读取 Twitter 专用标签，若缺失则回退到对应 OGP 标签：

- `twitter:title` → `og:title`
- `twitter:description` → `og:description`
- `twitter:image` → `og:image`

因此，如果 Twitter 专用标签未定义，OGP 标签也能生效。

**调试工具**：[Twitter Card Validator](https://cards-dev.twitter.com/)（需登录）

> 来源：Twitter/X 开发者文档（部分文档路径可能变更）

### LinkedIn

LinkedIn 支持 OGP，分享时展示富媒体卡片：

- 读取 `og:title`, `og:description`, `og:image`, `og:url`
- 图片建议尺寸：至少 1200x627
- 提供 [Post Inspector](https://www.linkedin.com/post-inspector/) 工具验证分享效果

### 其他平台

| 平台 | OGP 支持 |
|------|----------|
| Pinterest | 支持，用于 Pin 展示 |
| Slack | 支持，链接展开显示标题/图片 |
| WhatsApp | 支持，消息预览 |
| Telegram | 支持，消息预览 |
| iMessage | 支持，链接预览 |
| Discord | 支持，链接嵌入 |

### 搜索引擎

- **Google**：Google Search 不直接使用 OGP 进行排名，但可能用于搜索结果展示。Google 主要依赖 Schema.org 结构化数据。
- **Bing**：可能参考 OGP 信息，但同样更依赖 Schema.org。

> OGP 主要服务于社交平台分享，与 SEO 的结构化数据（Schema.org）目的不同但可并存。

## 最佳实践

### 图片规格建议

| 平台 | 推荐尺寸 | 比例 | 最小尺寸 |
|------|----------|------|----------|
| Facebook | 1200x630 | 1.91:1 | 200x200 |
| Twitter (large) | 1200x600 | 2:1 | 300x157 |
| LinkedIn | 1200x627 | 1.91:1 | 1200x627 |

**通用建议**：
- 使用 **1200x630** 兼容多数平台
- 图片文件大小 < 8MB（Facebook 限制）
- 使用 HTTPS URL
- 指定 `og:image:alt` 提供无障碍支持
- 声明 `og:image:width` 和 `og:image:height` 避免平台重新下载测量

### 标签放置位置

- 所有 OGP 标签放在 `<head>` 中
- `<html>` 标签声明 `prefix="og: https://ogp.me/ns#"`
- 结构化属性紧跟其根标签

### 与标准 Meta 标签配合

OGP 与 SEO meta 标签可并存：

```html
<head>
<!-- SEO meta -->
<title>页面标题 - 网站名</title>
<meta name="description" content="用于搜索引擎的描述" />

<!-- OGP -->
<meta property="og:title" content="页面标题" />
<meta property="og:description" content="用于社交分享的描述（可与 SEO 不同）" />
<meta property="og:type" content="article" />
<meta property="og:url" content="https://example.com/article" />
<meta property="og:image" content="https://example.com/social.jpg" />
</head>
```

**注意**：`og:description` 与 `meta name="description"` 可不同，前者面向社交用户，后者面向搜索引擎。

### URL 规范化

`og:url` 应指向规范 URL，与 `<link rel="canonical">` 保持一致：

```html
<link rel="canonical" href="https://example.com/article" />
<meta property="og:url" content="https://example.com/article" />
```

### 缓存刷新

社交平台会缓存 OGP 数据。内容更新后需手动刷新：

- **Facebook**：使用 Sharing Debugger 点击"Scrape Again"
- **Twitter/X**：使用 Card Validator 重新验证
- **LinkedIn**：使用 Post Inspector

## 常见问题

### 为什么分享时不显示图片？

可能原因：
1. 图片 URL 不可访问（检查 HTTPS、权限）
2. 图片尺寸过小（低于平台最小要求）
3. 平台缓存未更新（使用调试工具刷新）
4. OGP 标签语法错误（缺少 `property` 属性）

### OGP 与 Schema.org 的关系

| 特性 | OGP | Schema.org |
|------|-----|------------|
| 目的 | 社交平台分享 | 搜索引擎结构化数据 |
| 格式 | `<meta>` 标签 | JSON-LD / Microdata |
| 主要用户 | Facebook, Twitter, LinkedIn | Google, Bing |
| 内容类型 | 侧重社交媒体展示 | 侧重语义理解 |

两者**可并存**，针对不同场景。对于 SEO，Schema.org 更重要；对于社交传播，OGP 更重要。

### Twitter Card 与 OGP 如何选择

推荐做法：
1. 设置完整 OGP 标签（覆盖多数平台）
2. 额外添加 `twitter:card` 指定卡片类型
3. 其他 Twitter 标签可选（缺失时回退到 OGP）

最小配置：

```html
<meta property="og:title" content="标题" />
<meta property="og:description" content="描述" />
<meta property="og:image" content="https://example.com/img.jpg" />
<meta name="twitter:card" content="summary_large_image" />
```

### og:type 是否必需

官方规范要求 `og:type` 为必需属性。实践中：
- Facebook 对缺失 `og:type` 较宽容，默认视为 `website`
- 建议显式声明以符合规范

### 动态内容如何处理

对于动态生成内容的页面：
- 服务端渲染时注入 OGP 标签
- 单页应用（SPA）需确保 OGP 标签在 HTML 源码中（而非 JS 动态插入），因为社交平台爬虫可能不执行 JS
- 可使用预渲染方案（如 prerender.io）或 SSR

## 调试工具

| 工具 | 平台 | URL | 功能 |
|------|------|-----|------|
| Facebook Sharing Debugger | Facebook | [developers.facebook.com/tools/debug/](https://developers.facebook.com/tools/debug/) | 验证、预览、缓存刷新 |
| Twitter Card Validator | Twitter/X | [cards-dev.twitter.com](https://cards-dev.twitter.com/) | 验证卡片效果 |
| LinkedIn Post Inspector | LinkedIn | [linkedin.com/post-inspector](https://www.linkedin.com/post-inspector/) | 验证分享效果 |
| Google Rich Results Test | Google | [search.google.com/test/rich-results](https://search.google.com/test/rich-results) | 测试结构化数据（含部分 OGP） |

> 来源：[OGP - Implementations](https://ogp.me/#implementations)

## 参考资料

### 官方规范

- [Open Graph Protocol](https://ogp.me/) - 官方规范文档（GitHub 源码：[facebook/open-graph-protocol](https://github.com/facebook/open-graph-protocol)）

### 社交平台文档

- [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/) - 官方调试工具
- [Twitter Card Validator](https://cards-dev.twitter.com/) - Twitter 卡片验证
- [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/) - LinkedIn 分享验证

### 相关 Web 标准

- [MDN - meta name attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name) - 标准 meta 标签参考
- [RDFa](https://en.wikipedia.org/wiki/RDFa) - OGP 的技术基础
- [Dublin Core](https://en.wikipedia.org/wiki/Dublin_Core) - OGP 灵感来源之一

### 相关概念

- Schema.org 结构化数据（用于 SEO）
- JSON-LD（Google 推荐的结构化数据格式）
- Microformats、Microdata
