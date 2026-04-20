---
created: 2026-04-18
updated: 2026-04-18
tags:
  - react
  - devtools
  - performance
  - profiling
  - 调试
aliases:
  - React DevTools Profiler
  - React Profiler 使用指南
  - React 性能分析工具
source_type: official-doc
source_urls:
  - https://react.dev/learn/react-developer-tools
  - https://react.dev/reference/react/Profiler
  - https://legacy.reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html
  - https://github.com/facebook/react/tree/main/packages/react-devtools
status: verified
---

## 概述

React DevTools Profiler 是 React 官方浏览器扩展中的性能分析面板，用于记录和分析组件渲染性能，帮助开发者定位 React 应用中的性能瓶颈。它基于 React 内部的 Profiler API 收集每个组件的渲染时间信息，以可视化图表呈现。

> **与 Chrome DevTools Performance 面板的区别**：Chrome DevTools 分析的是浏览器级别的整体性能（长任务、布局重排、内存等），而 React DevTools Profiler 专注于 **React 组件级别的渲染耗时**，能直接告诉你哪个组件渲染慢、为什么渲染。

## 安装与环境要求

### 浏览器扩展（推荐）

React DevTools 提供主流浏览器的官方扩展：

- [Chrome 扩展](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi?hl=en)
- [Firefox 扩展](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/)
- [Edge 扩展](https://microsoftedge.microsoft.com/addons/detail/react-developer-tools/gpphkfbcpidddadnkolkpfckpihlkkil)

安装后，访问使用 React 构建的网站，DevTools 中会出现 **Components** 和 **Profiler** 两个面板。

### Safari 及其他浏览器

Safari 不支持浏览器扩展，需使用独立版本：

```bash
npm install -g react-devtools
# 或
yarn global add react-devtools
```

启动后在页面 `<head>` 中注入连接脚本：

```html
<script src="http://localhost:8097"></script>
```

### React Native

React Native 0.76+ 内置集成了 React DevTools（React Native DevTools），功能与浏览器扩展一致。更早版本需使用上述独立版本。

### 版本要求

- Profiler 功能需要 **react-dom 16.5+**
- 开发模式（DEV mode）下默认支持 profiling
- 生产模式需要使用特殊的 profiling 构建：`react-dom/profiling`（详见 [React 官方说明](https://fb.me/react-profiling)）

## 使用流程

### 1. 开始录制

打开 React DevTools，切换到 **Profiler** 面板，点击左上角的 **录制按钮**（灰色圆点变为红色）开始录制。

### 2. 执行操作

在应用中正常操作（点击、输入、滚动等），DevTools 会自动收集每次渲染的性能数据。

### 3. 停止录制

操作完成后再次点击录制按钮（此时为红色方块）停止录制，面板会展示分析结果。

### 4. 分析数据

停止录制后，Profiler 提供多种视图来分析数据。

## 核心视图解读

### Commit 柱状图

位于面板顶部，以柱状图展示每次 commit 的耗时：

- **柱子高度和颜色**：对应渲染耗时，越高越黄表示耗时越长，越矮越蓝表示耗时越短
- **黑色柱子**：当前选中的 commit
- **左右箭头**：切换查看不同 commit

> **Render 阶段 vs Commit 阶段**：React 的工作分为两个阶段。Render 阶段确定需要做什么变更（调用组件函数、比较 Virtual DOM），Commit 阶段才真正应用变更（插入/更新/删除 DOM 节点）。Profiler 按 commit 来组织数据。

### 火焰图（Flamegraph）

火焰图是定位性能问题最常用的视图：

- **每个柱子**：代表一个 React 组件
- **柱子宽度**：表示该组件（含子组件）**上次渲染时**的耗时。越宽表示渲染越慢
- **柱子颜色**：表示该组件在**当前 commit 中**的耗时
  - 🔴 黄色/红色：耗时较长
  - 🔵 蓝色：耗时较短
  - ⚫ 灰色：本次 commit 中未渲染

**交互操作**：
- 点击组件可以**放大/缩小**查看该子树
- 选中组件后，右侧面板显示该组件的 props、state 和 hooks 值
- 在不同 commit 间切换时，可以对比 props/state 的变化，找出**为什么渲染**

> **宽度 vs 颜色的区别**：宽度反映的是组件上次渲染的总耗时（可能是上一次 commit），颜色反映的是当前 commit 中的耗时。一个组件可能很宽（上次渲染慢）但颜色是灰色（本次未渲染）。

### 排序图（Ranked chart）

按渲染耗时从高到低排列所有组件，快速识别最耗时的组件：

- 耗时最长的组件排在顶部
- 渲染时间包含子组件的时间，所以耗时长的组件通常位于组件树较上层
- 同样支持点击放大/缩小查看子树

### 组件图（Component chart）

展示**单个组件在整个录制期间的所有渲染记录**：

- 每个柱子代表该组件的一次渲染
- 颜色和高度表示相对于同 commit 中其他组件的耗时
- 双击火焰图中的组件，或选中后点击右侧的蓝色柱状图图标即可查看

**用途**：识别哪些组件渲染过于频繁。如果一个简单组件在录制期间渲染了几十次，可能就是优化目标。

### 交互追踪（Interactions）

通过 React 的 [Interaction Tracing API](https://fb.me/react-interaction-tracing)（实验性），可以追踪每次更新的**触发原因**：

- 每一行代表一个被追踪的交互（如"点击搜索按钮"）
- 行上的彩色圆点代表与该交互相关的 commit
- 可以从火焰图中看到某个 commit 属于哪个交互

> **注意**：Interaction Tracing API 仍是实验性功能，需要手动使用 `unstable_trace` 等 API 标记交互。

## Commit 过滤

录制时间较长时，可能会产生大量 commit。Profiler 提供过滤功能：

- 设置时间阈值，隐藏所有**快于该值**的 commit
- 只保留耗时较长的 commit，便于聚焦真正的性能问题

## 多 Root 应用的处理

如果应用有多个 React root（例如多次调用 `ReactDOM.createRoot`），Profiler 默认只显示当前在 Components 面板中选中的 root 的数据。

**看不到数据时**：切换到 Components 面板，选择另一个 root，再回到 Profiler 面板查看。

## 极快 Commit 无数据

某些 commit 可能非常快，`performance.now()` 无法提供有意义的时间数据。此时会显示 "No timing data to display for the selected commit"，属于正常现象。

## 编程式 Profiler API

除了浏览器扩展的交互式 Profiler，React 还提供 `<Profiler>` 组件用于编程式测量：

```jsx
import { Profiler } from 'react';

function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  console.log(`${id} 在 ${phase} 阶段耗时 ${actualDuration}ms`);
}

<App>
  <Profiler id="Sidebar" onRender={onRender}>
    <Sidebar />
  </Profiler>
</App>
```

### onRender 回调参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | `<Profiler>` 的 id prop，用于区分多个 profiler |
| `phase` | string | `"mount"`（首次挂载）、`"update"`（更新）、`"nested-update"`（嵌套更新） |
| `actualDuration` | number | 本次更新中 `<Profiler>` 及其子树的实际渲染耗时（ms）。**反映 memoization 的效果**，理想情况下首次挂载后该值应显著下降 |
| `baseDuration` | number | 估计不使用优化时重新渲染整个子树的耗时（ms）。通过累加树中各组件最近一次渲染耗时计算得出。**对比 `actualDuration` 可评估 memoization 的收益** |
| `startTime` | number | React 开始渲染本次更新的时间戳 |
| `commitTime` | number | React 提交本次更新的时间戳。同一 commit 中所有 profiler 共享此值 |

### 使用场景

- 自动化性能回归测试
- 生产环境性能监控（需使用 profiling 构建）
- 对不同部分分别测量（可嵌套多个 `<Profiler>`）

### 注意事项

- Profiling 会带来额外开销，**生产构建默认禁用**
- 每个 `<Profiler>` 都会增加 CPU 和内存开销，应仅在必要时使用
- 被 `<Profiler>` 包裹的组件在 profiling 构建中会标记在 React Performance tracks 的 Components track 中

## 定位性能问题的典型流程

### 第一步：录制典型用户操作

不要录制空操作，而是执行真实用户会做的操作：

- 页面首次加载
- 点击按钮触发数据加载
- 在搜索框输入文字
- 滚动长列表
- 切换 Tab

### 第二步：在火焰图中找最宽/最黄的组件

- **最宽**：表示该组件上次渲染总耗时长
- **最黄**：表示在本次 commit 中耗时长
- 重点关注那些**每次 commit 都渲染且耗时较长**的组件

### 第三步：分析渲染原因

选中问题组件，切换 commit 查看右侧面板中 props/state 的变化：

- 如果是 **props 没变但依然渲染**：说明是父组件渲染导致的连带渲染，考虑用 `React.memo` 优化
- 如果是 **props 每次都变**：检查是否传入了新的引用（内联函数、内联对象），考虑用 `useCallback`/`useMemo` 稳定引用
- 如果是 **state 变化**：确认状态更新是否合理，是否可以拆分或 colocation

### 第四步：在组件图中找渲染频率异常

双击问题组件查看组件图：

- 如果组件在录制期间渲染了数十次，但每次内容变化不大，说明存在**不必要的重复渲染**
- 对比同 commit 中其他组件的耗时，判断是否是该 commit 的主要瓶颈

### 第五步：优化后再次录制验证

应用优化（`React.memo`、`useMemo`、状态拆分等）后，重复上述录制流程，对比优化前后的数据：

- commit 总耗时是否下降
- 问题组件的渲染次数是否减少
- `actualDuration` 相比 `baseDuration` 的差距是否增大（说明 memoization 生效）

## 常见问题

### Profiler 面板不显示

确认当前网站确实使用了 React。某些使用 SSR 且 hydrate 完成的页面，或 React 版本过低（< 16.5）可能不显示 Profiler 面板。

### 录制后没有数据

- 确认录制期间页面确实发生了渲染
- 如果是多 root 应用，尝试在 Components 面板切换 root
- 确认 React 版本 >= 16.5

### 所有组件都是蓝色/耗时很短

说明当前操作没有性能瓶颈，或者操作本身很简单。尝试录制更复杂的交互，或加载数据量更大的场景。

### 生产环境如何使用 Profiler

生产环境默认禁用了 profiling。如需在生产环境分析性能，需要使用特殊的 profiling 构建：

- 将 `react-dom` 替换为 `react-dom/profiling`
- 同时使用 `scheduler/tracing-profiling`（如使用 Interaction Tracing API）

详见 [React Profiling 构建说明](https://fb.me/react-profiling)。

## 与 React 性能优化的关系

React DevTools Profiler 是「测量 → 定位 → 优化 → 验证」闭环中的**测量和验证**环节。定位到问题组件后，具体的优化手段包括：

- 使用 `React.memo` 减少不必要的渲染
- 使用 `useMemo` / `useCallback` 稳定引用
- 状态 Colocation（状态就近原则）
- 使用 `startTransition` / `useDeferredValue` 调度优先级
- 使用虚拟列表减少 DOM 节点数量

详见 [[React 性能优化思路与常用工具]]。

## 相关概念

- [[React 性能优化思路与常用工具]] — React 性能优化方法论和具体手段
- [[React Fiber 架构详解]] — Fiber 是 React 性能调度的底层基础
- [[Chrome DevTools 性能与内存调试工具]] — 浏览器级别的性能分析工具，与 React DevTools 互补

## 参考资料

- [React Developer Tools](https://react.dev/learn/react-developer-tools) — React 官方开发者工具文档
- [`<Profiler>` API 参考](https://react.dev/reference/react/Profiler) — React 官方 Profiler 组件 API 文档
- [Introducing the React Profiler (React Blog, 2018)](https://legacy.reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html) — React Profiler 发布博客，包含详细的视图解读和示例
- [React DevTools 源码](https://github.com/facebook/react/tree/main/packages/react-devtools) — React DevTools 的 GitHub 仓库
