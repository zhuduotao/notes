---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - React
  - Context
  - performance
  - optimization
  - useContext
  - memo
  - frontend
aliases:
  - Context Performance
  - useContext optimization
  - Context 性能
source_type: official-doc
source_urls:
  - 'https://react.dev/reference/react/useContext'
  - 'https://react.dev/reference/react/memo'
  - 'https://react.dev/learn/scaling-up-with-reducer-and-context'
  - 'https://react.dev/learn/passing-data-deeply-with-context'
  - 'https://github.com/dai-shi/use-context-selector'
  - 'https://legacy.reactjs.org/blog/2018/03/29/react-v-16-3.html'
  - 'https://react.dev/blog/2022/03/29/react-v18'
  - 'https://react.dev/blog/2024/04/25/react-19'
  - 'https://github.com/reactjs/rfcs/pull/119'
status: verified
---
## Context 性能问题的根源

React Context 的核心性能问题是：**当 Provider 的 `value` 发生变化时，所有消费该 Context 的组件（通过 `useContext`）都会重新渲染**，无论它们是否实际使用了变化的部分[^1]。

### 渲染触发机制

React 通过 `Object.is` 比较 Provider 的新旧 `value`[^1]：
- 如果 `value` 引用变化 → 所有 `useContext` 的组件重渲染
- `React.memo` 无法阻止 Context 变化导致的重渲染[^2]

```jsx
const ThemeContext = createContext(null);

function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext value={theme}>
      <Greeting name="Taylor" />
    </ThemeContext>
  );
}

// 即使 Greeting 被 memo 包裹，theme 变化时仍会重渲染
const Greeting = memo(function Greeting({ name }) {
  const theme = useContext(ThemeContext); // 订阅 Context
  return <h3 className={theme}>Hello, {name}!</h3>;
});
```

**关键结论**[^2]：`React.memo` 仅控制 props 变化时的重渲染，无法阻止 Context 变化导致的重渲染。

---

## 性能问题场景

### 场景 1：内联对象作为 Context value

```jsx
function App() {
  const [user, setUser] = useState(null);

  // ❌ 每次渲染都创建新对象，所有 Consumer 都重渲染
  return (
    <UserContext value={{ user, setUser }}>
      <AppContent />
    </UserContext>
  );
}
```

### 场景 2：多个不相关状态放在同一 Context

```jsx
// ❌ theme 变化会触发 user 相关组件重渲染
<AppContext value={{ theme, user, notifications }}>
  <ThemeProvider />
  <UserProfile />
  <NotificationList />
</AppContext>
```

### 场景 3：频繁更新的状态

```jsx
// ❌ 每次输入都触发所有订阅者重渲染
<InputContext value={inputValue}>
  <InputField />
  <Suggestions />
  <ResultCount />
</InputContext>
```

---

## 优化方案

### 方案 1：使用 useMemo 保持引用稳定

当 Context value 是对象或函数时，使用 `useMemo` 保持引用稳定[^1]：

```jsx
function App() {
  const [currentUser, setCurrentUser] = useState(null);

  const login = useCallback((response) => {
    setCurrentUser(response.user);
  }, []);

  // ✅ 仅当 currentUser 或 login 变化时才创建新对象
  const contextValue = useMemo(() => ({
    currentUser,
    login
  }), [currentUser, login]);

  return (
    <AuthContext value={contextValue}>
      <Page />
    </AuthContext>
  );
}
```

**适用场景**：
- Context value 包含多个属性
- 部分属性变化频繁，但整体引用变化较少

---

### 方案 2：拆分 Context

将频繁变化和稳定变化的值拆分到不同 Context[^1][^3]：

```jsx
// ✅ 拆分为两个独立 Context
const ThemeContext = createContext('light'); // 很少变化
const UserContext = createContext(null);     // 可能变化

function App() {
  const [theme, setTheme] = useState('light');
  const [user, setUser] = useState(null);

  return (
    <ThemeContext value={theme}>
      <UserContext value={user}>
        <Layout />
        <UserProfile />
      </UserContext>
    </ThemeContext>
  );
}

// Layout 只订阅 theme，user 变化不会触发它重渲染
function Layout() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>...</div>;
}
```

**原则**：按变化频率拆分，高频变化的值独立成 Context。

---

### 方案 3：拆分 State Context 与 Dispatch Context

将状态和更新函数拆分为两个 Context[^3]：

```jsx
const TasksContext = createContext(null);
const TasksDispatchContext = createContext(null);

function TasksProvider({ children }) {
  const [tasks, dispatch] = useReducer(tasksReducer, initialTasks);

  return (
    <TasksContext value={tasks}>
      <TasksDispatchContext value={dispatch}>
        {children}
      </TasksDispatchContext>
    </TasksContext>
  );
}

// 只需要 dispatch 的组件不会因 tasks 变化而重渲染
function AddTask() {
  const dispatch = useContext(TasksDispatchContext); // dispatch 引用稳定
  return <button onClick={() => dispatch({ type: 'add' })}>Add</button>;
}
```

**原理**：`dispatch` 函数引用在组件生命周期内保持稳定（`useReducer` 保证）[^3]。

---

### 方案 4：将 Context 消费者拆分为 memo 子组件

在父组件读取 Context，将值作为 props 传递给 memo 子组件[^2]：

```jsx
function Parent() {
  const theme = useContext(ThemeContext);
  return <MemoizedChild theme={theme} />;
}

const MemoizedChild = memo(function Child({ theme }) {
  // 只有 theme 实际变化时才重渲染
  return <div className={theme}>...</div>;
});
```

**适用场景**：Context 中只有部分值需要传递给子组件。

---

### 方案 5：使用选择器模式（第三方库）

当 Context 包含大型对象，组件只需要其中一部分时，使用选择器库[^4]：

#### use-context-selector

```jsx
import { createContext, useContextSelector } from 'use-context-selector';

const PersonContext = createContext(null);

function FirstName() {
  // 仅订阅 firstName，其他字段变化不触发重渲染
  const firstName = useContextSelector(PersonContext, (v) => v.firstName);
  return <span>{firstName}</span>;
}

function FamilyName() {
  const familyName = useContextSelector(PersonContext, (v) => v.familyName);
  return <span>{familyName}</span>;
}
```

**原理**[^4]：通过 selector 函数选择特定字段，仅当选择结果变化时才触发重渲染。

**限制**[^4]：
- 必须使用库提供的 `createContext`，不能用 React 原生 `createContext`
- Provider 的 children 必须在 Provider 外部创建或用 `React.memo` 包裹
- 不支持 Class 组件和 Context Consumer

**适用场景**：
- Context 包含大型状态对象
- 组件只依赖其中少数字段
- 需要精细化控制重渲染

---

### 方案 6：状态下沉（State Colocation）

将状态放在最需要它的位置，而非提升到根组件[^5]：

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

// ✅ 好：状态放在 Form 内部
function Form() {
  const [formState, setFormState] = useState({});
  return <form>...</form>;
}
```

---

## 方案对比

| 方案 | 适用场景 | 实现复杂度 | 性能提升 |
|------|---------|-----------|---------|
| useMemo 稳定引用 | 对象/函数作为 value | 低 | 中 |
| 拆分 Context | 多个独立状态 | 低 | 高 |
| State + Dispatch 拆分 | Reducer + Context 组合 | 低 | 高 |
| memo 子组件 | Context 值传递给特定子组件 | 低 | 中 |
| use-context-selector | 大型状态对象、精细化订阅 | 中 | 最高 |
| 状态下沉 | 状态使用范围有限 | 低 | 高 |

---

## React.memo 与 Context 的关系

### memo 无法阻止 Context 重渲染

`React.memo` 仅阻止 props 未变化时的父组件传递重渲染，**不影响 Context 变化导致的重渲染**[^2]：

```jsx
const Greeting = memo(function Greeting({ name }) {
  const theme = useContext(ThemeContext);
  return <h3 className={theme}>Hello, {name}!</h3>;
});

// theme 变化时 Greeting 会重渲染，即使 name props 未变
```

### 如何让 memo 组件只响应部分 Context 值

**方法**：在父组件读取 Context，将值作为 props 传递给 memo 子组件[^2]：

```jsx
function GreetingWrapper({ name }) {
  const theme = useContext(ThemeContext);
  return <Greeting name={name} theme={theme} />;
}

const Greeting = memo(function Greeting({ name, theme }) {
  return <h3 className={theme}>Hello, {name}!</h3>;
});

// theme 实际变化时才重渲染 Greeting
```

---

## 与状态管理库的对比

当 Context 性能优化需求复杂时，状态管理库提供更完善的方案：

| 库 | 选择器支持 | 性能优化 |
|----|-----------|---------|
| Zustand | 内置 `useStore(selector)` | 仅 selector 结果变化时重渲染 |
| Redux Toolkit | `useSelector(selector)` | 仅 selector 结果变化时重渲染 |
| Jotai | 原子化状态 | 每个 atom 独立订阅 |
| Valtio | Proxy 状态 | 自动追踪依赖 |
| React Context + use-context-selector | 需第三方库 | 手动配置选择器 |

**建议**[^4]：
- 简单场景（少量状态、低频更新） → React Context + 拆分方案
- 复杂场景（大型状态、高频更新、精细化订阅） → Zustand / Redux Toolkit

---

## 最佳实践

### 使用前评估

1. **测量性能瓶颈**：使用 React DevTools Profiler 确认 Context 是否真的是重渲染源头[^5]
2. **评估状态变化频率**：高频变化的状态不适合直接放在 Context
3. **评估消费组件数量**：少量消费者时优化收益有限

### 设计原则

1. **按变化频率拆分**：频繁变化的值独立 Context
2. **最小化 Context value**：只包含必要值，避免大型对象
3. **State + Dispatch 分离**：dispatch 函数引用稳定，可独立 Context
4. **组合而非替代**：Context 不是状态管理库的替代品，复杂场景使用 Zustand/Redux

### 避免的陷阱

1. **不要过度拆分 Context**：增加代码复杂度，维护成本上升
2. **不要滥用 useMemo/useCallback**：缓存本身有开销，简单场景不需要
3. **不要过早优化**：先测量，确认瓶颈后再优化

---

## 限制与注意事项

### useMemo 的语义不确定性

`useMemo` 不是语义保证，React 可能在内存压力下丢弃缓存[^5]。但这通常不影响 Context value，因为 Provider 的 children 通常在组件外部创建。

### React Compiler 的影响

React Compiler 会自动应用类似 `memo` 的优化[^2]，但 **Compiler 不会自动解决 Context 性能问题**，仍需要手动拆分或使用选择器。

### use-context-selector 的并发模式兼容

`use-context-selector` v1.3+ 使用 `useSyncExternalStore` 模拟 Context 行为[^4]，与 React 18 并发模式兼容，但需注意：
- Provider children 必须 memoized
- 不支持 Class 组件

---

## React 各版本中 Context 的变化

### React 16.3（2018年3月）：新 Context API

React 16.3 引入了全新的官方 Context API，取代之前的实验性 legacy context API[^6]：

**之前的问题**：
- Legacy context API 存在固有的 API 问题
- 官方一直不推荐使用，并计划替换

**新 API 特性**[^6]：
- 更高效的实现
- 支持静态类型检查（TypeScript）
- 支持深度更新
- 使用 `React.createContext(defaultValue)` 创建 Context
- 使用 `<Context.Provider value={...}>` 提供值
- 使用 `<Context.Consumer>` 或 `useContext()` 消费值

```jsx
// React 16.3 新 API
const ThemeContext = React.createContext('light');

class ThemeProvider extends React.Component {
  state = { theme: 'light' };
  render() {
    return (
      <ThemeContext.Provider value={this.state.theme}>
        {this.props.children}
      </ThemeContext.Provider>
    );
  }
}

class ThemedButton extends React.Component {
  render() {
    return (
      <ThemeContext.Consumer>
        {theme => <Button theme={theme} />}
      </ThemeContext.Consumer>
    );
  }
}
```

**迁移说明**[^6]：Legacy context API 在所有 React 16.x 版本中继续工作，有足够时间迁移。

---

### React 18（2022年3月）：并发渲染与 Context

React 18 引入并发渲染，对 Context 行为有重要影响[^7]：

**移除的 API**：
- `unstable_changedBits`：一个未发布的 bailout 机制，用于优化 Context 传播，被移除

**并发模式的影响**[^7]：
- 渲染可中断：React 可能开始渲染更新，中间暂停，稍后继续
- Context 更新在并发渲染中可能被延迟处理
- Suspense 与 Context 结合改进：Context 在 suspended trees 中正确传播

**useSyncExternalStore**[^7]：
- 新增 Hook，帮助外部 store 库集成 React
- 强制更新同步执行，避免并发模式下的 tearing 问题
- 主要供库开发者使用，非应用代码

---

### React 19（2024年12月）：简化 Provider API

React 19 对 Context API 进行了简化[^8]：

**`<Context>` 作为 Provider**[^8]：
- 可以直接渲染 `<Context>` 作为 Provider，无需 `<Context.Provider>`
- 未来版本将弃用 `<Context.Provider>`

```jsx
// React 19 新写法
const ThemeContext = createContext('');

function App({ children }) {
  return (
    <ThemeContext value="dark">
      {children}
    </ThemeContext>
  );
}

// 旧写法（仍支持，但将弃用）
<ThemeContext.Provider value="dark">
  {children}
</ThemeContext.Provider>
```

**`use` API 读取 Context**[^8]：
- 新的 `use` API 可以读取 Context
- 与 `useContext` 不同，`use` 支持条件调用（可在 early return 后使用）

```jsx
import { use } from 'react';

function Heading({ children }) {
  if (children == null) {
    return null;
  }
  // useContext 不支持条件调用，但 use 支持
  const theme = use(ThemeContext);
  return <h1 style={{ color: theme.color }}>{children}</h1>;
}
```

---

### Context Selectors RFC：未被采纳的提案

**RFC #119（2019年提出）**[^9]：
- 提议添加 `useContextSelector(context, selector)` API
- 仅当 selector 返回值变化时才触发重渲染
- 目标：解决外部状态订阅与并发 React 的兼容问题

**未被采纳的原因**（React 团队 2025年说明）[^9]：

1. **React Compiler 提供替代方案**：
   - Compiler 自动 memoize 组件输出
   - 消费 Context 的组件仍会重渲染，但子组件输出被缓存
   - 如果组件只依赖 Context 中未变化的部分，子组件不会重渲染

2. **更好的数据建模是根本解决方案**：
   - 大型 immutable 状态 blob 会导致任何响应式系统出现性能问题
   - 更好的方案是 normalized store + selectors（如 Relay）
   - Context selectors 和 signals 虽然流行，但不是根本解决

3. **支持并发 store + selectors**：
   - React 团队正在开发支持并发 store 的方案
   - 用 Context 传递"哪个 store"，而非直接访问 store

**用户land 实现**[^4][^9]：
- `use-context-selector` 库提供了 RFC 提议的功能
- 但需注意其限制（不支持 Class 组件、children 必须 memoized）

---

### 版本演进总结

| 版本 | Context 相关变化 | 影响 |
|------|-----------------|------|
| React 16.3 | 新官方 Context API | 替代 legacy context，更高效、类型安全 |
| React 18 | 移除 `unstable_changedBits`，引入并发渲染 | Context 在并发模式下行为变化，新增 `useSyncExternalStore` |
| React 19 | `<Context>` 简化 Provider，`use` API | 更简洁的语法，支持条件读取 Context |
| RFC #119 | `useContextSelector` 提案 | 未被采纳，React Compiler + 更好的数据建模是替代方案 |

---

## 参考资料

[^1]: React 官方文档 - useContext: Optimizing re-renders when passing objects and functions. https://react.dev/reference/react/useContext#optimizing-re-renders-when-passing-objects-and-functions

[^2]: React 官方文档 - memo: Updating a memoized component using a context. https://react.dev/reference/react/memo#updating-a-memoized-component-using-a-context

[^3]: React 官方文档 - Scaling Up with Reducer and Context. https://react.dev/learn/scaling-up-with-reducer-and-context

[^4]: use-context-selector GitHub Repository. https://github.com/dai-shi/use-context-selector

[^5]: React 官方文档 - Passing Data Deeply with Context. https://react.dev/learn/passing-data-deeply-with-context

[^6]: React Blog - React v16.3.0: New lifecycles and context API. https://legacy.reactjs.org/blog/2018/03/29/react-v-16-3.html

[^7]: React Blog - React v18.0. https://react.dev/blog/2022/03/29/react-v18

[^8]: React Blog - React v19. https://react.dev/blog/2024/04/25/react-19

[^9]: React RFC #119: Context selectors. https://github.com/reactjs/rfcs/pull/119
