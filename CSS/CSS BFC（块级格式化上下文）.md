---
created: 2026-04-15
updated: 2026-04-15
tags:
  - CSS
  - 布局机制
  - 前端基础
  - 渲染机制
aliases:
  - BFC
  - Block Formatting Context
  - 块级格式化上下文
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Display/Block_formatting_context
  - https://drafts.csswg.org/css-display/#block-formatting-context
status: verified
---

## 是什么

**块级格式化上下文（Block Formatting Context，简称 BFC）** 是 CSS 视觉渲染模型中的一个独立布局区域。它是块级盒子（block boxes）进行布局的区域，也是浮动元素与其他元素交互的边界。

BFC 是一个**自包含的布局单元**：

- BFC 内部的子元素按照块级格式化规则进行布局。
- BFC 外部的浮动元素不会影响 BFC 内部的布局。
- BFC 内部的浮动元素不会溢出到 BFC 外部。
- BFC 可以嵌套，形成布局树的子结构。

## 为什么重要

理解 BFC 是解决许多 CSS 布局问题的关键。很多开发者遇到的"父元素高度塌陷"、"margin 重叠"、"浮动元素遮挡内容"等问题，根本原因都是对 BFC 的触发条件和布局规则不够清楚。

> 创建一个新的 BFC 可以隔离内部布局与外部环境，这是 CSS 中实现布局隔离的核心机制之一。

## 前置概念：格式化上下文

浏览器渲染页面时，会为每个元素分配一个**格式化上下文（Formatting Context）**，它决定了内部子元素的布局规则：

| 格式化上下文 | 说明 |
|-------------|------|
| **BFC（块级格式化上下文）** | 块级元素按照垂直方向排列，浮动元素参与布局 |
| **IFC（行内格式化上下文）** | 行内元素按照水平方向排列，形成行框（line boxes） |
| **Flex 格式化上下文** | Flex 容器内的子元素按照弹性布局规则排列 |
| **Grid 格式化上下文** | Grid 容器内的子元素按照网格布局规则排列 |
| **Table 格式化上下文** | 表格元素按照表格布局规则排列 |

BFC 是最基础的格式化上下文，普通文档流中的块级元素默认就在 BFC 中布局。

## 创建 BFC 的条件

以下任一情况会使元素**创建新的 BFC**：

| 触发条件 | 示例 |
|----------|------|
| 文档根元素 | `<html>` 始终创建根 BFC |
| 浮动元素 | `float: left;` 或 `float: right;` |
| 绝对定位元素 | `position: absolute;` 或 `position: fixed;` |
| 行内块元素 | `display: inline-block;` |
| 表格单元格 | `display: table-cell;`（HTML `<td>`、`<th>` 默认值） |
| 表格标题 | `display: table-caption;`（HTML `<caption>` 默认值） |
| 匿名表格单元格 | 由 `display: table`、`table-row`、`table-row-group` 等隐式创建 |
| `display: flow-root` | `display: flow-root;`（专门用于创建 BFC 的现代值） |
| Flex 子元素 | `display: flex` 或 `inline-flex` 的直接子元素（自身不是 flex/grid/table 容器时） |
| Grid 子元素 | `display: grid` 或 `inline-grid` 的直接子元素（自身不是 flex/grid/table 容器时） |
| `overflow` 不为 `visible` | `overflow: hidden;`、`overflow: auto;`、`overflow: scroll;` |
| `contain` 包含 `layout`、`content` 或 `paint` | `contain: strict;` / `contain: content;` |
| 多列容器 | `column-count` 或 `column-width` 不为 `auto`（包括 `column-count: 1`） |
| `column-span: all` | 即使不在多列容器内也会创建 BFC |

> **注意**：`display: flow-root` 是 CSS 规范中专门用于创建 BFC 而**不产生任何副作用**的值。相比 `overflow: hidden` 等历史方案，它语义更清晰，不会裁剪内容或产生滚动条。

## BFC 的布局规则

在同一个 BFC 内，块级盒子按以下规则布局：

1. **垂直排列**：块级盒子从包含块的顶部开始，垂直向下依次排列。
2. **间距由 margin 决定**：相邻块级盒子之间的垂直距离由 `margin` 决定（可能发生 margin 折叠）。
3. **左对齐**：块级盒子的左外边距边缘与包含块的左边缘接触（对于从左到右的格式化，右边缘同理），即使存在浮动也是如此。
4. **不与浮动重叠**：BFC 的区域不会与同一 BFC 内的浮动元素重叠（BFC 会"避开"浮动元素）。
5. **独立隔离**：BFC 是一个隔离的容器，内部元素不会影响外部布局，外部浮动也不会影响内部布局。

## BFC 的三大核心特性

### 1. 包含内部浮动（Clearfix）

当子元素浮动时，父元素会失去高度（高度塌陷），因为浮动元素脱离了正常文档流。

**问题示例**：

```html
<div class="container">
  <div class="float">浮动内容</div>
  <p>普通内容</p>
</div>
```

```css
.container {
  border: 2px solid red;
}
.float {
  float: left;
  height: 100px;
}
```

此时 `.container` 的高度只包含 `<p>` 的高度，浮动元素会溢出到底部之外。

**解决方案**：

```css
/* 方案 1：使用 display: flow-root（推荐） */
.container {
  display: flow-root;
}

/* 方案 2：使用 overflow（历史方案，可能有副作用） */
.container {
  overflow: auto;
}

/* 方案 3：使用 clearfix 伪元素（传统方案） */
.container::after {
  content: "";
  display: table;
  clear: both;
}
```

### 2. 排除外部浮动（自适应两栏布局）

BFC 区域不会与浮动元素重叠，可以利用这个特性实现自适应布局。

```html
<div class="sidebar">侧边栏（浮动）</div>
<div class="main">主内容区（BFC）</div>
```

```css
.sidebar {
  float: left;
  width: 200px;
}

.main {
  display: flow-root; /* 或 overflow: hidden */
  /* 主内容区会自动避开浮动侧边栏，占据剩余空间 */
}
```

### 3. 阻止 Margin 折叠（Margin Collapsing）

在普通文档流中，相邻块级元素的垂直 margin 会发生折叠（取较大值而非相加）。创建 BFC 可以阻止这种折叠。

**Margin 折叠示例**：

```html
<div class="blue"></div>
<div class="red"></div>
```

```css
.blue, .red {
  height: 50px;
  margin: 10px 0;
}
/* 实际间距为 10px（折叠），而非 20px */
```

**阻止折叠**：

```html
<div class="blue"></div>
<div class="wrapper">
  <div class="red"></div>
</div>
```

```css
.wrapper {
  overflow: hidden; /* 创建 BFC，阻止内部 margin 与外部折叠 */
}
```

## 常见误区

### 误区 1：`overflow: hidden` 是创建 BFC 的最佳方案

**错误**：`overflow` 的语义是处理溢出内容，用它来创建 BFC 是历史 hack 手段。可能导致内容被裁剪、阴影被截断、或出现意外滚动条。

**正确做法**：优先使用 `display: flow-root`，它是专门为此设计的值，无任何副作用。

### 误区 2：只有 `overflow` 和 `float` 才能创建 BFC

**错误**：`position: absolute/fixed`、`display: inline-block`、`display: flow-root`、Flex/Grid 子元素等都可以创建 BFC。现代 CSS 中 `display: flow-root` 是最语义化的选择。

### 误区 3：BFC 和层叠上下文是一回事

**错误**：BFC 控制的是**块级布局规则**（margin 折叠、浮动清除、布局隔离），而层叠上下文（Stacking Context）控制的是 **Z 轴渲染顺序**。两者是完全不同的概念，尽管某些属性（如 `overflow: hidden`）会同时触发两者。

### 误区 4：Flex/Grid 容器就是 BFC

**部分错误**：Flex/Grid 容器创建的是**独立的格式化上下文**（Flex Formatting Context / Grid Formatting Context），它们与 BFC 类似但不完全相同。Flex/Grid 容器内部没有浮动概念，但同样会排除外部浮动和阻止 margin 折叠。

## 实用技巧

### 推荐：使用 `display: flow-root` 清除浮动

```css
.clearfix {
  display: flow-root;
}
```

这是现代浏览器（Chrome 58+、Firefox 53+、Safari 11.1+、Edge 79+）推荐的方式，语义清晰且无副作用。

### 历史方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| `display: flow-root` | 语义清晰、无副作用 | 不支持 IE |
| `overflow: hidden` | 兼容性好 | 可能裁剪内容、截断阴影 |
| `overflow: auto` | 兼容性好 | 可能产生意外滚动条 |
| `float: left` + 父级清除 | 兼容性好 | 破坏正常文档流 |
| `::after { clear: both }` | 兼容性好、无视觉副作用 | 代码冗长 |

### 调试 BFC

浏览器开发者工具中，可以通过以下方式验证 BFC：

- 检查元素是否包含浮动子元素（父元素高度是否正常包裹浮动）。
- 检查相邻元素的 margin 是否发生折叠。
- 检查元素是否避开了同级浮动元素。

## 与相关概念的关系

| 概念 | 区别 |
|------|------|
| **层叠上下文（Stacking Context）** | 控制 Z 轴渲染顺序（`z-index`），与 BFC 的布局规则无关 |
| **IFC（行内格式化上下文）** | 控制行内元素的水平排列和行框生成，与 BFC 的块级布局互补 |
| **包含块（Containing Block）** | 决定定位元素的参考坐标系，与格式化上下文是不同维度的概念 |
| **Margin 折叠（Margin Collapsing）** | BFC 的一个副作用被阻止的现象，发生在普通文档流的相邻块级元素之间 |
| **Flex/Grid 格式化上下文** | 现代布局模式创建的独立上下文，行为类似 BFC 但内部无浮动概念 |

## 参考资料

- [MDN - Block formatting context](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Display/Block_formatting_context)
- [CSS Display Module Level 3 - Block Formatting Context](https://drafts.csswg.org/css-display/#block-formatting-context)
- [MDN - Margin collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Box_model/Margin_collapsing)
- [MDN - In flow and out of flow](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Display/In_flow_and_out_of_flow)
