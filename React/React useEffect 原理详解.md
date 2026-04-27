---
created: 2026-04-27T00:00:00.000Z
updated: '2026-04-27T00:00:00.000Z'
tags:
  - React
  - useEffect
  - Hooks
  - 原理
  - Frontend
aliases:
  - useEffect 原理
  - React Effect 机制
  - useEffect internal
source_type: official-doc
source_urls:
  - 'https://react.dev/reference/react/useEffect'
  - 'https://react.dev/learn/synchronizing-with-effects'
  - 'https://react.dev/learn/lifecycle-of-reactive-effects'
  - 'https://react.dev/learn/you-might-not-need-an-effect'
status: verified
---

## 是什么

`useEffect` 是 React 16.8 引入的 Hook，用于让组件**与外部系统同步**（如 DOM 操作、网络请求、第三方库、浏览器 API 等）。它本质上是 React 提供的一个"逃生舱"（escape hatch），用于处理那些无法在纯渲染逻辑中完成的副作用。

```js
useEffect(setup, dependencies?)
```

- `setup`：包含 Effect 逻辑的函数，可返回一个清理函数
- `dependencies`：依赖数组，决定 Effect 何时重新执行

## 为什么需要 useEffect

React 组件中的逻辑分为两类：

| 类型 | 触发方式 | 特点 | 示例 |
|------|----------|------|------|
| **渲染代码** | 组件函数体顶层 | 必须是纯计算，不能有副作用 | 计算 JSX、转换 props/state |
| **事件处理器** | 用户交互（点击、输入等） | 可以包含副作用 | 提交表单、导航 |
| **Effect** | 渲染本身（而非特定事件） | 用于同步外部系统 | 连接服务器、订阅事件、操作 DOM |

Effect 填补了一个关键场景：**某些副作用不是由特定用户操作引起的，而是由组件"出现在屏幕上"这一事实引起的**。例如，一个聊天室组件只要显示就需要连接服务器，无论用户是通过什么路径到达这个页面的。

## 执行时机与生命周期

### 在 React 渲染流程中的位置

React 每次更新经历三个阶段：Trigger → Render → Commit。`useEffect` 在 **Commit 阶段之后**异步执行：

```
Trigger（触发）→ Render（计算）→ Commit（DOM 更新）→ useEffect 执行
```

这意味着：
- Effect **总是在 DOM 已更新之后**才运行
- React 会先让浏览器完成屏幕绘制，再执行 Effect（非交互触发的 Effect）
- 如果 Effect 由用户交互引起，React **可能**在浏览器绘制前执行，以便事件系统能观察到 Effect 的结果

### 完整生命周期

```
组件挂载（Mount）
  └→ 执行 setup()

依赖变化（Re-sync）
  ├→ 执行 cleanup()（使用旧的 props/state）
  └→ 执行 setup()（使用新的 props/state）

组件卸载（Unmount）
  └→ 执行 cleanup()
```

**关键细节**：
1. 组件首次添加到页面时，React 执行 setup 函数
2. 每次依赖变化后的 commit，React **先执行清理函数（旧值），再执行 setup 函数（新值）**
3. 组件从页面移除时，React 最后一次执行清理函数

### 开发环境的双重执行

Strict Mode 下，React 在开发环境中会对每个组件**额外执行一次 setup → cleanup → setup 周期**：

```
开发环境：setup → cleanup → setup → （正常使用）→ cleanup
生产环境：setup → （正常使用）→ cleanup
```

这是压力测试，用于验证清理逻辑是否正确"镜像"了 setup 逻辑。如果清理函数实现正确，用户无法区分 Effect 执行了一次还是执行了 setup → cleanup → setup。

## 内部实现机制

### Fiber 架构中的 Effect 存储

在 React 的 Fiber 架构中，每个组件实例的 Fiber 节点维护了一个 **effect 链表**：

```
fiber.updateQueue = {
  lastEffect: effect1 → effect2 → effect3 → null
}
```

每个 effect 节点包含：
- `create`：setup 函数
- `destroy`：cleanup 函数（setup 的返回值）
- `deps`：依赖数组
- `tag`：标记 Effect 的类型和状态（如 `HookHasEffect` 表示需要重新执行）

### 依赖比较机制

React 使用 `Object.is()` 逐个比较依赖数组中的值：

```js
// 伪代码：React 内部的依赖比较逻辑
function areDepsEqual(prevDeps, nextDeps) {
  for (let i = 0; i < nextDeps.length; i++) {
    if (!Object.is(nextDeps[i], prevDeps[i])) {
      return false;
    }
  }
  return true;
}
```

`Object.is()` 的行为：
- `Object.is(NaN, NaN)` → `true`（与 `===` 不同）
- `Object.is(-0, +0)` → `false`（与 `===` 不同）
- 对象/函数比较的是**引用**，每次渲染新建的对象/函数会被视为不同

### 每次渲染的 Effect 是独立的

每个渲染周期都有自己独立的 Effect 实例，它们通过**闭包**捕获对应渲染周期的 props 和 state：

```js
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // 捕获的是当前渲染的 count 值
    }, 1000);
    return () => clearInterval(id);
  }, [count]);
}
```

当 `count` 从 0 变为 1 时：
1. React 先清理 `count = 0` 时的 Effect（清除旧的 interval）
2. 再执行 `count = 1` 时的 Effect（创建新的 interval）
3. 新 interval 中的 `console.log(count)` 捕获的值是 `1`

这意味着**每个 Effect 从各自渲染中看到的 props 和 state 是固定的**，不会随后续状态变化而改变。

### Commit 阶段的 Effect 调度

React 在 Commit 阶段分三个子阶段处理 Effect：

| 子阶段 | 执行时机 | Effect 类型 |
|--------|----------|-------------|
| **Before Mutation** | DOM 更新前 | `useLayoutEffect` 的清理函数 |
| **Mutation** | DOM 更新中 | 无 Effect 执行 |
| **Layout** | DOM 更新后、浏览器绘制前 | `useLayoutEffect` 的 setup 函数 |
| **Passive（异步）** | 浏览器绘制后 | `useEffect` 的 setup/cleanup 函数 |

`useEffect` 属于 **Passive Effect**，在浏览器完成绘制后异步执行，不会阻塞视觉更新。

## 依赖数组的三种行为

| 写法 | 行为 | 适用场景 |
|------|------|----------|
| 省略依赖数组 | 每次渲染后都执行 | 极少需要，通常会导致性能问题 |
| `[]`（空数组） | 仅挂载时执行一次（开发环境两次） | 仅需在组件生命周期内执行一次的逻辑 |
| `[dep1, dep2]` | 依赖变化时重新执行 | 需要同步外部系统且依赖某些响应式值 |

### 什么是"响应式值"

所有在组件体内声明且在 Effect 中使用的值都是响应式值，必须声明为依赖：
- props
- state（`useState`、`useReducer` 返回值）
- 在组件体内创建的变量和函数
- 其他 Hook 的返回值（如 `useContext`）

**例外**（具有稳定引用，可不声明为依赖）：
- `useRef` 返回的 ref 对象（React 保证每次渲染返回同一个对象）
- `useState` 返回的 `setState` 函数
- `useReducer` 返回的 `dispatch` 函数

### 不能"选择"依赖

依赖数组不是由开发者主观决定的，而是由 Effect 内部的代码决定的。如果 Effect 中使用了某个响应式值但未声明为依赖，React 的 lint 规则（`react-hooks/exhaustive-deps`）会报错。

## 常见模式与最佳实践

### 连接外部系统

```js
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();
  return () => {
    connection.disconnect(); // 清理函数必须"镜像"setup
  };
}, [serverUrl, roomId]);
```

### 订阅事件

```js
useEffect(() => {
  function handleScroll(e) {
    console.log(window.scrollX, window.scrollY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

### 基于前一个状态更新

当 Effect 中需要根据前一个状态值来更新时，使用 updater 函数避免将状态值加入依赖：

```js
// 错误：count 作为依赖会导致 interval 不断重置
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);

// 正确：使用 updater 函数，依赖数组为空
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### 数据获取与竞态条件

```js
useEffect(() => {
  let ignore = false; // 用于防止竞态条件
  fetchBio(person).then(result => {
    if (!ignore) {
      setBio(result);
    }
  });
  return () => {
    ignore = true; // 清理时标记为忽略
  };
}, [person]);
```

当 `person` 快速变化时，之前的请求可能晚于新请求返回。`ignore` 标志确保过期的响应不会更新状态。

### 减少不必要的依赖

**将对象创建移到 Effect 内部**：

```js
// 错误：每次渲染创建新对象，导致 Effect 重复执行
const options = { serverUrl, roomId };
useEffect(() => {
  const connection = createConnection(options);
  // ...
}, [options]);

// 正确：在 Effect 内部创建对象
useEffect(() => {
  const options = { serverUrl, roomId };
  const connection = createConnection(options);
  // ...
}, [serverUrl, roomId]);
```

**使用 Event 函数处理非响应式逻辑**（React 19+ 实验性 `useEffectEvent`）：

```js
// 将不需要触发 Effect 重新执行的逻辑提取为事件函数
const onConnected = useEffectEvent(() => {
  showNotification('Connected!');
});

useEffect(() => {
  const connection = createConnection(options);
  connection.on('connected', () => {
    onConnected(); // 调用不会触发 Effect 重新执行
  });
  connection.connect();
  return () => connection.disconnect();
}, [options]);
```

## 不需要 useEffect 的场景

官方明确指出：**如果不是在同步外部系统，可能不需要 Effect**。

### 派生状态（Derived State）

```js
// 错误：用 Effect 计算派生值
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);

// 正确：在渲染时计算
const fullName = firstName + ' ' + lastName;
```

### 事件过滤数据

```js
// 错误：用 Effect 过滤数据
const [filteredItems, setFilteredItems] = useState([]);
useEffect(() => {
  setFilteredItems(items.filter(item => item.includes(query)));
}, [items, query]);

// 正确：在事件处理器或渲染时处理
const filteredItems = items.filter(item => item.includes(query));
```

### 非 React 的初始化逻辑

```js
// 错误：在 Effect 中初始化应用
useEffect(() => {
  checkAuthToken();
  loadDataFromLocalStorage();
}, []);

// 正确：在组件外部执行（仅客户端）
if (typeof window !== 'undefined') {
  checkAuthToken();
  loadDataFromLocalStorage();
}
```

## 与 useLayoutEffect 的深度对比

### 核心差异

| 维度 | `useEffect` | `useLayoutEffect` |
|------|-------------|-------------------|
| **执行时机** | 浏览器绘制**后**（异步） | DOM 更新后、浏览器绘制**前**（同步） |
| **是否阻塞绘制** | 否 | 是 |
| **性能影响** | 小 | 可能显著（过度使用会导致卡顿） |
| **服务端渲染** | 不执行（仅客户端） | 不执行，且会报警告 |
| **内部调度阶段** | Passive Effect | Layout Effect |
| **典型用途** | 网络请求、订阅、日志、非关键 DOM 操作 | DOM 测量、同步滚动位置、避免视觉闪烁 |

### 执行时序详解

```
Trigger → Render → Commit（DOM 更新）
  ├── Before Mutation 阶段：useLayoutEffect 的 cleanup
  ├── Mutation 阶段：DOM 操作
  ├── Layout 阶段：useLayoutEffect 的 setup（同步，阻塞绘制）
  │   └→ 浏览器等待此阶段完成才绘制屏幕
  └── Passive 阶段：useEffect 的 setup（异步，不阻塞绘制）
      └→ 浏览器已完成绘制，用户已看到更新后的画面
```

### 何时选择 useLayoutEffect

**唯一场景**：你需要在浏览器绘制**之前**读取 DOM 布局信息，并基于该信息同步更新状态，以避免视觉闪烁。

典型例子：Tooltip 定位

```js
function Tooltip({ children, targetRect }) {
  const ref = useRef(null);
  const [tooltipHeight, setTooltipHeight] = useState(0);

  useLayoutEffect(() => {
    // 在浏览器绘制前测量高度
    const { height } = ref.current.getBoundingClientRect();
    setTooltipHeight(height); // 触发同步重新渲染
  }, []);

  // 根据测量高度决定 tooltip 显示在上方还是下方
  let tooltipY = targetRect.top - tooltipHeight;
  if (tooltipY < 0) {
    tooltipY = targetRect.bottom; // 空间不够，显示在下方
  }
  // ...
}
```

**执行流程**：
1. Tooltip 首次渲染，`tooltipHeight = 0`（位置可能不正确）
2. React 将节点放入 DOM，执行 `useLayoutEffect`
3. 测量 tooltip 实际高度，触发同步重新渲染
4. Tooltip 以正确的 `tooltipHeight` 再次渲染
5. React 更新 DOM，浏览器绘制最终结果

用户只看到最终正确的结果，不会看到闪烁。

### 使用 useEffect 替代 useLayoutEffect 的问题

同样的 Tooltip 逻辑如果用 `useEffect`：

1. Tooltip 首次渲染，`tooltipHeight = 0`
2. React 更新 DOM，**浏览器绘制屏幕**（用户看到位置错误的 tooltip）
3. `useEffect` 执行，测量高度，触发重新渲染
4. Tooltip 以正确高度渲染，浏览器再次绘制

**结果**：用户会短暂看到 tooltip 闪烁（先出现在错误位置，再跳到正确位置）。在慢速设备或渲染较慢时尤为明显。

### 从 useLayoutEffect 迁移到 useEffect

官方建议：**优先使用 `useEffect`，仅在必要时切换到 `useLayoutEffect`**。

如果你不确定是否需要 `useLayoutEffect`，可以先用 `useEffect`，观察是否有视觉闪烁。如果没有，保持 `useEffect`。

**迁移策略**：

```js
// 服务端兼容写法（避免 useLayoutEffect 的 SSR 警告）
import { useEffect, useLayoutEffect } from 'react';

// 在服务端渲染时降级为 useEffect
const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

### useLayoutEffect 中的状态更新行为

在 `useLayoutEffect` 中触发的状态更新会**同步阻塞**浏览器绘制，并且 React 会立即执行所有剩余的 Effect（包括 `useEffect`）：

```js
useLayoutEffect(() => {
  setState(newValue);
  // 这个 setState 会阻塞浏览器绘制
  // 同时，所有 useEffect 也会被立即触发执行
}, []);
```

这意味着 `useLayoutEffect` 中的状态更新会导致：
- 浏览器等待所有同步工作完成才绘制
- 用户感知到的延迟增加
- 可能触发额外的同步渲染链

### 三种 Effect Hook 完整对比

| Hook | 执行时机 | 是否阻塞绘制 | 典型用途 | 使用频率 |
|------|----------|-------------|----------|----------|
| `useInsertionEffect` | React 修改 DOM **之前**（同步） | 是 | CSS-in-JS 库注入样式 | 极少（库作者使用） |
| `useLayoutEffect` | DOM 更新后、浏览器绘制**前**（同步） | 是 | DOM 测量、同步滚动位置 | 偶尔 |
| `useEffect` | 浏览器绘制**后**（异步） | 否 | 网络请求、订阅、日志 | 默认选择 |

**选择原则**：
1. 默认使用 `useEffect`
2. 如果需要测量 DOM 布局且必须避免视觉闪烁，使用 `useLayoutEffect`
3. 仅在编写 CSS-in-JS 库时使用 `useInsertionEffect`

## 常见误区

### "useEffect 可以用来编排数据流"

**错误**。官方明确建议：如果不与外部系统交互，可能不需要 Effect。数据流应通过事件处理和状态更新来处理，而非 Effect。

### "空依赖数组 `[]` 意味着只执行一次"

**不完全正确**。在 Strict Mode 的开发环境中，即使依赖数组为空，Effect 也会执行两次（setup → cleanup → setup）。生产环境中确实只执行一次。

### "可以用 ref 阻止 Effect 执行两次"

**错误且危险**。使用 ref 标记"已执行"来跳过开发环境的双重执行，会导致组件卸载时不清理资源，造成内存泄漏和连接堆积：

```js
// 错误做法
const executedRef = useRef(false);
useEffect(() => {
  if (executedRef.current) return;
  executedRef.current = true;
  // 连接逻辑...
}, []);
```

正确的做法是实现清理函数，让 Effect 能够安全地重复执行。

### "Effect 中捕获的是最新的 state"

**错误**。Effect 捕获的是**创建它时那个渲染周期**的 state。如果需要读取最新值，应使用 ref 或将逻辑提取为事件函数。

### "在 Effect 中设置状态总是安全的"

**有条件安全**。如果 Effect 不是由用户交互引起的，React 会在浏览器绘制后执行，此时设置状态会触发新的渲染。如果 Effect 内部无条件地设置状态，可能导致无限循环：

```js
// 无限循环
useEffect(() => {
  setCount(count + 1); // 触发渲染 → Effect 再次执行 → 再次设置状态...
});
```

## 参考资料

- [useEffect API Reference - React 官方文档](https://react.dev/reference/react/useEffect)
- [Synchronizing with Effects - React 官方教程](https://react.dev/learn/synchronizing-with-effects)
- [Lifecycle of Reactive Effects - React 官方教程](https://react.dev/learn/lifecycle-of-reactive-effects)
- [You Might Not Need an Effect - React 官方教程](https://react.dev/learn/you-might-not-need-an-effect)
- [Removing Effect Dependencies - React 官方教程](https://react.dev/learn/removing-effect-dependencies)
- [Render and Commit - React 官方文档](https://react.dev/learn/render-and-commit)
