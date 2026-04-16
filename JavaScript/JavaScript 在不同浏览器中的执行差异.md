---
title: JavaScript 在不同浏览器中的执行差异
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - JavaScript
  - 浏览器
  - 引擎
  - 兼容性
  - ECMAScript
  - V8
  - SpiderMonkey
  - JavaScriptCore
aliases:
  - JS 浏览器执行差异
  - JavaScript 引擎差异
  - 浏览器 JavaScript 兼容性
source_type: mixed
source_urls:
  - >-
    https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/JavaScript_technologies_overview
  - >-
    https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features
  - 'https://v8.dev/docs'
  - 'https://tc39.es/ecma262/'
  - 'https://spidermonkey.dev/'
  - 'https://docs.webkit.org/Deep%20Dive/JSC/JavaScriptCore.html'
status: verified
---

## 核心结论

JavaScript 在不同浏览器中的执行**既有统一性，也有差异性**。统一性来源于 ECMAScript 标准的约束，差异性来源于各浏览器使用的 JavaScript 引擎不同、标准实现进度不同、以及宿主环境（Host Environment）提供的 Web API 不同。

理解这些差异对于编写跨浏览器兼容的前端代码至关重要。

---

## JavaScript 执行环境的三层结构

要理解浏览器间的执行差异，首先需要明确"JavaScript"在浏览器环境中实际包含三个层次：

### 1. ECMAScript 核心语言层

由 ECMA TC39 委员会标准化的语言核心，定义在 [ECMA-262 规范](https://tc39.es/ecma262/)中。这一层规定是**跨引擎统一的**，包括：

- 语言语法（解析规则、关键字、控制流、对象字面量初始化等）
- 错误处理机制（`throw`、`try...catch`、用户自定义 `Error` 类型）
- 数据类型（boolean、number、string、function、object 等）
- 基于原型的继承机制
- 内置对象和函数（`JSON`、`Math`、`Array` 方法、`parseInt`、`decodeURI` 等）
- 严格模式（Strict Mode）
- 模块系统
- 基础内存模型

只要浏览器实现了某个版本的 ECMAScript 标准，该版本定义的语言行为在所有浏览器中应当一致。

### 2. DOM 与 Web API 层

由 WHATWG 和 W3C 维护的宿主环境 API，**不属于 ECMAScript 标准**，而是各浏览器厂商自行实现的扩展能力：

- **DOM Core**：`Node`、`Element`、`Document`、`Event`、`EventTarget` 等语言无关接口
- **HTML DOM**：`className` 属性、`Document.body` 等 HTML 特定 API
- **其他 Web API**：`setTimeout()`、`setInterval()`、`XMLHttpRequest`、`Fetch API`、`WebWorkers`、`WebSockets`、`Canvas 2D`、`WebAssembly` 等

这一层是**浏览器差异的主要来源**，因为：
- 各浏览器对新 API 的实现进度不同
- 某些 API 存在历史遗留的非标准实现
- 部分 API 的行为边界在规范中定义不够明确

### 3. JavaScript 引擎实现层

各浏览器使用不同的 JavaScript 引擎来解析、编译和执行 ECMAScript 代码。引擎差异主要体现在：

- 新语言特性的支持时间不同
- 性能优化策略不同
- 错误消息和调试体验不同
- 对规范边缘情况的解释可能略有差异

---

## 主要浏览器 JavaScript 引擎对比

| 引擎 | 开发商 | 使用浏览器 | 非浏览器应用 |
|------|--------|-----------|-------------|
| **V8** | Google | Chrome、Edge、Opera、Brue 等 Chromium 系浏览器 | Node.js、Deno、Electron |
| **SpiderMonkey** | Mozilla | Firefox、Servo、Flow | MongoDB、CouchDB |
| **JavaScriptCore (JSC)** | Apple | Safari 及其他 WebKit 系浏览器 | Bun |
| **LibJS** | SerenityOS 社区 | Ladybird | - |

### 历史引擎（已淘汰）

- **Carakan**：旧版 Opera（Presto 引擎时期）
- **Chakra (JScript)**：Internet Explorer
- **Chakra (JavaScript)**：旧版 Edge（Chromium 化之前）

---

## 差异的具体表现

### 1. ECMAScript 新特性支持时间差

TC39 委员会每年发布新的 ECMAScript 版本，但各引擎实现新特性的时间不同：

- **Stage 3/4 提案**：通常在提案进入 Stage 3 或 Stage 4 时，各引擎会开始实现
- **V8**：通常最先实现新特性，因为资源投入最大
- **SpiderMonkey**：跟进速度较快，但某些特性实现时间晚于 V8
- **JavaScriptCore**：Apple 有自己的发布节奏，通常与 Safari 大版本绑定

**影响**：使用最新语法（如可选链 `?.`、空值合并 `??`、顶层 `await`、`Array.prototype.toSorted` 等）时，需要考虑目标浏览器是否已支持。

**最佳实践**：
- 使用 [caniuse.com](https://caniuse.com/) 或 MDN 兼容性表格查询特性支持情况
- 通过 Babel 等转译工具将新语法转换为目标浏览器支持的版本
- 使用 Polyfill 补充缺失的内置方法

### 2. 宿主对象（Host Objects）行为差异

DOM 和 Web API 由各浏览器独立实现，存在以下差异：

#### 2.1 事件模型差异

- 事件传播顺序、事件对象属性在不同浏览器中可能有细微差异
- 某些事件（如 `touch`、`pointer`）在移动端浏览器支持度不同

#### 2.2 渲染相关 API

- `requestAnimationFrame` 的帧率策略可能不同
- Canvas 2D 的抗锯齿、字体渲染存在差异
- WebGL 的扩展支持因浏览器和底层图形驱动而异

#### 2.3 网络 API

- `fetch()` 的 Cookie 策略、CORS 处理可能有差异
- `XMLHttpRequest` 的历史遗留行为不一致

#### 2.4 存储 API

- `localStorage`、`IndexedDB`、`Web SQL`（已废弃）的支持和限制不同
- Safari 的 ITP（Intelligent Tracking Prevention）对存储有特殊限制

### 3. 已废弃和非标准特性的支持差异

ECMAScript 规范中有一部分特性被标记为**已废弃（Deprecated）**或**过时（Obsolete）**，但各浏览器出于向后兼容考虑，支持情况不同：

#### 3.1 Annex B 特性（Normative Optional）

ECMAScript 规范的 Annex B 章节包含一些"规范可选"特性，**浏览器必须实现，但非浏览器环境可以不实现**：

- HTML 风格注释：`<!-- comment` 和 `--> comment`（仅在某些解析场景有效）
- `RegExp` 的静态属性：`$1-$9`、`input`、`lastMatch` 等
- `String.prototype.substr()`（定义在 Annex B，可能被部分环境移除）
- `escape()` / `unescape()` 函数
- `with` 语句（严格模式下不可用）

**影响**：在 Node.js 等非浏览器环境中，这些特性可能不可用。

#### 3.2 已废弃但仍可用的特性

- `Object.prototype.__proto__`（应使用 `Object.getPrototypeOf` / `Object.setPrototypeOf`）
- `Object.prototype.__defineGetter__` / `__defineSetter__`（应使用 `Object.defineProperty`）
- `Function.prototype.caller` / `arguments.callee`（严格模式下不可用）
- `Date.prototype.getYear()` / `setYear()`（受 Y2K 问题影响）

#### 3.3 已完全移除的特性

- `Object.observe()` / `Array.observe()`（已被 `Proxy` 替代）
- `WeakMap.prototype.clear()`（Firefox 46+ 移除）
- `for each...in` 语句（应使用 `for...of`）
- 表达式闭包（`function () 1` 简写）
- Sharp 变量（`#1 = {a: 1}; #1.b = #1;`）

### 4. 性能优化策略差异

各引擎采用不同的编译和优化策略，导致相同代码在不同浏览器中的执行性能不同：

| 引擎 | 解释器 | JIT 编译器 | 垃圾回收策略 |
|------|--------|-----------|-------------|
| **V8** | Ignition | TurboFan | 分代式、Stop-the-World、精准 GC |
| **SpiderMonkey** | 基线解释器 | IonMonkey | 增量式、分代式 GC |
| **JavaScriptCore** | LLInt | DFG + FTL JIT | 分代式、并发 GC |

**常见性能差异场景**：

- **数组操作**：V8 对密集数组有专门的优化（如 Elements Kind 转换），其他引擎策略不同
- **对象属性访问**：V8 使用 Hidden Classes（Maps）优化属性访问，JSC 和 SpiderMonkey 有类似但不同的机制
- **正则表达式**：各引擎的正则编译器和回溯策略不同，复杂正则的性能差异可能很大
- **闭包**：V8 的闭包实现经过多次优化，与 SpiderMonkey 的上下文链策略有差异

### 5. 错误消息和调试体验差异

同一错误在不同浏览器中的表现：

- **错误消息文本**：各引擎自定义错误消息，格式和内容不同
- **堆栈跟踪格式**：V8、SpiderMonkey、JSC 的堆栈格式有差异
- **非标准错误类型**：如 Firefox 的 `InternalError`（非标准，其他浏览器不支持）

---

## 如何确保跨浏览器兼容性

### 1. 遵循 ECMAScript 标准

- 优先使用已进入正式标准的特性（Stage 4）
- 避免使用 Annex B 中的特性编写新代码
- 不使用已废弃或已移除的特性

### 2. 使用兼容性查询工具

- [caniuse.com](https://caniuse.com/)：查询浏览器对特定特性的支持情况
- [MDN 兼容性表格](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array#browser_compatibility)：每个 MDN 参考页面底部都有兼容性表格
- [ECMAScript 兼容性表](https://compat-table.github.io/compat-table/es6/)：查询各引擎对 ES6+ 特性的支持

### 3. 使用转译和 Polyfill

- **Babel**：将新语法转译为目标浏览器支持的版本
- **core-js** / **es-shims**：为缺失的内置方法提供 Polyfill
- **TypeScript**：通过 `target` 和 `lib` 配置控制输出语法和类型定义

### 4. 特性检测而非浏览器检测

```javascript
// ✅ 推荐：特性检测
if (typeof Promise !== 'undefined' && typeof Promise.prototype.finally === 'function') {
  // 使用 Promise.finally
}

// ❌ 不推荐：浏览器检测
if (navigator.userAgent.includes('Chrome')) {
  // 假设 Chrome 支持某个特性
}
```

### 5. 使用自动化测试

- 在目标浏览器矩阵中运行自动化测试
- 使用 Playwright、Selenium 等工具进行跨浏览器 E2E 测试
- 关注 CI/CD 中的浏览器兼容性测试报告

---

## 常见误区

### 误区 1："JavaScript 是统一的，所以代码在所有浏览器中行为一致"

**事实**：ECMAScript 核心语言是统一的，但 DOM API、Web API、引擎实现进度和性能优化策略都存在差异。

### 误区 2："现代浏览器都支持最新的 ECMAScript"

**事实**：即使是最新版本的 Chrome、Firefox、Safari，也可能存在某些 Stage 4 特性尚未实现的情况。企业环境中的浏览器版本更新更慢。

### 误区 3："Polyfill 可以解决所有兼容性问题"

**事实**：Polyfill 只能补充缺失的 API，无法修复引擎层面的行为差异、性能问题或规范解释分歧。

### 误区 4："Annex B 特性可以安全使用"

**事实**：Annex B 特性虽然在浏览器中必须实现，但规范明确指出"程序员在编写新代码时不应使用或假设这些特性存在"，且非浏览器环境可能不支持。

---

## 相关概念

- [[原型和原型链]]：JavaScript 继承机制的核心
- [[JavaScript 闭包与 V8 实现]]：闭包在 V8 引擎中的具体实现
- [[V8 对象模型解析]]：V8 的 Hidden Classes 机制
- [[V8 数组处理机制]]：V8 对数组的优化策略
- [[内联缓存 (Inline Caching)]]：属性访问优化技术
- [[Turbofan 编译器解析]]：V8 的 JIT 编译器

---

## 参考资料

- [MDN - JavaScript technologies overview](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/JavaScript_technologies_overview)
- [MDN - Deprecated and obsolete features](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Deprecated_and_obsolete_features)
- [V8 官方文档](https://v8.dev/docs)
- [ECMA-262 语言规范](https://tc39.es/ecma262/)
- [SpiderMonkey 官网](https://spidermonkey.dev/)
- [JavaScriptCore 文档](https://docs.webkit.org/Deep%20Dive/JSC/JavaScriptCore.html)
- [ECMAScript 兼容性表](https://compat-table.github.io/compat-table/es6/)
- [caniuse.com](https://caniuse.com/)
