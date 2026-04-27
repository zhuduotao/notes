---
created: 2026-04-27
updated: 2026-04-27
tags:
  - React
  - useState
  - Hooks
  - Frontend
  - 原理
  - JavaScript
aliases:
  - useState 原理
  - React useState 机制
  - useState 源码解析
  - useState 实现原理
source_type: official-doc
source_urls:
  - https://react.dev/reference/react/useState
  - https://react.dev/learn/state-a-components-memory
  - https://react.dev/learn/queueing-a-series-of-state-updates
  - https://react.dev/learn/state-as-a-snapshot
  - https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js
status: verified
---

## 概述

`useState` 是 React 16.8 引入的内置 Hook，允许在函数组件中声明状态变量，使函数组件具备跨渲染周期的"记忆"能力。它是 React Hooks 体系中最基础、使用最频繁的 API。

```js
const [state, setState] = useState(initialState);
```

- `state`：当前状态值（首次渲染时等于 `initialState`）
- `setState`：更新函数，接受新值或 updater 函数，调用后会排队一次重新渲染

---

## 核心机制

### 状态快照（Snapshot）特性

React 中的 state 在一次渲染中是**只读的快照**，而非实时可变的引用：

```js
function handleClick() {
  console.log(count);  // 0
  setCount(count + 1); // 请求下次渲染使用 1
  console.log(count);  // 仍然是 0！
}
```

**原理**：`set` 函数不会改变当前执行代码中的 state 变量，它只影响下一次渲染时 `useState` 返回的值。每次渲染，React 都会为该次渲染生成一组全新的状态值，这些值在整个渲染周期内保持不变。

> 参考：[State as a Snapshot - react.dev](https://react.dev/learn/state-as-a-snapshot)

### 批处理（Batching）

React 会等待事件处理程序中**所有代码执行完毕**后再处理状态更新，这称为批处理：

```js
// 点击一次，number 只增加 1（不是 3）
setNumber(number + 1);
setNumber(number + 1);
setNumber(number + 1);
```

**关键行为**：
- React 只在安全的情况下进行批处理
- **不会跨多个独立事件批处理**（每次点击独立处理）
- React 18 后自动批处理覆盖了更多场景（包括 `setTimeout`、原生事件回调等），但行为保持一致

> 参考：[Queueing a Series of State Updates - react.dev](https://react.dev/learn/queueing-a-series-of-state-updates)

### 更新队列（Update Queue）

当需要在同一次事件中多次更新同一个 state 时，使用 **updater 函数**：

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

**规则**：
- **updater 函数**（`n => n + 1`）：添加到队列，基于前一个状态计算
- **其他值**（如 `5`）：添加到队列，忽略已排队的更新（相当于 `"replace with 5"`）

updater 函数在渲染阶段执行，因此**必须是纯函数**，不能包含副作用。

---

## 源码实现原理

### Hook 链表存储结构

在 React 源码中（`ReactFiberHooks.js`），每个组件实例有一个**链表**存储 Hooks：

```js
export type Hook = {
  memoizedState: any,     // 当前状态值
  baseState: any,         // 基础状态（用于处理更新队列）
  baseQueue: Update<any, any> | null,  // 待处理的更新队列
  queue: any,             // 更新队列引用
  next: Hook | null,      // 指向下一个 Hook
};
```

链表挂载在 Fiber 节点的 `memoizedState` 字段上：

```
fiber.memoizedState → hook1 → hook2 → hook3 → null
```

### 调用顺序依赖

`useState` 没有接收任何标识符来区分不同的状态变量。React 依赖 **Hook 的调用顺序**来关联状态：

```
组件首次渲染：
  useState('a') → 链表节点1（memoizedState = 'a'）
  useEffect(...) → 链表节点2
  useState('b') → 链表节点3（memoizedState = 'b'）

重新渲染时，React 按相同顺序遍历链表，恢复对应状态
```

如果在条件语句中调用 Hook，调用顺序变化会导致**状态错位**。这也是 Hooks 规则要求"只在顶层调用"的根本原因。

> 源码参考：[ReactFiberHooks.js - facebook/react](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)

### 双缓冲与 Fiber

React 维护两棵 Fiber 树：
- **current 树**：当前已渲染到屏幕上的树
- **workInProgress 树**：正在构建中的新树

每个 Hook 节点在 current 和 workInProgress 之间通过 `alternate` 指针关联。渲染完成后，workInProgress 树成为新的 current 树。

### 惰性初始化

`useState` 支持传入函数作为初始值，该函数**仅在首次渲染时执行**：

```js
const [todos, setTodos] = useState(() => createInitialTodos());
```

**源码行为**：
- 首次渲染（mount）：React 调用初始化函数，将返回值作为初始状态
- 后续渲染（update）：React 忽略该参数，直接返回已存储的状态
- Strict Mode 下，初始化函数会在开发环境中被调用两次（帮助发现不纯的函数），但只使用其中一次的结果

### `set` 函数的稳定身份

`set` 函数具有稳定的身份（stable identity），不会在重新渲染时变化。因此：
- 通常不需要将其加入 `useEffect` 的依赖数组
- 可以作为 props 传递给子组件而不会导致不必要的重新渲染

---

## 状态不可变性

React 将 state 视为**只读**，更新时必须**替换**而非**修改**现有对象：

```js
// 🚩 错误：直接修改 state 中的对象
form.firstName = 'Taylor';
setForm(form); // React 会忽略此更新（Object.is 比较结果相同）

// ✅ 正确：创建新对象替换
setForm({
  ...form,
  firstName: 'Taylor'
});
```

**比较规则**：React 使用 `Object.is()` 比较新旧状态，如果值相同则跳过重新渲染。

> 参考：[Updating Objects in State - react.dev](https://react.dev/learn/updating-objects-in-state)

---

## 渲染期间调用 `set` 函数

在极少数情况下，可以在组件渲染期间调用 `set` 函数来存储上一次渲染的信息：

```js
function CountLabel({ count }) {
  const [prevCount, setPrevCount] = useState(count);
  const [trend, setTrend] = useState(null);

  if (prevCount !== count) {
    setPrevCount(count);
    setTrend(count > prevCount ? 'increasing' : 'decreasing');
  }

  return (
    <>
      <h1>{count}</h1>
      {trend && <p>The count is {trend}</p>}
    </>
  );
}
```

**限制条件**：
- 必须在条件语句中调用（如 `prevCount !== count`），否则会进入无限渲染循环
- 只能更新**当前正在渲染的组件**的状态
- React 会立即用新状态重新渲染该组件（在渲染子组件之前），子组件不需要渲染两次

---

## 常见误区

### "setState 是异步的"

**不完全正确**。setState 在事件处理程序中表现为"异步"（因为批处理），但本质上是同步请求重新渲染。React 18 后自动批处理已覆盖更多场景（`setTimeout`、Promise 回调等）。

### "useState 返回的 state 是响应式的"

**错误**。React 的 state 不是响应式的（与 Vue 的 `ref` 不同）。state 是渲染快照，更新后不会自动反映到当前执行的代码中。

### "直接修改对象/数组后调用 setState 会生效"

**错误**。React 使用 `Object.is()` 比较，直接修改后引用不变，React 会跳过渲染。必须创建新对象/数组。

### "将函数存入 state 时直接传函数引用"

```js
// 🚩 React 会将其当作 updater 函数调用
const [fn, setFn] = useState(someFunction);

// ✅ 需要用箭头函数包裹
const [fn, setFn] = useState(() => someFunction);
```

因为 React 对函数类型的参数有特殊处理：作为 `initialState` 时视为初始化函数，作为 `setState` 参数时视为 updater 函数。

---

## 最佳实践

### 何时使用多个独立 state 变量

当状态之间**无关联**时，使用多个 `useState`：

```js
const [index, setIndex] = useState(0);
const [showMore, setShowMore] = useState(false);
```

### 何时合并为对象 state

当多个状态**经常一起更新**时，合并为对象并使用展开运算符更新：

```js
const [form, setForm] = useState({
  firstName: '',
  lastName: '',
  email: ''
});

setForm({ ...form, firstName: 'Taylor' });
```

### 何时使用 `useReducer`

当状态更新逻辑复杂、或下一个状态依赖于前一个状态的多个字段时，考虑 `useReducer`：

```js
const [state, dispatch] = useReducer(reducer, initialState);
```

### 避免不必要的重新渲染

- 使用 `Object.is()` 比较，如果值相同则跳过渲染
- 对于频繁变化的复杂对象，考虑拆分 state 或使用 `useReducer`
- 配合 `React.memo` 和 `useCallback` 优化子组件

---

## 与 Vue 响应式的对比

| 特性 | React `useState` | Vue `ref`/`reactive` |
|------|------------------|---------------------|
| 更新方式 | 手动调用 `setState` | 自动追踪（Proxy） |
| 读取方式 | 直接读取变量（快照值） | 通过 `.value` 或直接读取（自动解包） |
| 触发渲染 | 调用 setter 后排队渲染 | 自动触发依赖组件更新 |
| 同一事件多次更新 | 需要 updater 函数 | 自动基于最新值 |
| 实现机制 | 链表 + 调用顺序 | Proxy + 依赖收集 |

---

## 参考资料

- [useState API Reference - React 官方文档](https://react.dev/reference/react/useState)
- [State: A Component's Memory - React 官方文档](https://react.dev/learn/state-a-components-memory)
- [Queueing a Series of State Updates - React 官方文档](https://react.dev/learn/queueing-a-series-of-state-updates)
- [State as a Snapshot - React 官方文档](https://react.dev/learn/state-as-a-snapshot)
- [ReactFiberHooks.js 源码 - facebook/react](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)
- [React Hooks: Not Magic, Just Arrays - Ryan Yardley](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)
