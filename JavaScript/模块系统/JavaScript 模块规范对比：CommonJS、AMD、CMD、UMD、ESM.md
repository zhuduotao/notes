---
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - javascript
  - module-system
  - commonjs
  - amd
  - cmd
  - umd
  - esm
  - frontend-engineering
aliases:
  - JS模块加载方式区别
  - 模块规范对比
  - CommonJS vs AMD vs CMD vs ESM
  - JavaScript Module Formats
source_type: mixed
source_urls:
  - 'https://nodejs.org/api/modules.html'
  - 'https://nodejs.org/api/esm.html'
  - 'https://github.com/amdjs/amdjs-api/wiki/AMD'
  - 'https://github.com/seajs/seajs/issues/242'
  - 'https://github.com/cmdjs/specification/blob/master/draft/module.md'
  - 'https://262.ecma-international.org/6.0/'
status: verified
---

---
created: 2026-04-15
updated: 2026-04-15
tags:
  - javascript
  - module-system
  - commonjs
  - amd
  - cmd
  - umd
  - esm
  - frontend-engineering
aliases:
  - JS模块加载方式区别
  - 模块规范对比
  - CommonJS vs AMD vs CMD vs ESM
  - JavaScript Module Formats
source_type: mixed
source_urls:
  - https://nodejs.org/api/modules.html
  - https://nodejs.org/api/esm.html
  - https://github.com/amdjs/amdjs-api/wiki/AMD
  - https://github.com/seajs/seajs/issues/242
  - https://github.com/cmdjs/specification/blob/master/draft/module.md
  - https://262.ecma-international.org/6.0/
status: verified
---

## 概述

JavaScript 在发展过程中出现了多种模块规范，它们解决了不同场景下的代码组织与加载问题。理解这些规范的差异，有助于在工程实践中做出正确的技术选型，并理解现代打包工具（如 Webpack、Vite、Rollup）背后的工作原理。

## 为什么需要模块规范

早期 JavaScript 没有原生的模块系统，所有脚本共享同一个全局作用域，导致：

- **命名冲突**：多个脚本定义同名变量时互相覆盖
- **依赖管理困难**：无法显式声明模块间的依赖关系
- **加载顺序敏感**：必须手动保证 `<script>` 标签的引入顺序
- **无法私有化**：所有变量默认暴露在全局

模块规范的核心目标：**提供作用域隔离、显式依赖声明、按需加载能力**。

## 各模块规范详解

### CommonJS

**定义**：CommonJS 是服务端 JavaScript 的模块规范，由 Mozilla 工程师 Kevin Dangoor 于 2009 年发起（最初名为 ServerJS），旨在为 JavaScript 提供一套标准库和模块系统。

**核心语法**：

```js
// 导出
module.exports = { foo: 'bar' }
// 或
exports.foo = 'bar'

// 导入
const { foo } = require('./module')
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 加载方式 | **同步加载**，`require()` 执行时立即返回模块 |
| 执行时机 | 代码**运行时**加载并执行 |
| 依赖声明 | **运行时动态**，`require()` 可出现在代码任意位置 |
| 输出值 | **值的拷贝**（基本类型）/ **引用共享**（对象类型） |
| 循环依赖 | 返回**未完成的 exports 对象**，可能导致 `undefined` |
| 顶层变量 | 模块内 `var`/`let`/`const` 不污染全局（通过函数包装实现） |

**运行时环境**：

- Node.js 默认模块系统（`.cjs` 或无 `"type": "module"` 的 `.js` 文件）
- 浏览器**不原生支持**，需通过打包工具转换

**模块包装机制**（Node.js 内部实现）：

```js
(function(exports, require, module, __filename, __dirname) {
  // 你的模块代码实际在这里执行
});
```

这使得每个模块拥有独立作用域，同时注入 `exports`、`require`、`module`、`__filename`、`__dirname` 五个模块级变量。

**循环依赖行为**：

当 A 依赖 B、B 依赖 A 时，B 在加载 A 时会拿到 A 的**未完成副本**（即执行到 `require('./b')` 那一刻 A 已导出的内容）。这要求开发者谨慎设计循环依赖。

---

### AMD（Asynchronous Module Definition）

**定义**：异步模块定义规范，由 RequireJS 推广，专为**浏览器环境**设计，解决 CommonJS 同步加载在浏览器中导致的阻塞问题。

**核心语法**：

```js
// 完整形式
define('moduleId', ['dependency1', 'dependency2'], function(dep1, dep2) {
  // 模块代码
  return { exportValue: '...' }
})

// 简化形式（匿名模块）
define(['jquery', 'lodash'], function($, _) {
  // 依赖前置：所有依赖在工厂函数执行前已加载完成
  return { ... }
})
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 加载方式 | **异步加载**，不阻塞页面渲染 |
| 执行时机 | 依赖加载完成后执行工厂函数 |
| 依赖声明 | **依赖前置**（提前声明），在 `define` 的第二个参数数组中声明 |
| 输出值 | 工厂函数的 **返回值** |
| 循环依赖 | 通过延迟执行处理，但行为复杂 |
| 典型实现 | RequireJS、curl.js |

**设计哲学**：**提前执行**（Early Execution）——在定义模块时就声明所有依赖，加载器可以并行下载所有依赖文件。

**与 CommonJS 的兼容**：AMD 支持 "Simplified CommonJS Wrapping"：

```js
define(function(require, exports, module) {
  var a = require('a')
  var b = require('b')
  exports.action = function() {}
})
```

加载器会扫描函数体中的 `require()` 调用来提取依赖（依赖 `Function.prototype.toString()` 支持）。

---

### CMD（Common Module Definition）

**定义**：通用模块定义规范，由玉伯（lifesinger）在 Sea.js 中提出，旨在结合 CommonJS 的书写习惯与浏览器的异步加载需求。

**核心语法**：

```js
define(function(require, exports, module) {
  // 依赖就近声明
  var $ = require('jquery')
  
  // 延迟执行（按需加载）
  var moment = require('moment') // 执行到这里才加载
  
  // 异步加载
  require.async('./heavy-module', function(heavy) {
    heavy.doSomething()
  })
  
  exports.init = function() {}
})
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 加载方式 | **异步加载**，但依赖**就近执行** |
| 执行时机 | `require()` 调用时才执行依赖模块（懒执行） |
| 依赖声明 | **依赖就近**，在代码中需要时调用 `require()` |
| 输出值 | `exports` 对象或 `module.exports` 或 `return` 值 |
| 异步 API | 提供 `require.async()` 显式异步加载 |
| 典型实现 | Sea.js |

**设计哲学**：**懒执行**（Lazy Execution）——模块下载后不会立即执行，而是在 `require()` 调用时才执行。这使得依赖声明更自然，与 CommonJS 书写习惯一致。

**与 AMD 的核心区别**：

| 对比维度 | AMD | CMD |
|----------|-----|-----|
| 依赖声明位置 | 定义时（`define` 参数数组） | 使用时（代码中 `require()`） |
| 依赖执行时机 | 提前执行（下载完立即执行） | 懒执行（`require()` 时才执行） |
| 设计理念 | 依赖提前、执行提前 | 依赖就近、执行懒加载 |
| API 复杂度 | 较高（多种 `define` 签名） | 较低（保持简单，贴近 CommonJS） |

**注意**：Sea.js 已于 2016 年左右停止维护，CMD 规范在现代前端工程中已较少直接使用，但其设计理念影响了后续工具的发展。

---

### UMD（Universal Module Definition）

**定义**：通用模块定义，**不是一种独立的模块规范**，而是一种**兼容写法**，使同一份代码能在 CommonJS、AMD 和全局变量环境中都能正常工作。

**核心模式**：

```js
;(function(root, factory) {
  if (typeof define === 'function' && define.amd) {
    // AMD 环境
    define(['jquery'], factory)
  } else if (typeof module === 'object' && module.exports) {
    // CommonJS 环境
    module.exports = factory(require('jquery'))
  } else {
    // 浏览器全局环境
    root.myModule = factory(root.jQuery)
  }
})(typeof self !== 'undefined' ? self : this, function($) {
  // 模块实际代码
  return { ... }
})
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 本质 | 环境检测 + 条件导出 |
| 兼容范围 | CommonJS、AMD、全局变量（`<script>` 直接引入） |
| 适用场景 | 库/插件作者希望一份代码适配多种环境 |
| 缺点 | 代码冗长，无法利用 ESM 的 tree-shaking |

**典型应用**：jQuery、Lodash、Moment.js 等老牌库的 UMD 打包产物。

**现代替代**：随着 ESM 的普及，现代库更多采用 `"module"` 或 `"exports"` 字段提供 ESM 入口，UMD 逐渐被 relegating 到兼容旧环境的场景。

---

### ESM（ECMAScript Modules）

**定义**：JavaScript **官方标准**的模块系统，在 ECMAScript 2015（ES6）规范中首次定义（[ECMA-262 6th Edition, Section 15.2](https://262.ecma-international.org/6.0/#sec-modules)），是语言层面的模块解决方案。

**核心语法**：

```js
// 导出
export const foo = 'bar'
export function baz() {}
export default class MyClass {}

// 导入
import { foo, baz } from './module'
import MyClass from './module'
import * as all from './module'

// 仅执行副作用
import './setup-polyfill'
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| 加载方式 | **异步加载**（浏览器中通过 `<script type="module">`） |
| 执行时机 | 依赖**静态分析**阶段确定，代码执行前所有依赖已加载 |
| 依赖声明 | **静态声明**，`import`/`export` 只能出现在顶层作用域 |
| 输出值 | **Live Binding（实时绑定）**，导入的是只读引用，源模块值变化会反映到导入方 |
| 循环依赖 | 通过 TDZ（暂时性死区）处理，访问未初始化的绑定会抛出 `ReferenceError` |
| 顶层 await | ES2022 支持，允许在模块顶层使用 `await` |

**与 CommonJS 的核心区别**：

| 对比维度 | CommonJS | ESM |
|----------|----------|-----|
| 语法 | `require()` / `module.exports` | `import` / `export` |
| 加载时机 | 运行时动态加载 | 编译时静态确定 |
| 输出值 | 值的拷贝（基本类型） | Live Binding（实时引用） |
| `this` 指向 | 指向 `module.exports` | `undefined` |
| 动态导入 | `require(variable)` 天然支持 | `import()` 函数（返回 Promise） |
| Tree-shaking | 不支持（动态特性） | 支持（静态结构可分析） |

**Live Binding 示例**：

```js
// counter.js
export let count = 0
export function increment() { count++ }

// main.js
import { count, increment } from './counter'
console.log(count) // 0
increment()
console.log(count) // 1 —— 值已更新，不是快照
```

如果是 CommonJS，`count` 会是导入时的快照值，不会随源模块变化而更新。

**运行时支持**：

| 环境 | 支持情况 |
|------|----------|
| 现代浏览器 | 原生支持（`<script type="module">`） |
| Node.js | 原生支持（`.mjs` 或 `"type": "module"` 的 `.js`） |
| Deno / Bun | 默认使用 ESM |

**Node.js 中的 ESM**：

- 文件扩展名 `.mjs` 强制作为 ESM 解析
- `package.json` 中 `"type": "module"` 使该包下所有 `.js` 作为 ESM
- 支持 `require()` 加载同步 ESM（Node.js v22+，无顶层 `await` 时）
- ESM 中**不能**使用 `require`、`exports`、`module.exports`、`__filename`、`__dirname`（需用 `import.meta.url` 替代）

---

## 综合对比表

| 维度 | CommonJS | AMD | CMD | UMD | ESM |
|------|----------|-----|-----|-----|-----|
| **诞生时间** | 2009 | 2009 | 2011 | 2012（社区约定） | 2015（ES6 标准） |
| **设计目标** | 服务端模块化 | 浏览器异步模块化 | 贴近 CJS 的浏览器方案 | 多环境兼容 | 语言级统一方案 |
| **加载方式** | 同步 | 异步 | 异步 | 取决于环境 | 异步 |
| **依赖声明** | 运行时动态 | 定义时前置 | 使用时就近 | 取决于环境 | 编译时静态 |
| **执行时机** | 加载即执行 | 依赖加载完即执行 | `require()` 时懒执行 | 取决于环境 | 依赖图构建后执行 |
| **输出语义** | 值拷贝 | 返回值 | 返回值 | 取决于环境 | Live Binding |
| **循环依赖** | 未完成副本 | 复杂 | 类似 CJS | 取决于环境 | TDZ 报错 |
| **Tree-shaking** | 不支持 | 不支持 | 不支持 | 不支持 | **支持** |
| **浏览器原生** | 否 | 否（需 RequireJS） | 否（需 Sea.js） | 否 | **是** |
| **Node.js 原生** | **是** | 否 | 否 | 否 | **是**（v13.2+） |
| **现状** | Node.js 主流 | 逐渐淘汰 | 已停止维护 | 兼容旧环境 | **未来标准** |

---

## 常见误区

### 误区 1：「UMD 是一种独立的模块规范」

UMD 不是一种新的模块规范，而是一种**兼容层**。它通过环境检测，让同一份代码在 CommonJS、AMD 和全局变量环境中都能工作。UMD 产物通常由打包工具生成。

### 误区 2：「ESM 的 `import` 和 CommonJS 的 `require` 只是语法不同」

二者语义有本质差异：
- `require` 是**运行时函数调用**，可动态拼接路径、条件导入
- `import` 是**编译时语法结构**，路径必须是字符串字面量，依赖图在代码执行前已确定
- ESM 的导入是**只读实时绑定**，CommonJS 导入的是**值拷贝**

### 误区 3：「AMD 和 CMD 只是写法不同，本质一样」

虽然都用于浏览器异步加载，但执行模型不同：
- AMD **提前执行**所有依赖（定义时声明，加载完即执行）
- CMD **懒执行**依赖（`require()` 调用时才执行）
这在实际工程中会影响首屏加载性能和依赖管理策略。

### 误区 4：「Node.js 中 `require()` 和 `import()` 可以随意混用」

混用有诸多限制：
- CJS 中 `require()` 不能直接加载含顶层 `await` 的 ESM（会抛 `ERR_REQUIRE_ASYNC_MODULE`）
- ESM 中不能使用 `require()`（除非通过 `import { createRequire } from 'node:module'` 创建）
- 循环依赖在 CJS ↔ ESM 混用时行为不可预测

---

## 适用场景与最佳实践

### 当前推荐

| 场景 | 推荐方案 |
|------|----------|
| 新项目（浏览器） | **ESM**（配合 Vite/原生模块） |
| 新项目（Node.js） | **ESM**（`"type": "module"`）或 CJS（根据团队习惯） |
| 发布 npm 包 | 同时提供 **CJS + ESM** 双格式（通过 `exports` 字段条件导出） |
| 维护老项目 | 保持现有规范，逐步迁移 |
| 库作者（兼容旧环境） | 打包产物提供 UMD + ESM + CJS |

### npm 包 `package.json` 推荐配置

```json
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

- `main`：CommonJS 入口（旧工具兼容）
- `module`：ESM 入口（Webpack/Rollup 识别）
- `exports`：Node.js 条件导出（v12.16+），优先级最高

---

## 相关概念

- **打包工具**：Webpack、Rollup、Vite、esbuild 等负责将多种模块格式转换为目标环境可执行的代码
- **Tree-shaking**：依赖 ESM 的静态结构，移除未使用的导出
- **条件导出（Conditional Exports）**：Node.js `exports` 字段支持按环境返回不同入口
- **模块解析算法**：Node.js 的 `require()` 和 ESM 的 `import` 有不同的路径解析规则
- **Interoperability**：CJS 与 ESM 互操作是 Node.js 持续改进的领域（如 `require(esm)` 支持）

---

## 参考资料

- **CommonJS Modules 规范**：[CommonJS Modules/1.1.1](http://wiki.commonjs.org/wiki/Modules/1.1.1)
- **Node.js CommonJS 文档**：[Modules: CommonJS modules](https://nodejs.org/api/modules.html)
- **Node.js ESM 文档**：[Modules: ECMAScript modules](https://nodejs.org/api/esm.html)
- **AMD 规范**：[amdjs/amdjs-api - AMD](https://github.com/amdjs/amdjs-api/wiki/AMD)
- **CMD 规范**：[cmdjs/specification - draft/module.md](https://github.com/cmdjs/specification/blob/master/draft/module.md)
- **CMD 规范（Sea.js 官方说明）**：[seajs/seajs #242](https://github.com/seajs/seajs/issues/242)
- **ECMAScript 2015 规范（模块章节）**：[ECMA-262 6th Edition, Section 15.2](https://262.ecma-international.org/6.0/#sec-modules)
- **RequireJS 文档**：[https://requirejs.org](https://requirejs.org)
- **Node.js 包入口解析**：[Node.js Packages 文档](https://nodejs.org/api/packages.html)
