---
title: JavaScript 是如何执行的
created: '2026-04-27'
updated: '2026-04-27'
tags:
  - JavaScript
  - V8
  - 执行模型
  - 编译原理
  - 运行时
  - 浏览器
aliases:
  - JavaScript 执行流程
  - JS 执行原理
  - JavaScript Runtime
  - JavaScript Execution Pipeline
source_type: mixed
source_urls:
  - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model
  - https://v8.dev/blog/launching-ignition-and-turbofan
  - https://tc39.es/ecma262/
  - https://html.spec.whatwg.org/multipage/webappapis.html#event-loops
  - https://v8.dev/docs/
status: verified
---

## 是什么

JavaScript 代码从源码到最终在屏幕上产生效果，需要经历**解析 → 编译 → 执行 → 事件循环调度**的完整流程。这个过程涉及两个核心角色的协作：

- **JavaScript 引擎**：负责将源码转换为可执行的机器码并运行（如 V8、SpiderMonkey、JavaScriptCore）
- **宿主环境（Host Environment）**：提供引擎与外部世界交互的能力（如浏览器的 DOM/Web API、Node.js 的 fs/net 模块）

理解 JavaScript 的执行机制是掌握性能优化、调试技巧和异步编程的基础。

---

## 整体执行流程概览

```
JavaScript 源码
     ↓
┌─────────────────────────────────────────┐
│           JavaScript 引擎                 │
│                                         │
│  1. 词法分析 (Lexer) → Token 流          │
│     ↓                                   │
│  2. 语法分析 (Parser) → AST              │
│     ↓                                   │
│  3. 生成字节码 (Ignition 解释器)           │
│     ↓                                   │
│  4. 执行字节码 + 收集类型反馈              │
│     ↓                                   │
│  5. TurboFan JIT 编译 → 优化机器码        │
│     ↓                                   │
│  6. 执行优化后的机器码                    │
└─────────────────────────────────────────┘
     ↓
┌─────────────────────────────────────────┐
│           宿主环境（浏览器 / Node.js）      │
│                                         │
│  Event Loop 调度任务与微任务               │
│  DOM 操作 / 网络 I/O / 文件读写            │
│  渲染管线更新                              │
└─────────────────────────────────────────┘
```

---

## 第一阶段：解析（Parsing）

### 词法分析（Lexical Analysis）

词法分析器（Lexer / Tokenizer）将源码字符串拆分为有意义的**词法单元（Token）**。

```javascript
const sum = a + b;
```

被拆分为：
```
Keyword(const) → Identifier(sum) → Operator(=) → Identifier(a) → Operator(+) → Identifier(b) → Punctuator(;)
```

### 语法分析（Syntax Analysis）

语法分析器（Parser）将 Token 流转换为**抽象语法树（AST）**，这是一种树状数据结构，表示代码的语法结构。

```javascript
// 源码
const sum = a + b;

// 简化的 AST 结构
{
  type: "VariableDeclaration",
  kind: "const",
  declarations: [{
    type: "VariableDeclarator",
    id: { type: "Identifier", name: "sum" },
    init: {
      type: "BinaryExpression",
      operator: "+",
      left: { type: "Identifier", name: "a" },
      right: { type: "Identifier", name: "b" }
    }
  }]
}
```

AST 是后续编译和执行的基础。V8 使用 [Ignition 解释器](https://v8.dev/docs/ignition) 从 AST 生成字节码。

---

## 第二阶段：编译与执行（Compilation & Execution）

### V8 的执行管道（Ignition + TurboFan）

自 V8 v5.9（Chrome 59）起，V8 采用 **Ignition 解释器 + TurboFan 优化编译器** 的执行管道[^v8-pipeline]。

#### 1. Ignition 解释器

Ignition 是 V8 的寄存器式字节码解释器，负责：

- 将 AST 转换为**字节码（Bytecode）**——一种紧凑的中间表示
- **快速启动执行**：字节码生成速度快于直接编译为机器码
- **收集类型反馈（Type Feedback）**：记录运行时数据，供 TurboFan 优化使用

```javascript
function add(a, b) {
  return a + b;
}
add(1, 2);
```

Ignition 生成的字节码（简化）：
```
Ldar a0          // 加载参数 a 到累加器
Add a1           // 将参数 b 加到累加器
Return           // 返回累加器值
```

**为什么需要解释器？** 在 Ignition 之前，V8 使用 Full-codegen 直接生成机器码，占用大量内存（约占 JS 堆的 1/3）。Ignition 的字节码将内存占用降低了约 **9 倍**（ARM64 移动设备）[^ignition-memory]。

#### 2. TurboFan 优化编译器

当 Ignition 执行字节码并收集到足够的类型反馈后，TurboFan 会将**热点代码路径**编译为高度优化的机器码[^turbofan]。

**优化策略包括**：

| 优化技术 | 说明 | 示例 |
|---------|------|------|
| **类型推测（Type Speculation）** | 根据运行时反馈假设类型，生成专用指令 | 总是传入整数时，生成整数加法而非通用加法 |
| **内联缓存（Inline Caching）** | 缓存对象形状（Map）以加速属性访问 | `obj.x` 的偏移量被缓存，避免每次动态查找 |
| **内联（Inlining）** | 将小函数直接嵌入调用处，减少函数调用开销 | `add(1, 2)` 被替换为 `1 + 2` |
| **逃逸分析（Escape Analysis）** | 检测对象是否逃逸出函数，决定栈分配还是堆分配 | 局部对象可分配在栈上，减少 GC 压力 |
| **死代码消除** | 移除永远不会执行的代码 | `if (false) { ... }` 被删除 |

#### 3. 反优化（Deoptimization）

当 TurboFan 的优化假设不再成立时，引擎会触发**反优化**，回退到 Ignition 解释器执行：

- **类型变化**：之前一直是整数，突然传入浮点数
- **对象形状变化**：添加了新属性或删除了属性
- **函数重定义**：函数被重新赋值

```javascript
function add(a, b) {
  return a + b;
}

// TurboFan 优化为整数加法
add(1, 2);
add(3, 4);

// 反优化：传入字符串，整数加法假设失效
add("hello", "world");
```

反优化流程：
```
优化后的机器码（假设失效）
       ↓
  Deoptimization
       ↓
  恢复解释器状态
       ↓
  继续执行字节码
```

---

## 第三阶段：执行上下文与调用栈

### 执行上下文（Execution Context）

每个函数调用或脚本执行都会创建一个**执行上下文**（也称栈帧），记录：

- 代码评估状态
- 当前执行的模块/脚本/函数
- 当前 [Realm](#realm领域)
- 变量绑定（`var`、`let`、`const`、`function`、`class` 等）
- `this` 引用

### 调用栈（Call Stack）

调用栈跟踪当前的执行上下文，遵循**后进先出（LIFO）**原则。

```javascript
function foo(b) {
  const a = 10;
  return a + b + 11;
}

function bar(x) {
  const y = 3;
  return foo(x * y);
}

const baz = bar(7);
```

执行过程：

1. 创建全局上下文，定义 `foo`、`bar`、`baz`
2. 调用 `bar(7)`，创建 `bar` 的帧**入栈**
3. `bar` 内调用 `foo(21)`，创建 `foo` 的帧**入栈**
4. `foo` 返回 42，帧**出栈**
5. `bar` 返回 42，帧**出栈**
6. `baz` 被赋值为 42，栈为空，任务完成

### 闭包（Closures）

闭包是函数创建时记住其词法作用域的机制。函数内部保存了创建时的变量绑定引用，使这些绑定可以在执行上下文出栈后仍然存活。

```javascript
let f;
{
  let x = 10;
  f = () => x;  // 箭头函数捕获了 x 的绑定
}
console.log(f()); // 输出 10，尽管块级作用域已结束
```

闭包在 V8 中的具体实现涉及**上下文对象（Context Object）**的分配和引用链管理，详见 [[JavaScript 闭包与 V8 实现]]。

---

## 第四阶段：Event Loop 与异步调度

JavaScript 引擎本身只负责执行代码，**不包含 Event Loop**。Event Loop 由宿主环境（浏览器或 Node.js）提供，负责协调调用栈、任务队列和微任务队列之间的执行顺序。

### Agent 执行模型

ECMAScript 规范中，每个自主执行 JavaScript 的实体称为 **Agent**[^agent]。一个 Agent 包含：

| 数据结构 | 作用 | 特性 |
|---------|------|------|
| **Heap（堆）** | 存储对象 | 对象创建时分配内存 |
| **Stack（调用栈）** | 跟踪执行上下文 | 后进先出（LIFO） |
| **Queue（任务队列）** | 管理异步任务 | 先进先出（FIFO），即 Event Loop |

在浏览器中，一个 Agent 可以是主页面及其同源 iframe、Dedicated Worker、Service Worker 等。每个 Worker 创建独立的 Agent。

### Realm（领域）

每个 Agent 拥有一个或多个 **Realm**，代表一个独立的 JavaScript 全局执行环境[^realm]。一个 Realm 包含一组内置对象（`Array`、`Array.prototype` 等）、全局变量和 `globalThis`。

每个 `Window`、`WorkerGlobalScope` 对应一个 Realm。这也是为什么需要 `Array.isArray()` 而不是依赖 `instanceof Array` —— 不同 Realm 中的 `Array.prototype` 是不同的对象。

### 浏览器 Event Loop 处理模型

HTML 标准定义的 Event Loop 处理流程如下[^event-loop]：

```
1. 从任务队列中选取最老的任务
   ↓
2. 执行该任务（调用栈推入 → 运行代码 → 调用栈清空）
   ↓
3. 清空微任务队列（执行所有微任务，包括新产生的）
   ↓
4. 可选：更新渲染（重排、重绘）
   ↓
5. 回到步骤 1，进入下一轮循环
```

**任务（Task / Macrotask）** vs **微任务（Microtask）**：

| 特性 | 任务（宏任务） | 微任务 |
|------|-------------|--------|
| 来源 | `setTimeout`、`setInterval`、用户交互、I/O、`postMessage` | `Promise.then`、`queueMicrotask`、`MutationObserver`、`async/await` |
| 执行频率 | 每轮 Event Loop **最多执行一个** | 每轮**清空整个队列** |
| 优先级 | 低 | 高（在任务之后、渲染之前执行） |

### Run-to-Completion（运行至完成）

每个任务或微任务在执行期间**不会被中断**，会完整执行完毕后才让出控制权。这保证了执行顺序的可预测性，避免了竞态条件。

```javascript
const promise = Promise.resolve();
let i = 0;
promise.then(() => { i += 1; console.log(i); });
promise.then(() => { i += 1; console.log(i); });
// 输出始终是 1、2，不会发生竞态
```

### Node.js Event Loop 的差异

Node.js 使用 libuv 实现 Event Loop，分为多个阶段[^node-event-loop]：

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  上一轮未执行的 I/O 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  I/O 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──│      close callbacks      │  socket.on('close') 等
   └───────────────────────────┘
```

Node.js 还有 `process.nextTick()`，它在当前操作完成后、Event Loop 继续之前立即执行，**优先级高于 Promise 微任务**。

---

## 第五阶段：内存管理与垃圾回收

### 内存分配

JavaScript 使用两种内存区域：

- **栈（Stack）**：存储执行上下文和局部变量，自动分配和释放
- **堆（Heap）**：存储对象、闭包、大字符串等，由垃圾回收器管理

### 垃圾回收（Garbage Collection）

V8 采用**分代式垃圾回收**策略[^v8-gc]：

| 代 | 对象特征 | 回收策略 |
|----|---------|---------|
| **新生代（New Space）** | 新创建的对象，存活时间短 | 频繁回收，使用 Scavenge 算法 |
| **老生代（Old Space）** | 经过多次回收仍存活的对象 | 较少回收，使用标记-清除-整理（Mark-Sweep-Compact） |

**可达性分析**：垃圾回收器从根对象（全局变量、当前执行上下文中的引用）出发，遍历所有可达对象。不可达的对象被标记为垃圾并回收。

### 常见内存泄漏场景

```javascript
// 1. 意外的全局变量
function leak() {
  leakedVar = "I'm global now"; // 忘记使用 var/let/const
}

// 2. 被遗忘的定时器或回调
const interval = setInterval(() => {
  // 即使组件已卸载，回调仍在执行
}, 1000);

// 3. 闭包持有大对象引用
function createHandler() {
  const largeData = new Array(1000000).fill("data");
  return () => {
    // 即使不需要 largeData，闭包也会持有其引用
    console.log("handler called");
  };
}
```

---

## 其他引擎的执行管道

不同浏览器使用不同的 JavaScript 引擎，但整体架构相似：

| 引擎 | 解释器 | JIT 编译器 | 垃圾回收 |
|------|--------|-----------|---------|
| **V8**（Chrome、Edge、Node.js） | Ignition | TurboFan | 分代式、精准 GC |
| **SpiderMonkey**（Firefox） | 基线解释器 | IonMonkey | 增量式、分代式 GC |
| **JavaScriptCore**（Safari） | LLInt | DFG + FTL JIT | 分代式、并发 GC |

各引擎的执行管道都遵循**解释执行 → 收集反馈 → JIT 优化 → 反优化**的自适应优化模式，但具体实现策略和性能特征有所不同。详见 [[JavaScript 在不同浏览器中的执行差异]]。

---

## 限制与注意事项

### 长任务阻塞渲染

如果同步代码执行时间过长（通常 > 50ms），会阻塞 Event Loop，导致无法处理用户交互和渲染[^long-task]。

```javascript
// ❌ 坏：长同步循环阻塞 Event Loop
function processData(data) {
  for (let i = 0; i < data.length; i++) {
    // 处理大量数据...
  }
}

// ✅ 好：将大任务拆分为多个小任务
async function processDataChunked(data) {
  const chunkSize = 1000;
  for (let i = 0; i < data.length; i += chunkSize) {
    processChunk(data.slice(i, i + chunkSize));
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

### 过度优化反模式

```javascript
// ❌ 类型不稳定，导致 TurboFan 反复优化/反优化
function process(arr) {
  return arr.reduce((sum, n) => sum + n, ''); // 可能返回字符串或数字
}

// ✅ 保持类型稳定
function process(arr) {
  return arr.reduce((sum, n) => sum + n, 0);
}
```

### Tail Call Optimization（尾调用优化）

ECMAScript 规范要求实现**正确尾调用（Proper Tail Call, PTC）**，但出于调试考虑（尾调用会移除栈帧，导致堆栈跟踪不完整），目前**只有 Safari（JavaScriptCore）实现了 PTC**[^ptc]。

```javascript
// 理论上不会栈溢出（尾递归优化）
function factorial(n, acc = 1) {
  if (n <= 1) return acc;
  return factorial(n - 1, n * acc); // 尾调用位置
}
```

### 同步阻塞 API（遗留）

虽然 Event Loop 保证 JavaScript 执行"永不阻塞"，但一些遗留 API 仍然会阻塞：

- `alert()`、`confirm()`、`prompt()`
- 同步 `XMLHttpRequest`
- `document.write()`（在页面加载后调用会清空文档）

应避免使用这些 API。

---

## 相关概念

- [[JavaScript 事件循环（Event Loop）]]：Event Loop 的详细机制、任务与微任务、async/await 执行顺序
- [[JavaScript 在不同浏览器中的执行差异]]：各浏览器引擎的对比与兼容性
- [[JavaScript 闭包与 V8 实现]]：闭包在 V8 中的具体实现
- [[V8 对象模型解析]]：V8 的 Hidden Classes（Maps）机制
- [[内联缓存 (Inline Caching)]]：属性访问优化技术
- [[Turbofan 编译器解析]]：V8 的 JIT 编译器详解
- [[V8 数组处理机制]]：V8 对数组的优化策略
- [[渲染管线]]：浏览器渲染管线与 JavaScript 执行的关系

---

## 参考资料

- [MDN - JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)
- [V8 Blog - Launching Ignition and TurboFan](https://v8.dev/blog/launching-ignition-and-turbofan)
- [V8 官方文档](https://v8.dev/docs/)
- [ECMAScript® Language Specification](https://tc39.es/ecma262/)
- [WHATWG HTML Standard - Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)
- [V8 Ignition 解释器文档](https://v8.dev/docs/ignition)
- [V8 TurboFan 设计文档](https://v8.dev/docs/turbofan)
- [Mathias Bynens - JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics)
- [Node.js - The Event Loop, Timers, and nextTick()](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)

[^v8-pipeline]: 来源：[V8 Blog - Launching Ignition and TurboFan](https://v8.dev/blog/launching-ignition-and-turbofan)，V8 v5.9 引入 Ignition + TurboFan 执行管道。

[^ignition-memory]: 来源：[V8 Blog - Ignition Interpreter](https://v8.dev/blog/ignition-interpreter)，Ignition 在 ARM64 移动设备上将字节码内存占用降低约 9 倍。

[^turbofan]: 来源：[V8 TurboFan Design](https://v8.dev/docs/turbofan)，TurboFan 是 V8 的新一代优化编译器，取代了 Crankshaft。

[^agent]: 来源：[ECMAScript Spec - Agents](https://tc39.es/ecma262/#sec-agents)、[MDN - Agent execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#agent_execution_model)。

[^realm]: 来源：[ECMAScript Spec - Realms](https://tc39.es/ecma262/#sec-code-realms)、[MDN - Realms](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#realms)。

[^event-loop]: 来源：[WHATWG HTML Spec §8.1.7.3 Processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)。

[^node-event-loop]: 来源：[Node.js Docs - The Event Loop, Timers, and nextTick()](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)。

[^v8-gc]: 来源：[V8 Garbage Collection](https://v8.dev/blog/trash-talk)，V8 的分代式垃圾回收策略。

[^long-task]: 来源：[MDN - Long tasks](https://developer.mozilla.org/en-US/docs/Web/Performance/Long_tasks)，浏览器通常将 > 50ms 的连续执行视为长任务。

[^ptc]: 来源：[WebKit Blog - ECMAScript 6 Proper Tail Calls in WebKit](https://webkit.org/blog/6240/ecmascript-6-proper-tail-calls-in-webkit/)，目前仅 Safari 实现了 PTC。
