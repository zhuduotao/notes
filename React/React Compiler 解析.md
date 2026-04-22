---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - React
  - compiler
  - performance
  - memoization
  - optimization
  - frontend
aliases:
  - React Compiler
  - React 编译器
  - babel-plugin-react-compiler
source_type: official-doc
source_urls:
  - 'https://react.dev/learn/react-compiler'
  - 'https://react.dev/learn/react-compiler/introduction'
  - 'https://react.dev/blog/2024/04/25/react-19'
  - 'https://github.com/facebook/react/tree/main/compiler'
  - >-
    https://raw.githubusercontent.com/facebook/react/main/compiler/docs/DESIGN_GOALS.md
  - 'https://react.dev/reference/react-compiler/configuration'
status: verified
---
## 是什么

React Compiler 是 React 官方提供的**构建时优化工具**，自动为 React 应用添加 memoization，无需开发者手动使用 `useMemo`、`useCallback` 或 `React.memo`。

它通过编译时分析代码，自动识别可以优化的部分，生成带有自动 memoization 的代码，确保只有必要的组件和数据才会在状态变化时重新计算或渲染。

React Compiler 于 2024 年 10 月正式发布稳定版本，在 React 19 中成为核心特性。

---

## 为什么重要

### React 的性能瓶颈

React 的默认行为是：**当组件状态变化时，该组件及其所有子组件都会重新渲染**。这在以下场景会导致性能问题：

- 大型组件树中，父组件变化导致整个子树重渲染
- 昂贵的计算在每次渲染时重复执行
- 不稳定的 props（如内联函数、对象）导致 `React.memo` 失效

### 手动优化的痛点

传统优化方式需要开发者手动添加：

```jsx
import { useMemo, useCallback, memo } from 'react';

const ExpensiveComponent = memo(function ExpensiveComponent({ data, onClick }) {
  const processedData = useMemo(() => expensiveProcessing(data), [data]);

  const handleClick = useCallback((item) => {
    onClick(item.id);
  }, [onClick]);

  return (
    <div>
      {processedData.map(item => (
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </div>
  );
});
```

**痛点**：
- 代码冗长，维护成本高
- 容易出错（如上面代码中 `() => handleClick(item)` 仍然创建新函数，破坏 memoization）
- 需要理解依赖数组的正确用法
- 性能优化变成"负担而非收益"

---

## React Compiler 后的代码

同样的功能，使用 React Compiler 后可以简化为：

```jsx
function ExpensiveComponent({ data, onClick }) {
  const processedData = expensiveProcessing(data);

  const handleClick = (item) => {
    onClick(item.id);
  };

  return (
    <div>
      {processedData.map(item => (
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </div>
  );
}
```

Compiler 自动分析并添加最优的 memoization，确保：
- `processedData` 仅在 `data` 变化时重新计算
- `Item` 仅在相关 props 变化时重新渲染
- 即使有箭函数包裹，也能正确优化

---

## 核心原理

### 编译流程

React Compiler 的工作流程[^4]：

```
源代码 → HIR（高级中间表示） → SSA 转换 → 验证 → 类型推断 → 推断 Reactive Scopes → 优化 → 代码生成
```

1. **Lowering（BuildHIR）**：将 Babel AST 转换为高级中间表示（HIR），保留 JavaScript 的精确求值顺序
2. **SSA Conversion**：转换为静态单赋值形式，每个标识符只赋值一次
3. **Validation**：检查代码是否遵循 Rules of React（如条件内调用 hooks）
4. **Type Inference**：推断值类型（hooks、primitives 等）
5. **Inferring Reactive Scopes**：识别哪些值需要 memoization，以及它们的依赖关系
6. **Codegen**：生成带有 memoization 的优化代码

### Reactive Scopes

Compiler 的核心概念是 **Reactive Scope**[^4]：

- 一组需要一起创建/修改的值
- Compiler 分析这些值的依赖关系，决定何时需要重新计算
- 自动合并相邻的 scope 以减少开销
- 跳过包含 hook 调用的 scope（hook 调用不能条件化）

### 生成的代码示例

Compiler 生成的代码会引入 runtime：

```jsx
import { c as _c } from "react/compiler-runtime";

export default function MyApp() {
  const $ = _c(1);  // 创建 memoization cache
  let t0;
  if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
    t0 = <div>Hello World</div>;
    $[0] = t0;  // 缓存结果
  } else {
    t0 = $[0];  // 使用缓存
  }
  return t0;
}
```

---

## 优化范围

Compiler 主要优化两种场景[^1]：

### 1. 跳过级联重渲染

当父组件状态变化时，默认会重渲染整个子树。Compiler 自动应用类似 `React.memo` 的优化：

```jsx
function FriendList({ friends }) {
  const onlineCount = useFriendOnlineCount();
  
  return (
    <div>
      <span>{onlineCount} online</span>
      {friends.map((friend) => (
        <FriendListCard key={friend.id} friend={friend} />
      ))}
      <MessageButton />
    </div>
  );
}
```

Compiler 会自动确保：
- `<FriendListCard />` 仅在 `friend` prop 变化时重渲染
- `<MessageButton />` 不会因 `onlineCount` 变化而重渲染

### 2. 缓存昂贵计算

```jsx
function TableContainer({ items }) {
  // Compiler 自动缓存这个计算
  const data = expensivelyProcessAReallyLargeArrayOfObjects(items);
  // ...
}
```

**注意**[^1]：Compiler 只 memoize React 组件和 hooks 内的计算，不会优化：
- 普通函数（不在组件或 hook 内）
- 跨多个组件共享的计算（每个组件独立 memoize）

---

## 使用方式

### 安装

```bash
npm install -D babel-plugin-react-compiler@latest
```

### Babel 配置

```js
// babel.config.js
module.exports = {
  plugins: [
    'babel-plugin-react-compiler',  // 必须放在第一位
    // ... 其他 plugins
  ],
};
```

**重要**[^2]：Compiler 必须**第一个**运行，需要原始源码信息进行分析。

### Vite 配置

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      babel: {
        plugins: ['babel-plugin-react-compiler'],
      },
    }),
  ],
});
```

### Next.js

Next.js v15.3.1+ 内置支持，参考 [Next.js 文档](https://nextjs.org/docs/app/api-reference/next-config-js/reactCompiler)。

### ESLint 集成

```bash
npm install -D eslint-plugin-react-hooks@latest
```

ESLint 规则会：
- 报告违反 Rules of React 的代码
- 指出哪些组件无法被优化
- 提供修复建议

---

## React 版本兼容性

| React 版本 | 支持情况 | 需要额外配置 |
|-----------|---------|-------------|
| React 19 | 完全支持 | 无 |
| React 18 | 支持 | 需要 `react-compiler-runtime` + `target: '18'` |
| React 17 | 支持 | 需要 `react-compiler-runtime` + `target: '17'` |

React 17/18 配置[^3]：

```bash
npm install react-compiler-runtime@latest
```

```js
// babel.config.js
{
  target: '18'  // 或 '17'
}
```

---

## 与手动 memoization 的关系

### 已有的 useMemo/useCallback 怎么办

**新代码**：依赖 Compiler 自动优化，仅在需要精确控制时使用 `useMemo`/`useCallback`（如 effect 依赖）。

**现有代码**[^1]：
- 保留已有的 memoization（移除可能改变编译输出）
- 或在仔细测试后移除

### React.memo 还需要吗

启用 Compiler 后，通常不再需要手动 `React.memo`，Compiler 会自动应用等效优化。

### 保留 useMemo 的场景

- 作为 effect 依赖，确保 effect 不过度触发
- 需要精确控制哪些值被缓存

---

## 限制与注意事项

### 不优化 Class 组件

Compiler 不支持 Class 组件[^4]，因为其固有的可变状态和复杂生命周期难以建模。

### 不支持违反 Rules of React 的代码

Compiler 依赖 Rules of React 进行安全转换[^4]：
- 组件和 hooks 必须是纯函数
- Hooks 必须在顶层调用，不能条件化
- 不支持无条件 setState 调用

### 不追求完美优化

Compiler 的目标不是零不必要计算[^4]：
- 追求完美优化的 runtime 开销可能超过计算本身
- 条件依赖场景无法避免某些重复计算
- 代码膨胀会影响启动性能

### 不支持的 JavaScript 特性

- 嵌套类捕获闭包值（难以建模）
- `eval()`（不安全）
- 少见或不安全特性

### Memoization-for-correctness 问题

**常见问题**[^5]：代码依赖 memoization 来保证正确性，而非仅用于性能：

- Effect 依赖对象引用相等性
- 不稳定的依赖数组导致 effect 过度触发或无限循环
- 基于引用检查的条件逻辑

**修复方法**：移除所有手动 memoization，确认代码仍然正确工作；如果出错，说明有 Rules of React 违规需要修复。

---

## 调试与排错

### 验证 Compiler 是否工作

React DevTools 中，优化后的组件会显示 **"Memo ✨"** 标记[^2]。

### 临时禁用编译

使用 `"use no memo"` 指令跳过特定组件[^2][^5]：

```jsx
function ProblematicComponent() {
  "use no memo";
  // ...
}
```

### 增量采用策略

使用 `compilationMode: 'annotation'`，仅编译标记 `"use memo"` 的函数[^3]：

```js
{
  compilationMode: 'annotation'
}
```

逐步扩展优化范围。

---

## 设计目标与非目标

### 核心目标[^4]

- **性能可预测**：限制更新时的重渲染量
- **启动时间中立**：不增加显著代码大小或 memoization 开销
- **保持 React 编程模型**：不改变开发者写 React 的方式
- **无需显式注解**：普通产品代码无需类型或其他标记
- **易于理解**：开发者能快速形成对 Compiler 工作方式的大致直觉

### 非目标[^4]

- **零不必要计算**：追求完美优化不现实，runtime 开销可能更大
- **支持违反规则的代码**：Rules of React 是 React 改进的基础契约
- **支持 legacy React 特性**：不支持 Class 组件
- **支持 100% JavaScript**：不支持少见或不安全特性

---

## 与 Million.js 等第三方编译器的区别

| 特性 | React Compiler | Million.js |
|------|---------------|------------|
| 来源 | React 官方 | 第三方开源 |
| 优化策略 | 自动 memoization | Block Virtual DOM |
| 需要代码改动 | 无 | 需要 `block()` 包装或自动模式 |
| 支持 React 版本 | 17, 18, 19 | 18+ |
| 与 React.memo 关系 | 自动替代 | 并不替代，可配合使用 |

**本质区别**：
- React Compiler 优化的是**何时重渲染**（自动 memoization）
- Million.js 优化的是**diff 效率**（改变 diff 粒度）
- 两者解决不同层面的问题

---

## 相关概念

- [[React 性能优化思路与常用工具]] — React 性能优化方法论
- [[React Fiber 架构详解]] — React 底层调度与协调机制
- [[React Context 性能优化方案]] — Context 相关性能问题及解决方案
- [[Million.js 前端性能优化：原理与适用场景]] — Block Virtual DOM 优化方案
- [[React Hooks 详解]] — Hooks 原理与使用

---

## 参考资料

[^1]: React 官方文档 - React Compiler Introduction. https://react.dev/learn/react-compiler/introduction

[^2]: React 官方文档 - React Compiler Installation. https://react.dev/learn/react-compiler/installation

[^3]: React 官方文档 - React Compiler Configuration. https://react.dev/reference/react-compiler/configuration

[^4]: React Compiler Design Goals. https://raw.githubusercontent.com/facebook/react/main/compiler/docs/DESIGN_GOALS.md

[^5]: React 官方文档 - React Compiler Debugging. https://react.dev/learn/react-compiler/debugging

[^6]: React Blog - React v19. https://react.dev/blog/2024/04/25/react-19

[^7]: React 官方文档 - React Compiler. https://react.dev/learn/react-compiler

[^8]: React Compiler GitHub Repository. https://github.com/facebook/react/tree/main/compiler
