---
created: 2026-04-27
updated: 2026-04-27
tags:
  - Vue
  - 双向绑定
  - 响应式系统
  - 前端原理
  - v-model
aliases:
  - Vue two-way binding
  - Vue 双向绑定机制
  - Vue v-model 原理
source_type: official-doc
source_urls:
  - https://cn.vuejs.org/guide/extras/reactivity-in-depth.html
  - https://cn.vuejs.org/guide/components/v-model.html
  - https://cn.vuejs.org/guide/essentials/reactivity-fundamentals.html
  - https://v2.cn.vuejs.org/v2/guide/reactivity.html
  - https://v2.cn.vuejs.org/v2/guide/components-custom-events.html
status: verified
---

## 什么是双向绑定

双向绑定（Two-way Binding）指的是**数据模型（Model）与视图（View）之间的同步更新机制**：

- **数据 → 视图**：当数据发生变化时，视图自动更新
- **视图 → 数据**：当用户在视图中交互（如输入表单）时，数据自动同步

Vue 的双向绑定由两层机制共同实现：底层的**响应式系统**（负责数据变化的追踪与通知）和上层的 **`v-model` 指令**（负责视图与数据之间的语法糖绑定）。

---

## 底层机制：Vue 响应式系统

### Vue 3 的实现：Proxy 代理

Vue 3 使用 JavaScript 的 [`Proxy`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 来实现响应式。核心原理是拦截对象属性的 `get` 和 `set` 操作：

```js
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      track(target, key)  // 依赖收集：记录谁访问了这个属性
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      trigger(target, key)  // 触发更新：通知所有订阅者
    }
  })
}
```

**依赖追踪流程**：

1. **track（依赖收集）**：当一个响应式属性在副作用函数（如组件渲染函数、`watchEffect`、`computed`）中被读取时，Vue 会将当前活跃的副作用记录为该属性的订阅者
2. **trigger（触发更新）**：当属性被修改时，Vue 查找该属性的所有订阅者并重新执行它们
3. 订阅关系存储在全局 `WeakMap<target, Map<key, Set<effect>>>` 数据结构中

`ref()` 的实现原理类似，通过 getter/setter 包装 `.value` 属性：

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

> 来源：[Vue 3 深入响应式系统](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html#how-reactivity-works-in-vue)

### Vue 2 的实现：Object.defineProperty

Vue 2 使用 [`Object.defineProperty`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 将 `data` 对象的所有属性转换为 getter/setter：

```js
// Vue 2 初始化时的简化逻辑
Object.defineProperty(data, key, {
  get() {
    // 依赖收集
    return val
  },
  set(newVal) {
    val = newVal
    // 触发更新
  }
})
```

每个组件实例对应一个 **Watcher** 实例。组件渲染时，所有被"接触"过的数据属性会被记录为依赖；当依赖项的 setter 触发时，通知 Watcher 重新渲染组件。

> 来源：[Vue 2 深入响应式原理](https://v2.cn.vuejs.org/v2/guide/reactivity.html#如何追踪变化)

### Vue 2 与 Vue 3 响应式对比

| 特性 | Vue 2（Object.defineProperty） | Vue 3（Proxy） |
|------|-------------------------------|----------------|
| **新增属性检测** | 无法检测，需用 `Vue.set` / `this.$set` | 可以检测 |
| **删除属性检测** | 无法检测，需用 `Vue.delete` / `this.$delete` | 可以检测 |
| **数组索引赋值** | 无法检测，需用 `splice` 或 `Vue.set` | 可以检测 |
| **数组 length 修改** | 无法检测 | 可以检测 |
| **初始化性能** | 需递归遍历所有属性，大对象较慢 | 惰性代理，访问时才转换 |
| **浏览器兼容** | 支持 IE9+（ES5） | 不支持 IE（需 ES2015+） |

---

## 上层机制：v-model 双向绑定语法糖

`v-model` 是 Vue 提供的双向绑定语法糖，本质上是 **`:value`（或 `:model-value`）属性绑定 + `@input`（或 `@update:model-value`）事件监听**的组合。

### 原生表单元素上的 v-model

```html
<input v-model="searchText" />
```

等价于：

```html
<input
  :value="searchText"
  @input="searchText = $event.target.value"
/>
```

> 来源：[Vue 3 组件 v-model](https://cn.vuejs.org/guide/components/v-model.html#under-the-hood)

### 组件上的 v-model（Vue 3.4+ 推荐方式）

从 Vue 3.4 开始，推荐使用 [`defineModel()`](https://cn.vuejs.org/api/sfc-script-setup.html#definemodel) 宏：

**子组件（Child.vue）**：
```vue
<script setup>
const model = defineModel()

function update() {
  model.value++
}
</script>

<template>
  <div>Parent bound v-model is: {{ model }}</div>
  <button @click="update">Increment</button>
</template>
```

**父组件**：
```vue
<Child v-model="countModel" />
```

`defineModel()` 返回一个 ref，它的 `.value` 与父组件的 `v-model` 值同步，子组件修改它时会触发父组件更新。

### 组件上的 v-model 底层机制

`defineModel` 是编译器宏，展开为：

- 一个名为 `modelValue` 的 prop，本地 ref 的值与其同步
- 一个名为 `update:modelValue` 的事件，当本地 ref 值变更时触发

**3.4 之前的实现方式**：

```vue
<!-- Child.vue -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="props.modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

父组件中的 `v-model="foo"` 被编译为：

```vue
<Child
  :modelValue="foo"
  @update:modelValue="$event => (foo = $event)"
/>
```

> 来源：[Vue 3 组件 v-model 底层机制](https://cn.vuejs.org/guide/components/v-model.html#under-the-hood)

### 带参数的 v-model

支持多个 v-model 绑定，每个绑定同步不同的 prop：

```vue
<!-- 父组件 -->
<UserName
  v-model:first-name="first"
  v-model:last-name="last"
/>
```

```vue
<!-- 子组件 -->
<script setup>
const firstName = defineModel('firstName')
const lastName = defineModel('lastName')
</script>

<template>
  <input type="text" v-model="firstName" />
  <input type="text" v-model="lastName" />
</template>
```

### v-model 修饰符

支持自定义修饰符，通过解构 `defineModel()` 返回值获取：

```vue
<script setup>
const [model, modifiers] = defineModel({
  set(value) {
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
    return value
  }
})
</script>

<template>
  <input type="text" v-model="model" />
</template>
```

使用：`<MyComponent v-model.capitalize="myText" />`

---

## Vue 2 中的 v-model 差异

### 默认 prop 和事件名不同

| 版本 | 默认 prop 名 | 默认事件名 |
|------|-------------|-----------|
| Vue 2 | `value` | `input` |
| Vue 3 | `modelValue` | `update:modelValue` |

### model 选项

Vue 2 中可通过 `model` 选项自定义 prop 和事件名，避免与原生 `value` 属性冲突：

```js
Vue.component('base-checkbox', {
  model: {
    prop: 'checked',
    event: 'change'
  },
  props: {
    checked: Boolean
  },
  template: `
    <input
      type="checkbox"
      v-bind:checked="checked"
      v-on:change="$emit('change', $event.target.checked)"
    >
  `
})
```

> 来源：[Vue 2 自定义组件的 v-model](https://v2.cn.vuejs.org/v2/guide/components-custom-events.html#自定义组件的-v-model)

### .sync 修饰符

Vue 2 提供 `.sync` 修饰符作为多个 prop 双向绑定的简写：

```vue
<text-document v-bind:title.sync="doc.title"></text-document>
```

等价于：

```vue
<text-document
  :title="doc.title"
  @update:title="doc.title = $event"
></text-document>
```

Vue 3 中 `.sync` 已被移除，统一使用 `v-model:propName` 语法。

---

## DOM 更新时机：异步更新队列

Vue 的 DOM 更新是**异步**的。当响应式状态变化时，Vue 不会立即更新 DOM，而是：

1. 将所有数据变更缓冲到一个队列中
2. 在同一事件循环中去重（同一 Watcher 多次触发只入队一次）
3. 在下一个事件循环 "tick" 中批量执行更新

这意味着修改数据后，DOM 不会立即反映新值。如需在 DOM 更新后执行操作，使用 `nextTick()`：

```js
import { nextTick } from 'vue'

async function increment() {
  count.value++
  await nextTick()
  // 现在 DOM 已更新
}
```

> 来源：[Vue 3 DOM 更新时机](https://cn.vuejs.org/guide/essentials/reactivity-fundamentals.html#dom-update-timing)、[Vue 2 异步更新队列](https://v2.cn.vuejs.org/v2/guide/reactivity.html#异步更新队列)

---

## 常见误区与限制

### Vue 3 reactive() 的局限性

1. **只能用于对象类型**：不能持有 `string`、`number`、`boolean` 等原始类型
2. **不能替换整个对象**：替换会丢失响应性连接
   ```js
   let state = reactive({ count: 0 })
   state = reactive({ count: 1 })  // 响应性连接已丢失
   ```
3. **对解构操作不友好**：解构原始类型属性会丢失响应性
   ```js
   const { count } = state  // count 已是普通值，不再响应
   ```

因此 Vue 3 推荐使用 `ref()` 作为声明响应式状态的主要 API。

### Vue 2 响应式检测限制

1. **无法检测对象属性的添加或删除**：需使用 `Vue.set` / `this.$set`
2. **无法检测数组索引直接赋值**：`vm.items[index] = newValue` 无效
3. **无法检测数组 length 修改**：`vm.items.length = newLength` 无效

### defineModel 默认值陷阱

如果为 `defineModel` 设置了 `default` 值，但父组件未提供绑定值，会导致父子组件不同步：

```vue
<!-- 子组件 -->
<script setup>
const model = defineModel({ default: 1 })
</script>

<!-- 父组件 -->
<script setup>
const myRef = ref()  // undefined
</script>
<Child v-model="myRef"></Child>
<!-- 子组件 model 为 1，父组件 myRef 为 undefined -->
```

---

## 最佳实践

1. **优先使用 `ref()`**：相比 `reactive()`，`ref()` 没有对象类型限制，也不会因解构或替换而丢失响应性
2. **始终通过代理访问响应式数据**：Vue 3 中 `reactive()` 返回的是 Proxy，与原始对象不相等（`proxy !== raw`）
3. **组件双向绑定使用 `defineModel()`**（Vue 3.4+）：代码更简洁，语义更清晰
4. **浅层 ref 优化大对象性能**：对于不需要深层响应性的大型数据，使用 `shallowRef()` 避免响应性开销
5. **避免在子组件中直接修改 prop**：应通过 `$emit('update:xxx')` 通知父组件更新，保持数据流清晰

---

## 相关概念

- **响应式副作用（Reactive Effect）**：`watchEffect()` 等 API 的核心，自动追踪依赖并在依赖变化时重新执行
- **计算属性（computed）**：基于响应式副作用实现失效与重新计算，适合派生状态
- **虚拟 DOM 与渲染机制**：Vue 组件通过响应式副作用触发渲染，结合编译器优化的虚拟 DOM 高效更新视图
- **信号（Signals）**：Solid、Angular 等框架的响应式基础类型，与 Vue 的 `ref` 原理相似

---

## 参考资料

- [Vue 3 深入响应式系统](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html)
- [Vue 3 响应式基础](https://cn.vuejs.org/guide/essentials/reactivity-fundamentals.html)
- [Vue 3 组件 v-model](https://cn.vuejs.org/guide/components/v-model.html)
- [Vue 2 深入响应式原理](https://v2.cn.vuejs.org/v2/guide/reactivity.html)
- [Vue 2 自定义事件（v-model / .sync）](https://v2.cn.vuejs.org/v2/guide/components-custom-events.html)
- [MDN - Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [MDN - Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
