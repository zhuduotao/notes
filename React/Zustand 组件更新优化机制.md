---
created: "2026-04-21"
updated: "2026-04-21"
tags:
  - React
  - Zustand
  - performance
  - optimization
  - state-management
  - selector
  - frontend
aliases:
  - Zustand 性能优化
  - Zustand selector
  - Zustand 重渲染
source_type: official-repo
source_urls:
  - https://github.com/pmndrs/zustand
  - https://zustand.docs.pmnd.rs/
status: verified
---

## Zustand 组件更新机制概述

Zustand 是一个轻量、快速且可扩展的 React 状态管理库。其核心优势之一是**内置选择器订阅机制**，只有当组件订阅的数据实际变化时才会触发重渲染，这与 React Context 的"广播式"更新形成鲜明对比。

### 核心原理

Zustand 的 `useStore` Hook 实现了**选择性订阅**[^1]：
- 每个 `useStore(selector)` 调用创建一个独立的订阅
- selector 函数返回组件需要的部分状态
- **通过严格引用相等比较（`===`）判断新旧结果**
- 只有当 selector 返回值变化时，才会触发组件重渲染

```jsx
import { create } from 'zustand'

const useStore = create((set) => ({
  bears: 0,
  nuts: 0,
  honey: 0,
  increaseBears: () => set((state) => ({ bears: state.bears + 1 })),
}))

// ✅ 只订阅 bears，其他字段变化不会触发此组件渲染
function BearCounter() {
  const bears = useStore((state) => state.bears)
  return <h1>{bears} around here...</h1>
}
```

---

## 选择器机制详解

### 基本用法

```jsx
// 选择单个值（推荐）
const bears = useStore((state) => state.bears)

// 选择多个值（需要 useShallow）
const { nuts, honey } = useStore(useShallow((state) => ({ nuts: state.nuts, honey: state.honey })))
```

**关键点**[^1]：
- 默认使用严格引用相等比较（`===`）
- 对于原始值（字符串、数字、布尔），引用稳定，只有值变化才触发重渲染
- 对于对象和数组，每次 selector 返回新引用都会触发重渲染

### 引用比较的重要性

Zustand 使用 `===` 进行比较[^1]：

```jsx
// ❌ 问题：每次调用都返回新数组引用，组件总是重渲染
const items = useStore((state) => state.items.filter(item => item.active))

// ✅ 正确：返回原始值引用，只有值变化时才重渲染
const count = useStore((state) => state.count)
```

---

## 防止不必要的重渲染

### 方案 1：多次调用选择单个值

**最简单且高效的方式**：多次调用 `useStore`，每次返回单个原始值[^1]：

```jsx
function TodoItem({ id }) {
  // ✅ 多次调用，每次返回原始值引用
  const text = useStore((state) => state.todos[id]?.text)
  const completed = useStore((state) => state.todos[id]?.completed)
  const theme = useStore((state) => state.theme)

  return <div style={{ color: theme }}>{text}</div>
}
```

**原理**：原始值的引用稳定，只有值真正变化时才会触发重渲染。

### 方案 2：使用 useShallow

当需要从 store 返回对象或数组时，使用 `useShallow` 进行浅比较[^1]：

```jsx
import { useShallow } from 'zustand/react/shallow'

function TodoList() {
  // ✅ 使用 useShallow，只有 nuts 或 honey 实际变化时才重渲染
  const { nuts, honey } = useStore(
    useShallow((state) => ({ nuts: state.nuts, honey: state.honey }))
  )

  // 数组选择
  const [nuts, honey] = useStore(
    useShallow((state) => [state.nuts, state.honey])
  )

  // 对象键选择
  const keys = useStore(useShallow((state) => Object.keys(state.treats)))
}
```

**useShallow 行为**[^1]：
- 比较返回对象的每个字段引用，而非整个对象引用
- 比较返回数组的每个元素引用
- 只有当实际字段值变化时才触发重渲染

### 方案 3：自定义 equalityFn

使用 `createWithEqualityFn` 创建支持自定义比较函数的 store[^1]：

```jsx
import { createWithEqualityFn } from 'zustand'
import { shallow } from 'zustand/shallow'

const useStore = createWithEqualityFn((set) => ({
  items: [],
  // ...
}))

// 使用自定义比较函数
const items = useStore(
  (state) => state.items,
  (oldItems, newItems) => shallow(oldItems, newItems)
)
```

---

## Transient Updates（不触发渲染的订阅）

Zustand 提供了一种独特的机制：**订阅状态变化但不触发组件渲染**[^1]。

### 适用场景

- 高频更新的状态（如动画帧计数、鼠标位置）
- 组件需要响应变化但不需要显示状态值
- 性能敏感场景

### 实现方式

使用 `subscribe` API 结合 `useRef` 和 `useEffect`[^1]：

```jsx
const useStore = create((set) => ({
  scratches: 0,
  addScratch: () => set((state) => ({ scratches: state.scratches + 1 })),
}))

function ScratchPad() {
  // ✅ 获取初始状态，但不订阅渲染更新
  const scratchRef = useRef(useStore.getState().scratches)

  // 订阅变化，更新 ref，但不触发渲染
  useEffect(() =>
    useStore.subscribe((state) => (scratchRef.current = state.scratches))
  , [])

  // 在需要时手动使用 ref 值（如动画、事件处理）
  const handleClick = () => {
    console.log('Current scratches:', scratchRef.current)
  }

  return <button onClick={handleClick}>Scratch</button>
}
```

**关键点**[^1]：
- `subscribe` 返回 unsubscribe 函数，`useEffect` 会自动清理
- 状态变化时更新 ref，但组件不重渲染
- 可以产生显著的性能提升

---

## Zustand vs React Context

| 特性 | Zustand | React Context |
|------|---------|---------------|
| 更新机制 | 选择性订阅（仅订阅部分变化触发渲染） | 广播式更新（所有 Consumer 都渲染） |
| 选择器 | 内置 selector 函数 | 需第三方库（use-context-selector） |
| Provider | 不需要 Provider 包裹 | 必须使用 Provider |
| 浅比较 | `useShallow` wrapper | 需手动实现或第三方库 |
| Boilerplate | 极少 | 较多 |
| 状态变化频率 | 支持高频变化 | 高频变化性能问题严重 |
| 跨组件状态 | 集中式 store | 需多个 Context |

### Context 的问题

React Context 的核心问题是：**当 Provider value 变化时，所有 useContext 组件都会重渲染**，无论是否使用了变化的部分。

```jsx
// React Context：所有 Consumer 都重渲染
<UserContext.Provider value={{ user, theme }}>
  <UserProfile />  // 使用 user
  <ThemeSwitch />  // 使用 theme
</UserContext.Provider>

// user 变化时，ThemeSwitch 也会重渲染（即使它不使用 user）
```

```jsx
// Zustand：只有订阅了变化部分的组件重渲染
function UserProfile() {
  const user = useStore((state) => state.user)  // 只订阅 user
}

function ThemeSwitch() {
  const theme = useStore((state) => state.theme)  // 只订阅 theme
}

// user 变化时，ThemeSwitch 不重渲染
```

---

## Store 设计最佳实践

### 状态结构建议

```jsx
// ✅ 推荐：按逻辑功能拆分字段
const useStore = create((set) => ({
  // 用户相关
  user: null,
  isLoggedIn: false,
  // UI 相关
  theme: 'light',
  sidebarOpen: false,
  // 数据相关
  todos: [],
  loading: false,
}))
```

### Actions 与 State 分离

```jsx
// ✅ actions 引用稳定，不影响订阅 state 的组件
const useStore = create((set, get) => ({
  count: 0,
  // Actions（引用稳定）
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}))

// 只订阅 action，不订阅 state
function Controls() {
  const increment = useStore((state) => state.increment)
  return <button onClick={increment}>+</button>
}

// 只订阅 state，不订阅 actions
function Counter() {
  const count = useStore((state) => state.count)
  return <span>{count}</span>
}
```

**关键**：actions 函数引用在 store 创建后保持稳定，只订阅 action 的组件不会因 state 变化而重渲染[^1]。

---

## Slice Pattern（拆分 Store）

对于大型应用，可以将 store 拆分为多个 slice[^1]：

```jsx
// slices/userSlice.js
export const createUserSlice = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
})

// slices/todosSlice.js
export const createTodosSlice = (set) => ({
  todos: [],
  addTodo: (todo) => set((state) => ({ todos: [...state.todos, todo] })),
})

// store.js
import { create } from 'zustand'
import { createUserSlice } from './slices/userSlice'
import { createTodosSlice } from './slices/todosSlice'

export const useStore = create((...a) => ({
  ...createUserSlice(...a),
  ...createTodosSlice(...a),
}))
```

---

## 外部使用 Store

Zustand store 可以在 React 组件外部使用[^1]：

```jsx
const useStore = create((set) => ({ count: 0, increment: () => set(...) }))

// 在组件外部
const count = useStore.getState().count  // 获取当前状态（非响应式）
useStore.setState({ count: 10 })         // 更新状态

// 订阅变化（不触发渲染）
const unsubscribe = useStore.subscribe((state) => {
  console.log('State changed:', state)
})
unsubscribe()  // 取消订阅
```

---

## 常见性能陷阱

### 陷阱 1：返回整个 state

```jsx
// ❌ 订阅整个 state，任何字段变化都会触发重渲染
const state = useStore()
const count = state.count
```

### 陷阱 2：内联 selector 返回新引用

```jsx
// ❌ 每次调用都返回新数组
const activeItems = useStore((state) => state.items.filter(i => i.active))

// ✅ 使用 useShallow 或拆分订阅
const items = useStore((state) => state.items)
const activeItems = items.filter(i => i.active)  // 组件内部计算
```

### 陷阱 3：对象选择器不用 useShallow

```jsx
// ❌ 每次都返回新对象引用
const { a, b } = useStore((state) => ({ a: state.a, b: state.b }))

// ✅ 使用 useShallow
const { a, b } = useStore(useShallow((state) => ({ a: state.a, b: state.b })))
```

### 陷阱 4：不必要的计算在 selector 中执行

```jsx
// ❌ 每次状态变化都重新计算
const total = useStore((state) => state.items.reduce((sum, i) => sum + i.price, 0))

// ✅ 选择原始数据，组件内部计算（或使用 memoized selector）
const items = useStore((state) => state.items)
const total = useMemo(() => items.reduce(...), [items])
```

---

## 中间件与持久化

### Persist 中间件

```jsx
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

const useStore = create(
  persist(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 })),
    }),
    {
      name: 'counter-storage',
      storage: createJSONStorage(() => sessionStorage),
    }
  )
)
```

### Immer 中间件

简化复杂状态更新[^1]：

```jsx
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

const useStore = create(
  immer((set) => ({
    nested: { deep: { value: 0 } },
    updateNested: () => set((state) => {
      state.nested.deep.value += 1  // 直接"修改"
    }),
  }))
)
```

---

## DevTools 集成

Zustand 支持 Redux DevTools[^1]：

```jsx
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

const useStore = create(
  devtools((set) => ({
    count: 0,
    increment: () => set({ count: 1 }, false, 'increment'),
  }))
)
```

---

## 与 Redux 的对比

| 特性 | Zustand | Redux |
|------|---------|-------|
| 选择器 API | `useStore(selector)` | `useSelector(selector)` |
| 浅比较 | `useShallow` wrapper | `shallowEqual` 参数 |
| Memoized selector | 不内置，可选 | 需 Reselect |
| Boilerplate | 极少 | 较多（slice、actions、reducers） |
| Provider | 不需要 | 需要 `<Provider>` |
| 中间件 | 简化版 | 丰富生态 |
| DevTools | Redux DevTools 兼容 | Redux DevTools 原生 |
| 学习曲线 | 低 | 中高 |

---

## 参考资料

[^1]: Zustand GitHub Repository. https://github.com/pmndrs/zustand

[^2]: Zustand 官方文档. https://zustand.docs.pmnd.rs/