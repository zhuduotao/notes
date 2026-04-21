---
created: "2026-04-21"
updated: "2026-04-21"
tags:
  - React
  - Redux
  - Redux Toolkit
  - performance
  - optimization
  - useSelector
  - Reselect
  - frontend
aliases:
  - Redux 性能优化
  - useSelector 优化
  - Redux 选择器
source_type: official-doc
source_urls:
  - https://react-redux.js.org/api/hooks
  - https://redux.js.org/usage/deriving-data-selectors
  - https://redux-toolkit.js.org/tutorials/quick-start
  - https://github.com/reduxjs/reselect
status: verified
---

## Redux 组件更新机制概述

Redux 通过 React-Redux 库将 Redux store 与 React 组件连接。当 store 状态变化时，只有真正订阅了变化数据的组件才会重新渲染，这是 Redux 相比 React Context 的核心优势之一。

### 核心原理

React-Redux 的 `useSelector` Hook 实现了**选择性订阅**机制[^1]：
- 每个 `useSelector` 调用创建一个独立的订阅
- 当 action 被 dispatch 时，`useSelector` 会重新执行 selector 函数
- **通过引用比较（`===`）判断新旧结果是否相同**
- 只有当选择结果不同时，才会强制组件重新渲染

这与 React Context 的行为不同：Context value 变化时，所有 `useContext` 组件都会重渲染，无论是否使用了变化的部分。

---

## useSelector 的选择器机制

### 基本用法

```jsx
import { useSelector } from 'react-redux'

function Counter() {
  const count = useSelector((state) => state.counter.value)
  return <div>{count}</div>
}
```

**关键点**[^1]：
- selector 函数接收完整的 Redux root state 作为参数
- selector 应该是纯函数，因为它可能在任意时间点多次执行
- 默认使用严格引用相等比较（`===`），而非浅比较

### 引用比较的重要性

`useSelector` 默认使用 `===` 引用比较[^1]。这意味着：

```jsx
// ❌ 问题：每次调用都返回新数组引用，组件总是重渲染
const completedTodos = useSelector((state) =>
  state.todos.filter((todo) => todo.completed)
)

// ✅ 正确：返回原始值引用，只有值真正变化时才重渲染
const count = useSelector((state) => state.counter.value)
```

**与 `connect()` 的区别**[^1]：
- `connect()` 使用浅比较（shallow equality）检查 `mapState` 返回对象的各个字段
- `useSelector()` 使用严格引用比较（`===`）
- 这导致 `useSelector` 对返回对象的场景需要额外处理

---

## 防止不必要的重渲染

### 方案 1：多次调用选择单个值

最简单且高效的方式：**多次调用 `useSelector`，每次返回单个值**[^1]：

```jsx
function TodoListItem({ id }) {
  // ✅ 多次调用，每次返回原始值引用
  const todo = useSelector((state) => state.todos[id])
  const isEditing = useSelector((state) => state.editingId === id)
  const theme = useSelector((state) => state.theme)

  return <div style={{ color: theme }}>{todo.text}</div>
}
```

**原理**：原始值（字符串、数字、布尔）的引用稳定，只有值变化时才会触发重渲染。

### 方案 2：使用 shallowEqual

当需要返回对象时，使用 `shallowEqual` 作为第二个参数[^1]：

```jsx
import { shallowEqual, useSelector } from 'react-redux'

function TodoList() {
  // ✅ 使用 shallowEqual 比较对象字段
  const { todos, loading } = useSelector(
    (state) => ({ todos: state.todos, loading: state.loading }),
    shallowEqual
  )
}
```

`shallowEqual` 会比较返回对象的每个字段引用，而非整个对象引用。

### 方案 3：使用 Reselect 创建 memoized selector

Reselect 是 Redux 官方推荐的选择器库，用于创建**缓存（memoized）选择器**[^2]：

```jsx
import { createSelector } from '@reduxjs/toolkit' // RTK 内置

// 定义 memoized selector
const selectCompletedTodos = createSelector(
  [(state) => state.todos],
  (todos) => todos.filter((todo) => todo.completed)
)

function CompletedTodoList() {
  // ✅ 只有当 todos 数组引用变化时才重新计算
  const completedTodos = useSelector(selectCompletedTodos)
}
```

**Reselect 核心机制**[^2]：
- 输入 selector（input selectors）：提取原始数据
- 输出 selector（result function）：执行转换逻辑
- 只有当输入 selector 的结果变化时，才会重新执行输出 selector
- 输出 selector 缓存大小默认为 1

---

## Reselect 深入理解

### createSelector 结构

```jsx
const selectA = (state) => state.a
const selectB = (state) => state.b

const selectSum = createSelector(
  [selectA, selectB], // input selectors
  (a, b) => a + b     // output selector (result function)
)
```

**执行流程**[^2]：
1. 调用所有 input selectors，获取各自结果
2. 使用 `===` 比较每个 input 结果与上次缓存值
3. 如果所有 input 结果都相同 → 返回缓存的 output 结果
4. 如果任意 input 结果不同 → 重新执行 output selector，缓存新结果

### 缓存限制

**重要**：`createSelector` 默认缓存大小为 1[^2]：

```jsx
const a = selectSum(state, 1) // 第一次调用，执行计算
const b = selectSum(state, 1) // 相同输入，返回缓存
const c = selectSum(state, 2) // 不同输入，执行计算（覆盖缓存）
const d = selectSum(state, 1) // 与上次输入不同，执行计算（非缓存命中）
```

### 常见错误

**错误 1：output selector 只返回输入**

```jsx
// ❌ 无法正确缓存，且没有意义
const brokenSelector = createSelector(
  (state) => state.todos,
  (todos) => todos // output 不应该直接返回输入
)
```

**错误 2：使用 `state => state` 作为输入**

```jsx
// ❌ 每次 state 变化都会触发重新计算
const brokenSelector = createSelector(
  (state) => state, // 整个 state 作为输入
  (state) => state.todos.filter(...)
)
```

---

## 选择器工厂模式

当多个组件实例需要使用相同的 selector 但参数不同时，需要为每个实例创建独立的 selector[^2]：

### 函数组件：使用 useMemo

```jsx
const makeSelectItemsByCategory = () =>
  createSelector(
    [(state) => state.items, (state, category) => category],
    (items, category) => items.filter((item) => item.category === category)
  )

function CategoryList({ category }) {
  // ✅ 每个组件实例创建独立的 selector
  const selectItemsByCategory = useMemo(makeSelectItemsByCategory, [])
  const items = useSelector((state) => selectItemsByCategory(state, category))
}
```

### 带参数的 selector

```jsx
const selectTodoById = createSelector(
  [(state) => state.todos, (state, id) => id],
  (todos, id) => todos[id]
)

// 在组件中使用
function TodoItem({ todoId }) {
  const todo = useSelector((state) => selectTodoById(state, todoId))
}
```

---

## 开发模式检查

React-Redux v8.1.0+ 在开发模式下执行额外检查[^1]：

### 选择器结果稳定性检查

检测 selector 是否在相同输入下返回不同引用：

```jsx
// ❌ 会触发警告：每次都返回新对象
const { count, user } = useSelector((state) => ({
  count: state.count,
  user: state.user
}))
```

**配置方式**：

```jsx
// 全局配置
<Provider store={store} stabilityCheck="always">

// 单个 selector 配置
const count = useSelector(selectCount, {
  devModeChecks: { stabilityCheck: 'never' }
})
```

### Identity Function 检查

检测 selector 是否返回整个 root state（几乎总是错误）[^1]：

```jsx
// ❌ 错误：订阅整个 state，任何变化都会触发重渲染
const { count, user } = useSelector((state) => state)

// ✅ 正确：只订阅需要的部分
const count = useSelector((state) => state.count.value)
const user = useSelector((state) => state.auth.currentUser)
```

---

## useSelector 与 connect 的对比

| 特性 | useSelector | connect() |
|------|-------------|-----------|
| 比较方式 | `===` 引用比较 | shallow equality 浅比较 |
| 返回值类型 | 任意值 | 必须是对象 |
| props 访问 | 通过闭包或 curried selector | `ownProps` 参数 |
| 父组件重渲染影响 | 无法阻止父组件导致的重渲染 | 可阻止 |
| 类型支持 | 更好的 TypeScript 支持 | 类型推断复杂 |
| 订阅层级 | 无嵌套层级 | 有嵌套 Subscription 层级 |

### 阻止父组件导致的重渲染

`useSelector` 无法阻止父组件重渲染导致的子组件重渲染[^1]。解决方案：

```jsx
const CounterComponent = ({ name }) => {
  const counter = useSelector((state) => state.counter)
  return <div>{name}: {counter}</div>
}

// ✅ 使用 React.memo 包裹
export const MemoizedCounterComponent = React.memo(CounterComponent)
```

---

## 性能陷阱与最佳实践

### 常见陷阱

**陷阱 1：内联 selector 返回新引用**

```jsx
// ❌ 每次调用都返回新数组
const items = useSelector((state) => state.items.filter(...))

// ✅ 使用 Reselect
const selectFilteredItems = createSelector(...)
const items = useSelector(selectFilteredItems)
```

**陷阱 2：昂贵计算在 selector 中执行**

```jsx
// ❌ 每次 state 变化都执行昂贵计算
const data = useSelector((state) => {
  const filtered = expensiveFilter(state.data)
  const sorted = expensiveSort(filtered)
  return expensiveTransform(sorted)
})

// ✅ 使用 Reselect 缓存
const selectProcessedData = createSelector(
  [(state) => state.data],
  (data) => expensiveFilter(data) |> expensiveSort |> expensiveTransform
)
```

**陷阱 3：订阅过多数据**

```jsx
// ❌ 订阅整个 todos slice
const todos = useSelector((state) => state.todos)
const count = todos.length // 可以计算，但 todos 变化会触发重渲染

// ✅ 只订阅真正需要的值
const todos = useSelector((state) => state.todos.items)
```

### 最佳实践

1. **保持 selector 简单**[^2]：input selector 只提取数据，output selector 执行转换
2. **定义 selector 与 reducer 同位置**[^2]：便于维护状态结构
3. **不要过度 memoize**[^2]：简单查找不需要 Reselect
4. **优先多次调用**：返回单个原始值而非对象
5. **使用 TypeScript**：`useSelector` 与 TypeScript 配合更好

### 判断是否需要 memoize[^2]

```jsx
// ❌ 不需要 memoize：直接返回原始值引用
const selectTodos = (state) => state.todos
const selectNestedValue = (state) => state.some.deeply.nested.field

// 🤔 可能需要：计算结果，但返回原始值
const selectTotal = (state) => state.items.reduce((sum, item) => sum + item.total, 0)

// ✅ 需要 memoize：返回新引用
const selectDescriptions = (state) => state.todos.map((todo) => todo.text)
```

---

## Stale Props 与 Zombie Children 问题

### 问题描述[^1]

这两个边缘情况在使用 hooks 时可能出现：

**Stale Props**：selector 依赖组件 props，但 props 还未更新时 selector 已执行。

**Zombie Children**：子组件先订阅 store，父组件因数据删除不再渲染子组件，但子组件的订阅先执行并尝试访问已删除的数据。

### 解决方案[^1]

1. **不依赖 props**：避免在 selector 中使用 props 提取可能被删除的数据
2. **防御性编写**：先检查数据是否存在

```jsx
// ✅ 防御性 selector
const selectTodoText = (state, id) => {
  const todo = state.todos[id]
  return todo?.text ?? 'Not found'
}
```

3. **使用 connect 包裹父组件**：connect 提供嵌套订阅层级，确保父组件先更新

---

## Redux Toolkit 集成

Redux Toolkit (RTK) 内置了 Reselect 的 `createSelector`[^3]：

```jsx
import { createSlice, createSelector } from '@reduxjs/toolkit'

const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    todoAdded: (state, action) => state.push(action.payload)
  }
})

// 直接在 slice 文件中定义 selector
export const selectTodos = (state) => state.todos
export const selectCompletedTodos = createSelector(
  [selectTodos],
  (todos) => todos.filter((todo) => todo.completed)
)

export default todosSlice.reducer
```

---

## 替代选择器库

### proxy-memoize[^2]

使用 ES2015 Proxy 追踪实际访问的字段，提供更精准的缓存：

```jsx
import { memoize } from 'proxy-memoize'

const selectTodoDescriptions = memoize(
  (state) => state.todos.map((todo) => todo.text)
)
```

**优势**：只有实际访问的字段变化才触发重新计算，而非整个数组变化。

### re-reselect[^2]

提供基于 key 的多缓存实例管理：

```jsx
import { createCachedSelector } from 're-reselect'

const selectItemsByCategory = createCachedSelector(
  [(state) => state.items, (state, category) => category],
  (items, category) => items.filter(...)
)(
  // key selector：按 category 分缓存
  (_state, category) => category
)
```

---

## 与 Zustand 的对比

| 特性 | Redux | Zustand |
|------|-------|---------|
| 选择器 API | `useSelector(selector)` | `useStore(selector)` |
| 浅比较支持 | `shallowEqual` 参数 | `useShallow` wrapper |
| Memoized selector | 需 Reselect | 内置可选 |
| Boilerplate | 较多（slice、actions） | 极少 |
| DevTools | Redux DevTools | Redux DevTools 兼容 |
| 中间件生态 | 丰富 | 简化版 |

---

## 参考资料

[^1]: React-Redux 官方文档 - Hooks API. https://react-redux.js.org/api/hooks

[^2]: Redux 官方文档 - Deriving Data with Selectors. https://redux.js.org/usage/deriving-data-selectors

[^3]: Redux Toolkit 官方文档 - Quick Start. https://redux-toolkit.js.org/tutorials/quick-start

[^4]: Reselect GitHub Repository. https://github.com/reduxjs/reselect