---
title: HTML script 标签 async 与 defer 属性详解
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - HTML
  - 浏览器
  - 性能优化
  - 脚本加载
aliases:
  - script async defer
  - script async 和 defer 区别
  - async vs defer
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script
  - https://html.spec.whatwg.org/multipage/scripting.html#the-script-element
status: verified
---

## 是什么

`async` 和 `defer` 是 HTML `<script>` 标签的两个布尔属性，用于控制外部脚本（带有 `src` 属性）的**下载**和**执行**时机，核心目的是**避免脚本阻塞 HTML 解析**，从而提升页面加载性能。

两个属性都**仅对带有 `src` 属性的外部脚本生效**，对内联脚本（inline script）无效。

## 为什么重要

默认情况下，浏览器遇到 `<script>` 标签时会：

1. **停止解析 HTML**
2. **下载脚本**（如果是外部脚本）
3. **执行脚本**
4. **继续解析 HTML**

这种"解析阻塞"行为会导致页面渲染延迟，用户看到白屏的时间变长。`async` 和 `defer` 通过让脚本在后台异步下载，解决了这个问题。

## 核心区别

| 特性 | 默认（无属性） | `async` | `defer` |
|------|---------------|---------|---------|
| **下载时机** | 遇到标签时同步下载 | 与 HTML 解析并行下载 | 与 HTML 解析并行下载 |
| **是否阻塞解析** | 是 | 否（下载不阻塞） | 否（下载不阻塞） |
| **执行时机** | 下载完成后立即执行，阻塞解析 | 下载完成后立即执行，**短暂阻塞解析** | HTML 解析完成后执行 |
| **执行顺序** | 按出现顺序 | **不保证顺序**（谁先下载完谁先执行） | **严格按出现顺序** |
| **触发 DOMContentLoaded** | 脚本执行完后才触发 | 不等待 async 脚本 | 等待所有 defer 脚本执行完 |
| **适用场景** | 必须在解析期间执行的脚本 | 独立、无依赖的脚本 | 依赖 DOM 或其他脚本的代码 |

## 详细行为

### async（异步）

- 脚本在后台**并行下载**，不阻塞 HTML 解析
- 下载完成后**立即暂停 HTML 解析**并执行脚本
- 执行完毕后恢复 HTML 解析
- **多个 async 脚本的执行顺序不确定**，取决于哪个先下载完成
- 如果同时指定 `async` 和 `defer`，浏览器会**只按 `async` 行为处理**[^mdn-async-defer-conflict]

```html
<!-- 适用场景：独立的分析脚本、广告脚本等 -->
<script async src="analytics.js"></script>
<script async src="ad-script.js"></script>
```

### defer（延迟）

- 脚本在后台**并行下载**，不阻塞 HTML 解析
- 下载完成后**不立即执行**，而是等待 HTML 解析完成
- 在 `DOMContentLoaded` 事件触发**之前**，按脚本在文档中出现的**顺序依次执行**
- 多个 defer 脚本**保证执行顺序**
- 对 `<script type="module">` 无效，因为 module 脚本**默认就是 defer 行为**[^mdn-module-defer]

```html
<!-- 适用场景：依赖 DOM 的应用脚本、有依赖关系的脚本链 -->
<script defer src="vendor/jquery.js"></script>
<script defer src="app.js"></script>
<script defer src="init.js"></script>
<!-- 保证按顺序执行：jquery.js → app.js → init.js -->
```

## 可视化对比

根据 HTML Living Standard 规范中的执行时序图[^html-spec]：

```
默认脚本：
HTML 解析: |=====|====(阻塞)====|=====|
脚本下载:         |---下载---|
脚本执行:                   |执行|

async 脚本：
HTML 解析: |========================|
脚本下载:  |---下载---|
脚本执行:              |执行|(短暂阻塞)|

defer 脚本：
HTML 解析: |========================|
脚本下载:  |---下载---|
脚本执行:                            |执行|
```

## 限制与注意事项

### 仅对外部脚本生效

`async` 和 `defer` 都要求脚本带有 `src` 属性。对内联脚本使用这两个属性**不会产生任何效果**[^mdn-warning-inline]：

```html
<!-- ❌ defer 无效 -->
<script defer>
  console.log("This runs immediately, defer is ignored");
</script>

<!-- ✅ 正确用法 -->
<script defer src="app.js"></script>
```

### async 与 defer 同时存在

如果同时指定两个属性，`async` 优先级更高，脚本将按 `async` 行为执行[^mdn-async-defer-conflict]：

```html
<!-- 等同于 async，defer 被忽略 -->
<script async defer src="script.js"></script>
```

### module 脚本默认 defer

`<script type="module">` 自动具有 defer 行为，无需显式添加 `defer` 属性[^mdn-module]：

```html
<!-- 这两个等价 -->
<script type="module" src="app.js"></script>
<script defer src="app.js"></script>
```

### blocking="render" 属性

HTML 新增了 `blocking` 属性，可以与 `async` 配合使用，实现"不阻塞解析但阻塞渲染"的效果[^mdn-blocking]：

```html
<!-- 脚本异步下载和执行，但渲染被阻塞直到脚本完成 -->
<script blocking="render" async src="critical-async.js"></script>
```

## 常见误区

### 误区 1：async 脚本不阻塞任何东西

**错误**。async 脚本下载时不阻塞解析，但**执行时会短暂阻塞解析**。只是阻塞时间通常很短（因为下载已在后台完成）。

### 误区 2：defer 脚本在 window.onload 之后执行

**错误**。defer 脚本在 HTML 解析完成后、`DOMContentLoaded` 事件触发**之前**执行。`window.onload` 要等到所有资源（图片、样式等）加载完毕才触发，比 defer 脚本晚得多。

### 误区 3：多个 async 脚本按书写顺序执行

**错误**。async 脚本的执行顺序完全取决于下载完成时间，与书写顺序无关。如果脚本之间有依赖关系，使用 async 会导致不可预测的错误。

## 最佳实践

| 场景 | 推荐方案 |
|------|---------|
| 独立的第三方脚本（分析、广告） | `async` |
| 应用主逻辑脚本（依赖 DOM） | `defer` |
| 有依赖关系的脚本链 | `defer`（按依赖顺序排列） |
| ES Module 脚本 | `type="module"`（自带 defer 行为） |
| 必须在解析期间执行的脚本（如 `document.write`） | 默认行为（无属性） |
| 需要阻塞渲染的异步脚本 | `async` + `blocking="render"` |

## 与 DOMContentLoaded 的关系

执行顺序保证：

1. HTML 解析完成
2. **所有 defer 脚本按顺序执行**
3. `DOMContentLoaded` 事件触发
4. 其他资源（图片等）继续加载
5. `window.onload` 事件触发

`async` 脚本**不参与**这个顺序保证，它们可能在任何时间点执行。

## 参考资料

- [MDN - `<script>`: The Script element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script)
- [HTML Living Standard - The script element](https://html.spec.whatwg.org/multipage/scripting.html#the-script-element)

[^mdn-async-defer-conflict]: 来源：MDN - "If the attribute is specified with the `defer` attribute, the element will act as if only the `async` attribute is specified."
[^mdn-module-defer]: 来源：MDN - "The `defer` attribute has no effect on module scripts — they defer by default."
[^mdn-warning-inline]: 来源：MDN - "This attribute must not be used if the `src` attribute is absent (i.e., for inline scripts), in this case it would have no effect."
[^mdn-blocking]: 来源：MDN - `blocking` attribute 文档
[^html-spec]: 来源：[HTML Spec async/defer diagram](https://html.spec.whatwg.org/images/asyncdefer.svg)
