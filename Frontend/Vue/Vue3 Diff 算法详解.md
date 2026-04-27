---
created: 2026-04-27
updated: 2026-04-27
tags:
  - Vue
  - Diff算法
  - 虚拟DOM
  - 前端原理
  - 渲染机制
aliases:
  - Vue3 Diff Algorithm
  - Vue3 虚拟DOM比对
  - Vue3 协调算法
  - Vue3 Reconciliation
source_type: mixed
source_urls:
  - https://cn.vuejs.org/guide/extras/rendering-mechanism.html
  - https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts
  - https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts
status: verified
---

## 什么是 Diff 算法

Diff 算法（差异比对算法，也称协调算法 Reconciliation）是虚拟 DOM 框架的核心机制之一。当组件状态变化触发重新渲染时，框架会生成一棵新的虚拟 DOM 树，然后将其与旧的虚拟 DOM 树进行比对，找出最小差异，最终只将必要的变更应用到真实 DOM 上。

Vue 3 的 Diff 算法在 Vue 2 的基础上进行了重大优化，核心差异在于引入了**编译时信息**，使运行时 Diff 能够跳过大量静态节点，只处理真正需要更新的部分。

---

## 为什么需要 Diff

响应式系统解决了"何时更新"的问题，但无法解决"更新什么"的问题。一个组件可能包含大量 DOM 节点，数据变化往往只影响其中一小部分。如果每次状态变化都全量重建 DOM，会导致：

- 大量不必要的 DOM 操作（性能瓶颈）
- 丢失 DOM 状态（如输入框焦点、滚动位置）
- 不必要的重排重绘

Diff 算法通过虚拟 DOM 的内存比对，将 DOM 操作次数降到最低。

---

## Vue 3 Diff 的核心设计

### 编译时优化 + 运行时 Diff

Vue 3 的 Diff 策略与 React 有本质区别：

| 特性 | Vue 3 | React |
|------|-------|-------|
| 优化阶段 | 编译时 + 运行时 | 纯运行时 |
| Diff 范围 | 仅带 PatchFlags 的动态节点 | 全量遍历组件子树 |
| 静态节点处理 | 编译时缓存，运行时完全跳过 | 每次渲染重新创建 vnode |
| 更新粒度 | 组件内细粒度（节点级） | 组件级（默认整棵子树） |

Vue 同时控制编译器和运行时，这使得编译器可以在生成渲染函数时注入优化信息，运行时利用这些信息走捷径。

---

## PatchFlags（更新类型标记）

PatchFlags 是 Vue 3 编译器生成的优化标记，编码在 vnode 创建调用中，用于指导运行时 Diff 的行为。

### 完整 PatchFlags 定义

来源：[vuejs/core - patchFlags.ts](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts)

| 标志名 | 值 | 含义 |
|--------|-----|------|
| `TEXT` | `1` | 动态文本内容 |
| `CLASS` | `2` | 动态 class 绑定 |
| `STYLE` | `4` | 动态 style 绑定 |
| `PROPS` | `8` | 动态非 class/style 属性（带动态 key 数组） |
| `FULL_PROPS` | `16` | 动态属性且 key 不确定，需要完整 diff |
| `NEED_HYDRATION` | `32` | 需要 SSR 激活 |
| `STABLE_FRAGMENT` | `64` | 子节点顺序不变的 Fragment |
| `KEYED_FRAGMENT` | `128` | 带 key 或部分带 key 的 Fragment |
| `UNKEYED_FRAGMENT` | `256` | 无 key 的 Fragment |
| `NEED_PATCH` | `512` | 仅需非属性 patch（ref、指令钩子） |
| `DYNAMIC_SLOTS` | `1024` | 动态插槽，组件强制更新 |
| `DEV_ROOT_FRAGMENT` | `2048` | 开发环境根 Fragment（含注释） |
| `CACHED` | `-1` | 缓存的静态 vnode，完全跳过 diff |
| `BAIL` | `-2` | 退出优化模式，完整 diff（如手写渲染函数） |

### 位运算检查

运行时通过位运算快速判断需要执行哪些更新操作：

```js
if (vnode.patchFlag & PatchFlags.CLASS) {
  // 仅更新 class，跳过其他属性
}
if (vnode.patchFlag & PatchFlags.TEXT) {
  // 仅更新文本内容
}
```

### 编译示例

```vue
<!-- 模板 -->
<div :class="{ active }">{{ message }}</div>
```

编译后的渲染函数：

```js
createElementVNode("div", {
  class: _normalizeClass({ active: _ctx.active })
}, _toDisplayString(_ctx.message), 3 /* TEXT | CLASS */)
```

参数 `3` 是 `TEXT(1) | CLASS(2)` 的位运算结果，运行时只需比对 class 和文本内容。

---

## 树结构打平（Tree Flattening）

Vue 3 引入了**区块（Block）**概念。一个区块是内部结构稳定的节点集合，每个区块追踪其所有带 PatchFlags 的后代节点（不只是直接子节点）。

```vue
<div> <!-- 根区块 -->
  <div>...</div>         <!-- 静态，不追踪 -->
  <div :id="id"></div>   <!-- 动态，追踪 -->
  <div>                  <!-- 静态，不追踪 -->
    <div>{{ bar }}</div> <!-- 动态，追踪 -->
  </div>
</div>
```

编译结果打平为动态节点数组：

```
div (block root)
- div 带有 :id 绑定
- div 带有 {{ bar }} 绑定
```

重渲染时只需遍历这个打平的数组，而非整棵虚拟 DOM 树。`v-if` 和 `v-for` 会创建新的子区块，子区块在父区块的动态子节点数组中被追踪。

---

## 列表 Diff 算法

当处理带 key 的列表节点时，Vue 3 使用**最长递增子序列（LIS, Longest Increasing Subsequence）**算法来最小化 DOM 移动操作。

### 双端比较策略

新旧列表同时从两端向中间比较，处理以下四种情况：

1. **头头相同**：旧头与新头 key 相同，指针同时后移
2. **尾尾相同**：旧尾与新尾 key 相同，指针同时前移
3. **头尾交叉**：旧头与新尾 key 相同，需要移动节点到末尾
4. **尾头交叉**：旧尾与新头 key 相同，需要移动节点到开头

### 最长递增子序列优化

当双端比较无法匹配时，剩余节点需要计算最优移动方案：

```js
// 伪代码
const oldKeyToIndex = buildKeyToIndex(oldChildren)
const newIndexToOldIndexMap = [] // 新索引 → 旧索引的映射
let moved = false
let maxNewIndexSoFar = 0

for (let i = 0; i < newChildren.length; i++) {
  const newChild = newChildren[i]
  const oldIndex = oldKeyToIndex.get(newChild.key)
  
  if (oldIndex === undefined) {
    // 新节点，需要挂载
    mount(newChild)
  } else {
    // 已存在节点，需要比对和可能移动
    newIndexToOldIndexMap[i] = oldIndex + 1
    if (oldIndex < maxNewIndexSoFar) {
      moved = true
    }
    maxNewIndexSoFar = Math.max(maxNewIndexSoFar, oldIndex)
    patch(oldChildren[oldIndex], newChild)
  }
}

if (moved) {
  // 计算最长递增子序列，确定哪些节点不需要移动
  const lis = getLongestIncreasingSubsequence(newIndexToOldIndexMap)
  // 根据 LIS 结果执行最小移动操作
}
```

### LIS 算法示例

假设旧列表 `[A, B, C, D, E]` 变为新列表 `[A, E, B, C, D]`：

1. 构建映射：`newIndexToOldIndexMap = [1, 5, 2, 3, 4]`（1-based，0 表示新节点）
2. 计算 LIS：`[1, 2, 3, 4]`（对应 A, B, C, D）
3. 只有 E 需要移动，其余节点保持原位

这比逐个比较移动的效率更高，时间复杂度为 `O(n log n)`。

---

## Diff 完整流程

### 组件更新触发

```
响应式数据变化
  ↓
触发组件的响应式副作用（render effect）
  ↓
执行渲染函数，生成新的虚拟 DOM 树
  ↓
调用 patch(oldVNode, newVNode)
```

### patch 函数核心逻辑

```js
function patch(oldVNode, newVNode, container) {
  // 1. 类型不同，直接替换
  if (oldVNode.type !== newVNode.type) {
    unmount(oldVNode)
    mount(newVNode, container)
    return
  }
  
  // 2. 类型相同，根据节点类型分发
  switch (newVNode.shapeFlag) {
    case ShapeFlags.TEXT:
      patchText(oldVNode, newVNode)
      break
    case ShapeFlags.ELEMENT:
      patchElement(oldVNode, newVNode)
      break
    case ShapeFlags.COMPONENT:
      patchComponent(oldVNode, newVNode)
      break
    case ShapeFlags.TEXT_CHILDREN:
      patchChildren(oldVNode, newVNode)
      break
  }
}
```

### patchElement 优化路径

```js
function patchElement(oldVNode, newVNode) {
  const el = newVNode.el = oldVNode.el
  const oldProps = oldVNode.props
  const newProps = newVNode.props
  const patchFlag = newVNode.patchFlag
  
  // 有 patchFlag 走优化路径
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.CLASS) {
      // 仅更新 class
    }
    if (patchFlag & PatchFlags.STYLE) {
      // 仅更新 style
    }
    if (patchFlag & PatchFlags.TEXT) {
      // 仅更新文本
    }
    if (patchFlag & PatchFlags.PROPS) {
      // 更新动态属性（使用 compiler 提供的 dynamicProps 数组）
      for (const key of newVNode.dynamicProps) {
        patchProp(el, key, oldProps[key], newProps[key])
      }
    }
  } else if (patchFlag === 0) {
    // 无 patchFlag，完整比对所有 props
    patchProps(el, oldProps, newProps)
  }
}
```

---

## Vue 2 vs Vue 3 Diff 对比

| 特性 | Vue 2 | Vue 3 |
|------|-------|-------|
| Diff 策略 | 单端遍历 + 双端查找 | 双端比较 + LIS 优化 |
| 静态节点处理 | 每次重新创建 vnode | 编译时缓存，运行时复用 |
| 动态节点识别 | 运行时全量比对 | 编译时 PatchFlags 标记 |
| 列表更新 | 基于 key 的 map 查找 | map 查找 + LIS 最小移动 |
| 树遍历方式 | 递归遍历整棵树 | 区块打平，仅遍历动态节点 |
| 性能瓶颈 | 大模板全量 diff | 仅 diff 动态部分 |

---

## 常见误区

1. **"Vue 3 的 Diff 算法和 Vue 2 完全不同"**：不准确。核心的同层比较、key 匹配策略保持一致，主要差异在于编译时优化和列表 Diff 的 LIS 优化

2. **"有 key 就一定比没 key 快"**：不一定。如果列表顺序稳定且无插入/删除，无 key 的同位比较可能更快。key 的价值在于处理乱序、插入、删除场景

3. **"index 作为 key 总是错误的"**：过于绝对。如果列表是静态的（不会重排序、插入、删除），使用 index 作为 key 是安全的。但涉及动态操作时必须使用稳定唯一 key

4. **"Vue 3 不需要 key"**：错误。虽然编译时优化减少了 Diff 范围，但列表节点的移动和复用仍然依赖 key 进行正确匹配

5. **"PatchFlags 是运行时计算的"**：错误。PatchFlags 是编译时由模板编译器静态分析后注入的，运行时只做位运算检查

---

## 最佳实践

1. **优先使用模板而非渲染函数**：模板能让编译器注入 PatchFlags 等优化信息，手写渲染函数会触发 `BAIL` 标志退出优化模式

2. **列表使用稳定唯一的 key**：避免使用 index 作为 key（除非列表完全静态），推荐使用数据 ID

3. **善用 `v-once` 和 `v-memo`**：对于确实不需要更新的子树，显式标记跳过 Diff

4. **避免在模板中使用复杂表达式**：复杂表达式可能导致编译器无法准确推断更新类型，降级为 `FULL_PROPS`

5. **长列表使用虚拟滚动**：Diff 算法再高效也无法突破 DOM 节点数量的物理限制，超大数据量应使用虚拟滚动

---

## 相关概念

- **虚拟 DOM（VDOM）**：用 JavaScript 对象描述 DOM 结构的编程模式
- **渲染函数（Render Function）**：返回虚拟 DOM 树的函数，由模板编译或手写
- **响应式副作用（Reactive Effect）**：追踪依赖并在依赖变化时重新执行的机制，组件渲染基于此
- **静态提升（Static Hoisting）**：将静态 vnode 创建提升到渲染函数外部，避免重复创建
- **Fiber 架构**：React 的可中断渲染实现，Vue 3 因细粒度更新不需要此架构

---

## 参考资料

- [Vue 3 渲染机制 - 官方文档](https://cn.vuejs.org/guide/extras/rendering-mechanism.html)
- [Vue 3 PatchFlags 源码定义](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts)
- [Vue 3 渲染器核心实现](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts)
- [Vue 3 模板编译器](https://github.com/vuejs/core/tree/main/packages/compiler-core)
- [Vue 3 渲染机制（英文版）](https://vuejs.org/guide/extras/rendering-mechanism.html)
