---
created: 2026-04-27
updated: 2026-04-27
tags:
  - react
  - diff
  - reconciliation
  - virtual-dom
  - frontend
  - internals
aliases:
  - React Diff
  - React Reconciliation
  - React 协调算法
  - React 差异比较
source_type: mixed
source_urls:
  - https://legacy.reactjs.org/docs/reconciliation.html
  - https://react.dev/learn/preserving-and-resetting-state
  - https://github.com/acdlite/react-fiber-architecture
  - https://github.com/facebook/react/tree/main/packages/react-reconciler
status: verified
---

## 是什么

React Diff 算法（也称 **Reconciliation / 协调算法**）是 React 用于比较新旧两棵 React Element 树、计算最小 DOM 操作集的核心算法。它的目标是以尽可能少的真实 DOM 操作将 UI 从当前状态更新到目标状态。

> React Element 是不可变的 JS 对象（JSX 编译产物），描述"在某一时刻想在屏幕上看到什么"。Diff 算法比较的是这些对象树，而非真实 DOM。

## 为什么需要 Diff 算法

将一棵树转换为另一棵树的**最小编辑距离**问题，通用算法复杂度为 **O(n³)**（n 为节点数）。对于 1000 个元素的树，需要约 10 亿次比较，完全不可接受。

React 基于两个合理假设，将复杂度降低到 **O(n)**：

| 假设 | 说明 |
|------|------|
| **不同类型元素产生不同树** | 元素类型（tag / 组件）不同时，直接销毁旧树、重建新树 |
| **开发者通过 `key` 标记稳定子元素** | `key` 帮助 React 识别列表中哪些子元素跨渲染保持稳定 |

这两个假设在几乎所有实际场景中都成立，牺牲了极端情况下的最优解，换取了工程上的高效表现。

## 核心规则

### 1. 同层比较（同一层级才做 Diff）

React **只比较同一层级**的节点，不会跨层级移动节点。这是 O(n) 策略的关键前提。

```jsx
// 旧树                    // 新树
<div>                      <section>
  <Counter />                <Counter />
</div>                     </section>
```

虽然 `Counter` 组件本身没变，但其父节点从 `<div>` 变为 `<section>`（类型不同），React 会**销毁整个子树**（包括 `Counter` 及其状态），然后重新挂载。

### 2. 元素类型不同 → 整棵子树销毁重建

当比较的两个元素类型不同时：

- 销毁旧 DOM 节点及其所有子节点
- 触发旧组件的 `componentWillUnmount` / cleanup 函数
- 创建新 DOM 节点
- 触发新组件的挂载生命周期（`componentDidMount` / `useEffect` mount）
- **旧树关联的所有状态丢失**

```jsx
// 类型不同，Counter 会被销毁重建
<div>
  <Counter />
</div>

<span>
  <Counter />
</span>
```

即使两个 `Counter` 看起来完全一样，React 也不认为它们是同一个组件——因为它们在树中的"地址"（父节点类型）不同。

### 3. 元素类型相同 → 保留节点、更新属性

**DOM 元素类型相同**（如 `div` → `div`）：

- 保留底层 DOM 节点
- 仅更新变化的属性（`className`、`style`、事件监听器等）
- 递归比较子节点

```jsx
// 旧                          // 新
<div className="before" />     <div className="after" />
```

React 知道只需修改 `className`，不会重建整个 `<div>`。

**组件元素类型相同**（如 `Counter` → `Counter`）：

- 组件实例保持不变（状态保留）
- 更新 props
- 调用 `render()` / 重新执行函数组件
- 递归比较 render 输出

### 4. 文本节点的处理

文本节点被视为一种特殊类型的元素。文本内容变化时，React 直接更新 DOM 节点的 `textContent`，不涉及子树比较。

```jsx
// 文本变化：直接更新 DOM textContent
<p>Hello</p>   →   <p>World</p>
```

## 列表 Diff 算法

### 默认行为：按索引逐位比较

当父元素的子元素列表发生变化时，React 默认**同时遍历新旧两个子节点列表**，按索引位置逐位比较：

```jsx
// 旧                         // 新
<ul>                          <ul>
  <li>first</li>                <li>first</li>
  <li>second</li>               <li>second</li>
</ul>                           <li>third</li>
                              </ul>
```

末尾添加元素时表现良好——前两个 `<li>` 匹配，第三个插入。

### 问题场景：头部插入

```jsx
// 旧                         // 新
<ul>                          <ul>
  <li>Duke</li>                 <li>Connecticut</li>
  <li>Villanova</li>            <li>Duke</li>
</ul>                           <li>Villanova</li>
                              </ul>
```

没有 `key` 时，React 按索引比较：

| 索引 | 旧 | 新 | 操作 |
|------|----|----|------|
| 0 | Duke | Connecticut | 更新 |
| 1 | Villanova | Duke | 更新 |
| 2 | (无) | Villanova | 插入 |

**所有子节点都被修改**，而非理想的一次插入。这在列表较长时造成严重性能问题。

### key 的作用机制

`key` 是 React 用于**跨渲染识别同一元素**的特殊属性。有了 `key` 后：

```jsx
// 旧                                 // 新
<ul>                                  <ul>
  <li key="2015">Duke</li>              <li key="2014">Connecticut</li>
  <li key="2016">Villanova</li>         <li key="2015">Duke</li>
</ul>                                   <li key="2016">Villanova</li>
                                      </ul>
```

React 使用 `key` 匹配新旧元素：

- `key="2014"` → 新元素，执行**插入**
- `key="2015"` → 匹配成功，**复用**（可能移动位置）
- `key="2016"` → 匹配成功，**复用**

仅需一次插入操作，性能大幅提升。

### key 的要求

| 要求 | 说明 |
|------|------|
| **兄弟节点内唯一** | key 只需在同级兄弟中唯一，不需要全局唯一 |
| **稳定** | 同一数据项在不同渲染中应使用相同的 key |
| **可预测** | 不应使用 `Math.random()` 等随机值 |

### 为什么不建议用 index 作为 key

| 场景 | 使用 index 的问题 |
|------|------------------|
| **列表头部插入** | 所有后续元素 index 变化，导致全部重建 |
| **列表排序/过滤** | index 变化导致 React 误判元素变化 |
| **组件含内部状态** | 状态会跟随 index 而非数据项，造成**状态错位** |

**状态错位示例：** 列表项包含未受控输入框，使用 index 做 key 后删除中间项，输入框内容会"跟随"到错误的行上。

**可以使用 index 的唯一场景：** 列表是静态的、永远不会重新排序或过滤。

## 状态保留与重置规则

Diff 算法直接决定了组件状态的去留：

| 条件 | 状态行为 |
|------|----------|
| 同一组件类型 + 同一树位置 | **保留**状态 |
| 不同组件类型（同一位置） | **重置**状态（销毁重建） |
| 同一组件类型 + 不同树位置 | **重置**状态 |
| 显式指定不同 `key` | **重置**状态（视为不同组件） |

### 通过 key 强制重置状态

`key` 不仅用于列表，也可用于**任何需要强制重建的组件**：

```jsx
// 切换联系人时，Chat 组件完全重建（清空输入框）
<Chat key={contact.id} contact={contact} />
```

这在表单场景中非常有用——确保切换上下文时不会残留上一个上下文的输入状态。

## Fiber 架构下的 Diff 实现

React 16+ 的 Fiber 架构中，Diff 发生在 **Render 阶段**（可中断的纯计算阶段）。

### Fiber 节点的 Diff 流程

```
1. 从 workInProgress 树的根节点开始
2. 对每个 Fiber 节点：
   a. 比较 current fiber 的 memoizedProps 与新元素的 pendingProps
   b. 比较 element.type（决定复用还是重建）
   c. 对于子节点列表，使用 key 进行匹配
   d. 标记副作用（Placement / Update / Deletion）
3. 递归处理子节点
4. 收集所有副作用到 effect list
```

### 列表 Diff 中的最长递增子序列优化

React 在 Fiber 实现中对列表 Diff 做了进一步优化：当新旧列表都有 `key` 时，React 会计算**最长递增子序列（LIS）**，最小化需要移动的节点数量。

```jsx
// 旧 key 顺序: [A, B, C, D, E]
// 新 key 顺序: [A, C, B, E, D]
// LIS: [A, B, E] 或 [A, C, D]
// 只需移动不在 LIS 中的节点
```

这减少了不必要的 DOM 移动操作。

## 与 Stack Reconciler 的对比

| 维度 | Stack Reconciler（React 15） | Fiber Reconciler（React 16+） |
|------|------------------------------|-------------------------------|
| **Diff 遍历方式** | 递归（依赖 JS 调用栈） | 循环 + 链表（自定义 fiber 节点） |
| **可中断性** | 不可中断，必须一次性完成 | 可中断、可恢复（以 fiber 为单位） |
| **列表 Diff** | 朴素逐位比较 | 带 key 匹配 + LIS 优化 |
| **优先级** | 所有更新同等对待 | 支持多优先级调度 |

## 常见误区

| 误区 | 澄清 |
|------|------|
| "React 的 Diff 是完整的 O(n³) 算法" | React 使用**启发式 O(n)** 策略，基于类型比较和 key 匹配 |
| "key 用于提升渲染速度" | key 的核心作用是**正确识别元素**，性能优化是附带结果 |
| "index 做 key 只是性能问题" | index 做 key 还会导致**状态错位**，是正确性问题 |
| "Diff 发生在 DOM 上" | Diff 比较的是 **React Element 对象树**，不是真实 DOM |
| "Fiber 改变了 Diff 的规则" | Fiber 改变的是 Diff 的**执行方式**（可中断），核心规则不变 |
| "React 会跨层级移动节点" | React **只做同层比较**，不会将节点从一层移到另一层 |

## 最佳实践

| 场景 | 建议 |
|------|------|
| 列表渲染 | 使用数据中稳定且唯一的 ID 作为 key |
| 条件渲染切换组件 | 如需保留状态，保持组件类型和位置一致 |
| 条件渲染切换组件 | 如需重置状态，使用不同 key 或不同组件类型 |
| 表单切换上下文 | 使用 `key={contextId}` 强制重建，避免状态残留 |
| 避免嵌套组件定义 | 嵌套定义会导致每次渲染产生"不同"的组件函数，状态被意外重置 |

## 相关概念

- [[React Fiber 架构详解]] — Fiber 是 Diff 算法的运行环境，提供可中断的增量渲染
- [[React 面试原理与考点详解]] — 包含 Diff 算法的面试考点速查
- [[前端性能优化]] — 合理使用 key 和组件结构可减少不必要的 Diff 开销

## 参考资料

- [Reconciliation - React 官方文档（legacy）](https://legacy.reactjs.org/docs/reconciliation.html) — Diff 算法的完整官方说明，包含 O(n) 策略的两大假设、元素类型比较规则、key 的使用
- [Preserving and Resetting State - React 官方文档](https://react.dev/learn/preserving-and-resetting-state) — React 18 文档，状态保留与重置规则、key 的进阶用法
- [acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture) — React 核心成员 Andrew Clark 撰写的 Fiber 架构说明
- [facebook/react - react-reconciler](https://github.com/facebook/react/tree/main/packages/react-reconciler) — React 源码，Fiber reconciler 的 Diff 实现在此包中
