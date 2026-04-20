---
created: "2026-04-17"
updated: "2026-04-17"
tags:
  - react
  - fiber
  - reconciliation
  - scheduler
  - frontend
  - internals
aliases:
  - React Fiber
  - Fiber Reconciler
  - Fiber 架构
source_type: mixed
source_urls:
  - "https://github.com/acdlite/react-fiber-architecture"
  - "https://legacy.reactjs.org/docs/codebase-overview.html#fiber-reconciler"
  - "https://legacy.reactjs.org/docs/reconciliation.html"
  - "https://react.dev/learn/render-and-commit"
  - "https://github.com/facebook/react"
status: verified
---

## 是什么

React Fiber 是 React 16 引入的**核心算法的重新实现**，是 React 团队历时两年多研究的成果。它本质上是一个**专为 UI 渲染优化的自定义调用栈**，将组件树中的每个节点表示为一个 "fiber" 对象（可理解为虚拟栈帧）。

Fiber 的核心突破是实现了**增量渲染（incremental rendering）**——能够将渲染工作拆分为多个小单元，分散到多个浏览器帧中执行，从而避免长时间阻塞主线程。

> Fiber Reconciler 自 React 16 起成为默认协调器，替代了之前的 Stack Reconciler。

## 为什么需要 Fiber

### Stack Reconciler 的局限性

React 15 及之前版本使用的 Stack Reconciler 采用**递归方式**遍历组件树：

```
触发更新 → 递归调用 render() → 生成完整 Virtual DOM → 一次性提交到 DOM
```

这种同步递归存在根本性问题：

- **不可中断**：一旦开始更新，必须完整执行完整个组件树的 diff，无法中途暂停
- **阻塞主线程**：大型组件树的 diff 可能耗时数百毫秒，导致动画掉帧、用户交互延迟
- **无法区分优先级**：所有更新同等对待，无法优先处理用户交互相关的更新

### UI 渲染的特殊需求

React 设计原则文档明确指出：

> React 不是通用的数据处理库，它是构建用户界面的库。它 uniquely positioned 能够知道哪些计算现在相关，哪些不相关。

UI 场景下的关键需求：

| 需求 | 说明 |
|------|------|
| **可中断** | 允许暂停工作，稍后恢复 |
| **优先级** | 用户交互（点击、输入）> 动画 > 数据更新 > 后台任务 |
| **可复用** | 新更新到来时，可以复用已完成的工作 |
| **可放弃** | 如果工作不再需要（如被新更新覆盖），可以丢弃 |

这些需求正是 Fiber 架构要解决的核心问题。

## 核心概念

### Fiber 节点结构

一个 fiber 是包含组件信息、输入和输出的 JavaScript 对象。关键字段：

| 字段 | 说明 |
|------|------|
| `type` | 组件类型（函数/类组件本身，或 `'div'` 等宿主组件标签名） |
| `key` | 用于 reconciliation 时判断 fiber 是否可复用 |
| `child` | 第一个子 fiber（对应 `render()` 返回值） |
| `sibling` | 下一个兄弟 fiber，形成单向链表 |
| `return` | 父 fiber（概念上等同于栈帧的返回地址） |
| `pendingProps` | 即将传入的 props（函数执行开始时设置） |
| `memoizedProps` | 上次渲染使用的 props（函数执行结束时设置） |
| `alternate` | 指向另一个 fiber，用于双缓冲机制 |
| `pendingWorkPriority` | 工作优先级（数字越大优先级越低，`NoWork` 为 0） |
| `output` | fiber 的输出结果（由宿主组件在叶子节点创建） |

### 链表树结构

Fiber 树使用**child-sibling-return**三指针表示树结构，而非传统的嵌套对象：

```
        Parent
       /      \
    Child1 → Child2 → Child3
```

- `Parent.child` → `Child1`
- `Child1.sibling` → `Child2`
- `Child2.sibling` → `Child3`
- 所有子节点的 `return` → `Parent`

这种结构的优势是**可以中断遍历**——保存当前 fiber 引用即可恢复，无需依赖调用栈。

### 双缓冲（Double Buffering）

任何时刻，一个组件实例最多对应**两个 fiber**：

- **current fiber**：已经渲染到屏幕上的 fiber
- **work-in-progress fiber**：正在构建中的 fiber

两者通过 `alternate` 字段互相引用：

```
current.alternate === workInProgress
workInProgress.alternate === current
```

工作流程：

1. 更新触发时，基于 current fiber 创建 work-in-progress fiber
2. 在 work-in-progress 上执行 diff 和构建
3. 完成后，将 work-in-progress 切换为新的 current（指针交换）
4. 旧的 work-in-progress 在下一次更新时复用（减少内存分配）

这类似于图形学中的双缓冲技术——在后台缓冲区绘制，完成后一次性切换到前台，避免用户看到中间状态。

### 时间分片（Time Slicing）

Fiber 将渲染工作拆分为**以 fiber 为单位的增量任务**：

```
帧 1: 处理 10 个 fiber → 检查剩余时间 → 时间到，让出主线程
帧 2: 从上次位置继续处理 8 个 fiber → 让出
帧 3: 继续处理剩余 fiber → 完成 → 进入 commit 阶段
```

关键机制：

- 每处理一个 fiber 后检查**是否还有剩余时间**（通过 `requestIdleCallback` 或 Scheduler 包）
- 如果时间不足，**暂停**并让出主线程给浏览器处理用户输入和动画
- 下一帧空闲时**恢复**，从上次暂停的 fiber 继续

> React 18 引入的 `createRoot` 默认启用了并发特性，时间分片是并发的基础。

### 优先级调度

Fiber 为不同类型的工作分配优先级：

| 优先级类别 | 触发场景 | 示例 API |
|-----------|---------|---------|
| **Immediate** | 用户交互需要立即响应 | — |
| **User Blocking** | 用户可见的交互 | `useTransition` 外的更新 |
| **Normal** | 普通数据更新 | `setState` |
| **Low** | 可以延迟的工作 | — |
| **Idle** | 不需要立即完成的工作 | — |

调度器（Scheduler 包，原称 `ReactScheduler`）负责：

1. 维护一个**任务队列**，按优先级排序
2. 每帧检查是否有**更高优先级**的任务需要插入
3. 决定何时**暂停**当前工作、何时**恢复**

## 工作流程：两阶段模型

Fiber 将更新过程明确分为两个阶段：

### Render 阶段（可中断、异步）

**做什么**：从根节点开始遍历 fiber 树，执行组件函数，计算哪些部分需要更新。

关键特性：

- **纯计算**：只调用组件函数，不修改 DOM
- **可中断**：可以随时暂停，稍后恢复
- **可丢弃**：如果有更高优先级更新到来，可以丢弃当前工作重新开始
- **并发安全**：同一时间可能有多个 render 在进行（不同优先级）

具体步骤：

```
1. 创建 work-in-progress 树（基于 current 树的 alternate）
2. 从根节点开始深度优先遍历
3. 对每个 fiber：
   a. 比较 pendingProps 和 memoizedProps
   b. 如果相同，跳过（复用上次结果）
   c. 如果不同，调用组件函数，生成新的子 fiber
   d. 标记有副作用的 fiber（插入、更新、删除）
4. 收集所有副作用到 effect list
```

### Commit 阶段（同步、不可中断）

**做什么**：将 render 阶段计算出的变更应用到 DOM。

关键特性：

- **同步执行**：一旦开始，不会中断
- **直接操作 DOM**：执行插入、更新、删除操作
- **生命周期触发**：执行 `componentDidMount`、`componentDidUpdate`、`useLayoutEffect` 等

三个子阶段：

| 子阶段 | 说明 |
|--------|------|
| **Before Mutation** | 执行 `getSnapshotBeforeUpdate` |
| **Mutation** | 执行 DOM 操作（增删改） |
| **Layout** | 执行 `componentDidMount`/`Update`、`useLayoutEffect` |

> Commit 阶段必须同步完成，因为用户已经能看到中间状态。如果 commit 时间过长，仍会导致掉帧。

## 与 Stack Reconciler 对比

| 维度 | Stack Reconciler（React 15） | Fiber Reconciler（React 16+） |
|------|------------------------------|-------------------------------|
| **遍历方式** | 递归（依赖调用栈） | 循环 + 链表（自定义栈帧） |
| **可中断性** | 不可中断 | 可中断、可恢复 |
| **优先级** | 无优先级概念 | 支持多优先级调度 |
| **工作复用** | 不支持 | 支持（alternate 双缓冲） |
| **并发** | 不支持 | 支持（React 18 默认启用） |
| **错误处理** | 错误导致整棵树卸载 | Error Boundary 隔离错误 |
| **返回值** | 仅支持单个元素 | 支持返回数组（React 16+） |

## React 18 并发特性

Fiber 架构为 React 18 的并发特性奠定了基础：

### `startTransition`

将更新标记为**过渡更新**（低优先级），让位于紧急更新：

```jsx
import { startTransition } from 'react';

// 紧急更新：立即响应
setInputValue(input);

// 过渡更新：可以被打断
startTransition(() => {
  setSearchQuery(input);
});
```

### `useTransition`

返回 `isPending` 状态，可用于显示加载指示器：

```jsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setTab(newTab);
});

{isPending && <Spinner />}
```

### `useDeferredValue`

延迟某个值的更新，类似于防抖但由 React 调度：

```jsx
const deferredQuery = useDeferredValue(query);
// deferredQuery 会比 query 晚更新，让位于其他渲染
```

> 这些 API 的核心原理都是利用 Fiber 的优先级调度机制，将更新分配到不同的优先级队列中。

## 限制与注意事项

### Commit 阶段不可中断

虽然 render 阶段可以拆分，但 **commit 阶段必须同步完成**。如果一次更新涉及大量 DOM 操作（如渲染数千个节点），commit 阶段仍可能阻塞主线程。

**应对策略**：

- 使用虚拟列表（`react-window`）减少实际 DOM 节点数量
- 使用 `startTransition` 将非关键更新降级
- 避免在 `useLayoutEffect` 中执行耗时操作

### 内存开销

双缓冲机制意味着**需要维护两棵 fiber 树**，内存占用比 Stack Reconciler 更高。不过 React 通过复用 fiber 对象（`cloneFiber`）来减少分配开销。

### 调试复杂度

并发特性可能导致**渲染结果与代码执行顺序不一致**，增加调试难度：

- Strict Mode 下组件函数会被调用两次，帮助发现副作用问题
- 需要理解 "render 可能被重试" 的概念，避免在 render 中产生副作用

### 不解决所有性能问题

Fiber 优化的是**调度和响应性**，而非渲染本身的计算量：

- 如果组件本身渲染很慢（复杂计算、大量 DOM），Fiber 无法加速
- 仍需配合 `React.memo`、`useMemo`、`useCallback` 等减少不必要渲染
- 官方建议：**不要过早优化**，先测量再优化

## 常见误区

| 误区 | 澄清 |
|------|------|
| "Fiber 就是 Virtual DOM" | Fiber 是协调算法的实现，Virtual DOM 是组件输出的数据结构，两者不同层次 |
| "Fiber 让 React 变得更快" | Fiber 主要提升的是**响应性**（不丢帧），而非绝对渲染速度 |
| "时间分片 = Web Workers" | Fiber 仍在主线程运行，只是拆分任务；Web Workers 是完全独立的线程 |
| "React 16 就有并发" | React 16 引入了 Fiber，但并发特性在 React 18 才默认启用 |
| "所有更新都会被拆分" | 只有使用 `createRoot` 且通过并发 API 触发的更新才会被调度 |

## 相关概念

- [[前端性能优化]] — Fiber 解决的是运行时交互响应性（INP 指标相关）
- [[事件循环与宏微任务]] — 理解 Fiber 如何利用空闲时间片
- [[requestIdleCallback]] — 浏览器原生空闲回调，Scheduler 包的底层依赖之一

## 参考资料

- [acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture) — Andrew Clark（React 核心成员）撰写的 Fiber 架构官方说明文档
- [React Codebase Overview: Fiber Reconciler](https://legacy.reactjs.org/docs/codebase-overview.html#fiber-reconciler) — React 官方代码库概览，Fiber 设计目标
- [React Reconciliation](https://legacy.reactjs.org/docs/reconciliation.html) — 协调算法的 O(n) 启发式策略说明
- [React: Render and Commit](https://react.dev/learn/render-and-commit) — React 18 官方文档，渲染三步骤（Trigger → Render → Commit）
- [facebook/react](https://github.com/facebook/react) — React 源码仓库，Fiber 实现在 `packages/react-reconciler`
