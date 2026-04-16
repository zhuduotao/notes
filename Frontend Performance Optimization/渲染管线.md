---
created: 2026-04-15
updated: 2026-04-15
tags:
  - 浏览器原理
  - 渲染机制
  - 性能优化
  - Blink
  - 前端基础
aliases:
  - Rendering Pipeline
  - Critical Rendering Path
  - 渲染管线
  - 渲染流水线
  - 浏览器渲染流程
source_type: official-doc
source_urls:
  - https://developers.google.com/web/updates/2018/09/inside-browser-part3
  - https://developers.google.com/web/fundamentals/performance/critical-rendering-path/
  - https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work
  - https://html.spec.whatwg.org/multipage/rendering.html
status: verified
---

## 是什么

渲染管线（Rendering Pipeline）是浏览器渲染引擎（如 Blink、WebKit、Gecko）将 HTML、CSS 和 JavaScript 转换为屏幕上可见像素的**分阶段处理流程**。

渲染管线包含五个核心阶段：**解析（Parsing）→ 样式计算（Style Calculation）→ 布局（Layout）→ 绘制（Paint）→ 合成（Compositing）**。每个阶段使用前一步的输出作为输入，逐步构建页面的视觉表示。

> **注意**：渲染管线是渲染引擎的内部实现机制，不是 Web 标准的一部分。不同浏览器的具体实现可能不同，但整体流程相似。本文基于 Blink 渲染引擎（Chrome、Edge、Opera 等使用）进行描述。

---

## 为什么重要

理解渲染管线是前端性能优化的基础：

- **识别性能瓶颈**：知道每个阶段的代价，才能定位卡顿、掉帧的根因
- **避免不必要的计算**：某些 DOM 操作会触发整个管线重新执行，了解触发条件可避免性能浪费
- **选择正确的优化策略**：不同场景需要不同的优化手段（如使用 `transform` 替代 `top/left` 避免触发布局）
- **理解 Core Web Vitals**：LCP（最大内容绘制）、INP（交互到下一次绘制）、CLS（累积布局偏移）等指标与渲染管线直接相关

---

## 渲染管线五阶段

### 阶段 1：解析（Parsing）

将 HTML 和 CSS 文本转换为浏览器可理解的数据结构。

#### 构建 DOM（Document Object Model）

主线程将 HTML 文本解析为树状结构的 DOM 节点：

```html
<!-- HTML 源码 -->
<html>
  <head><title>Page</title></head>
  <body>
    <p>Hello <span>World</span></p>
  </body>
</html>
```

解析结果：

```
Document
└── html
    ├── head
    │   └── title
    │       └── "Page"
    └── body
        └── p
            ├── "Hello "
            └── span
                └── "World"
```

**关键特性**：

- HTML 解析器具有**容错机制**，能处理缺失闭合标签、标签交叉嵌套等不规范 HTML（详见 [HTML Standard - Parsing](https://html.spec.whatwg.org/multipage/parsing.html)）
- 解析是**增量进行**的，不需要等待整个 HTML 文档下载完成

#### 构建 CSSOM（CSS Object Model）

CSS 解析器将 CSS 规则转换为 CSSOM 树：

```css
p { font-size: 16px; color: #333; }
span { font-weight: bold; }
```

CSSOM 树结构与 DOM 类似，但存储的是样式规则而非文档结构。

#### JavaScript 会阻塞解析

当 HTML 解析器遇到 `<script>` 标签时，会**暂停 HTML 解析**，必须先加载、解析并执行 JavaScript：

- JavaScript 可以通过 `document.write()` 等方式修改 DOM 结构
- 因此解析器必须等待脚本执行完毕才能继续

**解决方案**：

- 使用 `async` 属性：异步加载，加载完成后立即执行（不保证执行顺序）
- 使用 `defer` 属性：异步加载，在 DOM 解析完成后按顺序执行
- 使用 `<script type="module">`：默认具有 `defer` 行为

#### 预加载扫描器（Preload Scanner）

预加载扫描器与 HTML 解析器**并发运行**，提前发现并请求外部资源：

- 查看 HTML 解析器生成的 token
- 发现 `<img>`、`<link>`、`<script>` 等标签时，提前向网络层发送请求
- 显著提升页面加载速度，因为网络请求不需要等待主线程解析到对应标签

---

### 阶段 2：样式计算（Style Calculation）

将 CSSOM 规则应用到 DOM 节点，计算每个节点的**计算样式（Computed Style）**。

**计算样式包含**：

- 从作者样式表、用户样式表、浏览器默认样式表继承和层叠后的最终样式值
- 相对单位（如 `em`、`rem`、`%`）转换为绝对值（如 `px`）
- 简写属性（如 `margin`）展开为长属性（`margin-top`、`margin-right` 等）

**示例**：

```css
/* 作者样式 */
body { font-size: 16px; }
p { color: blue; }
```

`<p>` 元素的计算样式：

```
font-size: 16px;    /* 从 body 继承 */
color: blue;        /* 自身定义 */
margin: 1em 0;      /* 浏览器默认样式 */
display: block;     /* 浏览器默认样式 */
```

**性能提示**：复杂的 CSS 选择器（如 `.a .b .c .d .e`）会增加样式计算时间，因为浏览器需要从右到左匹配每个选择器。

---

### 阶段 3：布局（Layout）

计算每个可见 DOM 节点的**几何信息**（位置、尺寸），生成**布局树（Layout Tree）**。

**布局树与 DOM 树的区别**：

| 特性 | DOM 树 | 布局树 |
|------|--------|--------|
| 包含 `display: none` 的元素 | ✅ | ❌ |
| 包含 `visibility: hidden` 的元素 | ✅ | ✅（占据空间） |
| 包含伪元素（如 `::before`） | ❌ | ✅ |
| 包含文本节点 | ✅ | ✅（作为匿名盒子） |

**布局过程**：

1. 主线程从布局树的根节点开始遍历
2. 对每个节点计算其在视口中的 x/y 坐标和宽高
3. 考虑字体大小、换行、浮动、溢出、书写方向等因素

**布局是一个递归过程**：父节点的尺寸可能影响子节点，子节点的尺寸也可能影响父节点（如 `height: auto` 时）。

---

### 阶段 4：绘制（Paint）

确定每个元素的**绘制顺序**，生成**绘制记录（Paint Records）**。

**为什么需要绘制阶段**：

- HTML 中元素的顺序不一定等于视觉上的绘制顺序
- `z-index`、层叠上下文、透明度等会影响绘制顺序
- 绘制记录是"绘制过程的笔记"，如"先画背景，再画文字，再画矩形"

**绘制记录示例**：

```
PaintRecord {
  type: "fillRect", color: "red", x: 0, y: 0, w: 100, h: 100
  type: "fillText", text: "Hello", x: 10, y: 50
  type: "drawImage", src: "logo.png", x: 200, y: 0
}
```

绘制记录是**声明式**的，不是实际的像素操作。真正的像素绘制在后续的光栅化阶段完成。

---

### 阶段 5：合成（Compositing）

将页面分成多个**图层（Layers）**，分别光栅化后组合成最终画面。

#### 为什么需要合成

- **滚动优化**：图层已光栅化，滚动时只需移动图层位置，无需重新绘制
- **动画优化**：动画可以通过移动/变换图层实现，不依赖主线程
- **部分更新**：页面某部分变化时，只需重新光栅化对应的图层

#### 图层划分

浏览器根据以下规则自动将元素提升为独立图层：

- 使用 3D 变换或透视的元素（`transform: translateZ(0)`、`perspective`）
- 使用加速 CSS 属性的元素（`transform`、`opacity`、`filter`）
- 具有裁剪或混合的元素（`overflow: hidden`、`mix-blend-mode`）
- `<video>`、`<canvas>`、`<iframe>` 等元素
- 使用 `will-change` 提示的元素

**警告**：给过多元素分配图层会导致内存开销增加和光栅化时间变长，反而降低性能。

#### 光栅化与合成流程

1. 主线程将图层树提交给**合成线程（Compositor Thread）**
2. 合成线程将每个图层划分为多个**瓦片（Tiles）**
3. 瓦片发送给**光栅线程（Raster Threads）**，光栅化为位图并存储在 GPU 内存中
4. 合成线程收集瓦片信息（**Draw Quads**）创建**合成帧（Compositor Frame）**
5. 合成帧通过 IPC 提交给浏览器进程，最终发送给 GPU 显示

**关键优势**：合成线程**不依赖主线程**，即使主线程被 JavaScript 阻塞，合成线程仍可处理滚动和动画。

---

## 渲染管线触发条件

### 首次加载

页面首次加载时，渲染管线完整执行一次：

```
HTML/CSS 下载 → 解析 → 样式计算 → 布局 → 绘制 → 合成 → 显示
```

### 运行时更新

页面加载后，JavaScript 或用户交互可能修改 DOM/CSS，触发管线重新执行。根据修改内容的不同，触发范围不同：

| 修改类型 | 触发阶段 | 性能代价 |
|----------|----------|----------|
| 修改 `transform`、`opacity` | 合成 | 最低 |
| 修改 `color`、`background-color`、`visibility` | 绘制 → 合成 | 中等 |
| 修改 `width`、`height`、`margin`、`padding`、`font-size` | 布局 → 绘制 → 合成 | 最高 |
| 添加/删除 DOM 节点 | 样式计算 → 布局 → 绘制 → 合成 | 最高 |

---

## 重排 / 重绘 / 重合成

### 重排（Reflow / Layout Thrashing）

**定义**：布局树发生变化，需要重新计算元素几何信息。

**触发条件**：

- 修改元素尺寸、位置、边距等几何属性
- 修改字体大小
- 添加/删除 DOM 节点
- 调用 `offsetWidth`、`offsetHeight`、`getBoundingClientRect()` 等读取几何属性的方法（会强制同步布局）

**性能代价**：最高。布局是渲染管线中最耗时的阶段之一，尤其是复杂页面。

### 重绘（Repaint）

**定义**：元素外观发生变化但几何信息不变，需要重新生成绘制记录。

**触发条件**：

- 修改 `color`、`background-color`、`border-color`
- 修改 `visibility`
- 修改 `outline`、`box-shadow`（部分情况）

**性能代价**：中等。不需要重新布局，但需要重新绘制和合成。

### 重合成（Recomposite）

**定义**：图层发生变化，需要重新光栅化或组合图层。

**触发条件**：

- 修改 `transform`（平移、旋转、缩放）
- 修改 `opacity`
- 修改 `filter`

**性能代价**：最低。合成线程独立于主线程执行，不阻塞 JavaScript。

---

## 强制同步布局（Layout Thrashing）

**问题**：在 JavaScript 中交替读取和写入几何属性会导致浏览器强制同步布局。

```javascript
// 反模式：强制同步布局
element.style.width = '100px';        // 写入，标记布局失效
const width = element.offsetWidth;    // 读取，强制同步布局
element.style.height = '200px';       // 写入，再次标记布局失效
const height = element.offsetHeight;  // 读取，再次强制同步布局
```

**优化方式**：批量读取和写入

```javascript
// 正模式：批量读取
const currentWidth = element.offsetWidth;
const currentHeight = element.offsetHeight;

// 批量写入
element.style.width = '100px';
element.style.height = '200px';
```

或使用 `requestAnimationFrame()` 分离读写操作：

```javascript
// 读取阶段
const currentWidth = element.offsetWidth;

requestAnimationFrame(() => {
  // 写入阶段（下一帧）
  element.style.width = (currentWidth + 10) + 'px';
});
```

---

## 性能优化最佳实践

### 使用合成器属性做动画

优先使用 `transform` 和 `opacity` 实现动画，避免触发布局和绘制：

```css
/* 推荐：使用 transform */
.element {
  transition: transform 0.3s;
}
.element:hover {
  transform: translateX(10px);
}

/* 不推荐：使用 left/top */
.element {
  transition: left 0.3s;
  position: relative;
}
.element:hover {
  left: 10px;
}
```

### 使用 `will-change` 提示浏览器

对即将动画化的元素使用 `will-change` 提示浏览器提前创建图层：

```css
.animated {
  will-change: transform, opacity;
}
```

**注意**：不要对所有元素使用 `will-change`，会导致内存浪费。仅在动画前添加，动画结束后移除。

### 避免强制同步布局

- 批量读取几何属性，再批量写入
- 使用 `requestAnimationFrame()` 分离读写操作
- 使用 `Element.animate()` API（浏览器自动优化）

### 减少布局范围

- 使用 CSS `contain` 属性限制布局范围：

```css
.card {
  contain: layout style paint;
}
```

- 使用 `display: contents` 避免生成布局节点

### 测量工具

- Chrome DevTools **Performance 面板**：记录渲染管线各阶段耗时
- Chrome DevTools **Rendering 面板**：开启 "Paint flashing" 查看重绘区域
- Chrome DevTools **Layers 面板**：查看页面图层划分
- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview)：审计页面性能

---

## 常见误区

### 误区 1：`display: none` 和 `visibility: hidden` 性能一样

- `display: none`：元素从布局树中移除，触发重排
- `visibility: hidden`：元素仍在布局树中（占据空间），只触发重绘

### 误区 2：所有动画性能都一样

只有使用合成器属性（`transform`、`opacity`）的动画才能在合成线程上运行。触发布局或重绘的动画（如改变 `width`、`height`、`top`、`left`）会阻塞主线程。

### 误区 3：`will-change` 越多越好

每个独立图层都有内存和光栅化开销。过多图层会导致性能下降。应仅在需要时添加，用完后移除。

### 误区 4：读取几何属性没有代价

读取 `offsetWidth`、`getBoundingClientRect()` 等方法会**强制同步布局**，如果之前有 DOM 修改未应用，浏览器会立即执行布局以返回最新值，导致性能问题。

### 误区 5：渲染管线只在页面加载时执行

任何 DOM/CSS 修改都可能触发渲染管线重新执行。理解触发条件是运行时性能优化的关键。

---

## 相关概念

- **[Chrome 从输入 URL 到页面渲染的完整流程](Chrome%20从输入%20URL%20到页面渲染的完整流程)**：完整导航流程，包含渲染管线在其中的位置
- **[Chrome 多进程模型](Chrome%20多进程模型)**：渲染管线运行在渲染进程中，涉及多个线程协作
- **V8 引擎**：JavaScript 执行可能阻塞渲染管线
- **事件循环（Event Loop）**：JavaScript 任务调度与渲染帧的协调机制
- **Core Web Vitals**：LCP、INP、CLS 等指标与渲染管线直接相关
- **Critical Rendering Path**：首次渲染的关键路径优化

---

## 参考资料

- [Chrome for Developers: Inside look at modern web browser (part 3) - Inner workings of a Renderer Process](https://developers.google.com/web/updates/2018/09/inside-browser-part3)
- [Chrome for Developers: Inside look at modern web browser (part 4) - Input is coming to the Compositor](https://developers.google.com/web/updates/2018/09/inside-browser-part4)
- [Google Web Fundamentals: Critical Rendering Path](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/)
- [Google Web Fundamentals: Optimize JavaScript Execution](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution)
- [Google Web Fundamentals: Stick to Compositor-Only Properties and Manage Layer Count](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count)
- [Google Web Fundamentals: Animations and Performance](https://developers.google.com/web/fundamentals/design-and-ux/animations/animations-and-performance)
- [MDN: How browsers work - Rendering](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work)
- [HTML Standard: Rendering](https://html.spec.whatwg.org/multipage/rendering.html)
- [CSS Containment Module Level 3](https://www.w3.org/TR/css-contain-3/)
- [CSS `will-change` - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/will-change)