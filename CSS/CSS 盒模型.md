---
tags:
  - CSS
  - 盒模型
  - 前端基础
  - 布局
created: '2026-04-15'
updated: '2026-04-15'
aliases:
  - CSS Box Model
  - Box Model
  - box-sizing
  - content-box
  - border-box
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Box_model/Introduction
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/box-sizing
  - https://drafts.csswg.org/css-sizing-3/
status: verified
---

## 定义

CSS 盒模型（Box Model）是浏览器渲染引擎用来描述每个元素所占空间的标准模型。根据 CSS 基本盒模型，文档中的每个元素都被表示为一个矩形盒子，CSS 决定了这些盒子的大小、位置和属性（颜色、背景、边框等）。

## 盒子的四个区域

每个盒子由四个部分组成，从内到外分别是：

### 1. 内容区（Content Area）

- 由**内容边缘**（content edge）包围
- 包含元素的"真实"内容，如文本、图像、视频等
- 尺寸由 `width`、`height`、`min-width`、`max-width`、`min-height`、`max-height` 属性定义
- 通常具有背景颜色或背景图像

### 2. 内边距区（Padding Area）

- 由**内边距边缘**（padding edge）包围
- 扩展内容区，包含元素的内边距
- 厚度由 `padding-top`、`padding-right`、`padding-bottom`、`padding-left` 或简写 `padding` 属性决定

### 3. 边框区（Border Area）

- 由**边框边缘**（border edge）包围
- 扩展内边距区，包含元素的边框
- 厚度由 `border-width` 或简写 `border` 属性决定
- 背景（`background-color` 或 `background-image`）会延伸到边框外缘

### 4. 外边距区（Margin Area）

- 由**外边距边缘**（margin edge）包围
- 扩展边框区，包含用于分隔元素的空白区域
- 大小由 `margin-top`、`margin-right`、`margin-bottom`、`margin-left` 或简写 `margin` 属性决定
- 当发生**外边距折叠**（margin collapsing）时，外边距区不是明确定义的，因为边距会在盒子之间共享

## 两种盒模型计算方式

通过 `box-sizing` 属性可以控制元素的总宽度和高度的计算方式。

### content-box（默认值）

- `width` 和 `height` 仅包含内容，**不包含**内边距、边框或外边距
- 最终渲染尺寸 = 内容尺寸 + 内边距 + 边框

```css
.box {
  box-sizing: content-box;
  width: 350px;
  border: 10px solid black;
}
/* 渲染宽度：350px + 20px（左右边框）= 370px */
```

**计算公式：**
- 总宽度 = `width` + `padding-left` + `padding-right` + `border-left-width` + `border-right-width`
- 总高度 = `height` + `padding-top` + `padding-bottom` + `border-top-width` + `border-bottom-width`

### border-box

- `width` 和 `height` 包含内容、内边距和边框，**不包含**外边距
- 内边距和边框会在盒子内部，内容区会相应缩小

```css
.box {
  box-sizing: border-box;
  width: 350px;
  border: 10px solid black;
}
/* 渲染宽度：350px（内容区自动缩小为 330px）*/
```

**计算公式：**
- 内容宽度 = `width` - `padding-left` - `padding-right` - `border-left-width` - `border-right-width`
- 内容高度 = `height` - `padding-top` - `padding-bottom` - `border-top-width` - `border-bottom-width`

**注意：** 内容框不能为负值，最小为 0，因此无法使用 `border-box` 使元素消失。

### 对比表格

| 属性 | `content-box` | `border-box` |
|------|---------------|--------------|
| `width` 包含 | 仅内容 | 内容 + 内边距 + 边框 |
| `height` 包含 | 仅内容 | 内容 + 内边距 + 边框 |
| 外边距 | 不包含 | 不包含 |
| 默认值 | ✅ 是 | ❌ 否 |
| 布局友好度 | 较低（需手动计算） | 较高（直观） |

## 为什么重要

### 默认行为的陷阱

在默认 `content-box` 模式下，设置 `width` 和 `height` 后，还需要额外计算内边距和边框的尺寸。例如：

```css
/* 问题：四个盒子无法在一行内排列 */
.box {
  width: 25%;
  padding: 10px;
  border: 1px solid black;
}
```

每个盒子的实际宽度 = `25% + 20px（左右内边距）+ 2px（左右边框）`，总和超过 100%，导致换行。

### border-box 的优势

使用 `border-box` 可以显著简化布局计算：

```css
/* 解决方案：四个盒子可以完美排列 */
.box {
  box-sizing: border-box;
  width: 25%;
  padding: 10px;
  border: 1px solid black;
}
```

每个盒子的总宽度就是 `25%`，内边距和边框包含在内。

## 核心用法

### 全局设置 border-box

现代项目推荐在全局样式表中统一设置：

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

这确保所有元素（包括伪元素）都使用 `border-box` 模型。

### 浏览器默认行为

以下元素默认使用 `border-box`：

- `<table>`
- `<select>`
- `<button>`
- `<input>` 类型为：`radio`、`checkbox`、`reset`、`button`、`submit`、`color`、`search`

### 何时使用 content-box

在某些场景下，`content-box` 仍然有用：

- 使用 `position: relative` 或 `position: absolute` 时，定位值相对于内容区会更直观
- 需要精确控制内容区尺寸，不受边框和内边距变化影响时

## 常见误区

### 误区 1：border-box 包含外边距

**错误理解：** `border-box` 的 `width` 包含外边距。

**正确理解：** `border-box` 的 `width` 仅包含内容、内边距和边框，**不包含外边距**。外边距始终在盒子外部。

### 误区 2：border-box 使元素消失

**错误理解：** 如果 `padding + border > width`，元素会消失。

**正确理解：** 内容框不能为负值，最小为 0。当 `padding + border >= width` 时，内容区会被压缩到 0，但元素仍然可见（边框和内边距仍会渲染）。

### 误区 3：所有元素默认都是 content-box

**错误理解：** 所有元素默认使用 `content-box`。

**正确理解：** 表单控件（如 `<button>`、`<input>`、`<select>`、`<table>`）默认使用 `border-box`。

### 误区 4：box-sizing 会继承

**错误理解：** 父元素设置 `box-sizing` 会影响子元素。

**正确理解：** `box-sizing` **不可继承**，需要显式设置或使用通配符选择器。

## 相关概念

### 外边距折叠（Margin Collapsing）

当两个垂直外边距相遇时，它们会折叠成一个单一的外边距，大小为两者中的较大值。

```css
.parent {
  margin-bottom: 30px;
}

.child {
  margin-top: 20px;
}
/* 实际间距：30px（取较大值），而非 50px */
```

### 替换元素（Replaced Elements）

某些元素（如 `<img>`、`<video>`、`<iframe>`）的内容不受 CSS 直接控制，其尺寸由外部资源决定。这些元素的盒模型行为可能有所不同。

### 行内非替换元素

对于非替换行内元素（如 `<span>`），其占用空间（对行高的贡献）由 `line-height` 属性决定，即使边框和内边距仍然会围绕内容显示。

## 参考资料

- [MDN: Introduction to the CSS box model](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Box_model/Introduction)
- [MDN: box-sizing](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/box-sizing)
- [CSS Box Sizing Module Level 3](https://drafts.csswg.org/css-sizing-3/)
- [MDN: Margin collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Box_model/Margin_collapsing)
