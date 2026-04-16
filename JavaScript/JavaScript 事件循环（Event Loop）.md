---
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - JavaScript
  - 异步编程
  - 浏览器
  - 运行时
  - 核心概念
aliases:
  - Event Loop
  - 事件循环
  - 宏任务与微任务
  - JavaScript 执行模型
  - 同步异步任务执行顺序
  - JS 任务执行顺序
source_type: mixed
source_urls:
  - 'https://html.spec.whatwg.org/multipage/webappapis.html#event-loops'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model
  - >-
    https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide
status: verified
---

## 是什么

Event Loop（事件循环）是 JavaScript 实现**单线程异步非阻塞**编程的核心机制。它由宿主环境（浏览器或 Node.js）提供，负责协调**调用栈（Call Stack）**、**任务队列（Task Queue / Macrotask Queue）**和**微任务队列（Microtask Queue）**之间的执行顺序。

JavaScript 引擎本身只负责执行代码，不包含 Event Loop。Event Loop 由 HTML 标准定义（[WHATWG HTML Spec §8.1.7](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)），浏览器和 Node.js 等宿主环境各自实现。

## 为什么重要

JavaScript 是单线程语言，同一时刻只能执行一段代码。如果没有 Event Loop：

- 等待 I/O（网络请求、文件读写、定时器）会**阻塞整个线程**，页面无法响应用户交互
- 浏览器会弹出"脚本运行时间过长"提示，甚至崩溃

Event Loop 通过**异步回调 + 队列调度**，让 JavaScript 在等待异步操作完成的同时，仍能处理其他任务，实现**永不阻塞（Never Blocking）**的执行模型[^never-blocking]。

[^never-blocking]: 来源：[MDN - JavaScript execution model: Never blocking](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#never_blocking)。注意 `alert()` 和同步 XHR 是遗留的阻塞 API，应避免使用。

## 前置概念

### Agent（代理）

ECMAScript 规范中，每个自主执行 JavaScript 的实体称为 **Agent**[^agent]。一个 Agent 包含三大数据结构：

| 数据结构 | 作用 | 特性 |
|---------|------|------|
| **Heap（堆）** | 存储对象 | 对象创建时分配内存 |
| **Stack（调用栈）** | 跟踪执行上下文 | 后进先出（LIFO） |
| **Queue（任务队列）** | 管理异步任务 | 先进先出（FIFO），即 Event Loop |

在浏览器中，一个 Agent 可以是：

- **Similar-origin window agent**：主页面及其同源 iframe
- **Dedicated worker agent**：单个 Dedicated Worker
- **Shared worker agent**：单个 Shared Worker
- **Service worker agent**：单个 Service Worker
- **Worklet agent**：单个 Worklet

每个 Worker 创建独立的 Agent；同一文档及其同源 iframe 通常共享一个 Agent。

[^agent]: 来源：[ECMAScript Spec - Agents](https://tc39.es/ecma262/#sec-agents)、[MDN - Agent execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#agent_execution_model)

### Realm（领域）

每个 Agent 拥有一个或多个 **Realm**，代表一个独立的 JavaScript 全局执行环境[^realm]。一个 Realm 包含：

- 一组内置对象（`Array`、`Array.prototype` 等）
- 全局变量和 `globalThis`
- 模板字面量数组缓存

在浏览器中，每个 `Window`、`WorkerGlobalScope` 或 `WorkletGlobalScope` 对应一个 Realm。因此每个 iframe 都在不同的 Realm 中执行，即使它们与父页面共享同一个 Agent。

这也是为什么需要 `Array.isArray()` 而不是依赖 `instanceof Array` —— 不同 Realm 中的 `Array.prototype` 是不同的对象。

[^realm]: 来源：[ECMAScript Spec - Realms](https://tc39.es/ecma262/#sec-code-realms)、[MDN - Realms](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#realms)

### 调用栈（Call Stack）

调用栈跟踪当前的**执行上下文（Execution Context）**，也称为栈帧（Stack Frame）。每个执行上下文记录：

- 代码评估状态
- 当前执行的模块/脚本/函数
- 当前 Realm
- 变量绑定（`var`、`let`、`const`、`function`、`class` 等）
- `this` 引用

**同步代码的执行流程**：

```js
function foo(b) {
  const a = 10;
  return a + b + 11;
}

function bar(x) {
  const y = 3;
  return foo(x * y);
}

const baz = bar(7); // 42
```

1. 执行 `bar(7)`，创建 `bar` 的帧入栈
2. `bar` 内调用 `foo(21)`，创建 `foo` 的帧入栈
3. `foo` 返回 42，帧出栈
4. `bar` 返回 42，帧出栈
5. 栈为空，当前任务完成

## Event Loop 运行机制

### 核心处理流程

HTML 标准定义的 Event Loop 处理模型如下[^event-loop-processing]：

1. **选择最老的任务**：从任务队列中选取最早加入的任务
2. **执行任务**：将该任务对应的执行上下文推入调用栈，运行代码
3. **任务完成**：调用栈清空后，任务标记为完成
4. **清空微任务队列**：**在执行完一个任务后、渲染前**，检查并清空微任务队列中的所有微任务
5. **更新渲染**：浏览器可选择性地更新页面渲染（重排、重绘）
6. **进入下一轮循环**：回到步骤 1

```
┌─────────────────────────────────────────┐
│           Event Loop（浏览器）            │
│                                         │
│  ┌───────┐    ┌───────┐    ┌─────────┐ │
│  │ 同步   │───▶│ 微任务 │───▶│ 渲染    │ │
│  │ 代码   │    │ 队列   │    │ 更新    │ │
│  └───────┘    └───────┘    └─────────┘ │
│      ▲              ▲                   │
│      │              │                   │
│  ┌───┴──────┐  ┌────┴──────┐           │
│  │ 任务队列  │  │ 微任务    │           │
│  │(宏任务)   │  │(高优先级)  │           │
│  └──────────┘  └───────────┘           │
└─────────────────────────────────────────┘
```

[^event-loop-processing]: 来源：[WHATWG HTML Spec §8.1.7.3 Processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)

### 任务（Task / Macrotask）

任务是宿主环境通过标准机制调度的异步操作，放入**任务队列（Task Queue）**[^tasks]。

以下情况会产生任务：

| 场景 | 触发方式 |
|------|---------|
| 执行新程序 | `<script>` 标签加载、控制台执行 |
| 用户交互 | 点击、键盘、滚动等事件回调 |
| 定时器 | `setTimeout()`、`setInterval()` 回调到期 |
| 网络 I/O | `fetch()` 响应到达、XMLHttpRequest 回调 |
| 消息传递 | `postMessage()`、`MessageChannel` |
| UI 渲染 | 页面加载、资源加载完成 |

**特点**：

- 每轮 Event Loop **最多执行一个任务**
- 按入队顺序依次执行（FIFO）
- 任务之间可能插入渲染更新

[^tasks]: 来源：[MDN - Microtask guide: Tasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide#tasks)

### 微任务（Microtask）

微任务是优先级更高的异步回调，放入**微任务队列（Microtask Queue）**[^microtasks]。

以下情况会产生微任务：

| 场景 | 触发方式 |
|------|---------|
| Promise 回调 | `.then()`、`.catch()`、`.finally()` |
| `queueMicrotask()` | 显式调度微任务 |
| MutationObserver | DOM 变化回调 |
| `async/await` | `await` 后的代码（本质是 Promise） |

**特点**：

- 每轮 Event Loop 中，**在当前任务执行完毕后、渲染前**，清空整个微任务队列
- 如果微任务执行过程中又产生了新的微任务，**这些新微任务也会在同一轮中被执行**，直到微任务队列为空
- 微任务**优先于下一个任务**执行

**警告**：由于微任务可以递归产生新微任务，如果处理不当，可能导致 Event Loop 永远在处理微任务，阻塞渲染和其他任务。需谨慎使用。

[^microtasks]: 来源：[MDN - Microtask guide: Microtasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide#microtasks)

### Run-to-Completion（运行至完成）

每个任务或微任务在执行期间**不会被中断**，会完整执行完毕后才让出控制权[^rtc]。这意味着：

```js
const promise = Promise.resolve();
let i = 0;
promise.then(() => {
  i += 1;
  console.log(i); // 1
});
promise.then(() => {
  i += 1;
  console.log(i); // 2
});
```

虽然两个回调看起来可能产生竞态条件，但输出完全可预测：`1`、`2`。因为每个微任务运行至完成后才执行下一个，执行顺序始终是 `i += 1; console.log(i); i += 1; console.log(i);`。

[^rtc]: 来源：[MDN - Run-to-completion](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model#run-to-completion)

## 执行顺序详解

### 单轮 Event Loop 的完整流程

```
1. 取出并执行一个任务（宏任务）
   ↓
2. 清空微任务队列（可能产生新微任务，继续清空直到为空）
   ↓
3. 可选：更新渲染（重排、重绘）
   ↓
4. 回到步骤 1，进入下一轮循环
```

### 代码示例与分析

```js
console.log('1. 同步代码 - script start');

setTimeout(() => {
  console.log('2. setTimeout 回调（宏任务）');
}, 0);

Promise.resolve()
  .then(() => {
    console.log('3. Promise.then 回调（微任务）');
  })
  .then(() => {
    console.log('4. 链式 .then 回调（微任务）');
  });

queueMicrotask(() => {
  console.log('5. queueMicrotask（微任务）');
});

console.log('6. 同步代码 - script end');
```

**输出顺序**：

```
1. 同步代码 - script start
6. 同步代码 - script end
3. Promise.then 回调（微任务）
5. queueMicrotask（微任务）
4. 链式 .then 回调（微任务）
2. setTimeout 回调（宏任务）
```

**分析**：

1. 同步代码 `1` 和 `6` 直接在当前执行上下文中运行
2. `setTimeout` 回调被放入**任务队列**（宏任务），等待下一轮 Event Loop
3. `Promise.resolve().then()` 的回调被放入**微任务队列**
4. `queueMicrotask()` 也被放入**微任务队列**
5. 同步代码执行完毕后，Event Loop 开始清空微任务队列：
   - 先执行第一个 `.then()` 回调（输出 `3`），它又链式产生了第二个 `.then()` 回调（进入微任务队列）
   - 然后执行 `queueMicrotask` 回调（输出 `5`）
   - 最后执行链式的第二个 `.then()` 回调（输出 `4`）
6. 微任务队列清空后，进入下一轮 Event Loop，执行 `setTimeout` 回调（输出 `2`）

### 微任务中产生新微任务

```js
console.log('start');

queueMicrotask(() => {
  console.log('microtask 1');
  queueMicrotask(() => {
    console.log('microtask 1.1');
  });
});

queueMicrotask(() => {
  console.log('microtask 2');
});

console.log('end');
```

**输出**：

```
start
end
microtask 1
microtask 2
microtask 1.1
```

注意 `microtask 1.1` 在 `microtask 2` 之后执行，因为它是 `microtask 1` 执行期间**新加入**微任务队列的，排在已有微任务之后。

## async/await 与 Event Loop

`async/await` 是 Promise 的语法糖，其执行顺序遵循 Promise 的微任务规则[^async-await]。

```js
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');

setTimeout(() => {
  console.log('setTimeout');
}, 0);

async1();

new Promise((resolve) => {
  console.log('promise executor');
  resolve();
}).then(() => {
  console.log('promise then');
});

console.log('script end');
```

**输出**：

```
script start
async1 start
async2
promise executor
script end
async1 end
promise then
setTimeout
```

**关键点**：

- `await` 后面的代码（`console.log('async1 end')`）本质上是一个 Promise `.then()` 回调，被放入微任务队列
- Promise 的 executor 函数（`console.log('promise executor')`）是**同步执行**的
- `.then()` 回调是微任务，在同步代码之后执行

[^async-await]: 来源：[MDN - async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)

## Node.js 中的 Event Loop

Node.js 的 Event Loop 实现（libuv）与浏览器有所不同，分为**多个阶段**[^node-event-loop]：

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout / setInterval 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  上一轮未执行的 I/O 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  I/O 回调、文件/网络事件
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──│      close callbacks      │  close 事件回调（如 socket.on('close')）
   └───────────────────────────┘
```

**与浏览器的关键差异**：

| 特性 | 浏览器 | Node.js |
|------|--------|---------|
| 微任务执行时机 | 每个任务之后、渲染前 | 每个阶段切换时 |
| `setTimeout` vs `setImmediate` | 只有 `setTimeout` | `setImmediate` 在 `setTimeout(fn, 0)` 之前执行（在 poll 阶段之后） |
| 渲染更新 | 在微任务清空后可能触发 | 不涉及渲染 |

```js
// Node.js 环境
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// 输出不确定，取决于 Node.js 启动速度
// 但如果在 I/O 回调中：
// setImmediate 总是先于 setTimeout(fn, 0) 执行
```

[^node-event-loop]: 来源：[Node.js Docs - The Event Loop, Timers, and nextTick()](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)

## `process.nextTick`（Node.js 特有）

Node.js 中还有一个 `process.nextTick()`，它**不属于 Event Loop 的任何阶段**，而是在当前操作完成后、Event Loop 继续之前立即执行[^nexttick]。

```js
// Node.js
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
// nextTick 总是先于 promise 执行
```

`process.nextTick()` 的优先级高于 Promise 微任务，过度使用会导致 I/O 和渲染饥饿，应谨慎使用。

[^nexttick]: 来源：[Node.js Docs - process.nextTick()](https://nodejs.org/api/process.html#processnexttickcallback-args)

## 常见 API 归类

### 产生宏任务（Task）的 API

- `setTimeout()`
- `setInterval()`
- `setImmediate()`（Node.js）
- I/O 操作（`fs.readFile`、网络请求回调）
- UI 渲染任务
- `requestAnimationFrame()`
- 用户交互事件回调（`click`、`keydown` 等）
- `postMessage()`

### 产生微任务（Microtask）的 API

- `Promise.then()` / `.catch()` / `.finally()`
- `queueMicrotask()`
- `MutationObserver` 回调
- `async/await`（await 后的代码）
- `process.nextTick()`（Node.js，优先级高于标准微任务）

## 常见误区

### 误区 1：`setTimeout(fn, 0)` 会立即执行

`setTimeout(fn, 0)` 的 `0` 表示**最小延迟**，不是立即执行。HTML 标准规定嵌套 `setTimeout` 的最小延迟为 **4ms**[^timer-min-delay]。回调必须等待当前任务和微任务队列全部完成后才能执行。

[^timer-min-delay]: 来源：[WHATWG HTML Spec §8.1.7.2](https://html.spec.whatwg.org/multipage/webappapis.html#timers)，嵌套定时器最小延迟 4ms。

### 误区 2：Promise 是异步的，所以 executor 也是异步的

Promise 的 executor 函数（`new Promise((resolve) => { ... })` 中的回调）是**同步执行**的。只有 `.then()`、`.catch()`、`.finally()` 回调才是异步的微任务。

### 误区 3：微任务队列一次只执行一个

微任务队列在每轮 Event Loop 中会**全部清空**。如果微任务执行过程中又产生了新微任务，这些新微任务也会在同一轮中被执行，直到队列为空。

### 误区 4：`async/await` 会阻塞 Event Loop

`await` 不会阻塞 Event Loop。它只是将后续代码包装成 Promise 回调放入微任务队列，然后让出控制权。其他任务（包括渲染）可以在 `await` 等待期间正常执行。

### 误区 5：Event Loop 是 JavaScript 语言的一部分

Event Loop 不是 ECMAScript 规范的一部分，而是由**宿主环境**（HTML 标准、Node.js）定义和实现的。ECMAScript 只定义了 Agent、Realm、Job Queue 等抽象概念。

## 最佳实践

### 避免长任务阻塞渲染

如果同步代码执行时间过长（通常 > 50ms），会阻塞 Event Loop，导致无法处理用户交互和渲染[^long-task]。

```js
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
    const chunk = data.slice(i, i + chunkSize);
    processChunk(chunk);
    // 让出控制权，允许渲染和其他任务执行
    await new Promise((resolve) => setTimeout(resolve, 0));
  }
}
```

### 使用微任务保证执行顺序

当需要确保某些操作在当前执行上下文中尽早执行，但又不想同步执行时，使用微任务：

```js
// 确保缓存命中和未命中时的事件触发时机一致
function getData(url) {
  if (cache[url]) {
    // 使用微任务，与 fetch 的 Promise 回调保持一致的执行时机
    queueMicrotask(() => {
      dispatchEvent(new CustomEvent('data-loaded', { detail: cache[url] }));
    });
  } else {
    fetch(url).then((data) => {
      cache[url] = data;
      dispatchEvent(new CustomEvent('data-loaded', { detail: data }));
    });
  }
}
```

### 避免微任务递归导致饥饿

```js
// ❌ 危险：微任务递归产生新微任务，可能导致渲染饥饿
function dangerousLoop() {
  doWork();
  queueMicrotask(dangerousLoop); // 微任务队列永远不会空
}

// ✅ 安全：使用 setTimeout 让出控制权，允许渲染
function safeLoop() {
  doWork();
  setTimeout(safeLoop, 0); // 放入任务队列，渲染有机会执行
}
```

[^long-task]: 来源：[MDN - Long tasks](https://developer.mozilla.org/en-US/docs/Web/Performance/Long_tasks)。浏览器通常将 > 50ms 的连续执行视为长任务。

## 相关概念

- **[Promise](./Promise)**：微任务的主要来源，异步编程的核心原语
- **[async/await](./async-await)**：Promise 的语法糖，简化异步代码书写
- **[浏览器的事件冒泡](./浏览器的事件冒泡)**：DOM 事件传播机制，事件回调作为任务进入 Event Loop
- **[MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)**：通过 `postMessage` 产生任务的跨线程通信 API
- **[requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)**：在下次渲染前执行回调，产生任务
- **[requestIdleCallback](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback)**：在浏览器空闲时执行回调，产生任务

## 参考资料

- [WHATWG HTML Standard - Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)（权威规范）
- [MDN - JavaScript execution model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Execution_model)
- [MDN - Using microtasks with queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [MDN - Event Loop (中文版)](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Event_loop)
- [Node.js - The Event Loop, Timers, and nextTick()](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Jake Archibald - Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)（经典深入解析）
- [Philip Roberts - What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)（JSConf 2014 演讲，可视化讲解）
