---
created: 2026-04-17
updated: 2026-04-17
tags:
  - React
  - Hooks
  - Frontend
  - JavaScript
aliases:
  - React Hooks
  - React 钩子
source_type: official-doc
source_urls:
  - https://react.dev/reference/react/hooks
  - https://react.dev/learn/state-a-components-memory
  - https://react.dev/learn/synchronizing-with-effects
  - https://react.dev/learn/reusing-logic-with-custom-hooks
status: verified
---

## 什么是 Hooks

Hooks 是 React 16.8 引入的特性，允许在函数组件中使用状态（state）和其他 React 特性，而无需编写 class 组件。

**核心价值**：
- 让函数组件具备状态管理能力
- 提供复用状态逻辑的机制（自定义 Hooks）
- 避免 class 组件中 `this` 绑定、生命周期拆分等痛点

## Hooks 使用规则

1. **只在顶层调用**：不要在循环、条件或嵌套函数中调用 Hook，确保每次渲染时调用顺序一致
2. **只在 React 函数中调用**：在函数组件或自定义 Hook 中调用，不要在普通 JavaScript 函数中调用

## State Hooks

### `useState`

声明一个状态变量，可直接更新。

```js
const [state, setState] = useState(initialState);
```

- `state`：当前状态值
- `setState`：更新函数，接受新值或 updater 函数 `(prev => newValue)`
- 更新会触发组件重新渲染
- **惰性初始化**：`useState(() => computeExpensiveValue())` 仅在首次渲染执行

### `useReducer`

适用于状态更新逻辑较复杂的场景，将更新逻辑抽离到 reducer 函数中。

```js
const [state, dispatch] = useReducer(reducer, initialArg, init?);
```

- `reducer`：`(state, action) => newState` 纯函数
- `dispatch`：触发状态更新
- 与 `useState` 等价，但更适合多子操作或复杂状态结构

## Context Hooks

### `useContext`

读取并订阅一个 Context，避免层层传递 props（prop drilling）。

```js
const value = useContext(MyContext);
```

- 返回最近的 `<MyContext.Provider>` 提供的值
- Context 值变化时，所有订阅该 Context 的组件都会重新渲染
- **性能注意**：频繁变化的值不适合放 Context，可考虑拆分 Context 或结合 `useMemo`

## Ref Hooks

### `useRef`

声明一个 ref 对象，`current` 属性可持有任何值。

```js
const ref = useRef(initialValue);
```

- 修改 `ref.current` **不会**触发重新渲染
- 常用于：持有 DOM 节点、定时器 ID、上一次渲染的值
- 是 React 范式之外的"逃生舱"，适合与非 React 系统交互

### `useImperativeHandle`

自定义组件暴露给父组件的 ref 实例。极少使用。

```js
useImperativeHandle(ref, createHandle, deps?);
```

## Effect Hooks

### `useEffect`

让组件连接并同步到外部系统（DOM 操作、订阅、网络请求等）。

```js
useEffect(() => {
  // 副作用逻辑
  return () => { /* 清理函数 */ };
}, [dependencies]);
```

- 依赖数组决定何时重新执行
- 返回的清理函数在组件卸载或依赖变化时执行
- **重要原则**：不要用 Effect 编排应用的数据流；如果不与外部系统交互，可能不需要 Effect

### `useLayoutEffect`

与 `useEffect` 签名相同，但在浏览器重绘之前同步执行。适用于测量 DOM 布局。

### `useInsertionEffect`

在 React 修改 DOM 之前同步执行。CSS-in-JS 库用于动态注入样式。

### `useEffectEvent`（实验性）

创建非响应式的事件函数，可在 Effect 中调用而不触发重新执行。

## Performance Hooks

### `useMemo`

缓存昂贵计算的结果。

```js
const cachedValue = useMemo(() => computeExpensive(a, b), [a, b]);
```

- 仅当依赖变化时重新计算
- **不要过度使用**：React 本身已做了大量优化，应先测量再优化

### `useCallback`

缓存函数定义，常用于传递给子组件。

```js
const cachedFn = useCallback(fn, [dependencies]);
```

- 等价于 `useMemo(() => fn, [dependencies])`
- 配合 `React.memo` 使用才有意义

### `useTransition`

将状态更新标记为非阻塞，允许其他更新打断。

```js
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setQuery(input); // 非阻塞更新
});
```

- `isPending`：过渡状态是否在进行中
- 适用于：搜索结果过滤、大数据列表渲染等场景

### `useDeferredValue`

延迟更新 UI 的非关键部分，让其他部分先更新。

```js
const deferredQuery = useDeferredValue(query);
```

- 与 `useTransition` 类似，但作用于值而非函数
- 适用于：输入框联动大量数据渲染

## 其他 Hooks

| Hook | 用途 |
|------|------|
| `useDebugValue` | 自定义 React DevTools 中显示的标签 |
| `useId` | 生成唯一 ID，常用于无障碍（a11y）场景 |
| `useSyncExternalStore` | 订阅外部 store，用于状态管理库集成 |
| `useActionState` | 管理 action 的状态（React 19+） |

## 自定义 Hooks

自定义 Hook 是以 `use` 开头的函数，可复用状态逻辑。

```js
function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }
    ChatAPI.subscribeToFriendStatus(friendID, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendID, handleStatusChange);
    };
  }, [friendID]);

  return isOnline;
}
```

**设计原则**：
- 命名必须以 `use` 开头（React 依赖此约定应用 Hooks 规则）
- 每次调用都是独立的状态实例
- 可组合多个内置 Hook 或其他自定义 Hook

## 常见问题与注意事项

### 为什么不能在条件语句中调用 Hook？

React 依赖 Hook 的调用顺序来关联状态。如果顺序变化，状态会错位。

### `useEffect` 的依赖数组何时为空？

- `[]`：仅在挂载和卸载时执行
- 省略：每次渲染后都执行
- `[a, b]`：仅当 `a` 或 `b` 变化时执行

### 何时使用 `useRef` 而非 `useState`？

- 值变化**不需要**触发重新渲染时用 `useRef`
- 值变化**需要**更新 UI 时用 `useState`

### `useMemo` / `useCallback` 的性能陷阱

- 缓存本身有开销（内存 + 依赖比较）
- 应先通过 React DevTools Profiler 确认瓶颈
- 对于简单计算或稳定组件，通常不需要

## 参考资料

- [React 官方文档 - Built-in Hooks](https://react.dev/reference/react/hooks)
- [React 官方文档 - State: A Component's Memory](https://react.dev/learn/state-a-components-memory)
- [React 官方文档 - Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [React 官方文档 - Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
