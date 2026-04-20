---
created: 2026-04-17
updated: 2026-04-17
tags:
  - react
  - performance
  - optimization
  - frontend
  - profiling
aliases:
  - React Performance Optimization
  - React 性能调优
  - React Profiling
source_type: mixed
source_urls:
  - https://react.dev/learn/render-and-commit
  - https://react.dev/reference/react/memo
  - https://react.dev/reference/react/useMemo
  - https://react.dev/reference/react/useCallback
  - https://react.dev/reference/react/startTransition
  - https://react.dev/learn/fix-performance-problems
  - https://legacy.reactjs.org/docs/optimizing-performance.html
  - https://developer.chrome.com/docs/devtools/evaluate-performance/
  - https://github.com/facebook/react/tree/main/packages/react-devtools
status: verified
---

## 优化思路总览

React 性能优化的核心目标是**减少不必要的渲染**和**降低单次渲染的计算成本**。优化应遵循「测量 → 定位 → 优化 → 验证」的闭环，避免过早优化。

### 优化层级

| 层级 | 关注点 | 典型手段 |
|------|--------|----------|
| **架构层** | 数据流设计、状态管理 | 状态拆分、Colocation、选择器订阅 |
| **组件层** | 渲染范围控制 | `React.memo`、`useMemo`、`useCallback` |
| **渲染层** | 渲染优先级调度 | `startTransition`、`useDeferredValue`、Suspense |
| **DOM 层** | 实际 DOM 操作量 | 虚拟列表、窗口化、CSS 优化 |
| **构建层** | Bundle 体积、加载策略 | Code Splitting、Tree Shaking、懒加载 |

### 优化原则

1. **先测量再优化**：使用 React DevTools Profiler 确认真正的性能瓶颈
2. **优化渲染次数优先于优化单次渲染**：减少不必要的 re-render 收益最大
3. **避免过度优化**：缓存本身有开销（内存 + 依赖比较），简单组件通常不需要 `memo`
4. **关注用户感知**：INP（交互响应性）比 FCP 更直接影响体验

## 渲染机制基础

理解 React 渲染流程是优化的前提。

### 渲染三阶段

```
Trigger（触发）→ Render（渲染计算）→ Commit（提交到 DOM）
```

| 阶段 | 说明 | 可中断？ |
|------|------|----------|
| **Trigger** | 状态变化触发重新渲染 | — |
| **Render** | 调用组件函数，计算 Virtual DOM 差异 | 可中断（React 18 并发模式） |
| **Commit** | 将差异应用到真实 DOM | 不可中断 |

### 什么时候组件会重新渲染？

- 组件自身的 state 发生变化
- 组件的 props 发生变化（引用比较）
- 组件的 context 值发生变化
- 父组件重新渲染时，子组件默认也会重新渲染（即使 props 未变）

> **关键点**：父组件渲染会导致所有子组件渲染，这是 React 的默认行为，也是优化的主要切入点。

## 减少不必要的渲染

### `React.memo` — 组件级记忆化

包裹组件，仅当 props 变化时才重新渲染：

```jsx
const MyComponent = React.memo(function MyComponent({ data }) {
  return <div>{data.name}</div>;
});
```

**工作原理**：对 props 进行浅比较（shallow compare），如果每个 prop 的引用都相同则跳过渲染。

**适用场景**：
- 组件渲染频繁且 props 相对稳定
- 组件渲染成本较高（复杂计算、大量子元素）
- 传递给子组件的 props 经常不变

**不适用场景**：
- props 每次都变化（`memo` 的比较开销反而增加成本）
- 组件本身很简单（比较开销可能超过渲染开销）
- 组件依赖 context（context 变化时 `memo` 无效）

**自定义比较函数**（极少需要）：

```jsx
React.memo(MyComponent, (prevProps, nextProps) => {
  // 返回 true 表示 props 相等，跳过渲染
  return prevProps.id === nextProps.id;
});
```

> **注意**：自定义比较函数每次渲染都会执行，实现不当反而降低性能。

### `useMemo` — 缓存计算结果

缓存昂贵计算的结果，避免每次渲染重新计算：

```jsx
const sortedList = useMemo(() => {
  return list.toSorted((a, b) => a.score - b.score);
}, [list]);
```

**适用场景**：
- 计算成本高（排序、过滤大数据集、复杂数学运算）
- 需要保持引用稳定以配合子组件的 `React.memo`

**常见误区**：
- `useMemo` 不是语义保证，React 可能在内存压力下丢弃缓存并重新计算
- 简单计算（如 `a + b`、`obj.name`）不需要 `useMemo`
- 缓存的开销 = 依赖比较 + 内存占用，需权衡

### `useCallback` — 缓存函数引用

缓存函数定义，保持引用稳定：

```jsx
const handleClick = useCallback((id) => {
  setSelectedId(id);
}, []);
```

**何时需要**：
- 函数作为 props 传递给 `React.memo` 包裹的子组件
- 函数作为其他 Hook 的依赖（如 `useEffect`、`useMemo`）

**何时不需要**：
- 函数仅在组件内部使用，不传递给子组件
- 函数不作为任何 Hook 的依赖

> `useCallback(fn, deps)` 等价于 `useMemo(() => fn, deps)`

### 状态 Colocation（状态就近原则）

将状态放在最需要它的位置，而非一律提升到根组件：

```jsx
// ❌ 不好：FormStatus 不需要 formState，但 formState 变化会导致它重渲染
function App() {
  const [formState, setFormState] = useState({});
  return (
    <>
      <Form state={formState} onChange={setFormState} />
      <FormStatus />
    </>
  );
}

// ✅ 好：状态放在 Form 内部，FormStatus 不受影响
function Form() {
  const [formState, setFormState] = useState({});
  return <form>...</form>;
}
```

**原则**：状态应该尽可能靠近使用它的组件，避免不必要的子树重渲染。

### 状态拆分与选择器订阅

当多个不相关的状态放在同一个 context 或 store 中时，任何状态变化都会导致所有订阅者重渲染。

**拆分 Context**：

```jsx
// 将频繁变化和稳定变化的值分开
const ThemeContext = createContext();    // 很少变化
const DataContext = createContext();     // 经常变化
```

**使用选择器（Selector）**：

```jsx
// Zustand 示例：仅订阅 count，total 变化不会触发此组件渲染
const count = useStore((state) => state.count);

// Redux Toolkit 示例
const count = useSelector((state) => state.counter.value);
```

## 渲染优先级调度（React 18+）

### `startTransition` — 标记非紧急更新

将状态更新标记为低优先级，允许被紧急更新打断：

```jsx
import { startTransition } from 'react';

function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  const handleSearch = (input) => {
    // 紧急更新：输入框立即响应
    setInputValue(input);

    // 过渡更新：搜索结果可以延迟
    startTransition(() => {
      fetchResults(input).then(setResults);
    });
  };
}
```

**特点**：
- Transition 更新可以被用户交互（点击、输入）打断
- 被打断的过渡更新会在紧急更新完成后继续
- 不会显示过期内容（Suspense fallback 会保持显示）

### `useTransition` — 获取过渡状态

```jsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setTab(newTab);
});

{isPending && <TabsSkeleton />}
```

`isPending` 在过渡更新进行中为 `true`，可用于显示加载指示器。

### `useDeferredValue` — 延迟值的更新

```jsx
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query);
  return <SearchResults query={deferredQuery} />;
}
```

**与 `useTransition` 的区别**：
- `useTransition` 包装的是**更新函数**
- `useDeferredValue` 包装的是**值本身**，适用于无法控制更新来源的场景（如第三方库传入的 prop）

### Suspense — 声明式加载状态

```jsx
<Suspense fallback={<Spinner />}>
  <ProfileDetails />
</Suspense>
```

Suspense 允许组件「等待」某些内容（数据、代码）加载完成后再渲染，避免显示不完整的 UI。

## 列表渲染优化

### 虚拟列表（Virtualization / Windowing）

当列表项数量巨大时，仅渲染视口内可见的元素：

```jsx
import { FixedSizeList } from 'react-window';

function MyList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

**常用库**：

| 库 | 特点 |
|----|------|
| [`react-window`](https://github.com/bvaughn/react-window) | 轻量，API 简洁，Dan Abramov 推荐 |
| [`react-virtualized`](https://github.com/bvaughn/react-virtualized) | 功能更全但较重，已不再积极维护 |
| [`@tanstack/virtual`](https://tanstack.com/virtual) | 框架无关，支持 React/Vue/Solid |

**适用场景**：列表项 > 100 条且高度固定或可计算

### `key` 属性的正确使用

```jsx
// ❌ 不好：使用 index 作为 key，列表重排时会导致状态错乱
items.map((item, index) => <Item key={index} data={item} />);

// ✅ 好：使用稳定唯一的 ID
items.map((item) => <Item key={item.id} data={item} />);
```

**key 的作用**：
- 帮助 React 识别哪些元素发生了变化、添加或删除
- 保持组件状态与正确的元素关联
- 影响 diff 算法的效率

**错误使用 index 的后果**：
- 列表插入/删除时，后续所有元素的 key 都变化，导致不必要的重渲染
- 如果列表项有内部状态（如输入框），状态会跟随 index 而非数据

## 代码分割与懒加载

### `React.lazy` + `Suspense`

按需加载组件，减少初始 bundle 体积：

```jsx
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
    </Suspense>
  );
}
```

**适用场景**：
- 路由级代码分割（不同页面按需加载）
- 模态框、弹窗等非首屏内容
- 大型图表库、富文本编辑器等重型依赖

### 路由级代码分割（React Router）

```jsx
import { lazy } from 'react';
import { createBrowserRouter } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      { index: true, element: <lazy(() => import('./pages/Home')) /> },
      { path: 'dashboard', element: <lazy(() => import('./pages/Dashboard')) /> },
      { path: 'settings', element: <lazy(() => import('./pages/Settings')) /> },
    ],
  },
]);
```

## 性能分析工具

### React DevTools Profiler

React 官方浏览器扩展，用于记录和分析组件渲染性能。

**核心功能**：
- **火焰图（Flamegraph）**：展示每个组件的渲染时间和调用层级
- **排序图（Ranked chart）**：按渲染耗时排序，快速定位瓶颈
- **记录交互**：追踪用户交互触发的完整渲染链路
- **为什么渲染（Why did this render?）**：显示 props/state/context 变化详情

**使用方法**：
1. 安装 [React DevTools](https://react.dev/learn/react-developer-tools) 浏览器扩展
2. 打开 DevTools，切换到 Profiler 标签
3. 点击录制按钮，执行操作，停止录制
4. 查看火焰图，找到耗时最长的组件

**关键指标解读**：
- **渲染时间**：组件函数执行 + 子组件渲染的总时间
- **自时间（Self time）**：组件自身渲染时间（不含子组件）
- **渲染原因**：props 变化、state 变化、或父组件导致

### Chrome DevTools Performance 面板

用于分析浏览器级别的性能问题（长任务、布局抖动、内存泄漏）。

**使用场景**：
- 识别 > 50ms 的长任务（long tasks）
- 分析 INP（交互到下一次绘制）
- 检测内存泄漏（Heap Snapshot 对比）
- 查看布局重排（Layout）和绘制（Paint）耗时

### `why-did-you-render` 调试库

第三方库，自动检测可能导致不必要重渲染的模式：

```jsx
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}
```

**检测内容**：
- `React.memo` 组件因引用变化导致的重渲染
- 每次渲染都创建新对象/函数的 props
- Hook 依赖项变化导致的 Effect 重执行

> **注意**：仅用于开发环境，生产环境务必移除。

### Performance API（浏览器原生）

使用 `performance.mark()` 和 `performance.measure()` 手动标记性能关键点：

```jsx
function ExpensiveComponent() {
  useEffect(() => {
    performance.mark('component-mount-start');
    return () => performance.mark('component-unmount');
  }, []);

  // 测量渲染耗时
  performance.mark('render-start');
  // ... 渲染逻辑
  performance.mark('render-end');
  performance.measure('render-duration', 'render-start', 'render-end');
}
```

结合 `PerformanceObserver` 可以监控长任务：

```jsx
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('Long task detected:', entry.duration, 'ms');
  }
});
observer.observe({ entryTypes: ['longtask'] });
```

## 常见性能陷阱

### 内联对象/函数导致子组件重渲染

```jsx
// ❌ 每次渲染都创建新对象，React.memo 失效
<Child style={{ color: 'red' }} onClick={() => handleClick()} />

// ✅ 提升到组件外部或使用 useMemo/useCallback
const defaultStyle = { color: 'red' };
const memoizedClick = useCallback(() => handleClick(), []);
<Child style={defaultStyle} onClick={memoizedClick} />
```

### Context 值频繁变化

```jsx
// ❌ 每次渲染都创建新的 value 对象，所有 Consumer 都会重渲染
<MyContext.Provider value={{ user, theme }}>
  <App />
</MyContext.Provider>

// ✅ 使用 useMemo 保持引用稳定
const contextValue = useMemo(() => ({ user, theme }), [user, theme]);
<MyContext.Provider value={contextValue}>
  <App />
</MyContext.Provider>
```

### Effect 中的同步阻塞操作

```jsx
// ❌ 在 Effect 中执行同步阻塞操作会阻塞 commit 阶段
useLayoutEffect(() => {
  heavySyncComputation(); // 阻塞主线程
}, []);

// ✅ 使用 useEffect 或将计算移出主线程
useEffect(() => {
  heavySyncComputation(); // 在 commit 之后异步执行
}, []);
```

### 过度使用 `useMemo`/`useCallback`

```jsx
// ❌ 不必要的缓存，增加代码复杂度和内存开销
const name = useMemo(() => user.name, [user.name]);
const handleClick = useCallback(() => {}, []);

// ✅ 简单值直接使用
const name = user.name;
const handleClick = () => {};
```

### 渲染大量 DOM 节点

即使使用 `React.memo`，渲染数千个 DOM 节点本身就很慢。应使用虚拟列表减少实际 DOM 数量。

## 优化检查清单

### 开发阶段

- [ ] 使用 React DevTools Profiler 记录典型用户操作
- [ ] 检查是否有不必要的子组件重渲染
- [ ] 确认 `key` 使用稳定唯一的 ID
- [ ] 状态是否放在了最合适的位置（Colocation）
- [ ] Context 值是否使用 `useMemo` 保持稳定引用

### 构建阶段

- [ ] 使用路由级代码分割减少初始 bundle
- [ ] 检查是否有未使用的依赖（Tree Shaking）
- [ ] 大型第三方库是否按需引入
- [ ] 图片是否使用现代格式并正确压缩

### 生产监控

- [ ] 部署后使用 RUM（Real User Monitoring）收集真实性能数据
- [ ] 关注 INP 指标（交互响应性）
- [ ] 设置性能预算和回归告警

## 相关概念

- [[React Fiber 架构详解]] — Fiber 是 React 性能调度的底层基础
- [[React Hooks 详解]] — `useMemo`、`useCallback`、`useTransition` 等性能 Hooks 的 API 文档
- [[前端性能优化]] — 通用前端性能优化方法论，包含 Core Web Vitals 指标体系
- [[渲染管线]] — 浏览器渲染流程，理解 React 渲染如何映射到浏览器渲染

## 参考资料

- [React: Fixing Performance Problems](https://react.dev/learn/fix-performance-problems) — React 18 官方性能优化指南
- [React: React.memo](https://react.dev/reference/react/memo) — 组件记忆化 API 文档
- [React: useMemo](https://react.dev/reference/react/useMemo) — 计算结果缓存 API 文档
- [React: useCallback](https://react.dev/reference/react/useCallback) — 函数引用缓存 API 文档
- [React: startTransition](https://react.dev/reference/react/startTransition) — 过渡更新 API 文档
- [React: useDeferredValue](https://react.dev/reference/react/useDeferredValue) — 延迟值 API 文档
- [React: Render and Commit](https://react.dev/learn/render-and-commit) — React 渲染流程官方文档
- [React Legacy: Optimizing Performance](https://legacy.reactjs.org/docs/optimizing-performance.html) — React 经典性能优化文档（仍有许多适用建议）
- [Chrome DevTools: Evaluate Performance](https://developer.chrome.com/docs/devtools/evaluate-performance/) — Chrome Performance 面板使用指南
- [React DevTools](https://github.com/facebook/react/tree/main/packages/react-devtools) — React 官方开发者工具源码
- [react-window](https://github.com/bvaughn/react-window) — 虚拟列表库，Dan Abramov 推荐
- [@welldone-software/why-did-you-render](https://github.com/welldone-software/why-did-you-render) — 不必要重渲染检测库
