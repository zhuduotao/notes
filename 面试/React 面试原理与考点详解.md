---
created: "2026-04-17"
updated: "2026-04-17"
tags:
  - React
  - 面试
  - 前端
  - 原理
  - Fiber
  - Reconciliation
aliases:
  - React 原理
  - React 面试考点
  - React 核心机制
  - React Internals
source_type: mixed
source_urls:
  - https://react.dev/learn/render-and-commit
  - https://react.dev/learn/queueing-a-series-of-state-updates
  - https://react.dev/reference/react/useState
  - https://react.dev/reference/react/memo
  - https://react.dev/reference/react/useTransition
  - https://react.dev/reference/react-dom/components/common#react-event-object
  - https://react.dev/learn/reconciliation
  - https://react.dev/learn/lists-and-keys
status: verified
---

## 渲染流程：Trigger → Render → Commit

React 每次屏幕更新都经历三个步骤：

### 1. Trigger（触发渲染）

两种情况会触发渲染：
- **首次渲染**：通过 `createRoot(rootNode).render(<App />)` 触发
- **状态更新**：组件或其祖先的 state 发生变化，自动排队渲染

### 2. Render（渲染组件）

React 调用组件函数来计算应该显示什么。

**关键特性**：
- 渲染是**纯计算**：相同输入必须产生相同输出
- 渲染期间不能修改渲染前已存在的对象或变量
- 递归过程：如果组件返回另一个组件，React 会继续渲染该组件，直到没有嵌套
- **渲染不等于 DOM 更新**：渲染只是计算差异，真正的 DOM 操作在 Commit 阶段

Strict Mode 下，React 会在开发环境中调用每个组件函数两次，以帮助发现不纯的函数。

### 3. Commit（提交到 DOM）

- **首次渲染**：使用 `appendChild()` 等 DOM API 创建节点
- **重新渲染**：应用最小必要的操作使 DOM 与最新渲染输出匹配
- **React 只在渲染结果有差异时才修改 DOM**

> 参考：[Render and Commit - react.dev](https://react.dev/learn/render-and-commit)

---

## Fiber 架构

### 什么是 Fiber

Fiber 是 React 16 引入的**内部数据结构**，用于表示组件树中的每个节点。它是 React 实现**增量渲染**和**任务调度**的核心。

每个 Fiber 节点包含：
- 组件的类型（函数组件、class 组件、原生 DOM 等）
- 指向父节点、子节点、兄弟节点的指针（链表结构）
- 当前状态（state）、props、副作用标记（effectTag）
- 指向 alternate（备用树）的指针

### 双缓冲机制（Double Buffering）

React 维护两棵 Fiber 树：
- **current 树**：当前已渲染到屏幕上的树
- **workInProgress 树**：正在构建中的新树

渲染完成后，workInProgress 树成为新的 current 树，旧的被丢弃。这确保了用户始终看到的是完整的 UI，不会出现中间状态。

### Fiber 的工作循环

Fiber 架构将渲染工作拆分为多个**时间切片（time slice）**：

```
渲染阶段（可中断）           提交阶段（不可中断）
├── beginWork              ├── beforeMutation
├── completeWork           ├── mutation（DOM 操作）
├── 检查是否有更高优先级任务  └── layout
└── 如有，让出控制权
```

- **渲染阶段（Render Phase）**：可中断，负责计算哪些节点需要更新
- **提交阶段（Commit Phase）**：不可中断，执行实际的 DOM 操作

### 调度器（Scheduler）

React 的调度器基于浏览器的 `requestIdleCallback` 理念（实际使用 `MessageChannel` 模拟），根据任务优先级决定执行顺序：

| 优先级 | 用途 |
|--------|------|
| Immediate | 用户交互相关的紧急更新 |
| UserBlocking | 用户可见的更新（如输入框） |
| Normal | 普通数据更新 |
| Low | 低优先级更新 |
| Idle | 空闲时执行的更新 |

> 面试考点：Fiber 架构解决了什么问题？答：解决了 React 15 中递归 diff 导致的长任务阻塞问题，通过可中断渲染实现了 Concurrent Mode。

---

## Reconciliation 与 Diff 算法

### 核心策略

React 的 Diff 算法采用**启发式 O(n)** 策略（而非传统 O(n³)），基于两个假设：

1. **不同类型的元素会产生不同的树**：如果元素类型不同（如 `<div>` 变为 `<p>`），React 会销毁旧树并重建新树
2. **开发者通过 `key` prop 标记稳定的子元素**：`key` 帮助 React 识别哪些子元素在列表中没有变化

### Diff 规则

**类型不同时**：
- 销毁旧的 DOM 节点及其子树
- 创建新的 DOM 节点
- 触发旧组件的 `componentWillUnmount`，新组件的挂载生命周期

**类型相同时**：
- 保留 DOM 节点，仅更新变化的属性
- 对组件实例，触发 `componentWillReceiveProps` / 重新渲染

**子元素 Diff**：
- React 同时遍历新旧子元素列表
- 遇到差异时开始生成操作（插入、移动、删除）
- 使用 `key` 来匹配新旧元素，避免不必要的销毁重建

### key 的作用

`key` 是 React 用于识别列表中哪些元素发生变化的**特殊属性**：

- **为什么不用 index 做 key**：
  - 当列表顺序变化（插入、删除、排序）时，index 会变化
  - 导致 React 误判元素变化，销毁重建不必要的节点
  - 可能导致状态错位（如输入框内容保留在错误的行上）
- **最佳实践**：使用数据中唯一且稳定的 ID（如数据库 ID）
- **何时可以用 index**：列表是静态的、不会重新排序或过滤

> 参考：[Lists and Keys - react.dev](https://react.dev/learn/reconciliation)

---

## 状态更新机制

### 快照特性（Snapshot）

React 的 state 在一次渲染中是**只读的快照**：

```js
function handleClick() {
  console.log(count);  // 0
  setCount(count + 1); // 请求下次渲染使用 1
  console.log(count);  // 仍然是 0！
}
```

- `setState` 不会改变当前执行代码中的 state 变量
- 它只影响下一次渲染时 `useState` 返回的值

### 批处理（Batching）

React 会等待事件处理程序中**所有代码执行完毕**后再处理状态更新：

```js
// 点击一次，number 只增加 1（不是 3）
setNumber(number + 1);
setNumber(number + 1);
setNumber(number + 1);
```

**批处理规则**：
- React 只在**安全的情况下**进行批处理
- **不会跨多个独立事件批处理**（每次点击独立处理）
- 确保第一个按钮点击禁用表单后，第二次点击不会再次提交

### Updater 函数

当需要在同一次事件中多次更新同一个 state 时，使用 updater 函数：

```js
// 正确：点击一次增加 3
setNumber(n => n + 1);
setNumber(n => n + 1);
setNumber(n => n + 1);
```

React 处理更新队列的算法：

| 排队更新 | `n` | 返回值 |
|----------|-----|--------|
| `"replace with 5"` | `0`（未使用） | `5` |
| `n => n + 1` | `5` | `6` |
| `"replace with 42"` | `6`（未使用） | `42` |

- **updater 函数**（`n => n + 1`）：添加到队列，基于前一个状态计算
- **其他值**（如 `5`）：添加到队列，忽略已排队的更新

> 参考：[Queueing a Series of State Updates - react.dev](https://react.dev/learn/queueing-a-series-of-state-updates)

### 状态更新的比较规则

React 使用 `Object.is()` 比较新旧状态：
- 如果值相同（`Object.is` 判断），React **跳过重新渲染**
- 这是优化手段，但组件函数可能仍会被调用一次（结果被丢弃）

---

## Concurrent Mode 与优先级调度

### 什么是 Concurrent Mode

Concurrent Mode 允许 React **同时准备多个版本的 UI**，并根据优先级选择渲染哪个版本。核心能力：

- **可中断渲染**：高优先级更新可以打断低优先级更新
- **响应式渲染**：保持用户交互的即时响应
- **避免不必要的 loading 状态**：Transition 不会触发闪烁的 loading

### useTransition

`useTransition` 将状态更新标记为**非阻塞**：

```js
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setQuery(input); // 非阻塞更新
});
```

**特性**：
- `isPending`：过渡状态是否在进行中
- Transition 中的更新可以被其他更新**打断**
- 适用于：搜索结果过滤、大数据列表渲染、路由导航
- **不能用于控制文本输入**（输入需要同步更新）

### useDeferredValue

延迟更新 UI 的非关键部分：

```js
const deferredQuery = useDeferredValue(query);
```

- 与 `useTransition` 类似，但作用于**值**而非函数
- 适用于：输入框联动大量数据渲染

### Suspense

Suspense 允许组件在数据未就绪时"挂起"（suspend），并显示 loading fallback：

```js
<Suspense fallback={<Spinner />}>
  <SomeComponent />
</Suspense>
```

- 与 `useTransition` 配合使用时，可以**防止已显示内容被 loading 替代**
- Suspense-enabled 路由默认将导航更新包装为 Transition

> 参考：[useTransition - react.dev](https://react.dev/reference/react/useTransition)

---

## Hooks 原理

### 为什么 Hooks 只能在顶层调用

React 依赖 **Hook 的调用顺序**来关联状态。内部实现：

```
组件首次渲染：
  useState('a') → 链表节点1
  useEffect(...) → 链表节点2
  useState('b') → 链表节点3

重新渲染时，React 按相同顺序遍历链表，恢复对应状态
```

如果在条件语句中调用 Hook，调用顺序变化会导致**状态错位**。

### Hooks 的存储结构

每个组件实例有一个**链表**存储 Hooks：

```
memoizedState → hook1 → hook2 → hook3 → null
```

每个 Hook 节点包含：
- `memoizedState`：当前状态值
- `baseState`：基础状态（用于处理更新队列）
- `baseQueue`：待处理的更新队列
- `next`：指向下一个 Hook

### 自定义 Hooks

自定义 Hook 是以 `use` 开头的函数，可复用状态逻辑：

- 每次调用都是**独立的状态实例**
- 命名必须以 `use` 开头（React 依赖此约定应用 Hooks 规则）
- 可组合多个内置 Hook 或其他自定义 Hook

### 常见 Hooks 原理要点

| Hook | 原理要点 |
|------|----------|
| `useState` | 状态快照 + 更新队列，updater 函数链式执行 |
| `useEffect` | 渲染完成后异步执行，依赖数组决定何时重新执行 |
| `useLayoutEffect` | 在浏览器重绘之前**同步**执行，适用于 DOM 测量 |
| `useRef` | 修改 `current` **不触发**重新渲染，持有可变值 |
| `useMemo` | 缓存计算结果，依赖变化时重新计算 |
| `useCallback` | 缓存函数定义，等价于 `useMemo(() => fn, deps)` |
| `useContext` | 订阅 Context，值变化时所有订阅组件重新渲染 |

> 参考：[Built-in Hooks - react.dev](https://react.dev/reference/react/hooks)

---

## 性能优化原理

### React.memo

`memo` 返回一个**记忆化版本**的组件，在 props 未变化时跳过重新渲染：

```js
const MemoizedComponent = memo(SomeComponent, arePropsEqual?);
```

**关键机制**：
- 默认使用 `Object.is()` 进行**浅比较**
- 是性能优化手段，**不是保证**（React 仍可能在某些情况下重新渲染）
- 配合 `useMemo` 和 `useCallback` 使用才有意义（避免 props 总是"新"的）

**memo 无效的场景**：
- props 中包含每次渲染都新建的对象或函数
- 组件使用了频繁变化的 Context
- 组件自身 state 变化

### useMemo 与 useCallback

```js
// 缓存计算结果
const cachedValue = useMemo(() => computeExpensive(a, b), [a, b]);

// 缓存函数定义
const cachedFn = useCallback(fn, [dependencies]);
```

**性能陷阱**：
- 缓存本身有开销（内存 + 依赖比较）
- 应先通过 React DevTools Profiler 确认瓶颈
- 对于简单计算或稳定组件，通常不需要

### React Compiler

React Compiler 自动对所有组件应用等价于 `memo` 的优化，减少手动记忆化的需求。启用后通常不再需要 `React.memo`。

> 参考：[memo - react.dev](https://react.dev/reference/react/memo)

---

## Context 渲染机制

### Context 的渲染行为

- Context 值变化时，**所有订阅该 Context 的组件都会重新渲染**
- 即使组件被 `memo` 包裹，Context 变化仍会触发重新渲染
- `useContext` 的组件无法通过 props 浅比较来避免渲染

### 避免 Context 引起的过度渲染

**策略 1：拆分 Context**
将频繁变化的值和稳定变化的值放在不同的 Context 中。

**策略 2：组件拆分 + memo**
```js
// 外层组件读取 Context
function Outer() {
  const theme = useContext(ThemeContext);
  return <MemoizedInner theme={theme} />;
}

// 内层组件被 memo 包裹
const MemoizedInner = memo(function Inner({ theme }) {
  return <div className={theme}>...</div>;
});
```

**策略 3：使用选择器模式**
通过自定义 Hook 或状态管理库（如 Zustand、Jotai）实现选择性订阅。

---

## 事件系统

### 合成事件（Synthetic Event）

React 事件是对浏览器原生事件的**跨浏览器封装**：

```js
<button onClick={e => {
  console.log(e); // React 合成事件对象
}}>
```

**特性**：
- 符合底层 DOM 事件的标准，但修复了浏览器差异
- `e.nativeEvent` 指向原生浏览器事件
- 部分事件在 React 中的行为与浏览器不同（如 `onBlur` 在 React 中会冒泡）

### 事件委托

React 不直接在 DOM 元素上绑定事件监听器，而是使用**事件委托**：

- 事件监听器绑定在**根节点**（React 17+ 绑定在 `root` 容器，而非 `document`）
- 事件冒泡到根节点后，React 根据组件树分发到对应组件
- 这意味着 `e.currentTarget` 在 React 中反映的是 React 组件树中的节点，而非原生 DOM 事件中的节点

### 事件处理阶段

- **冒泡阶段**：大多数 React 事件在冒泡阶段处理
- **捕获阶段**：使用 `onEventCapture` 变体（如 `onClickCapture`）
- `stopPropagation()` 停止事件在 **React 树**中的传播
- 注意：`onMouseLeave` / `onMouseEnter` 不冒泡，而是从一个元素传播到另一个元素

> 参考：[React Event Object - react.dev](https://react.dev/reference/react-dom/components/common#react-event-object)

---

## 常见面试考点汇总

### 基础概念

| 考点 | 核心要点 |
|------|----------|
| JSX 是什么 | JSX 是 `React.createElement()` 的语法糖，编译后生成 JS 对象（React Element） |
| 虚拟 DOM | 用 JS 对象描述 DOM 结构，通过 Diff 算法最小化真实 DOM 操作 |
| 受控组件 vs 非受控组件 | 受控组件的值由 React state 驱动；非受控组件使用 ref 直接读取 DOM 值 |
| 组件通信方式 | props、Context、状态管理库、自定义事件、ref |

### 渲染与性能

| 考点 | 核心要点 |
|------|----------|
| 什么触发重新渲染 | state 变化、props 变化、Context 变化、父组件渲染（默认行为） |
| 如何避免不必要的渲染 | `React.memo`、`useMemo`、`useCallback`、拆分组件、提升 state 到合适层级 |
| 长列表优化 | 虚拟滚动（react-window）、分页加载、`React.memo` 子项 |
| 渲染性能分析工具 | React DevTools Profiler、`why-did-you-render`、Chrome Performance 面板 |

### Hooks 相关

| 考点 | 核心要点 |
|------|----------|
| Hooks 规则 | 只在顶层调用、只在 React 函数中调用 |
| useEffect 依赖数组 | `[]` 仅挂载/卸载时执行；省略则每次渲染后执行；`[a, b]` 依赖变化时执行 |
| useEffect vs useLayoutEffect | useEffect 异步执行；useLayoutEffect 在浏览器重绘前同步执行 |
| useRef vs useState | 不需要触发渲染用 useRef；需要更新 UI 用 useState |
| 自定义 Hooks 设计 | 命名以 `use` 开头；每次调用独立实例；可组合多个 Hook |

### 状态管理

| 考点 | 核心要点 |
|------|----------|
| useState 快照特性 | 一次渲染中 state 是只读的，setState 不影响当前执行的代码 |
| 批处理机制 | React 等待事件处理程序完成后再处理更新 |
| updater 函数 | `setState(prev => newValue)` 用于同一次事件中多次更新同一 state |
| 状态不可变性 | 必须替换而非修改对象/数组，使用展开运算符或 immutable 库 |
| useReducer 适用场景 | 复杂状态逻辑、多子操作、状态更新依赖前一个状态 |

### 架构与原理

| 考点 | 核心要点 |
|------|----------|
| Fiber 解决的问题 | 长任务阻塞、无法中断渲染、优先级调度 |
| Diff 算法复杂度 | O(n) 启发式策略，基于类型比较和 key 匹配 |
| key 的作用 | 帮助 React 识别列表中稳定的元素，避免不必要的销毁重建 |
| 为什么不用 index 做 key | 列表顺序变化时导致状态错位和不必要的重建 |
| Concurrent Mode 核心 | 可中断渲染、优先级调度、Transition、Suspense |

---

## 常见误区

1. **"setState 是异步的"**：不完全正确。setState 在事件处理程序中表现为"异步"（因为批处理），但在 `setTimeout` 或原生事件回调中是同步的（React 18 后自动批处理已覆盖更多场景）

2. **"useMemo 总是提升性能"**：错误。缓存本身有开销，应先测量再优化

3. **"Context 性能很差"**：Context 本身不慢，问题在于 Context 值变化会导致所有订阅者重新渲染。通过拆分 Context、memo 子组件或使用选择器模式可以解决

4. **"React 的 Diff 是完整的 O(n³) 算法"**：错误。React 使用启发式 O(n) 策略，牺牲了某些极端情况的最优解，换取了实际场景中的高性能

5. **"useEffect 可以用来编排数据流"**：错误。官方明确建议：如果不与外部系统交互，可能不需要 Effect。数据流应通过事件处理和状态更新来处理

---

## 与 Vue 的对比（常见追问）

| 特性 | React | Vue |
|------|-------|-----|
| 响应式 | 手动 state 更新触发渲染 | 自动依赖追踪（Proxy） |
| 渲染粒度 | 组件级别（默认渲染整个子树） | 组件内细粒度（虚拟 DOM + 依赖追踪） |
| 时间分片 | 需要（Fiber 架构） | Vue 3 不需要（编译时优化 + 细粒度更新） |
| Diff 优化 | key + 类型比较 | 编译时静态标记（PatchFlag）+ 静态提升 |
| 状态管理 | useState/useReducer/外部库 | ref/reactive + Pinia |

---

## 参考资料

- [Render and Commit - React 官方文档](https://react.dev/learn/render-and-commit)
- [Queueing a Series of State Updates - React 官方文档](https://react.dev/learn/queueing-a-series-of-state-updates)
- [useState - React API Reference](https://react.dev/reference/react/useState)
- [memo - React API Reference](https://react.dev/reference/react/memo)
- [useTransition - React API Reference](https://react.dev/reference/react/useTransition)
- [React Event Object - React API Reference](https://react.dev/reference/react-dom/components/common#react-event-object)
- [Reconciliation - React 官方文档](https://react.dev/learn/reconciliation)
- [Lists and Keys - React 官方文档](https://react.dev/learn/lists-and-keys)
- [React Fiber 架构解析 - 官方博客（英文）](https://reactjs.org/docs/codebase-overview.html#fiber)
- [Inside Fiber: in-depth overview of the new reconciliation algorithm in React](https://medium.com/react-in-depth/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react-e1c04707ef06)
