---
created: "2026-04-20"
updated: "2026-04-20"
tags:
  - React
  - 事件系统
  - SyntheticEvent
  - EventDelegation
  - 前端
aliases:
  - React Event System
  - React 合成事件
  - React 事件委托
source_type: official-doc
source_urls:
  - https://react.dev/blog/2024/04/25/react-19
  - https://react.dev/blog/2024/04/25/react-19-upgrade-guide
  - https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html
  - https://legacy.reactjs.org/docs/events.html
  - https://legacy.reactjs.org/docs/legacy-event-pooling.html
  - https://react.dev/reference/react-dom/components/common
status: verified
---

React 事件系统是框架的核心基础设施，提供跨浏览器的事件标准化和性能优化机制。理解其设计思路和演变历史有助于编写更可靠的事件处理代码。

## 核心概念

### SyntheticEvent（合成事件）

React 不直接使用浏览器原生事件，而是创建 `SyntheticEvent` 对象——一个跨浏览器的事件包装器：

- 具有与原生事件相同的接口（`stopPropagation()`、`preventDefault()` 等）
- 在所有浏览器中行为一致
- 通过 `e.nativeEvent` 可访问底层原生事件

```js
function handleClick(e) {
  e.preventDefault();      // 阻止默认行为
  e.stopPropagation();     // 阻止冒泡
  console.log(e.nativeEvent);  // 原生事件对象
}
```

**设计目的**：解决不同浏览器事件实现的细微差异，提供统一的编程接口。

### 事件委托（Event Delegation）

React 采用事件委托策略，而非将事件监听器直接绑定到每个 DOM 元素：

| 版本 | 事件监听器挂载位置 |
|------|-------------------|
| React ≤16 | `document` 节点 |
| React ≥17 | React 树的根容器（`rootNode`） |

**React 17 的变化原因**（来源：[React v17 RC 公告](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html)）：

1. **支持渐进式升级**：允许多个 React 版本共存于同一页面
2. **改善与非 React 代码的集成**：`e.stopPropagation()` 行为更符合预期
3. **修复历史问题**：解决与 jQuery 等框架集成时的事件传播混乱

```js
// React 17+
const rootNode = document.getElementById('root');
ReactDOM.render(<App />, rootNode);
// 事件监听器挂载在 rootNode 上，而非 document
```

## 事件池（Event Pooling）的演变

### React ≤16：事件池机制

早期版本为优化性能，复用事件对象：

```js
// React 16 中这段代码会报错
function handleChange(e) {
  setTimeout(() => {
    console.log(e.target.value);  // 错误：e.target 已被清空
  }, 100);
}
```

解决方案：调用 `e.persist()` 保留事件对象：

```js
function handleChange(e) {
  e.persist();  // 阻止 React 重置事件属性
  setTimeout(() => {
    console.log(e.target.value);  // 正常工作
  }, 100);
}
```

### React ≥17：移除事件池

**完全移除事件池机制**（来源：[React v17 RC 公告](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#no-event-pooling)）：

- 现代浏览器性能已足够，事件池优化收益不明显
- 消除开发者困惑（`e.persist()` 是常见绊脚石）
- `e.persist()` 方法仍存在，但不再执行任何操作

```js
// React 17+ 中可直接使用
function handleChange(e) {
  setTimeout(() => {
    console.log(e.target.value);  // 正常工作，无需 persist()
  }, 100);
}
```

## React 17+ 事件行为调整

### 不冒泡的事件

| 事件 | React 行为 | 浏览器原生行为 |
|------|-----------|---------------|
| `onScroll` | 不冒泡 | 不冒泡（React 17 开始匹配浏览器） |

来源：[React v17 RC 公告](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html)

### 焦点事件

React `onFocus` 和 `onBlur` 在底层使用 `focusin` 和 `focusout`：

- `focusin`/`focusout` 本身会冒泡
- React 保持 `onFocus`/`onBlur` 的冒泡行为（更实用的默认值）
- 提供更多上下文信息（如 `relatedTarget`）

### 捕获阶段事件

`onClickCapture` 等事件在 React 17+ 使用真正的浏览器捕获阶段监听器，而非模拟实现。

## React 19 事件相关特性

React 19 对传统事件系统无重大改动，新增的 **Actions** 特性与表单提交相关：

### Form Actions

`<form>` 元素支持 `action` prop 接收函数：

```js
<form action={async (formData) => {
  await updateName(formData.get("name"));
}}>
  <input name="name" />
  <button type="submit">提交</button>
</form>
```

来源：[React 19 发布公告](https://react.dev/blog/2024/04/25/react-19)

### 相关 Hooks

- `useActionState`：管理 Action 的状态和结果
- `useOptimistic`：乐观更新 UI
- `useFormStatus`：获取表单提交状态

这些特性简化异步数据提交，不改变核心事件系统。

## 支持的事件类型

React 支持的标准事件类型（来源：[React 官方文档](https://react.dev/reference/react-dom/components/common)）：

| 分类 | 事件示例 |
|------|---------|
| Clipboard | `onCopy`, `onCut`, `onPaste` |
| Composition | `onCompositionStart`, `onCompositionEnd`, `onCompositionUpdate` |
| Keyboard | `onKeyDown`, `onKeyPress`（已弃用）, `onKeyUp` |
| Focus | `onFocus`, `onBlur` |
| Form | `onChange`, `onInput`, `onSubmit`, `onReset` |
| Mouse | `onClick`, `onMouseDown`, `onMouseUp`, `onMouseEnter`, `onMouseLeave` |
| Pointer | `onPointerDown`, `onPointerMove`, `onPointerUp` |
| Touch | `onTouchStart`, `onTouchEnd`, `onTouchMove`, `onTouchCancel` |
| Animation | `onAnimationStart`, `onAnimationEnd`, `onAnimationIteration` |
| Transition | `onTransitionEnd` |

每个事件都支持 Capture 版本（如 `onClickCapture`）。

## 常见问题与解决方案

### 异步访问事件对象

| 版本 | 解决方案 |
|------|---------|
| React ≤16 | 调用 `e.persist()` 或立即读取所需属性 |
| React ≥17 | 直接访问，无需特殊处理 |

### 与非 React 代码集成

React 17+ 改善了事件边界处理：

- React 内调用 `e.stopPropagation()` 不会影响外部框架的事件监听
- 外部调用 `stopPropagation()` 不会影响 React 树内的事件处理

### 获取原生事件信息

```js
function handleClick(e) {
  // React 事件类型可能与原生事件不同
  // 例如 onMouseLeave 对应原生 mouseout
  console.log(e.type);           // React 事件类型
  console.log(e.nativeEvent.type);  // 原生事件类型
}
```

## 设计思路总结

| 设计决策 | 目的 |
|---------|------|
| SyntheticEvent | 跨浏览器一致性 |
| 事件委托 | 性能优化（减少监听器数量） |
| 根节点委托（React 17+） | 支持多版本共存、改善集成 |
| 移除事件池（React 17+） | 消除开发者困惑、现代浏览器已无需此优化 |
| 冒泡行为标准化 | 更接近原生 DOM 行为 |

## 参考资料

- [React v17 Release Candidate](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html)
- [React v19 发布公告](https://react.dev/blog/2024/04/25/react-19)
- [React v19 升级指南](https://react.dev/blog/2024/04/25/react-19-upgrade-guide)
- [Common components 文档](https://react.dev/reference/react-dom/components/common)
- [Legacy Event Pooling 文档](https://legacy.reactjs.org/docs/legacy-event-pooling.html)
- [SyntheticEvent 文档](https://legacy.reactjs.org/docs/events.html)