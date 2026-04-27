---
created: 2026-04-27
updated: 2026-04-27
tags:
  - JavaScript
  - ECMAScript
  - 元编程
  - Proxy
  - Reflect
aliases:
  - Proxy
  - JS代理
  - Proxy对象
source_type: official-doc
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
  - https://tc39.es/ecma262/multipage/reflection.html#sec-proxy-objects
status: verified
---

## 是什么

`Proxy` 是 ES6（ES2015）引入的内置构造函数，用于**创建一个对象的代理，从而拦截并重新定义该对象的基本操作**（如属性读取、赋值、枚举、函数调用等）。

```js
const proxy = new Proxy(target, handler)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `target` | `Object` | 要代理的目标对象（可以是任何类型对象，包括数组、函数、甚至其他 Proxy） |
| `handler` | `Object` | 处理器对象，包含定义各种拦截行为的 trap 函数 |

**返回值**：一个新的 Proxy 对象。

## 为什么重要

- **元编程能力**：允许开发者在语言层面拦截和自定义对象行为，是 JavaScript 最强大的元编程工具之一。
- **响应式系统的核心**：Vue 3 的响应式机制完全基于 Proxy 实现。
- **超越 Object.defineProperty()**：可拦截整个对象而非单个属性，支持检测新属性添加、数组索引和长度变化等。
- **13 种拦截陷阱**：几乎覆盖了对象的所有底层内部方法。

## 核心概念

### 术语

| 术语 | 说明 |
|------|------|
| `target`（目标） | 被 Proxy 虚拟化的原始对象，通常作为存储后端 |
| `handler`（处理器） | 作为第二个参数传入 Proxy 构造函数的对象，包含各种 trap |
| `trap`（陷阱） | 提供属性访问拦截的函数，对应对象的内部方法 |
| `invariants`（不变量） | 自定义操作时必须保持的语义约束，违反会抛出 `TypeError` |

### 对象内部方法与对应 Trap

所有对象交互最终都归结为调用内部方法，Proxy 允许自定义这些方法：

| 内部方法 | 对应 Trap | 触发场景 |
|----------|-----------|----------|
| `[[GetPrototypeOf]]` | `getPrototypeOf()` | `Object.getPrototypeOf()`、`__proto__`、`Object.prototype.isPrototypeOf()` |
| `[[SetPrototypeOf]]` | `setPrototypeOf()` | `Object.setPrototypeOf()` |
| `[[IsExtensible]]` | `isExtensible()` | `Object.isExtensible()` |
| `[[PreventExtensions]]` | `preventExtensions()` | `Object.preventExtensions()` |
| `[[GetOwnProperty]]` | `getOwnPropertyDescriptor()` | `Object.getOwnPropertyDescriptor()`、`Object.defineProperty()` 前置检查 |
| `[[DefineOwnProperty]]` | `defineProperty()` | `Object.defineProperty()`、`Object.defineProperties()` |
| `[[HasProperty]]` | `has()` | `in` 操作符 |
| `[[Get]]` | `get()` | 属性读取：`obj.x`、`obj[x]` |
| `[[Set]]` | `set()` | 属性赋值：`obj.x = 1`、`obj[x] = 1` |
| `[[Delete]]` | `deleteProperty()` | `delete obj.x` |
| `[[OwnPropertyKeys]]` | `ownKeys()` | `Object.keys()`、`Object.getOwnPropertyNames()`、`Object.getOwnPropertySymbols()`、`Object.assign()`、`for...in` |
| `[[Call]]` | `apply()` | 函数调用：`proxy(...args)`、`proxy.call()`、`proxy.apply()` |
| `[[Construct]]` | `construct()` | `new proxy(...args)` |

## 13 个 Trap 详解

### `get(target, prop, receiver)`

拦截属性读取操作。

```js
const handler = {
  get(target, prop, receiver) {
    if (prop in target) {
      return target[prop];
    }
    return "default"; // 属性不存在时返回默认值
  },
};
```

| 参数 | 说明 |
|------|------|
| `target` | 目标对象 |
| `prop` | 属性名（`string` 或 `Symbol`） |
| `receiver` | 最初被调用的 Proxy 对象（或继承链上的 Proxy） |

### `set(target, prop, value, receiver)`

拦截属性赋值操作。**必须返回布尔值**，`true` 表示成功，`false` 表示失败（严格模式下抛 `TypeError`）。

```js
const validator = {
  set(obj, prop, value) {
    if (prop === "age" && !Number.isInteger(value)) {
      throw new TypeError("age must be an integer");
    }
    obj[prop] = value;
    return true; // 必须返回 true 表示成功
  },
};
```

### `has(target, prop)`

拦截 `in` 操作符。

```js
const handler = {
  has(target, prop) {
    // 隐藏以 _ 开头的私有属性
    if (prop.startsWith("_")) {
      return false;
    }
    return prop in target;
  },
};
```

**不变量约束**：如果目标对象有不可配置的自有属性，`has()` 必须返回 `true`，不能隐藏它。

### `deleteProperty(target, prop)`

拦截 `delete` 操作。必须返回布尔值。

```js
const handler = {
  deleteProperty(target, prop) {
    if (prop.startsWith("_")) {
      return false; // 禁止删除私有属性
    }
    delete target[prop];
    return true;
  },
};
```

**不变量约束**：不可配置的自有属性不能被删除，trap 必须返回 `false`。

### `ownKeys(target)`

拦截属性枚举操作。必须返回一个数组。

```js
const handler = {
  ownKeys(target) {
    // 过滤掉以 _ 开头的属性
    return Reflect.ownKeys(target).filter(key => !key.startsWith("_"));
  },
};
```

**不变量约束**：
- 返回值必须是数组
- 必须包含所有不可配置自有属性的键
- 如果目标对象不可扩展，必须包含所有自有属性的键

### `getOwnPropertyDescriptor(target, prop)`

拦截 `Object.getOwnPropertyDescriptor()`。

```js
const handler = {
  getOwnPropertyDescriptor(target, prop) {
    const desc = Object.getOwnPropertyDescriptor(target, prop);
    if (prop.startsWith("_")) {
      return undefined; // 隐藏属性描述符
    }
    return desc;
  },
};
```

### `defineProperty(target, prop, descriptor)`

拦截 `Object.defineProperty()`。必须返回布尔值。

### `getPrototypeOf(target)`

拦截原型读取。必须返回对象或 `null`。

### `setPrototypeOf(target, prototype)`

拦截原型设置。必须返回布尔值。

### `isExtensible(target)`

拦截可扩展性检查。必须返回布尔值，且结果必须与 `Object.isExtensible(target)` 一致。

### `preventExtensions(target)`

拦截 `Object.preventExtensions()`。必须返回布尔值。

### `apply(target, thisArg, argumentsList)`

拦截函数调用。

```js
const sum = (...args) => args.reduce((a, b) => a + b, 0);

const handler = {
  apply(target, thisArg, args) {
    console.log(`Calling with args: ${args}`);
    return target(...args);
  },
};

const proxy = new Proxy(sum, handler);
proxy(1, 2, 3); // 输出: Calling with args: 1,2,3 → 返回 6
```

### `construct(target, argumentsList, newTarget)`

拦截 `new` 操作。必须返回一个对象。

```js
function User(name) {
  this.name = name;
}

const handler = {
  construct(target, args) {
    console.log(`Creating ${target.name} with: ${args}`);
    return new target(...args);
  },
};

const ProxyUser = new Proxy(User, handler);
const u = new ProxyUser("Alice"); // 输出: Creating function User with: Alice
```

## 核心用法

### 与 Reflect 配合使用

`Reflect` 提供了与 Proxy trap 同名的方法，用于执行默认行为。最佳实践是在 trap 中使用 `Reflect` 方法转发操作：

```js
const handler = {
  get(target, prop, receiver) {
    if (prop === "message2") {
      return "world";
    }
    // 使用 Reflect.get 执行默认行为
    return Reflect.get(target, prop, receiver);
  },
};
```

**为什么用 Reflect 而不是直接访问 `target[prop]`？**

- `Reflect.get()` 正确处理 getter 的 `this` 绑定
- 对于 Proxy 链，`Reflect` 方法会正确传递 `receiver` 参数
- 返回值更一致（如 `Reflect.defineProperty()` 返回布尔值而非抛异常）

### 可撤销的 Proxy

```js
const { proxy, revoke } = Proxy.revocable(target, handler);

proxy.foo; // 正常工作

revoke(); // 撤销后

proxy.foo; // TypeError: Cannot perform 'get' on a proxy that has been revoked
```

撤销后的 Proxy 无法再执行任何操作，且撤销不可逆。适用于临时授权场景。

### 默认值代理

```js
const handler = {
  get(obj, prop) {
    return prop in obj ? obj[prop] : 37;
  },
};

const p = new Proxy({}, handler);
p.a = 1;
console.log(p.a); // 1
console.log(p.c); // 37 (默认值)
```

### 数据验证

```js
const validator = {
  set(obj, prop, value) {
    if (prop === "age") {
      if (!Number.isInteger(value)) {
        throw new TypeError("The age is not an integer");
      }
      if (value > 200) {
        throw new RangeError("The age seems invalid");
      }
    }
    obj[prop] = value;
    return true;
  },
};

const person = new Proxy({}, validator);
person.age = 100; // 成功
person.age = "young"; // TypeError
```

### 扩展额外属性

```js
const products = new Proxy(
  { browsers: ["Firefox", "Chrome"] },
  {
    get(obj, prop) {
      if (prop === "latestBrowser") {
        return obj.browsers[obj.browsers.length - 1];
      }
      return obj[prop];
    },
    set(obj, prop, value) {
      if (prop === "latestBrowser") {
        obj.browsers.push(value);
        return true;
      }
      if (typeof value === "string") {
        value = [value];
      }
      obj[prop] = value;
      return true;
    },
  }
);
```

## 限制与注意事项

### 私有字段（Private Fields）无法直接代理

Proxy 无法访问目标对象的私有字段（`#field`），因为 Proxy 是不同身份的对象：

```js
class Secret {
  #secret;
  constructor(secret) {
    this.#secret = secret;
  }
  get secret() {
    return this.#secret;
  }
}

const secret = new Secret("123456");
const proxy = new Proxy(secret, {});
console.log(proxy.secret); // TypeError: Cannot read private member #secret
```

**修复方式**：在 getter 中将 `this` 绑定回原始对象：

```js
const proxy = new Proxy(secret, {
  get(target, prop, receiver) {
    return target[prop]; // 使用 target 而非 receiver
  },
});
```

对于方法，还需要修复 `this` 绑定：

```js
const proxy = new Proxy(secret, {
  get(target, prop, receiver) {
    const value = target[prop];
    if (value instanceof Function) {
      return function (...args) {
        return value.apply(this === receiver ? target : this, args);
      };
    }
    return value;
  },
});
```

### 内置对象（Internal Slots）无法简单代理

`Map`、`Set`、`Date`、`Promise` 等内置对象有内部槽（internal slots），简单的空 handler 代理会失败：

```js
const proxy = new Proxy(new Map(), {});
console.log(proxy.size); // TypeError: get size method called on incompatible Proxy
```

需要使用上述 `this` 恢复模式才能正确代理。

### 不变量（Invariants）约束

Proxy 不是完全自由的，必须遵守以下不变量，否则会抛出 `TypeError`：

| 不变量 | 说明 |
|--------|------|
| `[[GetPrototypeOf]]` | 必须返回对象或 `null` |
| `[[SetPrototypeOf]]` | 如果目标不可扩展，新原型必须与 `Object.getPrototypeOf(target)` 相同 |
| `[[IsExtensible]]` | 返回值必须与 `Object.isExtensible(target)` 一致 |
| `[[PreventExtensions]]` | 如果返回 `true`，`Object.isExtensible(proxy)` 必须为 `false` |
| `[[GetOwnProperty]]` | 不可配置自有属性必须返回描述符 |
| `[[DefineOwnProperty]]` | 不可配置自有属性不能被删除或修改为不兼容的描述符 |
| `[[HasProperty]]` | 不可配置自有属性必须返回 `true` |
| `[[Get]]` | 不可配置的只读数据属性必须返回其值 |
| `[[Delete]]` | 不可配置自有属性不能被删除 |
| `[[OwnPropertyKeys]]` | 必须包含所有不可配置自有属性的键 |

### 性能考量

- 早期 V8 引擎中 Proxy 性能较差，但现代引擎已大幅优化
- 频繁拦截大量属性访问时仍有开销
- 对于性能敏感场景，建议基准测试后再决定是否使用

### `[[Set]]` 与 `[[DefineOwnProperty]]` 的区别

- `obj.x = 1` 使用 `[[Set]]`，会触发 setter
- `Object.defineProperty(obj, "x", { value: 1 })` 使用 `[[DefineOwnProperty]]`，**不会触发 setter**
- 类字段（class fields）也使用 `[[DefineOwnProperty]]` 语义

## 典型应用场景

| 场景 | 使用的 Trap | 说明 |
|------|-------------|------|
| 响应式系统 | `get`、`set` | Vue 3 的 `reactive()` 核心实现 |
| 数据验证 | `set` | 赋值前校验类型、范围 |
| 属性隐藏 | `has`、`ownKeys`、`getOwnPropertyDescriptor` | 过滤私有属性 |
| 默认值 | `get` | 属性不存在时返回默认值 |
| 日志/调试 | 所有 trap | 记录所有对象操作 |
| 负索引数组 | `get`、`set` | 支持 `arr[-1]` 访问末尾元素 |
| 只读视图 | `set`、`deleteProperty` | 返回 `false` 阻止修改 |
| 撤销授权 | `Proxy.revocable()` | 临时访问权限控制 |
| DOM 操作封装 | `set` | 如切换 `aria-selected` 属性 |

## 与 Object.defineProperty() 的对比

| 维度 | `Object.defineProperty()` | `Proxy` |
|------|---------------------------|---------|
| 引入版本 | ES5 (2009) | ES6 (2015) |
| 拦截粒度 | 单个属性 | 整个对象 |
| 新属性检测 | 不支持（需手动 defineProperty） | 支持（`set` trap） |
| 数组索引检测 | 有限（需重写方法） | 原生支持 |
| 拦截操作数量 | 有限（仅 get/set） | 13 种 |
| 性能 | 较好（已高度优化） | 早期较差，现代引擎已优化 |
| 典型应用 | Vue 2 响应式 | Vue 3 响应式、微前端沙箱 |

## 规范与兼容性

- **ECMAScript 规范**：ES6（ES2015）引入，后续版本持续维护。最新规范见 [ECMAScript 2027 § Proxy Objects](https://tc39.es/ecma262/multipage/reflection.html#sec-proxy-objects)。
- **浏览器兼容性**：自 2016 年 9 月起成为 Baseline Widely Available 特性，所有现代浏览器均支持。
- **Node.js**：所有 LTS 版本均支持。

## 参考资料

- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [ECMAScript® 2027 Language Specification — Proxy Objects](https://tc39.es/ecma262/multipage/reflection.html#sec-proxy-objects)
- [ECMAScript® 2027 — Ordinary and Exotic Objects Behaviors](https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-object-internal-methods-and-internal-slots)
- [Brendan Eich: Proxies are awesome (JSConf 2014)](https://youtu.be/sClk6aB_CPk)
