---
created: 2026-04-27
updated: 2026-04-27
tags:
  - JavaScript
  - ECMAScript
  - 对象
  - 属性描述符
  - 元编程
aliases:
  - Object.defineProperty
  - 属性描述符
  - Property Descriptor
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty
  - https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.defineproperty
status: verified
---

## 是什么

`Object.defineProperty()` 是 JavaScript 的内置静态方法，用于**直接在对象上定义新属性，或修改对象的现有属性**，并返回该对象。它提供了对属性行为的精细控制，远超普通赋值操作的默认行为。

```js
Object.defineProperty(obj, prop, descriptor)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `obj` | `Object` | 要定义或修改属性的目标对象 |
| `prop` | `string \| Symbol` | 要定义或修改的属性名 |
| `descriptor` | `Object` | 属性的描述符对象 |

**返回值**：传入的目标对象（链式调用友好）。

## 为什么重要

- **精细控制属性行为**：可以控制属性是否可写、可枚举、可配置。
- **实现响应式系统的基石**：Vue 2 的响应式机制就是基于 `Object.defineProperty()` 的 getter/setter 劫持。
- **与普通赋值的本质区别**：普通赋值创建的属性默认是 `writable: true`、`enumerable: true`、`configurable: true`，而 `Object.defineProperty()` 创建的属性默认值全部为 `false`。

## 前置概念：属性描述符的两种类型

属性描述符分为两种**互斥**的类型，一个描述符不能同时是两者：

### 数据描述符（Data Descriptor）

具有值的属性，该值可能是可写的也可能不是。

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `value` | 任意 | `undefined` | 属性关联的值 |
| `writable` | `boolean` | `false` | 是否可通过赋值运算符修改 |

### 访问器描述符（Accessor Descriptor）

由 getter-setter 函数对描述的属性。

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `get` | `Function \| undefined` | `undefined` | 读取属性时调用的函数，无参数，`this` 指向访问该属性的对象 |
| `set` | `Function \| undefined` | `undefined` | 写入属性时调用的函数，接收一个参数（新值），`this` 指向赋值的目标对象 |

### 共享键（两种描述符都有）

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `configurable` | `boolean` | `false` | 是否可删除、是否可修改描述符的其他属性、是否可切换描述符类型 |
| `enumerable` | `boolean` | `false` | 是否出现在属性枚举中（`for...in`、`Object.keys()`、展开运算符等） |

## 核心用法

### 创建属性

```js
const obj = {};

// 数据描述符
Object.defineProperty(obj, "a", {
  value: 37,
  writable: true,
  enumerable: true,
  configurable: true,
});

// 访问器描述符
let bValue = 38;
Object.defineProperty(obj, "b", {
  get() {
    return bValue;
  },
  set(newValue) {
    bValue = newValue;
  },
  enumerable: true,
  configurable: true,
});
```

### 修改现有属性

当属性已存在时，`Object.defineProperty()` 会根据当前属性的配置和新的描述符尝试修改：

```js
const obj = {};
Object.defineProperty(obj, "a", { value: 1, writable: true, configurable: false });

// 即使 configurable 为 false，writable: true 的数据属性仍可修改值
Object.defineProperty(obj, "a", { value: 2 }); // 成功
obj.a = 3; // 也成功

// 但不能再切换为访问器描述符或修改 enumerable
Object.defineProperty(obj, "a", { get() { return 1; } }); // TypeError
```

### 与普通赋值的对比

```js
const obj = {};

// 普通赋值 — 默认全部为 true
obj.a = 1;
// 等价于：
Object.defineProperty(obj, "a", {
  value: 1,
  writable: true,
  enumerable: true,
  configurable: true,
});

// Object.defineProperty() — 默认全部为 false
Object.defineProperty(obj, "b", { value: 1 });
// 等价于：
Object.defineProperty(obj, "b", {
  value: 1,
  writable: false,
  enumerable: false,
  configurable: false,
});
```

## 关键属性详解

### `configurable` 的锁定效应

`configurable: false` 是**单向锁定**，一旦设为 `false` 就无法恢复：

- 属性不可被 `delete` 删除
- 描述符类型不可切换（数据描述符 ↔ 访问器描述符）
- 除 `value` 和 `writable` 外的其他属性不可修改

**例外**：对于 `writable: true` 的数据属性，即使 `configurable: false`，仍可修改 `value` 或将 `writable` 从 `true` 改为 `false`（但不可逆）。

```js
const obj = {};
Object.defineProperty(obj, "b", { writable: true, configurable: false });

obj.b = 1; // 可以修改
Object.defineProperty(obj, "b", { writable: false }); // 可以关闭可写性
// 此后 b 属性完全锁定，无法再做任何修改
```

### `enumerable` 的影响范围

`enumerable: false` 的属性会被以下操作**忽略**：

- `for...in` 循环
- `Object.keys()`
- `Object.assign()` / 展开运算符 `{...obj}`

但仍可通过 `Object.getOwnPropertyNames()` 获取（该方法返回所有自有属性，不论是否可枚举）。

### `writable` 与严格模式

- 非严格模式下：对 `writable: false` 的属性赋值会**静默失败**（不报错，值不变）。
- 严格模式下：同样的操作会抛出 `TypeError`。

```js
"use strict";
const obj = {};
Object.defineProperty(obj, "a", { value: 1, writable: false });
obj.a = 2; // TypeError: Cannot assign to read only property 'a'
```

## 常见模式与示例

### 自归档对象（访问器描述符典型用例）

```js
function Archiver() {
  let temperature = null;
  const archive = [];

  Object.defineProperty(this, "temperature", {
    get() {
      return temperature;
    },
    set(value) {
      temperature = value;
      archive.push({ val: temperature });
    },
  });

  this.getArchive = () => archive;
}

const arc = new Archiver();
arc.temperature = 11;
arc.temperature = 13;
arc.getArchive(); // [{ val: 11 }, { val: 13 }]
```

### 只读常量

```js
const obj = {};
Object.defineProperty(obj, "PI", {
  value: 3.14159,
  writable: false,
  enumerable: false,
  configurable: false,
});
```

### 惰性求值（缓存模式）

```js
Object.defineProperty(obj, "expensiveValue", {
  get() {
    const value = computeExpensiveValue();
    Object.defineProperty(this, "expensiveValue", {
      value,
      writable: false,
      enumerable: true,
      configurable: true,
    });
    return value;
  },
  configurable: true,
});
```

## 限制与注意事项

### 不能混用描述符类型

同时包含数据描述符键（`value` 或 `writable`）和访问器描述符键（`get` 或 `set`）会抛出 `TypeError`：

```js
Object.defineProperty(obj, "conflict", {
  value: 42,
  get() { return 0; },
}); // TypeError: Invalid property descriptor
```

### 继承的访问器属性共享状态

如果访问器属性定义在原型上，所有实例共享同一个闭包变量：

```js
function MyClass() {}
let value;
Object.defineProperty(MyClass.prototype, "x", {
  get() { return value; },
  set(x) { value = x; },
});

const a = new MyClass();
const b = new MyClass();
a.x = 1;
console.log(b.x); // 1 — 被 a 的赋值影响了
```

**修复方式**：在 getter/setter 中使用 `this` 存储到实例自身：

```js
Object.defineProperty(MyClass.prototype, "x", {
  get() { return this.storedX; },
  set(x) { this.storedX = x; },
});
```

### 对数组的限制

`Object.defineProperty()` 可以定义数组索引属性，但无法高效地拦截数组长度变化或新增元素。这也是 Vue 2 无法检测以下数组变动的原因：

- 通过索引直接赋值：`arr[index] = newValue`
- 修改数组长度：`arr.length = newLength`

Vue 2 通过重写数组的变异方法（`push`、`pop`、`splice` 等）来部分解决此问题。

### 不触发 setter

`Object.defineProperty()` 使用内部的 `[[DefineOwnProperty]]` 方法，而非 `[[Set]]`。因此**即使属性已存在且有 setter，`defineProperty` 也不会调用它**。

### 描述符对象的原型链陷阱

描述符对象的属性会继承原型链上的值。如果原型被污染，可能导致意外行为：

```js
// 不安全的做法
const descriptor = { value: "static" };
Object.defineProperty(obj, "key", descriptor);

// 安全的做法 — 使用 null 原型对象
const descriptor = Object.create(null);
descriptor.value = "static";
Object.defineProperty(obj, "key", descriptor);
```

## 相关 API

| API | 说明 |
|-----|------|
| `Object.defineProperties()` | 批量定义多个属性 |
| `Object.getOwnPropertyDescriptor()` | 获取单个属性的描述符 |
| `Object.getOwnPropertyDescriptors()` | 获取所有自有属性的描述符 |
| `Object.create()` | 创建对象时可同时传入属性描述符 |
| `Reflect.defineProperty()` | `Object.defineProperty()` 的 Reflect 版本，返回布尔值而非抛异常 |
| `Object.freeze()` | 冻结对象（所有属性 `writable: false, configurable: false`） |
| `Object.seal()` | 密封对象（所有属性 `configurable: false`） |
| `Object.preventExtensions()` | 禁止添加新属性 |

## 与 Proxy 的对比

| 维度 | `Object.defineProperty()` | `Proxy` |
|------|---------------------------|---------|
| 引入版本 | ES5 (2009) | ES6 (2015) |
| 拦截粒度 | 单个属性 | 整个对象 |
| 新属性检测 | 不支持（需手动 defineProperty） | 支持（`set` trap） |
| 数组索引检测 | 有限（需重写方法） | 原生支持 |
| 性能 | 较好（已高度优化） | 早期较差，现代引擎已优化 |
| 典型应用 | Vue 2 响应式 | Vue 3 响应式 |

## 规范与兼容性

- **ECMAScript 规范**：ES5 引入，后续版本持续维护。最新规范见 [ECMAScript 2027 § Object.defineProperty](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.defineproperty)。
- **浏览器兼容性**：自 2015 年 7 月起成为 Baseline Widely Available 特性，所有现代浏览器均支持。

## 参考资料

- [MDN: Object.defineProperty()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
- [ECMAScript® 2027 Language Specification — Object.defineProperty](https://tc39.es/ecma262/multipage/fundamental-objects.html#sec-object.defineproperty)
- [ECMAScript® 2027 — Property Descriptor Specification Type](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-property-descriptor-specification-type)
