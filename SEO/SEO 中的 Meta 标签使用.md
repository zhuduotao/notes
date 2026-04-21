---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - SEO
  - Meta标签
  - HTML
  - 搜索引擎优化
  - Web开发
  - meta-description
  - robots
aliases:
  - Meta标签SEO
  - Meta Tags SEO
  - HTML Meta 元素
source_type: official-doc
source_urls:
  - 'https://developers.google.com/search/docs/crawling-indexing/special-tags'
  - 'https://developers.google.com/search/docs/appearance/snippet'
  - 'https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag'
  - 'https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta'
  - 'https://ziyuan.baidu.com/college/documentinfo?id=193'
status: verified
---
本文整理 Meta 标签在搜索引擎优化中的实际作用、各搜索引擎的支持差异、最佳实践与常见误区。基于 Google Search Central、MDN Web Docs、百度搜索资源平台等官方文档。

## 什么是 Meta 标签

`<meta>` 是 HTML 中用于提供页面元数据的元素，位于 `<head>` 部分。Meta 标签通过 `name` 和 `content` 属性以名称-值对的形式描述页面信息，或通过 `http-equiv` 属性模拟 HTTP 头指令。

> 来源：[MDN `<meta>: The metadata element`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta)

### Meta 标签分类

根据属性不同，Meta 标签提供以下类型的信息：

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | 文档级元数据 | 适用于整个页面的描述性信息 |
| `http-equiv` | pragma 指令 | 模拟 HTTP 头指令 |
| `charset` | 字符集声明 | 声明文档编码 |
| `itemprop` | 用户定义元数据 | 用于结构化数据 |

## Google 支持的 Meta 标签

> 来源：[Google Search Central - Meta tags and attributes that Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)（2025-12-10 更新）

Google 明确支持以下 Meta 标签：

### description

```html
<meta name="description" content="页面描述内容">
```

- **用途**：提供页面的简短描述
- **SEO 影响**：可能用于搜索结果中的摘要（snippet）显示
- **注意事项**：不直接参与排名计算，但影响搜索结果展示和点击率

详见下文 "Meta Description 最佳实践" 章节。

### robots 和 googlebot

```html
<meta name="robots" content="noindex, nofollow">
<meta name="googlebot" content="noindex, nofollow">
```

- **用途**：控制搜索引擎的抓取和索引行为
- **区别**：`robots` 适用于所有搜索引擎，`googlebot` 仅针对 Google
- **冲突处理**：多个冲突规则时，更严格的规则生效

详见下文 "Robots Meta 标签规则详解" 章节。

### notranslate

```html
<meta name="googlebot" content="notranslate">
```

- **用途**：阻止 Google 在搜索结果中提供翻译版本的标题和摘要
- **适用场景**：不希望页面内容被翻译展示时使用

### nopagereadaloud

```html
<meta name="google" content="nopagereadaloud">
```

- **用途**：阻止 Google 文本转语音（TTS）服务朗读页面内容

### google-site-verification

```html
<meta name="google-site-verification" content="验证码">
```

- **用途**：验证 Search Console 网站所有权
- **注意事项**：`name` 和 `content` 值必须精确匹配提供的值（大小写敏感）

### charset

```html
<meta charset="utf-8">
```

- **用途**：声明页面字符编码
- **要求**：HTML5 文档必须使用 UTF-8，声明须位于文档前 1024 字节内

### refresh（Meta Refresh）

```html
<meta http-equiv="refresh" content="3;url=https://example.com">
```

- **用途**：定时刷新或跳转到新 URL
- **SEO 建议**：不推荐用于重定向，应使用服务器端 301 重定向
- **问题**：部分浏览器不支持，可能造成用户体验混乱

### viewport

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

- **用途**：告诉浏览器如何在移动设备上渲染页面
- **SEO 影响**：此标签存在表明页面是移动友好的

### rating

```html
<meta name="rating" content="adult">
```

- **用途**：标记页面包含成人内容，触发 SafeSearch 过滤

### theme-color

```html
<meta name="theme-color" content="#4285f4">
```

- **用途**：定义浏览器地址栏或 UI 的主题颜色

## Google 不支持的 Meta 标签

> 来源：[Google Search Central - Unsupported tags and attributes](https://developers.google.com/search/docs/crawling-indexing/special-tags)

### keywords（已废弃）

```html
<meta name="keywords" content="关键词1, 关键词2">
```

**重要结论**：Google Search **完全不使用** keywords meta 标签，对索引和排名没有任何影响。

这是 SEO 常见误区之一。Google 在 2009 年就已明确声明不再使用 keywords meta 标签[^1]。原因是：keywords 标签容易被滥用堆砌关键词，无法准确反映页面真实内容，且用户不可见，缺乏可信度。

### lang 属性

Google 根据页面文本内容检测语言，不依赖 HTML `lang` 属性。

### next/prev rel 属性

Google 不再使用 `<link rel="next">` 和 `<link rel="prev">` 标签，对索引无影响。

### nositelinkssearchbox

此规则已废弃，站内搜索框功能已不存在。

## Meta Description 最佳实践

> 来源：[Google Search Central - Control your snippets in search results](https://developers.google.com/search/docs/appearance/snippet)（2026-02-04 更新）

### 搜索摘要（Snippet）的生成机制

搜索摘要主要由页面内容自动生成，但 Google 可能使用 meta description 如果它比页面内容更能准确描述页面。

**关键点**：
- 搜索摘要针对不同查询可能显示不同内容
- Meta description 不保证被使用
- 没有长度限制，但搜索结果会根据设备宽度截断

### 最佳实践

#### 1. 每个页面使用独特描述

相同或相似的描述无法帮助用户区分不同页面。优先为首页、热门页面创建描述，数据库驱动的大型网站可考虑程序化生成。

#### 2. 包含相关信息

Meta description 不必须是完整句子，可以包含结构化信息：

```html
<meta name="description" content="作者：张三，绘者：李四，价格：¥99，页数：300">
```

#### 3. 程序化生成

对于产品聚合类网站，可基于产品数据自动生成描述，确保描述多样化和可读性。

#### 4. 避免

- 关键词堆砌
- 全站使用相同描述
- 与页面内容无关的描述
- 过短或空洞的描述

### 示例对比

**差的写法（关键词列表）**：
```html
<meta name="description" content="缝纫用品, 毛线, 彩色铅笔, 缝纫机, 线, 针">
```

**好的写法（具体描述）**：
```html
<meta name="description" content="提供所有缝纫所需用品。周一至周五 8:00-17:00 营业，位于时尚街区。">
```

**差的写法（过于笼统）**：
```html
<meta name="description" content="当地新闻，送到家门口。了解今日发生的事件。">
```

**好的写法（具体新闻摘要）**：
```html
<meta name="description" content="小镇发生意外事件：一位老人在重要活动前夕偷走了所有人的礼物。关注后续实时报道。">
```

## Robots Meta 标签规则详解

> 来源：[Google Search Central - Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)（2026-03-24 更新）

### 基本用法

Robots meta 标签放置于 `<head>` 部分：

```html
<meta name="robots" content="noindex, nofollow">
```

可通过逗号分隔组合多个规则，或使用多个 meta 标签：

```html
<meta name="robots" content="max-snippet:20, max-image-preview:large">
```

### 针对特定爬虫

- `robots`：适用于所有搜索引擎
- `googlebot`：仅针对 Google
- `googlebot-news`：仅针对 Google 新闻结果

### 完整规则列表

| 规则 | 作用 |
|------|------|
| `all` | 无限制（默认值） |
| `noindex` | 不在搜索结果中显示此页面 |
| `nofollow` | 不跟踪此页面上的链接 |
| `none` | 等同于 `noindex, nofollow` |
| `nosnippet` | 不显示文本摘要和视频预览，也阻止内容用于 AI Overviews |
| `max-snippet:[数字]` | 摘要最多使用指定字符数（0=不显示，-1=无限制） |
| `max-image-preview:[设置]` | 图片预览最大尺寸（none/standard/large） |
| `max-video-preview:[数字]` | 视频预览最多使用指定秒数（0=仅静态图，-1=无限制） |
| `notranslate` | 不提供翻译版本 |
| `noimageindex` | 不索引页面上的图片 |
| `unavailable_after:[日期]` | 指定日期后不再显示此页面 |
| `indexifembedded` | 允许嵌入内容被索引（需配合 `noindex`） |

### 已废弃规则

- `noarchive`：Google 已不再显示缓存链接功能
- `nocache`：Google 不使用此规则
- `nositelinkssearchbox`：站内搜索框功能已不存在

### X-Robots-Tag HTTP 头

对于非 HTML 文件（PDF、图片、视频），可使用 HTTP 头：

```
HTTP/1.1 200 OK
X-Robots-Tag: noindex, nofollow
```

示例配置：

**Apache (.htaccess)**：
```apache
<Files ~ "\.pdf$">
  Header set X-Robots-Tag "noindex, nofollow"
</Files>
```

**NGINX**：
```nginx
location ~* \.pdf$ {
  add_header X-Robots-Tag "noindex, nofollow";
}
```

### data-nosnippet 属性

可使用 `data-nosnippet` HTML 属性排除特定元素内容不被用于摘要：

```html
<p>这部分内容可能出现在摘要中
<span data-nosnippet>这部分不会被用于摘要</span>。</p>

<div data-nosnippet>整个区块不用于摘要</div>
```

**要求**：仅支持 `span`、`div`、`section` 元素；HTML 必须有效且标签正确闭合。

## 百度对 Meta 标签的处理

> 来源：[百度搜索引擎优化指南2.0](https://ziyuan.baidu.com/college/documentinfo?id=193)（2014-12-12）

### Meta Description

- **不参与权值计算**：Meta description 不是排名因素
- **仅用于摘要选择**：作为搜索结果摘要的候选来源
- **推荐做法**：
  - 为首页、频道页、产品页等创建独特描述
  - 准确描述内容，不堆砌关键词
  - 每个页面使用不同描述

### Robots Meta 标签

百度支持 `nofollow` 和 `noarchive` 两种 meta 标签：

```html
<meta name="robots" content="noarchive">
<meta name="robots" content="nofollow">
```

- `noarchive`：阻止所有搜索引擎显示网页快照
- `nofollow`：不追踪链接且不传递链接权重

### Keywords Meta 标签

百度官方指南中未提及 keywords meta 标签的作用。根据行业共识，现代搜索引擎普遍不再依赖此标签。

## Bing 对 Meta 标签的处理

> 来源：[Bing SEO 入门指南](https://www.bing.com/webmasters)（参考）

### Meta Description

- 不直接影响排名
- 影响搜索结果展示和点击率
- 建议简洁概括页面内容（约 150-160 字符）

### Keywords Meta 标签

Bing 可能曾使用 keywords 标签，但现代算法中权重极低或已废弃，不建议投入精力。

### 与 Google 的差异

- Bing 对 meta description 的使用可能更直接
- Bing 更重视精确关键词匹配

## 常见误区澄清

### 1. Keywords Meta 标签无用论

**误区**：以为 keywords meta 标签对 SEO 有帮助，花大量时间优化。

**事实**：Google 完全不使用，百度也未明确支持，其他搜索引擎权重极低或已废弃。

### 2. Meta Description 影响排名

**误区**：认为优化 meta description 可以提升排名。

**事实**：
- Google 和百度都明确声明 meta description **不是排名因素**
- 仅用于搜索结果摘要展示，影响点击率而非排名

### 3. Robots Meta 可替代 robots.txt

**误区**：用 robots meta 标签替代 robots.txt。

**事实**：
- robots.txt 控制抓取许可
- robots meta 控制索引和展示
- 若 robots.txt 禁止抓取，则 robots meta 标签不会被读取

### 4. Meta Description 必定被使用

**误区**：写了 meta description 就会在搜索结果中显示。

**事实**：Google 可能根据用户查询从页面内容中提取更相关的摘要，不保证使用 meta description。

### 5. Meta Description 镒度限制

**误区**：meta description 必须在 160 字符内。

**事实**：没有长度限制，但搜索结果会截断。关键是内容质量而非长度。

## 实施检查清单

### 必要 Meta 标签

- [ ] `charset`：使用 UTF-8，位于文档前 1024 字节
- [ ] `viewport`：移动端适配
- [ ] `description`：每个页面独特描述

### 控制类 Meta 标签（按需）

- [ ] `robots`：控制索引行为
- [ ] `google-site-verification`：Search Console 验证
- [ ] `notranslate`：阻止翻译（必要时）

### 不推荐使用

- [ ] ~~`keywords`~~：已被主流搜索引擎废弃
- [ ] ~~`refresh`~~：用于重定向，改用 301
- [ ] ~~`noarchive`~~：Google 已不再显示缓存链接

### 其他注意事项

- [ ] Meta 标签位于 `<head>` 部分
- [ ] HTML 有效且标签正确闭合
- [ ] 避免使用 JavaScript 动态修改 meta 标签（可能不被及时读取）
- [ ] 使用 Search Console URL Inspection Tool 检查 Google 如何解读 meta 标签

## 参考资料

### Google 官方文档

- [Meta tags and attributes that Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)（2025-12-10）
- [Control your snippets in search results](https://developers.google.com/search/docs/appearance/snippet)（2026-02-04）
- [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)（2026-03-24）
- [Influencing your title links in search results](https://developers.google.com/search/docs/appearance/title-link)（2025-12-10）

### HTML 规范

- [MDN `<meta>: The metadata element`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta)（2025-11-07）
- [MDN Standard metadata names](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name)

### 百度官方文档

- [百度搜索引擎优化指南2.0](https://ziyuan.baidu.com/college/documentinfo?id=193)（2014-12-12）

### 相关笔记

- [[SEO/Google SEO 入门指南]]
- [[SEO/百度 SEO 入门指南]]
- [[SEO/Bing SEO 入门指南]]

[^1]: Google 搜索中心博客于 2009 年声明 Google 不使用 keywords meta 标签。见 [Google does not use the keywords meta tag in web ranking](https://developers.google.com/search/blog/2009/09/google-does-not-use-keywords-meta-tag)。
