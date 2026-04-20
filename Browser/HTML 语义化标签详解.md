---
title: HTML 语义化标签详解
created: 2026-04-18
updated: 2026-04-18
tags:
  - HTML
  - 语义化
  - 可访问性
  - SEO
  - 前端基础
aliases:
  - Semantic HTML
  - 语义化 HTML
  - HTML 语义元素
  - Semantic Elements
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Glossary/Semantics
  - https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements
  - https://html.spec.whatwg.org/multipage/semantics.html
status: verified
---

## 是什么

**语义化（Semantics）** 指的是代码的"含义"——即某个 HTML 元素**代表什么角色或意义**，而非"它长什么样"。

**语义化标签** 是指那些能够清晰描述其内容含义的 HTML 元素。例如 `<h1>` 表示"页面最高级别标题"，`<article>` 表示"独立完整的内容块"，`<nav>` 表示"导航链接区域"。

与之相对的是**无语义标签**（如 `<div>`、`<span>`），它们只是通用容器，本身不传达任何内容含义。

```html
<!-- ❌ 无语义写法 -->
<div class="header">
  <div class="title">文章标题</div>
</div>

<!-- ✅ 语义化写法 -->
<header>
  <h1>文章标题</h1>
</header>
```

## 为什么重要

### 1. 可访问性（Accessibility）

屏幕阅读器（Screen Reader）依赖语义标签来理解页面结构，帮助视障用户导航。例如：

- 屏幕阅读器可以**直接跳转到 `<nav>` 区域**或**列出所有标题层级**
- `<button>` 比 `<div onclick="...">` 提供完整的键盘支持和焦点管理
- `<main>` 告诉辅助技术"这是页面的核心内容区域"

### 2. 搜索引擎优化（SEO）

搜索引擎爬虫将语义标签视为**内容重要性的信号**：

- `<h1>`~`<h6>` 中的文本被当作关键词影响搜索排名
- `<article>` 和 `<section>` 帮助搜索引擎理解内容结构
- `<time>` 的 `datetime` 属性让日期对机器可读

### 3. 代码可维护性

- 语义化标签让开发者**一眼看出内容类型**，而非在无数 `<div class="xxx">` 中猜测
- 减少了对 class 命名的依赖，样式与结构解耦更清晰
- 团队协作时降低理解成本

### 4. 默认行为与内置功能

许多语义标签自带浏览器默认行为：

| 标签 | 内置行为 |
|------|---------|
| `<button>` | 键盘可聚焦、支持 Enter/Space 触发、表单提交 |
| `<details>` | 原生展开/折叠交互，无需 JavaScript |
| `<dialog>` | 原生模态框，支持 `showModal()` 和 ESC 关闭 |
| `<form>` | 表单验证、提交、重置 |
| `<a href>` | 链接跳转、右键菜单、历史记录 |

## 常用语义化标签分类

### 页面结构标签

| 标签 | 含义 | 使用场景 |
|------|------|---------|
| `<header>` | 页眉/介绍性内容 | 页面顶部、文章头部、section 头部 |
| `<nav>` | 导航链接区域 | 主导航、侧边栏导航、面包屑 |
| `<main>` | 页面主体内容 | 每页仅一个，排除侧边栏和页脚 |
| `<article>` | 独立完整的内容块 | 博客文章、新闻、评论、产品卡片 |
| `<section>` | 通用内容分区 | 有明确主题的内容块，应包含标题 |
| `<aside>` | 侧边/间接相关内容 | 侧边栏、引用框、广告位 |
| `<footer>` | 页脚 | 页面底部、文章作者信息、版权 |
| `<address>` | 联系信息 | 作者联系方式、公司地址 |
| `<search>` | 搜索功能区域 | 搜索表单、筛选控件（HTML 新增） |

### 文本语义标签

| 标签 | 含义 | 示例 |
|------|------|------|
| `<h1>`~`<h6>` | 标题层级 | `<h1>` 每页建议仅一个 |
| `<p>` | 段落 | 文本段落 |
| `<strong>` | 重要/严肃内容 | 浏览器默认加粗 |
| `<em>` | 强调内容 | 浏览器默认斜体 |
| `<mark>` | 高亮标记 | 搜索结果高亮 |
| `<time>` | 时间/日期 | `<time datetime="2026-04-18">今天</time>` |
| `<abbr>` | 缩写 | `<abbr title="World Wide Web">WWW</abbr>` |
| `<cite>` | 作品标题 | 书籍、电影、论文名称 |
| `<code>` | 代码片段 | 内联代码 |
| `<kbd>` | 用户输入 | 键盘按键提示 |
| `<dfn>` | 定义术语 | 术语首次出现时 |
| `<blockquote>` | 长引用 | 块级引用，通常缩进 |
| `<q>` | 短引用 | 内联引用，浏览器自动加引号 |

### 列表与数据标签

| 标签 | 含义 |
|------|------|
| `<ul>` | 无序列表 |
| `<ol>` | 有序列表 |
| `<li>` | 列表项 |
| `<dl>` | 描述列表（键值对） |
| `<dt>` | 描述术语 |
| `<dd>` | 描述内容 |
| `<figure>` | 独立内容（图、表、代码块） |
| `<figcaption>` | figure 的标题/说明 |
| `<table>` | 表格数据 |
| `<caption>` | 表格标题 |

### 交互语义标签

| 标签 | 含义 |
|------|------|
| `<button>` | 按钮 |
| `<form>` | 表单 |
| `<input>` | 输入控件 |
| `<select>` | 下拉选择 |
| `<textarea>` | 多行文本输入 |
| `<label>` | 表单控件标签 |
| `<fieldset>` | 表单控件分组 |
| `<details>` | 可展开/折叠区域 |
| `<summary>` | details 的标题 |
| `<dialog>` | 对话框 |

## 正确使用示例

### 典型文章页面结构

```html
<body>
  <header>
    <nav>
      <ul>
        <li><a href="/">首页</a></li>
        <li><a href="/about">关于</a></li>
      </ul>
    </nav>
  </header>

  <main>
    <article>
      <header>
        <h1>HTML 语义化标签详解</h1>
        <p>作者：<a href="/author/tristan">Tristan</a></p>
        <time datetime="2026-04-18">2026年4月18日</time>
      </header>

      <section>
        <h2>什么是语义化</h2>
        <p>语义化指的是...</p>
      </section>

      <section>
        <h2>为什么重要</h2>
        <p>语义化的好处包括...</p>
      </section>

      <aside>
        <h3>相关阅读</h3>
        <ul>
          <li><a href="/accessibility">可访问性指南</a></li>
        </ul>
      </aside>

      <footer>
        <p>© 2026 版权声明</p>
      </footer>
    </article>
  </main>

  <footer>
    <address>
      联系邮箱：<a href="mailto:example@zoom.us">example@zoom.us</a>
    </address>
  </footer>
</body>
```

### 表单语义化

```html
<form action="/search" method="GET">
  <fieldset>
    <legend>搜索条件</legend>

    <label for="search-input">关键词</label>
    <input type="search" id="search-input" name="q" required>

    <label for="category">分类</label>
    <select id="category" name="category">
      <option value="all">全部</option>
      <option value="docs">文档</option>
    </select>

    <button type="submit">搜索</button>
  </fieldset>
</form>
```

## 限制与注意事项

### `<div>` 和 `<span>` 并非禁用

语义化不等于完全不用 `<div>` 和 `<span>`。它们适用于：

- **纯样式包裹**：仅用于 CSS 布局，无内容含义
- **组件容器**：React/Vue 等框架中的组件根节点
- **无语义分组**：确实没有更合适的语义标签时

```html
<!-- ✅ 合理：仅用于布局 -->
<div class="flex-container">
  <article>...</article>
  <aside>...</aside>
</div>
```

### `<section>` 不等于 `<div>`

`<section>` 用于**有明确主题的内容分区**，通常应包含标题。如果只是为样式或脚本方便而分组，应使用 `<div>`。

```html
<!-- ❌ 错误：无语义分区 -->
<section class="wrapper">
  <p>一些内容</p>
</section>

<!-- ✅ 正确：有明确主题 -->
<section>
  <h2>用户评价</h2>
  <blockquote>...</blockquote>
</section>
```

### `<article>` 可嵌套

`<article>` 可以嵌套使用，内层的 `<article>` 应与外层内容相关但可独立存在：

```html
<article>
  <h1>博客文章标题</h1>
  <p>正文内容...</p>

  <section>
    <h2>评论</h2>
    <article>
      <h3>用户 A 的评论</h3>
      <p>评论内容...</p>
    </article>
  </section>
</article>
```

### `<header>` 和 `<footer>` 可多次使用

它们不仅限于页面级别，也可以用于 `<article>`、`<section>` 等内部：

```html
<article>
  <header>
    <h2>文章标题</h2>
  </header>
  <p>内容...</p>
  <footer>
    <time datetime="2026-04-18">发布于 2026-04-18</time>
  </footer>
</article>
```

### `<main>` 每页仅一个

一个页面只能有一个 `<main>` 元素（不可见的 `<main hidden>` 除外），它代表页面的核心内容。

### 标题层级不应跳跃

标题应按层级顺序使用，不应跳过级别：

```html
<!-- ❌ 错误：h1 直接跳到 h3 -->
<h1>主标题</h1>
<h3>子标题</h3>

<!-- ✅ 正确：逐级递减 -->
<h1>主标题</h1>
<h2>子标题</h2>
<h3>孙标题</h3>
```

## 常见误区

### 误区 1：语义化标签只是为了 SEO

**错误**。语义化的首要价值是**可访问性**，让所有用户（包括使用辅助技术的用户）都能理解和使用页面。SEO 只是附带收益。

### 误区 2：语义化标签在视觉上一定不同

**错误**。语义化标签的默认样式可以通过 CSS 完全自定义。语义关注的是"含义"，而非"外观"。

```html
<!-- 语义正确，样式可任意定制 -->
<h1 style="font-size: 14px; font-weight: normal;">这是一个小标题</h1>
```

### 误区 3：`<b>` 和 `<strong>` 等价，`<i>` 和 `<em>` 等价

**错误**。它们的语义完全不同：

| 标签 | 语义 | 视觉 |
|------|------|------|
| `<b>` | 引起注意，无额外重要性 | 加粗 |
| `<strong>` | 内容重要/严肃/紧急 | 加粗 |
| `<i>` | 语气不同（术语、外来语等） | 斜体 |
| `<em>` | 强调（重读） | 斜体 |

```html
<!-- b：只是视觉上突出 -->
<p>请输入<b>用户名</b>和密码</p>

<!-- strong：内容重要 -->
<p><strong>警告：</strong>此操作不可撤销</p>
```

### 误区 4：`<section>` 比 `<div>` 更"高级"，应该全部替换

**错误**。`<section>` 有特定语义（主题分区），不应盲目替换所有 `<div>`。只有当内容确实构成一个有意义的分区时才使用 `<section>`。

## 与 ARIA 的关系

当 HTML 语义标签不足以表达复杂交互时，可配合 **ARIA（Accessible Rich Internet Applications）** 属性使用：

```html
<!-- 语义标签不足时补充 ARIA -->
<nav aria-label="主导航">
  <ul>...</ul>
</nav>

<!-- 优先使用原生语义标签 -->
<button>点击</button>
<!-- 而非 -->
<div role="button" tabindex="0" onclick="...">点击</div>
```

**ARIA 第一规则**：如果存在能表达所需语义的 HTML 元素，优先使用 HTML 元素而非 ARIA 角色[^aria-first-rule]。

## 最佳实践

| 场景 | 推荐做法 |
|------|---------|
| 页面主内容 | `<main>` |
| 独立可分发内容 | `<article>` |
| 有主题的内容分区 | `<section>` + 标题 |
| 导航链接集合 | `<nav>` |
| 页面/区块头部 | `<header>` |
| 页面/区块底部 | `<footer>` |
| 侧边/相关内容 | `<aside>` |
| 图片+说明 | `<figure>` + `<figcaption>` |
| 时间日期 | `<time datetime="...">` |
| 代码片段 | `<code>` |
| 键盘输入提示 | `<kbd>` |
| 纯样式容器 | `<div>` |
| 纯样式内联容器 | `<span>` |

## 参考资料

- [MDN - Semantics (Glossary)](https://developer.mozilla.org/en-US/docs/Glossary/Semantics)
- [MDN - HTML elements reference](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements)
- [HTML Living Standard - Semantics](https://html.spec.whatwg.org/multipage/semantics.html)
- [MDN - Using HTML sections and outlines](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/Heading_Elements#usage_notes)
- [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/practices/semantic-structure/)

[^aria-first-rule]: 来源：[ARIA Specification - First Rule of ARIA](https://www.w3.org/TR/using-aria/#rule1) — "If you can use a native HTML element with the semantics and behavior you need already built in, instead of re-purposing an element and adding an ARIA role, state or property to make it accessible, then do so."
