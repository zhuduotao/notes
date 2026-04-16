---
created: 2026-04-15
updated: 2026-04-15
tags:
  - CSS
  - 渲染机制
  - 前端基础
  - z-index
aliases:
  - Stacking Context
  - 层叠上下文
  - z-index 层叠规则
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Stacking_context
  - https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Understanding_z-index
status: verified
---
## 是什么

**层叠上下文（Stacking Context）** 是 HTML 元素沿 Z 轴（垂直于屏幕的虚拟轴）进行三维渲染排序的概念模型。它决定了当元素发生视觉重叠时，哪个元素渲染在上方、哪个在下方。

层叠上下文是一个**自包含的渲染单元**：

- 上下文内部的子元素独立参与层叠排序，不受外部元素干扰。
- 上下文整体在其父级层叠上下文中被视为一个原子单元，内部子元素的 `z-index` 不会跨越上下文边界与外部元素直接比较。
- 层叠上下文可以嵌套，形成一棵层叠树，它是 DOM 树的子集（只有特定元素才会创建层叠上下文）。

## 为什么重要

理解层叠上下文是正确使用 `z-index` 的前提。很多开发者遇到的"z-index 不生效"问题，根本原因是忽略了层叠上下文的隔离性：

> 子元素的 `z-index` 只在它所属的层叠上下文内有效。父级上下文的整体层级低于其他兄弟上下文时，无论子元素的 `z-index` 设得多大，都无法超越父级的渲染顺序。

## 前置概念：Z 轴与层叠层

浏览器渲染页面时，每个元素盒子除了 X/Y 平面位置，还位于一个虚拟的 Z 轴上。`z-index` 的值（整数，可正可负或零）决定了元素在 Z 轴上的层级：

| 层级 | 说明 |
|------|------|
| Top layer | 最靠近观察者（如全屏元素、popover） |
| 正 `z-index` | 值越大越靠前 |
| Layer 0 | 默认渲染层（未设置 `z-index` 的元素） |
| 负 `z-index` | 值越小越靠后 |
| Bottom layer | 最远离观察者 |

## 创建层叠上下文的条件

以下任一情况会使元素**创建新的层叠上下文**：

| 触发条件 | 示例 |
|----------|------|
| 文档根元素 | `<html>` 始终创建根层叠上下文 |
| `position` 为 `absolute` 或 `relative`，且 `z-index` 不为 `auto` | `position: relative; z-index: 1;` |
| `position` 为 `fixed` 或 `sticky` | `position: fixed;`（无论 `z-index` 值） |
| `container-type` 为 `size` 或 `inline-size` | 容器查询相关 |
| Flex 子元素，且 `z-index` 不为 `auto` | `display: flex;` 下的直接子元素 |
| Grid 子元素，且 `z-index` 不为 `auto` | `display: grid;` 下的直接子元素 |
| `opacity` 小于 `1` | `opacity: 0.9;` |
| `mix-blend-mode` 不为 `normal` | `mix-blend-mode: multiply;` |
| `transform` 不为 `none` | `transform: rotate(10deg);` |
| `scale` 不为 `none` | `scale: 1.1;` |
| `rotate` 不为 `none` | `rotate: 15deg;` |
| `translate` 不为 `none` | `translate: 10px 20px;` |
| `filter` 不为 `none` | `filter: blur(5px);` |
| `backdrop-filter` 不为 `none` | `backdrop-filter: blur(10px);` |
| `perspective` 不为 `none` | `perspective: 500px;` |
| `clip-path` 不为 `none` | `clip-path: circle(50%);` |
| `mask` / `mask-image` / `mask-border` 不为 `none` | `mask-image: url(...);` |
| `isolation` 为 `isolate` | `isolation: isolate;`（专门用于强制创建层叠上下文） |
| `will-change` 指定了会创建层叠上下文的属性（非初始值） | `will-change: transform;` |
| `contain` 包含 `layout` 或 `paint` | `contain: strict;` / `contain: content;` |
| 位于 Top layer 的元素及其 `::backdrop` | 全屏 API、Popover API 元素 |
| 在 `@keyframes` 中动画了创建层叠上下文的属性，且 `animation-fill-mode` 为 `forwards` | 动画结束后仍保持层叠上下文 |

> **注意**：以上属性只要满足条件就会创建层叠上下文，**不需要**同时设置 `position` 或 `z-index`。例如 `opacity: 0.9` 单独使用就会创建新的层叠上下文。

## 层叠排序规则

在同一个层叠上下文内，元素按以下顺序从后到前渲染（CSS 2.1 规范定义）：

1. 层叠上下文的背景和边框
2. `z-index` 为负值的子层叠上下文
3. `z-index` 为负值的非定位子元素
4. 非定位的普通子元素（按 DOM 顺序）
5. `z-index` 为 `auto` 的定位子元素（按 DOM 顺序）
6. `z-index` 为 0 或正值的子层叠上下文（按 `z-index` 值排序，值相同按 DOM 顺序）
7. `z-index` 为正值的定位子元素（按 `z-index` 值排序）

## 嵌套层叠上下文示例

```html
<article id="a1" style="position: relative; z-index: 5; opacity: 0.9;">
  <h1>Article #1</h1>
</article>

<article id="a2" style="position: relative; z-index: 2; opacity: 0.9;">
  <h1>Article #2</h1>
</article>

<article id="a3" style="position: absolute; z-index: 4; opacity: 0.9;">
  <section id="s1" style="position: relative; z-index: 6;">Section #6</section>
  <section id="s2" style="position: relative; z-index: 1;">Section #1</section>
</article>
```

渲染顺序分析（用"父层级-子层级"版本号类比）：

| 元素 | 所属上下文 | 渲染顺序 |
|------|-----------|---------|
| Article #2 | Root | `2-0` |
| Article #3 | Root | `4-0` |
| → Section #2 | Article #3 (z:4) | `4-1` |
| → Section #1 | Article #3 (z:4) | `4-6` |
| Article #1 | Root | `5-0` |

关键点：

- Section #1 的 `z-index: 6` **只在 Article #3 的上下文内有效**，不会与 Article #1 的 `z-index: 5` 直接比较。
- 因为 Article #3 (`z:4`) 整体低于 Article #1 (`z:5`)，所以 Article #3 内部的所有元素都渲染在 Article #1 下方。
- 同理，Article #2 (`z:2`) 整体低于 Section #2 (`z:1`，但其父 Article #3 为 `z:4`)，所以 Section #2 渲染在 Article #2 上方。

## 常见误区

### 误区 1：z-index 越大越靠前

**错误**：`z-index` 的比较只在**同一个层叠上下文内**有效。不同上下文的 `z-index` 不能直接跨级比较。

### 误区 2：只有 position + z-index 才会创建层叠上下文

**错误**：`opacity`、`transform`、`filter`、`will-change` 等现代 CSS 属性也会隐式创建层叠上下文，这是很多"z-index 不生效"的隐藏原因。

### 误区 3：z-index 可以是小数

**错误**：`z-index` 的值必须是**整数**。浏览器会忽略小数部分（如 `z-index: 1.5` 等同于 `z-index: 1`）。

### 误区 4：设置了 z-index 就一定会生效

**错误**：`z-index` 只对**定位元素**（`position` 为 `relative`、`absolute`、`fixed`、`sticky`）或 Flex/Grid 子元素生效。普通块级元素设置 `z-index` 无效。

## 实用技巧

### 强制创建层叠上下文而不改变布局

使用 `isolation: isolate` 可以在不改变元素位置、透明度、变换等视觉效果的前提下，强制创建一个新的层叠上下文。这在需要隔离子元素 `z-index` 时非常有用：

```css
.isolated {
  isolation: isolate;
}
```

### 避免 z-index 地狱

- **不要使用过大的 z-index 值**（如 `9999`、`99999`），应使用合理的小范围值（如 `1`~`10`）。
- **优先通过 DOM 顺序控制层叠**，而非依赖 `z-index`。
- **使用 CSS 自定义属性管理 z-index 层级**，便于统一维护：
  ```css
  :root {
    --z-dropdown: 10;
    --z-modal: 20;
    --z-tooltip: 30;
  }
  ```

### 调试层叠上下文

浏览器开发者工具中，部分浏览器（如 Firefox）支持可视化层叠上下文。Chrome 可通过 DevTools 的 Layers 面板查看渲染层分布。

## 与相关概念的关系

| 概念 | 区别 |
|------|------|
| **BFC（块级格式化上下文）** | 控制块级元素的布局规则（如 margin 折叠、浮动清除），与 Z 轴层叠无关 |
| **CSS 优先级（Specificity）** | 决定哪个 CSS 规则生效，与渲染顺序无关 |
| **层叠顺序（Stacking Order）** | 层叠上下文内的具体排序规则，是层叠上下文的子概念 |
| **Top layer** | 浏览器提供的特殊层叠层（如 `<dialog>`、popover），位于所有层叠上下文之上 |

## 参考资料

- [MDN - Stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Stacking_context)
- [MDN - Understanding z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Understanding_z-index)
- [MDN - Using z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Using_z-index)
- [MDN - Stacking without z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Positioned_layout/Stacking_without_z-index)
