---
title: JavaScript 对象内存占用详解
created: 2026-04-22
updated: 2026-04-22
tags:
  - javascript
  - v8
  - memory
  - performance
  - internals
aliases:
  - JavaScript 对象内存大小
  - JS Object Memory Size
  - JavaScript 对象占多少内存
  - V8 对象内存布局
source_type: mixed
source_urls:
  - https://v8.dev/blog/fast-properties
  - https://mathiasbynens.be/notes/shapes-ics
  - https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management
  - https://github.com/v8/v8/blob/main/src/objects/js-objects.h
status: verified
---

## 核心结论

**JavaScript 语言规范（ECMAScript）没有定义对象的具体内存大小**。一个对象占多少内存完全由 JavaScript 引擎的实现决定。以 Chrome 和 Node.js 使用的 V8 引擎为例，64 位平台上一个空对象 `{}` 的 **shallow size 约为 24-48 字节**，实际占用取决于属性数量、属性类型、存储模式等因素。

> **重要前提**：以下所有具体数值均基于 V8 引擎（64 位架构），其他引擎（SpiderMonkey、JavaScriptCore）的实现细节不同，但基本原理相似。

---

## 为什么这个问题没有简单答案

JavaScript 对象的内存占用不是固定值，而是由多个维度共同决定：

1. **引擎实现**：ECMAScript 规范只定义对象的行为语义，不规定内存布局
2. **架构差异**：32 位 vs 64 位平台的指针大小不同（4 字节 vs 8 字节）
3. **对象形状**：属性数量、属性添加顺序影响隐藏类（Map）和内联属性数量
4. **存储模式**：快属性（fast properties）vs 慢属性/字典模式（dictionary mode）
5. **元素类型**：数组的 Elements 类型（Smi、Double、Generic、Holey 等）影响存储密度

---

## V8 中 JSObject 的内存布局

### 对象头（Object Header）

在 V8 中，每个 JavaScript 对象（JSObject）都包含一个对象头，64 位平台上的基本结构如下：

```
┌─────────────────────────────────┐
│         JSObject Header         │
├─────────────────────────────────┤
│ Map Pointer      (8 bytes)      │  ← 隐藏类指针，描述对象形状
│ Properties Ptr   (8 bytes)      │  ← 命名属性存储区指针
│ Elements Ptr     (8 bytes)      │  ← 数组元素存储区指针
│ In-object Props  (0-N × 8B)    │  ← 内联属性（直接嵌入对象）
└─────────────────────────────────┘
```

**对象头固定开销（64 位）**：

| 字段 | 大小 | 说明 |
|------|------|------|
| Map Pointer | 8 字节 | 指向隐藏类（V8 称 Map，其他引擎称 Shape/Structure） |
| Properties Pointer | 8 字节 | 指向外部属性数组（若无外部属性则为空） |
| Elements Pointer | 8 字节 | 指向元素数组（普通对象通常为空） |
| **对象头最小值** | **24 字节** | 无内联属性时的固定开销 |

> 来源：[V8 Blog - Fast properties in V8](https://v8.dev/blog/fast-properties)、[V8 源码 js-objects.h](https://github.com/v8/v8/blob/main/src/objects/js-objects.h)

### 空对象的实际占用

```javascript
const obj = {};
```

一个空对象 `{}` 在 V8 64 位平台上的内存占用：

- **对象头**：24 字节（Map + Properties + Elements 指针）
- **Map 对象本身**：约 64-96 字节（**多个相同形状的对象共享同一个 Map**，不重复分配）
- **实际 shallow size**：约 **24-48 字节**（取决于 V8 版本和内联属性预留空间）

> 注意：Map（隐藏类）是元数据，被所有相同形状的对象共享。计算单个对象内存时，Map 的开销应平摊或忽略。

---

## 属性如何影响内存

### 三种属性存储模式

V8 根据使用模式将命名属性存储在三个不同位置：

| 模式 | 存储位置 | 访问速度 | 内存开销 |
|------|----------|----------|----------|
| **In-object Properties** | 直接嵌入对象内部 | 最快（零间接） | 低，但受对象创建时预留空间限制 |
| **Fast Properties** | 外部属性数组 + Map 上的 DescriptorArray | 快（一次间接） | 中等，DescriptorArray 被同形状对象共享 |
| **Slow Properties（字典模式）** | 自包含哈希表 | 慢（哈希查找） | 高，每个对象独立存储完整元数据 |

### In-object Properties（内联属性）

```javascript
const point = { x: 1, y: 2 };
```

- 属性值直接存储在对象体内，无需额外指针跳转
- V8 在对象创建时根据初始属性数量预留内联空间
- 这是**最快的属性访问方式**，也是**内存效率最高**的方式
- 每个内联属性值占用 **8 字节**（64 位指针）

### Fast Properties（快属性）

当属性数量超过内联容量时，超出部分存储在外部属性数组中：

```javascript
const obj = { a: 1, b: 2, c: 3 };
obj.d = 4;  // 可能溢出到外部属性数组
obj.e = 5;
```

- 通过 `Properties Pointer` 访问，增加一次指针解引用
- 属性数组可以动态扩展
- 属性名和偏移量信息存储在共享的 DescriptorArray 中

### Slow Properties / Dictionary Mode（字典模式）

当对象经历大量属性删除或动态添加时，V8 会切换到字典模式：

```javascript
const obj = {};
for (let i = 0; i < 100; i++) {
  obj[`key${i}`] = i;
  if (i % 3 === 0) delete obj[`key${i-1}`];  // 频繁删除触发字典模式
}
```

- 属性存储在哈希表中，每个条目包含：key + value + 属性描述符
- **内存开销显著增加**：每个属性条目约 48-64 字节（含哈希表 overhead）
- 失去隐藏类优化，属性访问变慢
- 不再与其他对象共享元数据

> 来源：[V8 Blog - Fast properties in V8](https://v8.dev/blog/fast-properties)

---

## 数组的内存占用

数组在 V8 中有独立的 Elements 存储区，根据内容类型分为多种 Elements Kind：

### Elements 类型层级

```
PACKED_SMI_ELEMENTS          ← 最紧凑（小整数直接编码在指针中）
        ↓ (添加浮点数)
PACKED_DOUBLE_ELEMENTS       ← 每个元素 8 字节（原始 double）
        ↓ (添加对象/字符串)
PACKED_ELEMENTS              ← 每个元素 8 字节（指针）
        ↓ (删除元素/产生空洞)
HOLEY_SMI_ELEMENTS
        ↓
HOLEY_DOUBLE_ELEMENTS
        ↓
HOLEY_ELEMENTS               ← 最宽松，但性能最差
```

### 不同类型数组的内存对比

```javascript
// PACKED_SMI_ELEMENTS - 最紧凑
const smiArr = [1, 2, 3, 4, 5];
// 小整数（Smi）直接编码在值中，无需额外对象分配
// 数组元素存储：约 40 字节（5 个 Smi 值）

// PACKED_DOUBLE_ELEMENTS
const doubleArr = [1.1, 2.2, 3.3];
// 原始 double 值直接存储，每个 8 字节
// 数组元素存储：约 24 字节（3 × 8 字节）

// PACKED_ELEMENTS
const objArr = [{a:1}, {b:2}];
// 每个元素是指向对象的指针（8 字节）
// 数组元素存储：约 16 字节（2 × 8 字节指针）
// 加上每个对象自身的内存开销
```

### 数组对象本身的开销

```javascript
const arr = [1, 2, 3];
```

- **对象头**：24 字节（Map + Properties + Elements 指针）
- **Elements 存储区**：根据类型和长度而定
- **length 属性**：通常作为内联属性存储

一个包含 3 个 Smi 的数组总占用约：**24（对象头）+ 24（elements 存储）≈ 48 字节**

> 来源：[V8 Blog - Fast properties in V8](https://v8.dev/blog/fast-properties)

---

## 字符串的内存占用

字符串在 V8 中有多种内部表示：

| 类型 | 适用场景 | 内存特点 |
|------|----------|----------|
| **Smi String** | 极短字符串 | 值直接编码 |
| **SeqString（顺序字符串）** | 普通字符串 | 对象头 + 字符数组 |
| **ConsString** | 字符串拼接中间结果 | 两个子字符串的引用对 |
| **SlicedString** | 子字符串操作 | 父字符串引用 + 偏移 + 长度 |
| **ThinString** | 去重字符串 | 指向另一个字符串的薄包装 |

**SeqString 内存计算（64 位）**：

```
对象头（约 16-24 字节）+ 长度字段（4 字节）+ 字符数据
```

- Latin1 编码：每字符 1 字节
- UTF-16 编码：每字符 2 字节

```javascript
const str = "hello";  // 5 个字符
// 约 24（对象头）+ 4（长度）+ 5（Latin1 字符）≈ 33 字节
// 实际会按 8 字节对齐，约 40 字节
```

---

## 内置对象的额外开销

JavaScript 内置对象除了基础对象头，还有额外的内部状态：

| 对象类型 | 额外开销 | 说明 |
|----------|----------|------|
| `Date` | +16 字节 | 存储时间戳（double）+ 缓存字段 |
| `RegExp` | + 编译后的字节码 | 正则表达式编译结果 |
| `Map` | + 哈希表结构 | 键值对存储，比 plain object 更重 |
| `Set` | + 哈希表结构 | 唯一值集合 |
| `Promise` | + 状态机字段 | 状态、结果、回调链 |
| `Function` | + 字节码/机器码 | 函数体编译结果，可能很大 |
| `ArrayBuffer` | + 外部缓冲区 | 数据存储在 V8 堆外 |

---

## 如何测量对象内存

### Chrome DevTools Heap Snapshot

1. 打开 DevTools → Memory 面板
2. 选择 **Heap snapshot** 并点击录制
3. 搜索对象，查看 **Shallow Size** 和 **Retained Size**

- **Shallow Size**：对象自身直接占用的内存
- **Retained Size**：对象被 GC 后能释放的总内存（含其引用的所有对象）

### `performance.memory` API（仅 Chrome）

```javascript
// 仅限 Chrome，非标准 API
console.log(performance.memory);
// {
//   jsHeapSizeLimit: 4294700000,    // JS 堆上限
//   totalJSHeapSize: 23000000,      // 当前 JS 堆总大小
//   usedJSHeapSize: 18000000        // 已使用的 JS 堆大小
// }
```

> **限制**：此 API 仅提供堆级别的统计，无法测量单个对象的大小。且仅在 Chrome 中可用，非 Web 标准。

### Node.js `--trace-gc` 和 `v8.getHeapStatistics()`

```javascript
const v8 = require('v8');
console.log(v8.getHeapStatistics());
// {
//   total_heap_size: 7340032,
//   total_heap_size_executable: 1048576,
//   total_physical_size: 7340032,
//   total_available_size: 4341612128,
//   used_heap_size: 4204568,
//   heap_size_limit: 4345298944,
//   ...
// }
```

### V8 内建函数（需 `--allow-natives-syntax`）

```bash
node --allow-natives-syntax script.js
```

```javascript
const obj = { x: 1, y: 2 };
%DebugPrint(obj);  // 打印对象的详细内存布局信息
```

---

## Shallow Size vs Retained Size

理解这两个概念对分析内存至关重要：

```javascript
const largeData = new Array(10000).fill({ value: 42 });
const wrapper = { data: largeData };
```

- **`wrapper` 的 Shallow Size**：约 48 字节（对象头 + 一个指针）
- **`wrapper` 的 Retained Size**：约 80KB+（包含 `largeData` 数组及其所有元素）

**实际意义**：
- 优化内存时，关注 **Retained Size** 而非 Shallow Size
- 一个看似很小的对象可能通过引用链持有大量内存
- 释放一个大对象的关键是让 GC Roots 无法到达它

---

## 内存优化最佳实践

### 1. 保持对象形状一致

```javascript
// 好 - 相同属性顺序，共享 Map
function createPoint(x, y) {
  return { x, y };
}

// 差 - 不同顺序产生不同 Map
function createPointBad(x, y) {
  if (x > 0) return { x, y };
  return { y, x };  // 不同的隐藏类！
}
```

### 2. 避免删除属性

```javascript
// 差 - 删除属性触发字典模式风险
delete obj.temp;

// 好 - 设为 null/undefined 保持形状
obj.temp = null;
```

### 3. 使用 TypedArray 处理大量数值

```javascript
// 普通数组 - 每个数字有对象开销
const arr = new Array(1000000).fill(0);

// TypedArray - 紧凑存储，无对象开销
const typed = new Float64Array(1000000);  // 精确 8MB
```

### 4. 使用 `Map` 替代动态键对象

```javascript
// 动态键对象 - 可能退化为字典模式
const obj = {};
for (let i = 0; i < 1000; i++) {
  obj[`key_${i}`] = i;
}

// Map - 原生哈希表实现，更高效
const map = new Map();
for (let i = 0; i < 1000; i++) {
  map.set(`key_${i}`, i);
}
```

### 5. 利用 WeakMap 管理可选引用

```javascript
// 强引用 - 阻止 GC 回收
const cache = new Map();
cache.set(domNode, data);  // domNode 不会被 GC

// 弱引用 - 允许 GC 回收
const weakCache = new WeakMap();
weakCache.set(domNode, data);  // domNode 可被 GC，entry 自动清理
```

---

## 常见误区

### 误区 1："空对象只占几个字节"

实际上，空对象 `{}` 在 V8 中至少有 24 字节的对象头开销，加上 Map 元数据。大量创建空对象时，累积开销不容忽视。

### 误区 2："`delete` 属性会释放内存"

`delete` 操作只是将属性标记为删除，不一定立即释放内存。频繁 `delete` 反而可能触发字典模式，**增加**内存开销。

### 误区 3："数组比对象省内存"

不一定。稀疏数组（如 `arr[999999] = 1`）会退化为字典模式，内存开销远超紧凑的普通对象。

### 误区 4："`null` 比 `undefined` 省内存"

两者都是原始值，在 V8 中都有特殊编码，内存占用相同。区别在于语义：`null` 表示"有意为空"，`undefined` 表示"未定义"。

---

## 与其他引擎对比

| 特性 | V8 (Chrome/Node.js) | SpiderMonkey (Firefox) | JavaScriptCore (Safari) |
|------|---------------------|------------------------|-------------------------|
| 隐藏类 | Map | Shape | Structure |
| 对象头 | ~24 字节（64 位） | 类似 | 类似 |
| 内联属性 | 支持 | 支持 | 支持 |
| 字典模式 | 支持 | 支持 | 支持 |
| Smi 编码 | 31 位小整数 | 31 位小整数 | 31 位小整数 |
| 元素类型优化 | 20+ 种 | 类似 | 类似 |

> 各引擎具体数值可能随版本变化，以上为概念性对比。

---

## 限制与注意事项

1. **引擎版本差异**：V8 持续优化对象模型，具体数值随版本变化
2. **无法精确控制**：JavaScript 不提供手动内存管理 API，对象布局由引擎决定
3. **GC 影响**：对象在 Young Generation 和 Old Generation 中的布局可能不同
4. **平台差异**：32 位平台指针为 4 字节，对象头更小，但地址空间受限
5. **测量困难**：没有标准 API 获取单个对象的精确内存大小

---

## 相关概念

| 概念 | 关联笔记 |
|------|----------|
| V8 对象模型 | `V8/V8 对象模型解析` |
| 隐藏类与内联缓存 | `V8/内联缓存 (Inline Caching)` |
| Chrome 内存限制 | `Browser/Chrome 的内存限制` |
| 内存调试工具 | `Frontend Performance Optimization/Chrome DevTools 性能与内存调试工具` |
| JavaScript 数据类型 | `JavaScript/基础数据类型深度解析` |
| 闭包与内存 | `JavaScript/JavaScript 闭包与 V8 实现` |

---

## 参考资料

- [V8 Blog - Fast properties in V8](https://v8.dev/blog/fast-properties) - V8 官方博客，详解属性存储机制
- [JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics) - Mathias Bynens & Benedikt Meurer，V8 团队成员撰写的引擎原理
- [MDN - Memory management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management) - MDN 内存管理指南
- [V8 Source - js-objects.h](https://github.com/v8/v8/blob/main/src/objects/js-objects.h) - V8 JSObject 定义源码
- [V8 Source - objects.h](https://github.com/v8/v8/blob/main/src/objects/objects.h) - V8 对象系统基础定义
- [V8 Blog - Trash talk: the Orinoco garbage collector](https://v8.dev/blog/trash-talk) - V8 垃圾回收器原理
