---
created: 2026-04-27
updated: 2026-04-27
tags:
  - React
  - ClassComponent
  - FunctionComponent
  - Lifecycle
  - Hooks
  - Frontend
aliases:
  - React Class vs Function Component
  - React 生命周期对比
  - React 组件类型区别
source_type: official-doc
source_urls:
  - https://react.dev/reference/react/Component
  - https://react.dev/reference/react/hooks
  - https://react.dev/reference/react/useState
  - https://react.dev/reference/react/useEffect
status: verified
---

## 是什么

React 提供了两种定义组件的方式：**Class Component**（类组件）和 **Function Component**（函数组件）。这是 React 演进过程中形成的两种编程范式，理解它们的区别和生命周期映射关系，是掌握 React 核心机制的基础。

**官方立场**：React 官方明确推荐在新代码中使用函数组件 + Hooks，类组件仍被支持但不再推荐。[来源](https://react.dev/reference/react/Component)

---

## 核心区别对比

| 维度 | Class Component | Function Component |
|------|-----------------|-------------------|
| **定义方式** | `class MyComponent extends React.Component` | `function MyComponent() { ... }` |
| **状态管理** | `this.state` + `this.setState()` | `useState()` / `useReducer()` |
| **生命周期** | 类方法（`componentDidMount` 等） | `useEffect()` / `useLayoutEffect()` |
| **this 绑定** | 需要处理 `this` 指向（构造函数绑定或箭头函数） | 无 `this` 问题 |
| **逻辑复用** | HOC（高阶组件）、Render Props | 自定义 Hooks |
| **代码组织** | 生命周期方法强制拆分相关逻辑 | Hooks 可按功能聚合相关逻辑 |
| **性能优化** | `shouldComponentUpdate` / `PureComponent` | `React.memo()` / `useMemo()` / `useCallback()` |
| **Context 读取** | `static contextType` + `this.context`（仅一个） | `useContext()`（可多个） |
| **Error Boundary** | 支持（`componentDidCatch` / `getDerivedStateFromError`） | 暂无直接等价（需使用类组件或第三方库） |
| **渲染行为** | 类实例在多次渲染间保持稳定 | 每次渲染都是全新的函数调用 |
| **闭包陷阱** | 不存在（通过 `this` 引用最新值） | 存在（需注意依赖数组和最新值捕获） |

---

## Class Component 生命周期详解

### 完整生命周期流程

```
挂载阶段（Mount）
  ↓
constructor()
  ↓
static getDerivedStateFromProps()
  ↓
render()
  ↓
componentDidMount()
  ↓
更新阶段（Update）
  ↓
static getDerivedStateFromProps()
  ↓
shouldComponentUpdate()
  ↓
render()
  ↓
getSnapshotBeforeUpdate()
  ↓
componentDidUpdate()
  ↓
卸载阶段（Unmount）
  ↓
componentWillUnmount()
```

### 各阶段方法说明

#### 挂载阶段

| 方法 | 触发时机 | 用途 | 注意事项 |
|------|---------|------|---------|
| `constructor(props)` | 组件实例化时 | 初始化 `this.state`、绑定事件处理器 | 唯一可直接赋值 `this.state` 的地方；不应有副作用 |
| `static getDerivedStateFromProps(props, state)` | 挂载前和每次更新前 | 根据 props 派生 state | 静态方法，无 `this` 访问；应返回对象或 `null` |
| `render()` | 挂载和每次更新时 | 返回 JSX | **唯一必需的方法**；必须是纯函数，无副作用 |
| `componentDidMount()` | 组件挂载到 DOM 后 | 数据获取、订阅、DOM 操作 | 可调用 `setState` 触发额外渲染（谨慎使用） |

#### 更新阶段

| 方法 | 触发时机 | 用途 | 注意事项 |
|------|---------|------|---------|
| `static getDerivedStateFromProps()` | 每次更新前 | 根据新 props 派生 state | 挂载时也会调用 |
| `shouldComponentUpdate(nextProps, nextState)` | 更新前、render 前 | 性能优化，决定是否重新渲染 | 默认返回 `true`；返回 `false` 则跳过 render 和后续生命周期 |
| `render()` | 更新时 | 返回新的 JSX | 同上 |
| `getSnapshotBeforeUpdate(prevProps, prevState)` | DOM 更新前 | 捕获 DOM 信息（如滚动位置） | 返回值作为第三个参数传给 `componentDidUpdate` |
| `componentDidUpdate(prevProps, prevState, snapshot)` | DOM 更新后 | 操作更新后的 DOM、网络请求 | 必须用条件判断避免无限循环 |

#### 卸载阶段

| 方法 | 触发时机 | 用途 | 注意事项 |
|------|---------|------|---------|
| `componentWillUnmount()` | 组件从 DOM 移除前 | 清理订阅、定时器、取消请求 | 应与 `componentDidMount` 逻辑对称 |

### 已废弃的生命周期（UNSAFE_ 前缀）

以下方法在 React 16.3 被标记为不安全，17 添加 `UNSAFE_` 前缀，未来版本将完全移除：

| 旧名称 | 新名称 | 为什么不安全 | 替代方案 |
|--------|--------|-------------|---------|
| `componentWillMount` | `UNSAFE_componentWillMount` | 在 Suspense 下不保证一定挂载 | 使用 `constructor` 或 `componentDidMount` |
| `componentWillReceiveProps` | `UNSAFE_componentWillReceiveProps` | 在 Suspense 下不保证一定收到 props | 使用 `static getDerivedStateFromProps` 或 `componentDidUpdate` |
| `componentWillUpdate` | `UNSAFE_componentWillUpdate` | 在 Suspense 下不保证一定更新 | 使用 `componentDidUpdate` 或 `getSnapshotBeforeUpdate` |

> 参考：[Update on Async Rendering - React Blog](https://legacy.reactjs.org/blog/2018/03/27/update-on-async-rendering.html)

---

## Function Component 生命周期（Hooks 方式）

函数组件没有传统意义上的"生命周期方法"，而是通过 **Hooks** 实现等效功能。

### 生命周期与 Hooks 映射表

| Class 生命周期 | Function Component 等效实现 | 说明 |
|---------------|---------------------------|------|
| `constructor` | `useState` 初始化 / `useRef` 初始化 | 惰性初始化：`useState(() => computeExpensive())` |
| `static getDerivedStateFromProps` | 渲染期间调用 `setState` / 使用 `key` 重置组件 | 直接在渲染时根据 props 计算值，或提升状态到父组件 |
| `render` | 函数组件本身 | 函数体即 render 逻辑 |
| `componentDidMount` | `useEffect(() => { ... }, [])` | 空依赖数组，仅在挂载时执行 |
| `componentDidUpdate` | `useEffect(() => { ... }, [deps])` | 依赖数组决定何时执行 |
| `componentWillUnmount` | `useEffect` 返回的清理函数 | `return () => { cleanup }` |
| `componentDidMount + componentDidUpdate` | `useEffect(() => { ... })` | 无依赖数组：每次渲染后执行 |
| `getSnapshotBeforeUpdate` | **暂无直接等价** | 需用 `useLayoutEffect` + ref 手动实现，或保留类组件 |
| `shouldComponentUpdate` | `React.memo()` 组件包裹 | 浅比较 props，可自定义比较函数 |
| `componentDidCatch` / `getDerivedStateFromError` | **暂无直接等价** | 需使用类组件 Error Boundary 或 `react-error-boundary` 库 |

### useEffect 生命周期对应关系

```js
// 等效 componentDidMount
useEffect(() => {
  // 挂载时执行
  return () => {
    // 等效 componentWillUnmount
  };
}, []);

// 等效 componentDidMount + componentDidUpdate（特定依赖）
useEffect(() => {
  // 挂载时执行 + deps 变化时执行
  return () => {
    // deps 变化前清理 + 卸载时清理
  };
}, [deps]);

// 等效 componentDidMount + componentDidUpdate（每次渲染）
useEffect(() => {
  // 每次渲染后执行
  return () => {
    // 下次执行前清理 + 卸载时清理
  };
});
```

### useLayoutEffect 与 useEffect 的区别

| 特性 | `useEffect` | `useLayoutEffect` |
|------|-------------|-------------------|
| **执行时机** | 浏览器绘制**后**异步执行 | 浏览器绘制**前**同步执行 |
| **对应类组件** | `componentDidMount` / `componentDidUpdate` | 更接近这些方法的同步行为 |
| **适用场景** | 数据获取、订阅、大多数副作用 | DOM 测量、同步重新渲染 |
| **SSR 兼容性** | 兼容 | 不兼容（服务端会警告） |
| **性能影响** | 不阻塞渲染 | 可能阻塞渲染 |

---

## 关键机制差异

### 1. 实例稳定性

**Class Component**：组件实例在多次渲染间保持稳定，`this` 始终指向同一实例。

```js
class Counter extends Component {
  state = { count: 0 };

  handleClick = () => {
    setTimeout(() => {
      console.log(this.state.count); // 总是读取最新值
    }, 3000);
  };
}
```

**Function Component**：每次渲染都是全新的函数调用，存在闭包陷阱。

```js
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setTimeout(() => {
      console.log(count); // 捕获的是点击时的值，不是最新值
    }, 3000);
  };
}
```

**解决方案**：
- 使用 `useRef` 持有最新值
- 使用 `useReducer` 的 dispatch（dispatch 引用稳定）
- 使用 `useEffectEvent`（实验性）分离事件与响应式值

### 2. 状态更新行为

**Class Component**：`setState` 是对象浅合并。

```js
this.setState({ count: 1 });  // 只更新 count，其他 state 保留
this.setState({ name: 'x' }); // name 被添加/更新
```

**Function Component**：`useState` 的 setter 是替换而非合并。

```js
const [state, setState] = useState({ count: 0, name: '' });
setState({ count: 1 });  // name 丢失！需要展开
setState(prev => ({ ...prev, count: 1 }));  // 正确做法
```

### 3. 逻辑组织方式

**Class Component**：相关逻辑被拆分到不同生命周期。

```js
class ChatRoom extends Component {
  componentDidMount() {
    this.subscribeToChat(this.props.roomId);
  }

  componentDidUpdate(prevProps) {
    if (prevProps.roomId !== this.props.roomId) {
      this.unsubscribeFromChat(prevProps.roomId);
      this.subscribeToChat(this.props.roomId);
    }
  }

  componentWillUnmount() {
    this.unsubscribeFromChat(this.props.roomId);
  }
}
```

**Function Component**：相关逻辑可聚合在同一 Hook。

```js
function ChatRoom({ roomId }) {
  useEffect(() => {
    const unsubscribe = subscribeToChat(roomId);
    return () => unsubscribe();
  }, [roomId]);  // roomId 变化时自动清理并重新订阅
}
```

---

## 限制与注意事项

### Class Component 的局限

1. **难以复用状态逻辑**：需要 HOC 或 Render Props，导致组件嵌套过深（wrapper hell）
2. **复杂组件难以理解**：相关逻辑分散在多个生命周期方法中
3. **this 指向问题**：需要手动绑定或使用箭头函数，容易出错
4. **不利于优化和工具**：类组件的实例化开销较大，minification 效果差
5. **官方不再推荐**：新特性优先支持函数组件

### Function Component 的局限

1. **Error Boundary 无法用 Hooks 实现**：仍需类组件或第三方库
2. **getSnapshotBeforeUpdate 无直接等价**：需用 `useLayoutEffect` + ref 手动模拟
3. **闭包陷阱**：新手容易踩坑，需要理解 Hooks 的依赖捕获机制
4. **Hooks 规则限制**：只能在顶层调用、只能在 React 函数中调用
5. **依赖数组心智负担**：需要正确声明依赖，否则可能导致无限循环或过期闭包

### 何时仍需使用 Class Component

- 实现 Error Boundary（`componentDidCatch` / `getDerivedStateFromError`）
- 使用 `getSnapshotBeforeUpdate` 捕获 DOM 快照
- 维护遗留代码库（迁移成本过高时）

---

## 迁移指南（Class → Function）

### 简单组件迁移

```js
// Class
class Greeting extends Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}

// Function
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}
```

### 带状态的组件迁移

```js
// Class
class Counter extends Component {
  state = { count: 0 };

  handleClick = () => {
    this.setState({ count: this.state.count + 1 });
  };

  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}

// Function
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

### 带生命周期的组件迁移

```js
// Class
class Timer extends Component {
  state = { seconds: 0 };

  componentDidMount() {
    this.interval = setInterval(() => {
      this.setState(prev => ({ seconds: prev.seconds + 1 }));
    }, 1000);
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  render() {
    return <span>{this.state.seconds}s</span>;
  }
}

// Function
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  return <span>{seconds}s</span>;
}
```

---

## 常见误区

1. **"Function Component 是无状态组件"**：错误。React 16.8 引入 Hooks 后，函数组件完全支持状态管理。

2. **"useEffect 完全等同于 componentDidMount + componentDidUpdate + componentWillUnmount"**：不完全正确。`useEffect` 的执行时机和依赖追踪机制与类生命周期有差异，特别是在 Concurrent Mode 下。

3. **"Class Component 性能更好"**：错误。现代 React 中两者性能差异可忽略，React 18 的优化对两者一视同仁。

4. **"setState 和 useState setter 行为完全一致"**：错误。`setState` 是浅合并，`useState` setter 是替换。

5. **"useEffect 的清理函数只在卸载时执行"**：错误。清理函数在**每次依赖变化前**和**卸载时**都会执行。

---

## 相关概念

- [React Hooks 详解](./React%20Hooks%20详解)：所有内置 Hooks 的完整参考
- [React 面试原理与考点详解](../面试/React%20面试原理与考点详解)：Fiber 架构、渲染流程、Reconciliation
- [React 性能优化思路与常用工具](./React%20性能优化思路与常用工具)：`React.memo`、`useMemo`、`useCallback` 等优化手段

---

## 参考资料

- [Component (Class API) - React 官方文档](https://react.dev/reference/react/Component)
- [Built-in React Hooks - React 官方文档](https://react.dev/reference/react/hooks)
- [useState - React API Reference](https://react.dev/reference/react/useState)
- [useEffect - React API Reference](https://react.dev/reference/react/useEffect)
- [You Might Not Need an Effect - React 官方文档](https://react.dev/learn/you-might-not-need-an-effect)
- [Update on Async Rendering - React Blog (2018)](https://legacy.reactjs.org/blog/2018/03/27/update-on-async-rendering.html)
- [You Probably Don't Need Derived State - React Blog (2018)](https://legacy.reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)
