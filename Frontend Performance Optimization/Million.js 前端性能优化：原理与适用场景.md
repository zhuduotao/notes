---
created: '2026-04-18'
updated: '2026-04-18'
tags:
  - frontend
  - react
  - performance
  - virtual-dom
  - compiler
  - millionjs
aliases:
  - Million.js
  - Block Virtual DOM
  - Million
source_type: mixed
source_urls:
  - 'https://github.com/aidenybai/million'
  - 'https://old.million.dev/blog/virtual-dom'
  - 'https://krausest.github.io/js-framework-benchmark/'
status: verified
---

## 是什么

Million.js 是一个针对 React 的**优化编译器**（optimizing compiler），通过引入 **Block Virtual DOM（块虚拟 DOM）** 技术，使 React 组件的更新性能最高提升 **70%**（基于 JS Framework Benchmark 数据）。

它不是独立的前端框架，而是作为 React 的 drop-in 优化层，无需重写现有组件即可接入。

## 为什么重要

React 的性能瓶颈主要来自 **reconciliation（协调）** 阶段的 diff 算法：

- 每次状态变化时，React 会对整个组件树进行递归 diff
- 组件树越大，diff 检查的节点越多，时间复杂度为 `O(n)`
- 在数据密集型或 UI 复杂的应用中，频繁更新会导致明显的交互延迟

Million.js 通过改变 diff 的粒度，将 reconciliation 从 `O(n)` 降至 `O(1)`，从根本上减少更新开销。

## 核心原理：Block Virtual DOM

### 传统 Virtual DOM 的问题

React 的 Virtual DOM 在更新时需要：
1. 生成新的虚拟 DOM 树
2. 与旧的虚拟 DOM 树逐节点 diff
3. 找出差异后批量更新真实 DOM

节点越多，diff 耗时越长。Rich Harris 在 2018 年的文章 ["Virtual DOM is pure overhead"](https://svelte.dev/blog/virtual-dom-is-pure-overhead) 中就指出这个问题。

### Block Virtual DOM 的工作方式

Million.js 借鉴了 [Blockdom](https://github.com/ged-odoo/blockdom) 的思路，将更新过程分为两步：

#### 1. 静态分析（Static Analysis）

在编译期或首次渲染时，将组件的虚拟 DOM 树分析为：
- **静态部分**：不会变化的 DOM 结构，直接缓存
- **动态部分（Edit Map）**：提取动态值与对应 DOM 节点的映射关系

```jsx
// 原始组件
function Count() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <ul>
        <li>{count + 1}</li>
        <li>{count + 2}</li>
      </ul>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

经过 Block 包装后，编译器会生成类似这样的更新逻辑：

```jsx
// 编译器生成的伪代码
if (count !== prevCount) {
  li1.textContent = count + 1;  // 直接更新，无需 diff
  li2.textContent = count + 2;
}
```

#### 2. 脏检查（Dirty Checking）

更新时不再 diff 整个虚拟 DOM 树，而是：
- 只比较**数据值**（state/props）是否变化
- 通过 Edit Map 直接定位到需要更新的 DOM 节点
- 跳过所有静态部分

**核心差异**：传统 Virtual DOM diff 的是**结构**，Block Virtual DOM diff 的是**数据**。数据量远小于 DOM 树，且浅比较成本极低。

### 复杂度对比

| 方案 | 更新时复杂度 | 说明 |
|------|-------------|------|
| React Virtual DOM | `O(n)` | n 为虚拟 DOM 节点数 |
| Million.js Block VM | `O(1)` | 仅比较动态数据值 |

## 使用方式

### 安装

```bash
npx million@latest
```

CLI 会自动安装依赖并配置项目。

### 基本用法：`block()` HOC

```jsx
import { block } from 'million/react';

function MyComponent(props) {
  return <div>{props.value}</div>;
}

// 包装为 Block 组件
const BlockComponent = block(MyComponent);
```

### `<For />` 组件：优化列表渲染

```jsx
import { For } from 'million/react';

function List({ items }) {
  return (
    <For each={items}>
      {(item) => <div>{item.name}</div>}
    </For>
  );
}
```

`<For />` 专门针对列表场景优化，避免 `Array.map()` 产生的不必要 diff。

### Automatic Mode

Million.js 还提供自动模式，无需手动包装组件，编译器会自动识别可优化的部分：

```js
// million.config.js
export default {
  automatic: true,
};
```

## 适用场景

### ✅ 适合使用 Block Virtual DOM 的场景

| 场景 | 原因 |
|------|------|
| **大量静态内容 + 少量动态数据** | Block VM 跳过静态部分，只更新动态节点 |
| **数据密集型表格/列表** | 如管理后台、数据看板，频繁更新大量行 |
| **UI 树结构稳定** | Edit Map 只需创建一次，不会因结构变化重建 |
| **React 应用遇到渲染瓶颈** | 无需迁移框架，drop-in 接入 |

```jsx
// ✅ 好：静态结构 + 少量动态值
<div>
  <header>固定标题</header>
  <main>
    <p>{dynamicValue}</p>
    大量静态文本...
  </main>
</div>
```

### ❌ 不适合或收益有限的场景

| 场景 | 原因 |
|------|------|
| **大量动态内容** | 动态值过多时，数据 diff 成本接近甚至超过传统 diff |
| **UI 树结构不稳定** | 非确定性返回会导致 Edit Map 频繁重建 |
| **计算型组件** | 数据计算成本远高于 DOM diff 成本 |

```jsx
// ❌ 差：动态值过多
<div>
  <div>{a}</div>
  <div>{b}</div>
  <div>{c}</div>
  <div>{d}</div>
  <div>{e}</div>
</div>

// ❌ 差：非确定性返回
function Component() {
  return Math.random() > 0.5 ? <div>{x}</div> : <p>fallback</p>;
}

// ❌ 差：计算成本高于 diff
function Component({ a, b, c, d, e }) {
  // 5 个数据值 diff vs 1 个 DOM 节点
  return <div>{a + b + c + d + e}</div>;
}
```

## 限制与注意事项

### 1. 不是银弹

Block Virtual DOM 并非在所有场景下都快。官方明确指出：
> "It's not a silver bullet."

在 JS Framework Benchmark 中的优异表现主要来自对**大规模表格渲染**的优化，不代表所有真实场景都能获得同等提升。

### 2. 组件结构限制

- Block 组件的返回值必须**稳定/确定**（deterministic）
- 条件渲染不同结构的组件会导致性能下降
- 列表场景应使用 `<For />` 而非条件返回

### 3. 不要全局使用

新手常见错误是将所有组件都用 `block()` 包装。正确做法是：
- 识别性能瓶颈组件
- 在静态内容多、更新频繁的部分使用 Block VM
- 小型表单、简单组件保持原生 React 即可

### 4. 与 React.memo 的区别

Block Virtual DOM 不是 `React.memo` 的替代品：
- `React.memo` 解决的是**不必要的重新渲染**（跳过整个组件）
- Block VM 解决的是**渲染内部的 diff 效率**（更新时更快）
- 两者可以配合使用

### 5. 兼容性

- 支持 React 18+
- 兼容 Next.js、Vite 等主流构建工具
- 部分高级功能（如 Automatic Mode）可能有额外限制

## 性能基准

| 指标 | React | Million.js |
|------|-------|-----------|
| JS Framework Benchmark（几何平均） | 基准 | 最高 +70% |
| 更新 1000 行表格 | 较慢 | 显著更快 |

> 数据来源：[JS Framework Benchmark](https://krausest.github.io/js-framework-benchmark/2023/table_chrome_112.0.5615.49.html)（Chrome 112）
> 
> 注意：基准测试不代表真实应用性能，实际提升取决于组件结构和数据量。

## 相关概念

- [[前端性能优化]] — 通用前端性能优化方法论
- [[React 性能优化思路与常用工具]] — React 生态内的性能优化手段
- [[React Fiber 架构详解]] — React 的底层调度与协调机制
- [[渲染管线]] — 浏览器渲染流程，理解 DOM 操作成本的基础

## 参考资料

- [Million.js GitHub 仓库](https://github.com/aidenybai/million)
- [Virtual DOM: Back in Block（官方博客）](https://old.million.dev/blog/virtual-dom)
- [JS Framework Benchmark](https://krausest.github.io/js-framework-benchmark/)
- [Virtual DOM is pure overhead（Rich Harris, Svelte 博客）](https://svelte.dev/blog/virtual-dom-is-pure-overhead)
- [Blockdom（Million.js 灵感来源）](https://github.com/ged-odoo/blockdom)
