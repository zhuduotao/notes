---
created: "2026-04-27"
updated: "2026-04-27"
tags:
  - Vue
  - 面试
  - 前端
  - 原理
  - 响应式
  - Composition API
aliases:
  - Vue 原理
  - Vue 面试考点
  - Vue 核心机制
  - Vue Internals
source_type: mixed
source_urls:
  - https://cn.vuejs.org/guide/extras/reactivity-in-depth.html
  - https://cn.vuejs.org/guide/extras/composition-api-faq.html
  - https://cn.vuejs.org/guide/essentials/lifecycle.html
  - https://cn.vuejs.org/guide/essentials/computed.html
  - https://cn.vuejs.org/guide/extras/rendering-mechanism.html
  - https://router.vuejs.org/guide/essentials/dynamic-matching.html
  - https://pinia.vuejs.org/
status: verified
---

## 响应式系统原理

### 什么是响应性

响应性是一种编程范式，使我们能够以声明式的方式处理变化。典型例子是 Excel 表格：单元格 A2 通过公式 `= A0 + A1` 定义，当 A0 或 A1 变化时，A2 自动更新。

JavaScript 原生不具备这种能力。要实现响应性，需要解决三个核心问题：

1. **追踪依赖**：当一个变量被读取时进行记录
2. **订阅副作用**：如果变量在副作用函数中被读取，将该函数注册为订阅者
3. **触发更新**：当变量被修改时，通知所有订阅者重新执行

### Vue 2 vs Vue 3 响应式实现

| 特性 | Vue 2（getter/setter） | Vue 3（Proxy） |
|------|----------------------|----------------|
| 实现方式 | `Object.defineProperty` 劫持属性 | `Proxy` 拦截对象所有属性访问 |
| 新增/删除属性 | 需要 `Vue.set` / `Vue.delete` | 原生支持 |
| 数组索引修改 | 需要特殊处理 | 原生支持 |
| 深层嵌套 | 递归遍历所有属性（初始化性能差） | 惰性代理（访问时才转换） |
| Map/Set 支持 | 不支持 | 支持 |

### Vue 3 响应式核心机制

**`reactive()` — 基于 Proxy 的响应式对象**：

```js
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      track(target, key)  // 追踪依赖
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      trigger(target, key)  // 触发更新
    }
  })
}
```

**`ref()` — 基于 getter/setter 的响应式基本类型**：

```js
function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
```

**依赖存储结构**：全局 `WeakMap<target, Map<key, Set<effect>>>`

- `WeakMap` 的 key 是目标对象，值是一个 `Map`
- 内层 `Map` 的 key 是属性名，值是 `Set`（存储所有订阅了该属性的副作用函数）
- 使用 `WeakMap` 确保当目标对象不再被引用时，可以被垃圾回收

### 响应式副作用（Reactive Effect）

Vue 通过 `watchEffect()` 提供创建响应式副作用的能力：

```js
function whenDepsChange(update) {
  const effect = () => {
    activeEffect = effect      // 设置当前活跃副作用
    update()                   // 执行更新（期间触发 track）
    activeEffect = null        // 清除
  }
  effect()
}
```

每个 Vue 组件实例内部都创建了一个响应式副作用，用于渲染和更新 DOM。

### `ref` vs `reactive` 的选择

| 场景 | 推荐 | 原因 |
|------|------|------|
| 基本类型（string/number/boolean） | `ref` | `reactive` 只对对象有效 |
| 对象/数组 | 两者均可 | `ref` 需 `.value` 访问，`reactive` 直接访问 |
| 需要替换整个对象 | `ref` | 直接赋值 `.value = newObj` |
| 需要保持引用不变 | `reactive` | 修改属性即可，引用不变 |
| 从函数返回 | `ref` | 解构后仍保持响应性 |

**`reactive()` 的局限性**：
- 解构或赋值给局部变量会丢失响应性（不再经过 Proxy）
- `reactive()` 返回的代理对象与原对象 `===` 比较不相等

### 常见面试追问

**Q: Vue 有了数据响应式，为何还需要 Diff？**

响应式系统解决了"何时更新"的问题（知道数据变了），但无法解决"更新什么"的问题。一个组件可能包含大量 DOM 节点，数据变化可能只影响其中一小部分。Diff 算法通过比较新旧虚拟 DOM 树，找出最小差异，只更新必要的 DOM 节点，避免全量重渲染。

**Q: Vue 3 为什么不需要时间分片（Time Slicing）？**

- React 采用组件级别的粗粒度更新，状态变化会触发整个组件子树重新渲染，因此需要 Fiber 架构 + 时间分片来避免长任务阻塞
- Vue 3 通过**编译时优化**（静态标记 PatchFlag、静态提升）+ **运行时细粒度依赖追踪**，在编译阶段就标记出哪些节点是动态的，运行时只更新这些动态节点，天然避免了大规模重渲染，因此不需要时间分片

> 参考：[深入响应式系统 - Vue 官方文档](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html)

---

## Composition API

### 为什么引入 Composition API

**选项式 API 的限制**：
- 同一逻辑关注点的代码被分散在不同选项（`data`、`methods`、`mounted` 等）中
- 大型组件中需要反复上下滚动才能理解一个完整功能
- 提取复用逻辑需要使用 mixins，但 mixins 存在命名冲突、来源不清晰、隐式依赖等问题

**Composition API 的优势**：

| 优势 | 说明 |
|------|------|
| 更好的逻辑复用 | 通过组合式函数（Composables）替代 mixins，解决所有 mixins 缺陷 |
| 更灵活的代码组织 | 同一关注点的代码聚合在一起，降低重构成本 |
| 更好的类型推导 | 基于普通变量和函数，天然类型友好，无需复杂类型体操 |
| 更小的生产包体积 | `<script setup>` 编译为内联函数，变量名可被压缩 |

### 与 React Hooks 的对比

| 特性 | Vue Composition API | React Hooks |
|------|---------------------|-------------|
| 调用时机 | `setup()` 只调用一次 | 每次渲染都重新调用 |
| 调用顺序 | 不限制，可条件调用 | 必须严格按顺序，不可条件调用 |
| 依赖收集 | 自动（响应式系统运行时收集） | 手动（需传依赖数组） |
| 闭包陷阱 | 不存在（只执行一次） | 存在（需 `useRef` 解决） |
| 性能优化 | 自动细粒度更新，无需手动 `memo` | 需手动 `useMemo`、`useCallback` |
| 缓存开销 | 无需手动缓存 | 缓存本身有开销 |

### 组合式函数（Composables）

组合式函数是以 `use` 开头的函数，用于封装和复用状态逻辑：

```js
// useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}
```

**设计原则**：
- 命名以 `use` 开头（约定，便于识别）
- 每次调用返回**独立的状态实例**
- 可组合多个内置 Hook 或其他 Composables
- 负责清理副作用（在 `onUnmounted` 中）

> 参考：[组合式 API 常见问答 - Vue 官方文档](https://cn.vuejs.org/guide/extras/composition-api-faq.html)

---

## 生命周期

### 生命周期钩子对照表

| 阶段 | Options API | Composition API | 说明 |
|------|-------------|-----------------|------|
| 创建前 | `beforeCreate` | 不需要 | `setup()` 本身就在创建前执行 |
| 创建后 | `created` | 不需要 | `setup()` 替代了 `created` |
| 挂载前 | `beforeMount` | `onBeforeMount` | DOM 尚未创建 |
| 挂载后 | `mounted` | `onMounted` | DOM 已创建，可操作 DOM |
| 更新前 | `beforeUpdate` | `onBeforeUpdate` | 数据变化，DOM 尚未更新 |
| 更新后 | `updated` | `onUpdated` | DOM 已更新 |
| 卸载前 | `beforeUnmount` | `onBeforeUnmount` | 组件即将被销毁 |
| 卸载后 | `unmounted` | `onUnmounted` | 组件已销毁，清理资源 |
| 错误捕获 | `errorCaptured` | `onErrorCaptured` | 捕获子孙组件错误 |

### 生命周期执行顺序

```
创建阶段                    更新阶段                    销毁阶段
├── setup()                ├── onBeforeUpdate         ├── onBeforeUnmount
├── onBeforeMount          ├── onUpdated              └── onUnmounted
└── onMounted
```

### 关键注意事项

**1. 生命周期钩子必须同步注册**：

```js
// 错误：异步注册无效
setTimeout(() => {
  onMounted(() => { /* 不会执行 */ })
}, 100)

// 正确：同步注册
onMounted(() => { /* 正常执行 */ })
```

`onMounted` 等钩子需要在 `setup()` 的同步调用栈中注册，Vue 会自动将回调与当前组件实例关联。但回调函数本身可以是异步的：

```js
onMounted(async () => {
  const data = await fetchData()
})
```

**2. 避免在 `setup()` 中使用箭头函数声明生命周期**（Options API 中）：

```js
// Options API 中避免箭头函数
export default {
  mounted: () => {
    // this 不指向组件实例
  }
}
```

Composition API 中无此问题，因为不依赖 `this`。

**3. 父组件监听子组件生命周期**：

```vue
<!-- 父组件中监听子组件 mounted -->
<ChildComponent @vue:mounted="onChildMounted" />
```

### 常见面试追问

**Q: Vue 组件中写的原生 `addEventListener` 要手动销毁吗？**

**需要**。在 `onMounted` 中添加的事件监听器，必须在 `onUnmounted` 中手动移除：

```js
onMounted(() => {
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
```

原因：组件卸载时，Vue 只会清理其内部的响应式订阅和 DOM 引用，不会自动移除原生事件监听器。不移除会导致**内存泄漏**和**幽灵回调**（组件已销毁但回调仍执行）。

> 参考：[生命周期 - Vue 官方文档](https://cn.vuejs.org/guide/essentials/lifecycle.html)

---

## 计算属性与侦听器

### 计算属性（Computed）

**核心特性**：
- 基于响应式依赖**自动缓存**：仅当依赖变化时才重新计算
- getter 中不应有副作用（不修改状态、不发请求、不操作 DOM）
- 返回值视为只读（除非显式定义 setter）

**计算属性 vs 方法**：

```js
// 计算属性 — 有缓存
const msg = computed(() => expensiveCalc(data.value))

// 方法 — 每次渲染都执行
function msg() {
  return expensiveCalc(data.value)
}
```

| 对比项 | `computed` | `methods` |
|--------|-----------|-----------|
| 缓存 | 依赖不变则返回缓存 | 每次渲染都执行 |
| 适用场景 | 耗性能的派生数据 | 事件处理、无缓存需求的计算 |
| 模板中使用 | `{{ msg }}` | `{{ msg() }}` |

**computed 为什么可以依赖另一个 computed？**

因为每个 `computed` 内部都是一个响应式副作用。当 computed A 的 getter 中读取了 computed B 的 `.value` 时，B 的 getter 会被触发，同时 A 订阅了 B 的变化。B 变化时标记 A 为"脏"，下次访问 A 时重新计算。这形成了一个**依赖链**：

```
ref → computed A → computed B → 模板
```

**可写计算属性**（Vue 3.4+ 支持获取上一个值）：

```js
const fullName = computed({
  get() {
    return firstName.value + ' ' + lastName.value
  },
  set(newValue) {
    [firstName.value, lastName.value] = newValue.split(' ')
  }
})
```

### 侦听器（Watch）

**`watchEffect` vs `watch`**：

| 特性 | `watchEffect` | `watch` |
|------|--------------|---------|
| 依赖收集 | 自动（运行时追踪） | 手动（显式指定源） |
| 执行时机 | 立即执行一次 | 默认懒执行（可配 `immediate`） |
| 访问旧值 | 不支持 | 支持 `(newVal, oldVal)` |
| 适用场景 | 副作用依赖多个响应式源 | 需要精确控制监听源和旧值 |

```js
// watchEffect — 自动追踪
watchEffect(() => {
  console.log(count.value, name.value)  // 任一变化都会触发
})

// watch — 显式指定
watch(count, (newVal, oldVal) => {
  console.log(`count changed from ${oldVal} to ${newVal}`)
})
```

**侦听器的常见陷阱**：

```js
// 错误：直接监听 reactive 对象的属性（无法追踪）
watch(state.count, (newVal) => {})

// 正确：用函数返回属性
watch(() => state.count, (newVal) => {})

// 正确：监听整个 reactive 对象（深监听）
watch(state, (newVal) => {})
```

> 参考：[计算属性 - Vue 官方文档](https://cn.vuejs.org/guide/essentials/computed.html)

---

## 组件通信

### 通信方式汇总

| 方式 | 方向 | 适用场景 |
|------|------|----------|
| `props` | 父 → 子 | 传递数据给子组件 |
| `emit` | 子 → 父 | 子组件通知父组件事件 |
| `v-model` | 双向 | 表单组件、双向绑定 |
| `provide / inject` | 祖先 → 后代 | 跨层级传递（避免 props drilling） |
| `ref` | 父 → 子 | 父组件直接调用子组件方法 |
| 事件总线（已移除） | 任意 | Vue 3 移除 `$on/$off/$emit` 实例方法 |
| 状态管理（Pinia） | 全局 | 跨组件共享状态 |

### 事件机制

**Vue 3 的变化**：

Vue 2 中组件实例上有 `$on`、`$off`、`$emit`、`$once` 方法，可用于实现事件总线。但 Vue 3 **移除了**这些实例方法，原因是：
- 事件总线模式导致事件来源不清晰
- 难以追踪事件流向，不利于维护
- 推荐使用 `mitt`、`tiny-emitter` 等第三方库替代

**手写 `$on/$off/$emit/$once`**：

```js
class EventEmitter {
  constructor() {
    this.events = new Map()
  }

  $on(event, fn) {
    const listeners = this.events.get(event) || []
    listeners.push(fn)
    this.events.set(event, listeners)
  }

  $off(event, fn) {
    if (!fn) {
      this.events.delete(event)
      return
    }
    const listeners = this.events.get(event) || []
    const index = listeners.indexOf(fn)
    if (index > -1) listeners.splice(index, 1)
  }

  $emit(event, ...args) {
    const listeners = this.events.get(event) || []
    listeners.forEach(fn => fn(...args))
  }

  $once(event, fn) {
    const onceFn = (...args) => {
      this.$off(event, onceFn)
      fn(...args)
    }
    this.$on(event, onceFn)
  }
}
```

### 父组件监听子组件生命周期

```vue
<!-- 方式 1：@hook: 前缀（Vue 2/3 通用） -->
<Child @hook:mounted="onChildMounted" />

<!-- 方式 2：@vue: 前缀（Vue 3 推荐） -->
<Child @vue:mounted="onChildMounted" />
```

### `provide / inject`

```js
// 祖先组件
import { provide, ref } from 'vue'
const theme = ref('light')
provide('theme', theme)

// 后代组件
import { inject } from 'vue'
const theme = inject('theme')
```

**注意事项**：
- `provide` 的值如果是 `ref`，后代组件 `inject` 后修改会同步到祖先
- 可设置默认值：`inject('theme', 'default')`
- 适合主题、配置等跨层级共享数据

---

## Vue Router

### 路由传参 `query` vs `params`

| 特性 | `query` | `params` |
|------|---------|----------|
| URL 表现 | `/path?name=foo` | `/path/foo` |
| 刷新后保留 | 是 | 是（需配置动态路由） |
| 定义方式 | 无需在路由配置中定义 | 需在路由 `path` 中定义 `:paramName` |
| 可选性 | 始终可选 | 可配置为必需或可选 |
| 适用场景 | 搜索条件、分页、过滤 | 资源 ID（如 `/users/:id`） |

```js
// query 传参
router.push({ path: '/search', query: { keyword: 'vue' } })
// 结果：/search?keyword=vue
// 获取：route.query.keyword

// params 传参
router.push({ name: 'user', params: { id: 123 } })
// 结果：/users/123（需路由配置 { path: '/users/:id' }）
// 获取：route.params.id
```

### 动态路由参数变化时的响应

当从 `/users/1` 导航到 `/users/2` 时，**同一个组件实例会被复用**，生命周期钩子不会重新触发。需要主动响应参数变化：

```js
// 方式 1：watch route.params
watch(() => route.params.id, (newId, oldId) => {
  // 响应变化
})

// 方式 2：导航守卫
onBeforeRouteUpdate(async (to, from) => {
  userData.value = await fetchUser(to.params.id)
})
```

### 导航守卫

| 守卫 | 作用域 | 触发时机 |
|------|--------|----------|
| `beforeEach` | 全局 | 每次导航前 |
| `beforeResolve` | 全局 | 所有组件内守卫和异步路由加载后 |
| `afterEach` | 全局 | 导航完成后（不能阻止导航） |
| `beforeEnter` | 路由独享 | 进入特定路由前 |
| `beforeRouteEnter` | 组件内 | 进入组件前（无法访问 `this`） |
| `beforeRouteUpdate` | 组件内 | 路由变化但组件复用时 |
| `beforeRouteLeave` | 组件内 | 离开组件前 |

> 参考：[Vue Router 官方文档](https://router.vuejs.org/guide/essentials/dynamic-matching.html)

---

## 渲染机制与 Diff 优化

### 编译时优化（Vue 3 核心优势）

Vue 3 的模板编译器会在编译阶段注入优化信息，大幅减少运行时 Diff 的工作量：

**1. 静态标记（PatchFlags）**：

编译器会标记哪些节点是动态的、哪些是静态的：

```vue
<!-- 模板 -->
<div>
  <span>静态文本</span>
  <span>{{ dynamicText }}</span>
</div>
```

编译后，动态节点会被标记：

```js
createElement('span', { text: dynamicText }, PatchFlags.TEXT)
```

运行时 Diff 时，只更新带有 PatchFlags 的节点，跳过静态节点。

**2. 静态提升（Static Hoisting）**：

静态节点会被提升到渲染函数外部，只创建一次：

```js
// 静态节点只创建一次，不在每次渲染时重复创建
const _hoisted_1 = createElement('span', '静态文本')

function render() {
  return createElement('div', [
    _hoisted_1,  // 直接复用
    createElement('span', dynamicText, PatchFlags.TEXT)
  ])
}
```

**3. 事件监听缓存**：

静态的事件监听器会被缓存，避免不必要的更新。

### Diff 算法策略

| 策略 | 说明 |
|------|------|
| 同层比较 | 只比较同一层级的节点，不跨层级 |
| 类型不同直接替换 | 元素类型变化（如 `div` → `p`），销毁旧节点重建 |
| key 优化列表 Diff | 使用 key 匹配新旧节点，减少移动操作 |
| 双端比较 | 同时从新旧列表两端向中间比较，减少移动次数 |

---

## 性能优化

### 编译时优化（自动生效）

- **静态标记**：只更新动态节点
- **静态提升**：静态节点只创建一次
- **事件缓存**：静态事件监听器复用
- **Tree-shaking**：未使用的 API 不会被打包

### 运行时优化

**`v-once`**：只渲染一次，后续不再更新：

```vue
<span v-once>{{ staticData }}</span>
```

**`v-memo`**（Vue 3.2+）：缓存子树，依赖不变时跳过更新：

```vue
<div v-memo="[item.id]">
  {{ item.name }} - {{ item.price }}
</div>
```

**长列表优化**：
- 使用 `vue-virtual-scroller` 等虚拟滚动库
- 只渲染可视区域内的 DOM 节点

**`shallowRef` / `shallowReactive`**：

对于深层嵌套但不需要响应式的大对象，使用浅层响应式避免递归代理开销：

```js
// 只追踪 .value 的替换，内部对象不递归代理
const largeData = shallowRef({ /* 大量数据 */ })
```

### 常见面试追问

**Q: Vue 3 引入 Composition API 后，选项式 API 会被废弃吗？**

不会。官方明确表示选项式 API 仍是 Vue 不可分割的一部分。两者的取舍：

- 选项式 API 适合中小型项目，约定式结构降低上手门槛
- Composition API 适合大型项目，提供更好的逻辑复用和类型推导
- 可以配置编译时标记移除选项式 API 运行时代码，减小包体积

---

## 常见面试考点汇总

### 响应式系统

| 考点 | 核心要点 |
|------|----------|
| Vue 2 响应式原理 | `Object.defineProperty` 劫持 getter/setter，无法检测新增/删除属性 |
| Vue 3 响应式原理 | `Proxy` 拦截所有属性访问 + `ref` 的 getter/setter |
| 依赖存储结构 | `WeakMap<target, Map<key, Set<effect>>>` |
| ref vs reactive | 基本类型用 ref，对象两者均可，需替换整体用 ref |
| 为什么有了响应式还要 Diff | 响应式解决"何时更新"，Diff 解决"更新什么" |
| Vue 3 为何不需要时间分片 | 编译时优化 + 细粒度依赖追踪，天然避免大规模重渲染 |

### Composition API

| 考点 | 核心要点 |
|------|----------|
| 为什么引入 | 解决选项式 API 逻辑分散、mixins 缺陷、更好的 TS 支持 |
| 与 React Hooks 区别 | Vue 只调用一次、自动依赖收集、无闭包陷阱、无需手动 memo |
| 组合式函数规范 | 以 `use` 开头、每次调用独立实例、负责清理副作用 |
| 选项式 API 会被废弃吗 | 不会，两者共存，可按需选择 |

### 生命周期

| 考点 | 核心要点 |
|------|----------|
| setup 与 created 关系 | `setup()` 在 `beforeCreate` 之前执行，替代了 `created` |
| 生命周期注册要求 | 必须在 setup 的同步调用栈中注册 |
| 原生事件监听器 | 必须在 `onUnmounted` 中手动移除，否则内存泄漏 |
| 父组件监听子组件生命周期 | `@vue:mounted` 或 `@hook:mounted` |

### 计算属性与侦听器

| 考点 | 核心要点 |
|------|----------|
| computed 缓存机制 | 依赖不变返回缓存，避免重复计算 |
| computed 链式依赖 | 每个 computed 是响应式副作用，自动订阅上游依赖 |
| watchEffect vs watch | watchEffect 自动追踪、立即执行；watch 显式指定、支持旧值 |
| computed getter 副作用 | getter 中不应修改状态、发请求或操作 DOM |

### 组件通信

| 考点 | 核心要点 |
|------|----------|
| Vue 3 事件总线变化 | 移除 `$on/$off/$emit` 实例方法，推荐第三方库 |
| provide/inject | 跨层级传递，provide ref 可实现双向响应 |
| query vs params | query 在 URL 参数中，params 需配置动态路由 |
| 动态路由参数变化 | 组件实例复用，需 watch 或导航守卫响应 |

### 渲染与性能

| 考点 | 核心要点 |
|------|----------|
| Vue 3 编译时优化 | 静态标记 PatchFlag、静态提升、事件缓存 |
| Diff 策略 | 同层比较、类型不同直接替换、key 优化、双端比较 |
| v-memo 使用场景 | 缓存子树，依赖数组不变时跳过整块更新 |
| 长列表优化方案 | 虚拟滚动、分页加载、shallowRef 减少代理开销 |

---

## 常见误区

1. **"`reactive()` 可以监听所有变化"**：错误。解构 `reactive` 对象的属性会丢失响应性，因为赋值给局部变量后不再经过 Proxy

2. **"computed 和方法没区别"**：错误。computed 有基于依赖的缓存，方法每次渲染都执行。耗性能计算必须用 computed

3. **"Composition API 就是 Vue 版的 React Hooks"**：不准确。虽然灵感来自 Hooks，但 Vue 的响应式模型解决了 Hooks 的闭包陷阱、依赖数组、手动缓存等问题

4. **"Vue 3 的 Diff 比 React 快是因为算法不同"**：不完全。核心差异在于 Vue 通过编译器提前标记动态节点，运行时只需处理带标记的节点，而 React 需要全量遍历

5. **"`watchEffect` 和 `watch` 可以随意替换"**：错误。`watchEffect` 无法获取旧值，且会立即执行一次；需要精确控制监听源或访问旧值时必须用 `watch`

6. **"响应式数据变化后组件立即重新渲染"**：不准确。Vue 的更新是异步的，同一 tick 内的多次数据变化会合并为一次更新（类似 React 的批处理）

---

## 与 React 的对比（常见追问）

| 特性 | Vue 3 | React |
|------|-------|-------|
| 响应式 | 自动依赖追踪（Proxy + 编译时优化） | 手动 state 更新触发渲染 |
| 渲染粒度 | 组件内细粒度（只更新动态节点） | 组件级别（默认渲染整个子树） |
| 时间分片 | 不需要 | 需要（Fiber 架构） |
| Diff 优化 | 编译时静态标记 + 静态提升 | key + 类型比较 |
| 状态管理 | `ref`/`reactive` + Pinia | `useState`/`useReducer` + 外部库 |
| 逻辑复用 | Composables（组合式函数） | Custom Hooks |
| 模板 | 编译时模板（SFC） | JSX（运行时） |
| 性能优化心智负担 | 低（自动细粒度更新） | 高（需手动 memo/useCallback） |

---

## 参考资料

- [深入响应式系统 - Vue 官方文档](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html)
- [组合式 API 常见问答 - Vue 官方文档](https://cn.vuejs.org/guide/extras/composition-api-faq.html)
- [生命周期 - Vue 官方文档](https://cn.vuejs.org/guide/essentials/lifecycle.html)
- [计算属性 - Vue 官方文档](https://cn.vuejs.org/guide/essentials/computed.html)
- [渲染机制 - Vue 官方文档](https://cn.vuejs.org/guide/extras/rendering-mechanism.html)
- [动态路由匹配 - Vue Router 官方文档](https://router.vuejs.org/guide/essentials/dynamic-matching.html)
- [Pinia 官方文档](https://pinia.vuejs.org/)
- [Vue 3 源码 - reactivity 模块](https://github.com/vuejs/core/tree/main/packages/reactivity)
