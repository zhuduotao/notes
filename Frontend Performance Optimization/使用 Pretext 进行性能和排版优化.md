---
created: '2026-04-18'
updated: '2026-04-18'
tags:
  - frontend
  - performance-optimization
  - text-layout
  - javascript
  - typescript
aliases:
  - Pretext
  - '@chenglou/pretext'
  - 文本测量与排版
source_type: official-repo
source_urls:
  - 'https://github.com/chenglou/pretext'
  - 'https://chenglou.me/pretext/'
status: verified
---

## 是什么

Pretext 是由 [chenglou](https://github.com/chenglou) 开发的纯 JavaScript/TypeScript 库，专注于**多行文本测量与排版**。核心卖点是：

- **快速**：`prepare()` 一次性完成文本分析与测量，`layout()` 纯算术计算，无需 DOM 操作
- **准确**：以浏览器字体引擎为基准，在 Chrome、Safari、Firefox 上经过大量语料验证
- **多语言支持**：覆盖 CJK、阿拉伯文、泰文、缅甸文、希伯来文等数十种书写系统
- **多渲染目标**：计算结果可用于 DOM、Canvas、SVG，以及未来的服务端渲染

安装：

```bash
npm install @chenglou/pretext
```

## 为什么重要：避免 DOM 测量导致的性能问题

浏览器中获取文本尺寸的传统方式是读取 DOM 属性（如 `getBoundingClientRect()`、`offsetHeight`）。这些操作会**触发同步布局（layout reflow）**，是浏览器中最昂贵的操作之一。

当 UI 组件独立测量文本高度时，如果读写交替发生，浏览器会反复重新布局整个文档。典型场景：

- 虚拟列表/滚动容器需要预先知道每行高度
- 响应式窗口 resize 时需要重新计算文本折行
- 瀑布流/Masonry 布局需要准确的卡片高度
- 聊天消息气泡需要紧凑的多行排版

Pretext 通过**两阶段架构**彻底规避这个问题：

1. `prepare(text, font)`：一次性完成文本分割、Canvas 测量、宽度缓存
2. `layout(prepared, maxWidth, lineHeight)`：纯算术运算，零 DOM 读取

## 核心用法

### 用例 1：测量段落高度（无需触碰 DOM）

最常见的场景：给定文本、字体和容器宽度，计算渲染后的总高度和行数。

```typescript
import { prepare, layout } from '@chenglou/pretext'

// 一次性分析 + 测量
const prepared = prepare('AGI 春天到了. بدأت الرحلة 🚀', '16px Inter')

// 纯算术计算，无 DOM 布局与 reflow
const { height, lineCount } = layout(prepared, textWidth, 20)
```

关键点：

- **不要对相同文本和配置重复调用 `prepare()`**，这违背了预计算的设计意图
- 窗口 resize 时只需重新调用 `layout()`，无需重新 `prepare()`
- `font` 格式与 Canvas `context.font` 一致，如 `16px Inter`、`500 17px Inter`
- `lineHeight` 需与 CSS `line-height` 声明同步

#### 模拟 textarea 行为

如果需要保留普通空格、`\t` 制表符和 `\n` 硬换行（类似 textarea 的 `white-space: pre-wrap`）：

```typescript
const prepared = prepare(textareaValue, '16px Inter', { whiteSpace: 'pre-wrap' })
const { height } = layout(prepared, textareaWidth, 20)
```

#### 保持 CJK/Hangul 不拆字

```typescript
const prepared = prepare(text, '16px Inter', { wordBreak: 'keep-all' })
```

### 用例 2：手动逐行排版

适用于需要将文本逐行渲染到 Canvas、SVG、WebGL 等场景。

#### 获取所有行信息

```typescript
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments('AGI 春天到了. بدأت الرحلة 🚀', '18px "Helvetica Neue"')
const { lines } = layoutWithLines(prepared, 320, 26)

for (let i = 0; i < lines.length; i++) {
  ctx.fillText(lines[i].text, 0, i * 26)
}
```

#### 轻量统计（不构建行字符串）

```typescript
import { measureLineStats, walkLineRanges } from '@chenglou/pretext'

const { lineCount, maxLineWidth } = measureLineStats(prepared, 320)

let maxW = 0
walkLineRanges(prepared, 320, line => {
  if (line.width > maxW) maxW = line.width
})
// maxW 即为最宽行 —— 文本的"紧缩包裹"宽度
```

#### 逐行流式排版（可变宽度）

适用于文字环绕浮动元素等场景：

```typescript
import { layoutNextLineRange, materializeLineRange, prepareWithSegments, type LayoutCursor } from '@chenglou/pretext'

const prepared = prepareWithSegments(article, BODY_FONT)
let cursor: LayoutCursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = 0

while (true) {
  const width = y < image.bottom ? columnWidth - image.width : columnWidth
  const range = layoutNextLineRange(prepared, cursor, width)
  if (range === null) break

  const line = materializeLineRange(prepared, range)
  ctx.fillText(line.text, 0, y)
  cursor = range.end
  y += 26
}
```

### 富文本内联流

Pretext 提供了 `@chenglou/pretext/rich-inline` 子模块，用于处理包含代码片段、链接、@提及、标签芯片（chips）等富文本的内联排版：

```typescript
import { prepareRichInline, walkRichInlineLineRanges, materializeRichInlineLineRange } from '@chenglou/pretext/rich-inline'

const prepared = prepareRichInline([
  { text: 'Ship ', font: '500 17px Inter' },
  { text: '@maya', font: '700 12px Inter', break: 'never', extraWidth: 22 },
  { text: "'s rich-note", font: '500 17px Inter' },
])

walkRichInlineLineRanges(prepared, 320, range => {
  const line = materializeRichInlineLineRange(prepared, range)
  // 每个片段保留源 item 索引、文本切片、gapBefore 和游标
})
```

富文本模块的设计约束：

- 仅支持 `white-space: normal`
- `break: 'never'` 保持原子性（如芯片、@提及不被拆行）
- `extraWidth` 由调用方控制水平装饰宽度（如 padding + border）
- 不是嵌套标记树，也不是通用 CSS inline 格式化引擎

## API 速查

### 用例 1 API

| API | 参数 | 返回值 | 说明 |
|-----|------|--------|------|
| `prepare(text, font, options?)` | `text: string`, `font: string`, `options?: { whiteSpace?: 'normal' \| 'pre-wrap', wordBreak?: 'normal' \| 'keep-all' }` | `PreparedText` | 一次性文本分析 + 测量，返回不透明句柄 |
| `layout(prepared, maxWidth, lineHeight)` | `prepared: PreparedText`, `maxWidth: number`, `lineHeight: number` | `{ height: number, lineCount: number }` | 纯算术计算文本高度和行数 |

### 用例 2 API

| API | 说明 |
|-----|------|
| `prepareWithSegments(text, font, options?)` | 同 `prepare()`，但返回更丰富的结构用于手动排版 |
| `layoutWithLines(prepared, maxWidth, lineHeight)` | 返回所有行信息（含文本、宽度、游标） |
| `walkLineRanges(prepared, maxWidth, onLine)` | 低级 API，逐行回调，不构建行字符串，适合二分搜索最佳宽度 |
| `measureLineStats(prepared, maxWidth)` | 仅返回行数和最宽行宽度，避免字符串分配 |
| `measureNaturalWidth(prepared)` | 返回不受宽度限制的最宽强制行宽度 |
| `layoutNextLineRange(prepared, start, maxWidth)` | 迭代器式 API，逐行计算范围 |
| `layoutNextLine(prepared, start, maxWidth)` | 逐行排版，每行可使用不同宽度 |
| `materializeLineRange(prepared, line)` | 将行范围转换为完整行对象 |

### 其他辅助 API

| API | 说明 |
|-----|------|
| `clearCache()` | 清除内部缓存，适用于频繁切换字体/文本变体的场景 |
| `setLocale(locale?)` | 设置 locale（默认使用当前 locale），内部会调用 `clearCache()` |

## 性能数据

以下为 Chrome 上的基准测试数据（冷启动单次运行）：

| 操作 | 耗时 | 说明 |
|------|------|------|
| `prepare()` | ~19.4ms | 500 段文本的一次性测量 |
| `layout()` | ~0.15ms | 归一化热路径吞吐量（每 500 段） |
| DOM 批量读写 | ~3.75ms | 单次 400→300px resize，先写后读 |
| DOM 交替读写 | ~41.75ms | 单次 400→300px resize，每个 div 写+读交替 |

关键对比：**DOM 交替读写比 Pretext 的 `layout()` 慢约 285 倍**。这正是 Pretext 解决的核心性能问题。

长文本语料表现（300px 宽度）：

| 语料 | 字符数 | `prepare()` | `layout()` | 行数 |
|------|--------|-------------|------------|------|
| 阿拉伯文长文 | 106,857 | ~72.6ms | ~0.27ms | 2,643 |
| 泰文长文 | 34,033 | ~16ms | ~0.09ms | 1,024 |
| 中文长文（祝福） | 9,428 | ~20.8ms | ~0.08ms | 626 |
| 日文长文（罗生门） | 5,702 | ~13.1ms | ~0.05ms | 380 |
| 韩文长文 | 10,272 | ~11.3ms | ~0.08ms | 428 |

## 限制与注意事项

### 不支持的 CSS 属性

Pretext 目前仅覆盖最常用的文本排版场景：

- `white-space`: 仅支持 `normal` 和 `pre-wrap`
- `word-break`: 仅支持 `normal` 和 `keep-all`
- `overflow-wrap`: 固定为 `break-word`（极窄宽度下仅在字素边界拆字）
- `line-break`: 固定为 `auto`
- 制表符遵循浏览器默认 `tab-size: 8`

### `system-ui` 字体在 macOS 上不安全

Canvas 和 DOM 在 macOS 上对 `system-ui` 的解析不同：小字号使用 SF Pro Text，大字号使用 SF Pro Display，且切换阈值不一致。实测不匹配集中在 `10-12px`、`14px`、`26px`。

**建议**：需要准确测量时使用具名字体（如 `Inter`、`Helvetica Neue`），避免使用 `system-ui`。

### 空字符串行为

`layout()` 对空字符串返回 `{ lineCount: 0, height: 0 }`。浏览器仍会将空块级元素渲染为一行 `line-height` 高度。如需模拟浏览器行为：

```typescript
const finalHeight = Math.max(1, lineCount) * lineHeight
```

### 段宽度 vs 精确字形位置

`segLevels` 可用于自定义双向文本（bidi）渲染，但段宽度是浏览器 Canvas 测量值，**不是**精确的字形位置数据，不能用于阿拉伯文或混合方向的精确 x 坐标重建。

### 软连字符

如果软连字符（soft hyphen）赢得断行，生成的行文本会包含可见的尾部 `-`。

### Emoji 宽度差异

Chrome 和 Firefox 在 macOS 上对小字号 Emoji 的 Canvas 测量值比 DOM 中更宽。Safari 无此差异。Pretext 内部已通过检测并缓存修正值来处理此问题，不在热路径中。

## 适用场景

| 场景 | 说明 |
|------|------|
| 虚拟列表/滚动容器 | 预先计算文本高度，无需 DOM 读取即可确定滚动范围 |
| 响应式 resize | 窗口缩放时只需重新调用 `layout()`，成本极低 |
| 瀑布流/Masonry 布局 | 准确预测卡片中文本高度，避免占位符和布局偏移 |
| Canvas/SVG/WebGL 文本渲染 | 手动逐行排版，支持浮动元素环绕 |
| 聊天消息气泡 | 紧凑多行排版，减少空白区域 |
| 开发时验证 | 无需浏览器即可验证按钮标签等是否会溢出折行 |
| 富文本内联流 | 代码片段、@提及、标签芯片等混合排版 |
| 手风琴/折叠面板 | 展开/收起时文本高度由 Pretext 计算 |
| 文本两端对齐比较 | 支持 CSS 对齐、贪婪连字符、Knuth-Plass 排版算法对比 |

## 架构设计哲学

Pretext 的架构核心来自 Sebastian Markbage 十年前的 [text-layout](https://github.com/chenglou/text-layout) 原型，经过多年迭代演进：

- **`prepare()` / `layout()` 分离**：昂贵工作只做一次，热路径保持纯算术
- **语义化预处理优于运行时修正**：如标点符号合并到前词测量、尾部可折叠空格悬挂
- **浏览器基准**：以 Canvas `measureText()` 直接调用浏览器字体引擎，而非自建字体渲染器
- **拒绝引入 DOM 测量**：所有尝试（隐藏 DOM 元素、SVG `getComputedTextLength()`、完整候选行字符串测量）均因性能或准确性被否决

## 相关概念

- **Layout Reflow**：浏览器同步布局，由 DOM 读取操作触发，是前端性能瓶颈之一
- **Canvas measureText()**：Canvas 2D API 的文本测量方法，直接调用浏览器字体引擎
- **Intl.Segmenter**：ECMAScript 国际化 API，用于文本分割（Pretext 内部使用）
- **Knuth-Plass 排版算法**：TeX 使用的段落排版算法，Pretext demo 中有对比展示
- **Bidi（双向文本）**：从右到左（RTL）与从左到右（LTR）混合书写系统的排版

## 参考资料

- [chenglou/pretext GitHub 仓库](https://github.com/chenglou/pretext)
- [Pretext 在线 Demo](https://chenglou.me/pretext/)
- [社区 Demo 集合](https://somnai-dreams.github.io/pretext-demos/)
- [RESEARCH.md — 研究日志](https://github.com/chenglou/pretext/blob/main/RESEARCH.md)
- [STATUS.md — 当前状态](https://github.com/chenglou/pretext/blob/main/STATUS.md)
- [Sebastian Markbage 的 text-layout 原型](https://github.com/chenglou/text-layout)
