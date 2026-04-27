---
created: 2026-04-27
updated: 2026-04-27
tags:
  - Vue
  - Composition API
  - 前端
  - 组件化
  - 逻辑复用
  - TypeScript
aliases:
  - Vue 组合式 API
  - Vue3 Composition API
  - Composables
  - setup script
source_type: official-doc
source_urls:
  - https://cn.vuejs.org/guide/extras/composition-api-faq.html
  - https://cn.vuejs.org/guide/reusability/composables.html
  - https://cn.vuejs.org/api/composition-api-setup.html
  - https://cn.vuejs.org/api/sfc-script-setup.html
  - https://cn.vuejs.org/guide/typescript/composition-api.html
status: verified
---

## 什么是 Composition API

Composition API（组合式 API）是 Vue 3 引入的一组 API 集合，允许开发者使用**函数**而非声明式选项的方式书写 Vue 组件。它涵盖以下核心 API：

| API 类别 | 代表 API | 用途 |
|----------|----------|------|
| 响应式 API | `ref()`, `reactive()`, `computed()`, `watch()` | 创建响应式状态、计算属性和侦听器 |
| 生命周期钩子 | `onMounted()`, `onUnmounted()`, `onBeforeUpdate()` | 在组件各生命周期阶段添加逻辑 |
| 依赖注入 | `provide()`, `inject()` | 利用 Vue 的依赖注入系统传递响应式数据 |

在 Vue 3 中，Composition API 通常配合 `<script setup>` 语法在单文件组件（SFC）中使用：

```vue
<script setup>
import { ref, onMounted } from 'vue'

// 响应式状态
const count = ref(0)

// 更改状态、触发更新的函数
function increment() {
  count.value++
}

// 生命周期钩子
onMounted(() => {
  console.log(`The initial count is ${count.value}.`)
})
</script>

<template>
  <button @click="increment">Count is: {{ count }}</button>
</template>
```

> 注意：Composition API 并非函数式编程。它以 Vue 的数据可变、细粒度响应性系统为基础，与函数式编程强调的数据不可变有本质区别。

> 参考：[组合式 API 常见问答 - Vue 官方文档](https://cn.vuejs.org/guide/extras/composition-api-faq.html#what-is-composition-api)

---

## 为什么引入 Composition API

### 1. 更好的逻辑复用

选项式 API 中主要的逻辑复用机制是 **mixins**，但 mixins 存在三个根本性缺陷：

| 缺陷 | 说明 | Composition API 如何解决 |
|------|------|--------------------------|
| 数据来源不清晰 | 多个 mixin 的属性合并到同一实例，无法追溯来源 | 组合式函数返回值显式解构，来源一目了然 |
| 命名空间冲突 | 不同作者的 mixin 可能注册相同属性名 | 解构时可重命名变量，避免冲突 |
| 隐式跨 mixin 通信 | 多个 mixin 依赖共享属性名隐式耦合 | 组合式函数返回值可作为参数传入另一函数，显式传递 |

通过**组合式函数（Composables）**，Vue 提供了清晰、高效的逻辑复用模式。社区也由此诞生了 [VueUse](https://vueuse.org/) 等高质量工具库。

### 2. 更灵活的代码组织

选项式 API 强制将代码分散到 `data`、`methods`、`mounted` 等选项中。当组件逻辑复杂时，**同一逻辑关注点的代码被拆分到文件的不同位置**，需要反复上下滚动才能理解完整功能。

Composition API 允许将同一关注点的代码聚合在一起，并可轻松抽取为外部组合式函数，大幅降低大型项目的重构成本。

### 3. 更好的类型推导

选项式 API 设计于 2013 年，未考虑类型推导，需要复杂的类型体操才能实现 TypeScript 支持，且在处理 mixins 和依赖注入时仍有局限。

Composition API 基于普通变量和函数，天然类型友好。大多数情况下，TypeScript 编写的代码与 JavaScript 几乎无差别，纯 JS 用户也能从 IDE 中享受部分类型推导。

### 4. 更小的生产包体积

`<script setup>` 编译为内联函数，模板可直接访问其中定义的变量，无需通过 `this` 代理。本地变量名可被压缩，而对象属性名不能，因此对代码压缩更友好。

> 参考：[组合式 API 常见问答 - Vue 官方文档](https://cn.vuejs.org/guide/extras/composition-api-faq.html#why-composition-api)

---

## 核心 API 详解

### 响应式基础

#### `ref()` — 响应式基本类型

```js
import { ref } from 'vue'

const count = ref(0)
console.log(count.value) // 访问需要 .value
count.value++
```

- 适用于基本类型（`string`、`number`、`boolean`）和对象
- 在 `<script setup>` 模板中使用时，**无需 `.value`**，自动解包
- 从函数返回时，解构后仍保持响应性

#### `reactive()` — 响应式对象

```js
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Vue' })
state.count++ // 直接访问，无需 .value
```

- 仅适用于对象类型（对象、数组、Map、Set 等）
- **局限性**：
  - 解构属性会丢失响应性：`const { count } = state` → `count` 变为普通值
  - 替换整个对象会丢失响应性连接：`state = reactive({ count: 1 })` 无效

#### `ref` vs `reactive` 选择建议

| 场景 | 推荐 | 原因 |
|------|------|------|
| 基本类型 | `ref` | `reactive` 只对对象有效 |
| 对象/数组 | 两者均可 | `ref` 需 `.value`，`reactive` 直接访问 |
| 需要替换整个对象 | `ref` | 直接赋值 `.value = newObj` |
| 需要保持引用不变 | `reactive` | 修改属性即可，引用不变 |
| 从函数返回 | `ref` | 解构后仍保持响应性 |

> Vue 官方推荐：**优先使用 `ref()`** 作为声明响应式状态的主要 API。

### 计算属性与侦听器

#### `computed()` — 派生状态

```js
import { ref, computed } from 'vue'

const count = ref(0)
const double = computed(() => count.value * 2)
```

- **基于依赖自动缓存**：依赖不变时返回缓存值
- getter 中不应有副作用（不修改状态、不发请求）
- 支持定义 setter 实现可写计算属性

#### `watch()` — 显式侦听

```js
import { ref, watch } from 'vue'

const count = ref(0)

watch(count, (newVal, oldVal) => {
  console.log(`count changed from ${oldVal} to ${newVal}`)
})
```

- 显式指定侦听源
- 支持访问旧值
- 默认懒执行（可配置 `immediate: true`）

#### `watchEffect()` — 自动追踪

```js
import { ref, watchEffect } from 'vue'

const count = ref(0)

watchEffect(() => {
  console.log(`count is ${count.value}`) // 自动追踪 count
})
```

- 自动追踪回调中访问的响应式依赖
- 立即执行一次
- 无法获取旧值

### 生命周期钩子

| Options API | Composition API | 说明 |
|-------------|-----------------|------|
| `beforeCreate` | 不需要 | `setup()` 本身在创建前执行 |
| `created` | 不需要 | `setup()` 替代了 `created` |
| `beforeMount` | `onBeforeMount` | DOM 尚未创建 |
| `mounted` | `onMounted` | DOM 已创建，可操作 DOM |
| `beforeUpdate` | `onBeforeUpdate` | 数据变化，DOM 尚未更新 |
| `updated` | `onUpdated` | DOM 已更新 |
| `beforeUnmount` | `onBeforeUnmount` | 组件即将被销毁 |
| `unmounted` | `onUnmounted` | 组件已销毁，清理资源 |
| `errorCaptured` | `onErrorCaptured` | 捕获子孙组件错误 |

**关键规则**：
- 生命周期钩子必须在 `setup()` 的**同步调用栈**中注册
- 回调函数本身可以是异步的：`onMounted(async () => { ... })`

### 依赖注入

```js
// 祖先组件
import { provide, ref } from 'vue'

const theme = ref('light')
provide('theme', theme)

// 后代组件
import { inject } from 'vue'

const theme = inject('theme')
```

- `provide` 的值如果是 `ref`，后代 `inject` 后修改会同步到祖先
- 可设置默认值：`inject('theme', 'default')`

---

## `<script setup>` 语法

`<script setup>` 是 Vue 3.2+ 推荐的 Composition API 使用方式，作为 SFC 的编译时语法糖：

### 核心特性

| 特性 | 说明 |
|------|------|
| 顶层绑定自动暴露 | 顶层变量、函数、导入自动对模板可用 |
| 无需 `return` | 不需要像 `setup()` 函数那样显式返回 |
| 更小的包体积 | 编译为内联函数，变量名可被压缩 |
| 更好的类型推导 | 编译器自动推导 props 和 emits 类型 |

### 编译器宏

在 `<script setup>` 中可直接使用以下编译器宏（无需导入）：

| 宏 | 用途 | 版本 |
|----|------|------|
| `defineProps()` | 声明组件 props | 3.0+ |
| `defineEmits()` | 声明组件事件 | 3.0+ |
| `defineExpose()` | 暴露组件实例属性给父组件 | 3.0+ |
| `defineSlots()` | 声明插槽类型 | 3.3+ |
| `defineModel()` | 声明 v-model 双向绑定 | 3.4+ |
| `defineOptions()` | 设置组件名、inheritAttrs 等 | 3.3+ |
| `withDefaults()` | 为 props 设置默认值 | 3.0+ |
| `useSlots()` / `useAttrs()` | 访问 slots 和 attrs | 3.0+ |

**`defineModel()` 示例**（Vue 3.4+ 推荐）：

```vue
<!-- 子组件 -->
<script setup>
const model = defineModel()

function update() {
  model.value++
}
</script>

<template>
  <button @click="update">{{ model }}</button>
</template>
```

父组件使用 `<Child v-model="count" />` 即可实现双向绑定。

> 参考：[`<script setup>` - Vue 官方 API](https://cn.vuejs.org/api/sfc-script-setup.html)

---

## 组合式函数（Composables）

### 什么是组合式函数

组合式函数是以 `use` 开头的函数，用于封装和复用**有状态逻辑**。它是 Composition API 逻辑复用的核心模式。

### 设计原则

| 原则 | 说明 |
|------|------|
| 命名约定 | 以 `use` 开头（如 `useMouse`、`useFetch`） |
| 独立实例 | 每次调用返回独立的状态实例，互不影响 |
| 返回值 | 返回包含多个 `ref` 的普通对象（非响应式），解构后仍保持响应性 |
| 副作用清理 | 在 `onUnmounted` 中清理副作用（事件监听器、定时器等） |
| 调用限制 | 只能在 `<script setup>` 或 `setup()` 中**同步**调用 |

### 示例：鼠标追踪

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

```vue
<!-- 组件中使用 -->
<script setup>
import { useMouse } from './useMouse.js'

const { x, y } = useMouse()
</script>

<template>Mouse position: {{ x }}, {{ y }}</template>
```

### 接收响应式参数

组合式函数可接收 ref 或 getter 作为参数，配合 `toValue()`（Vue 3.3+）和 `watchEffect()` 实现响应式更新：

```js
import { ref, watchEffect, toValue } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const error = ref(null)

  watchEffect(() => {
    data.value = null
    error.value = null

    fetch(toValue(url))
      .then(res => res.json())
      .then(json => data.value = json)
      .catch(err => error.value = err)
  })

  return { data, error }
}
```

此实现支持三种调用方式：
- 静态字符串：`useFetch('/api/data')`（只请求一次）
- ref：`useFetch(urlRef)`（url 变化时重新请求）
- getter：`useFetch(() => `/posts/${props.id}`)`（props.id 变化时重新请求）

### 与 Mixins 对比

| 特性 | Mixins | Composables |
|------|--------|-------------|
| 数据来源 | 不清晰（合并到实例） | 清晰（显式解构） |
| 命名冲突 | 可能 | 可通过重命名避免 |
| 隐式耦合 | 是（共享属性名） | 否（显式参数传递） |
| 类型推导 | 差 | 优秀 |
| 组件实例开销 | 无 | 无 |

### 与无渲染组件对比

| 特性 | 无渲染组件 | Composables |
|------|-----------|-------------|
| 逻辑复用 | 支持 | 支持 |
| 视图复用 | 支持 | 不支持 |
| 组件实例开销 | 有 | 无 |

> 推荐：纯逻辑复用使用 Composables；需要同时复用逻辑和视图时使用无渲染组件。

> 参考：[组合式函数 - Vue 官方文档](https://cn.vuejs.org/guide/reusability/composables.html)

---

## 与 React Hooks 对比

| 特性 | Vue Composition API | React Hooks |
|------|---------------------|-------------|
| 调用时机 | `setup()` 只调用一次 | 每次渲染都重新调用 |
| 调用顺序 | 不限制，可条件调用 | 必须严格按顺序，不可条件调用 |
| 依赖收集 | 自动（响应式系统运行时收集） | 手动（需传依赖数组） |
| 闭包陷阱 | 不存在（只执行一次） | 存在（需 `useRef` 解决） |
| 性能优化 | 自动细粒度更新，无需手动 `memo` | 需手动 `useMemo`、`useCallback` |
| 缓存开销 | 无需手动缓存 | 缓存本身有开销 |
| 状态可变性 | 可变（Mutation-based） | 不可变（Immutable） |

Vue 的响应式模型天然解决了 React Hooks 的闭包陷阱、依赖数组遗漏、手动缓存等问题。

> 参考：[组合式 API 常见问答 - 与 React Hooks 对比](https://cn.vuejs.org/guide/extras/composition-api-faq.html#comparison-with-react-hooks)

---

## 与选项式 API 的关系

### 共存而非替代

- **选项式 API 不会被废弃**，仍是 Vue 不可分割的一部分
- 两者可在同一组件中混用（通过 `setup()` 选项），但不推荐
- 可通过编译时标记移除选项式 API 运行时代码，减小包体积（约几 KB）

### 适用场景

| 项目规模 | 推荐 API | 原因 |
|----------|----------|------|
| 中小型项目 | 选项式 API | 约定式结构，上手门槛低 |
| 大型项目 | Composition API | 更好的逻辑复用、类型推导、可维护性 |
| 需要 TS 强支持 | Composition API | 天然类型友好 |
| 逻辑复用需求高 | Composition API | Composables 替代 mixins |

---

## TypeScript 集成

Composition API 与 TypeScript 天然兼容，大多数情况下无需额外类型标注：

### Props 类型声明

```vue
<script setup lang="ts">
// 运行时声明（推荐）
const props = defineProps<{
  title: string
  count?: number
}>()

// 或使用 withDefaults 设置默认值
const props = withDefaults(defineProps<{
  size: 'small' | 'medium' | 'large'
}>(), {
  size: 'medium'
})
</script>
```

### Emits 类型声明

```vue
<script setup lang="ts">
const emit = defineEmits<{
  change: [id: number]
  update: [value: string]
}>()
</script>
```

### Ref 类型推导

```ts
// 自动推导为 Ref<number>
const count = ref(0)

// 显式标注联合类型
const status = ref<'loading' | 'success' | 'error'>('loading')
```

> 参考：[TypeScript 与组合式 API - Vue 官方文档](https://cn.vuejs.org/guide/typescript/composition-api.html)

---

## 常见误区

1. **"Composition API 就是 Vue 版的 React Hooks"**
   - 不准确。虽然灵感来自 Hooks，但 Vue 的响应式模型解决了闭包陷阱、依赖数组、手动缓存等问题

2. **"`reactive()` 可以监听所有变化"**
   - 错误。解构 `reactive` 对象的属性会丢失响应性，替换整个对象也会丢失连接

3. **"选项式 API 会被废弃"**
   - 错误。官方明确表示不会废弃，两者共存，按项目规模选择

4. **"组合式函数可以在任何地方调用"**
   - 错误。只能在 `<script setup>` 或 `setup()` 中**同步**调用（`<script setup>` 是唯一在 `await` 之后仍可调用的地方，编译器会自动恢复组件实例）

5. **"从组合式函数返回 `reactive()` 对象更好"**
   - 不推荐。解构时会丢失响应性连接。应返回包含 `ref` 的普通对象

6. **"`watchEffect` 和 `watch` 可以随意替换"**
   - 错误。`watchEffect` 无法获取旧值且立即执行；需要精确控制监听源或访问旧值时必须用 `watch`

---

## 最佳实践

1. **优先使用 `ref()`**：无对象类型限制，不会因解构或替换丢失响应性
2. **组合式函数以 `use` 开头**：便于识别和约定
3. **组合式函数返回普通对象**：包含多个 `ref`，解构后仍保持响应性
4. **在 `onUnmounted` 中清理副作用**：避免内存泄漏
5. **使用 `defineModel()` 实现双向绑定**（Vue 3.4+）：代码更简洁
6. **大对象使用 `shallowRef()`**：避免深层响应性开销
7. **基于逻辑关注点组织代码**：将相关状态、计算、副作用聚合在一起

---

## 相关概念

- **响应式系统**：`ref`/`reactive` 底层基于 Proxy 和依赖追踪机制
- **虚拟 DOM 与渲染机制**：组件通过响应式副作用触发渲染
- **Pinia 状态管理**：基于 Composition API 设计的新一代状态管理库
- **VueUse**：社区组合式函数集合，是学习 Composables 的优质资源

---

## 参考资料

- [组合式 API 常见问答 - Vue 官方文档](https://cn.vuejs.org/guide/extras/composition-api-faq.html)
- [组合式函数 - Vue 官方文档](https://cn.vuejs.org/guide/reusability/composables.html)
- [`setup()` 函数 - Vue 官方 API](https://cn.vuejs.org/api/composition-api-setup.html)
- [`<script setup>` - Vue 官方 API](https://cn.vuejs.org/api/sfc-script-setup.html)
- [TypeScript 与组合式 API - Vue 官方文档](https://cn.vuejs.org/guide/typescript/composition-api.html)
- [响应式 API - Vue 官方 API](https://cn.vuejs.org/api/reactivity-core.html)
- [生命周期钩子 - Vue 官方 API](https://cn.vuejs.org/api/composition-api-lifecycle.html)
- [VueUse 组合式函数集合](https://vueuse.org/)
