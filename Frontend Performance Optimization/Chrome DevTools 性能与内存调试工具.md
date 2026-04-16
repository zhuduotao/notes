---
created: 2026-04-16
updated: 2026-04-16
tags:
  - Chrome
  - DevTools
  - 性能调试
  - 内存分析
  - 火焰图
  - 前端调试
aliases:
  - Chrome DevTools Performance and Memory Debugging
  - Chrome 火焰图分析
  - Chrome 内存 Dump
  - Chrome 性能分析工具
source_type: official-doc
source_urls:
  - https://developer.chrome.com/docs/devtools/memory-problems
  - https://developer.chrome.com/docs/devtools/memory-problems/get-started
  - https://developer.chrome.com/docs/devtools/memory-problems/heap-snapshots
  - https://developer.chrome.com/docs/devtools/memory-problems/allocation-profiler
  - https://developer.chrome.com/docs/devtools/performance
  - https://developer.chrome.com/docs/devtools/performance/save-trace
  - https://developer.chrome.com/docs/devtools/memory-inspector
  - https://developer.chrome.com/docs/devtools/coverage
  - https://developer.chrome.com/docs/devtools/rendering/performance
  - https://developer.chrome.com/docs/devtools/performance-monitor
status: verified
---

## 概述

Chrome DevTools 提供了完整的性能与内存调试工具链，覆盖 **CPU 执行分析（火焰图）**、**内存快照（Heap Snapshot）**、**内存分配追踪**、**二进制内存检查（Memory Inspector）**、**代码覆盖率**、**渲染性能监控** 等场景。本文档系统整理各工具的用途、核心用法、适用场景和限制。

> 本文基于 Chrome 129+ 版本编写。部分功能在旧版本中可能不存在或 UI 不同，可通过 `chrome://version/` 查看当前版本。

---

## 一、Performance 面板 — 火焰图与运行时性能分析

### 用途

Performance 面板用于记录和分析页面运行时的性能数据，生成 **火焰图（Flame Chart）**，帮助定位 JavaScript 执行瓶颈、长任务（Long Tasks）、强制重排（Forced Reflow）等问题。

### 核心用法

#### 1. 录制性能数据

1. 打开 DevTools（`Cmd+Option+I` / `Ctrl+Shift+I`），切换到 **Performance** 面板
2. 建议先在无痕模式下打开页面，避免扩展程序干扰
3. 在 Capture Settings 中设置 **CPU Throttling**（推荐 `4x slowdown` 模拟移动设备）
4. 启用 **Memory** 复选框（可选，用于同时观察内存变化）
5. 点击 **Record** 按钮开始录制
6. 执行需要分析的操作
7. 点击 **Stop** 停止录制，DevTools 自动处理数据并展示结果

#### 2. 火焰图解读

火焰图位于 **Main** 区域，展示主线程上的函数调用栈：

- **X 轴**：时间线，越宽表示耗时越长
- **Y 轴**：调用栈深度，上层函数由下层函数调用
- **颜色编码**：
  - 黄色：JavaScript 执行
  - 紫色：样式计算与布局（Layout）
  - 绿色：绘制与栅格化（Paint & Composite）
  - 蓝色：解析 HTML
  - 浅蓝：脚本解析与编译

#### 3. 关键指标图表

| 图表 | 含义 | 关注点 |
|------|------|--------|
| **FPS** | 帧率，目标 60 FPS | 红色条表示帧率过低 |
| **CPU** | CPU 使用率分解 | 颜色填满表示 CPU 满载 |
| **NET** | 网络请求时间线 | 识别阻塞性请求 |
| **HEAP** | JS 堆内存（启用 Memory 后） | 观察内存增长趋势 |
| **Counter** | DOM Nodes、Listeners、GPU Memory | 检测节点/监听器泄漏 |

#### 4. 分析流程

1. 查看 **Summary** 标签页，了解时间主要消耗在哪个阶段（Scripting / Rendering / Painting / System / Idle）
2. 在 **Main** 火焰图中寻找宽条（耗时长的任务）
3. 注意事件右上角的 **红色三角警告**，表示可能存在性能问题（如长任务、强制重排）
4. 点击具体事件，在 Summary 中查看详细信息，包括 **Initiated by**（触发源）和源码链接
5. 使用 **WASD** 键导航：W 放大、S 缩小、A 左移、D 右移

### 保存与分享 Trace

录制完成后，可点击 **Download** 保存 trace 文件，选项包括：

| 选项 | 说明 | 默认 |
|------|------|------|
| **Include annotations** | 包含标注信息 | 开启 |
| **Include resource content** | 包含 HTML/JS/CSS 内容（可在 Sources 面板查看） | 关闭 |
| **Include script source maps** | 包含源码映射（需先开启 resource content） | 关闭 |
| **Compress with gzip** | 使用 gzip 压缩（Chrome 142+ 默认开启） | 开启 |

> **注意**：若资源内容或 source maps 包含敏感信息，应将 trace 文件视为敏感文件处理。

### 适用场景

- 定位动画卡顿（jank）原因
- 分析长任务（> 50ms）
- 检测强制同步布局（Forced Synchronous Layout）
- 识别渲染瓶颈（Scripting vs Rendering vs Painting）
- 对比优化前后的性能差异

### 限制

- 录制过程本身会引入性能开销，数据不完全等同于真实运行
- 无法测量真实用户数据（需结合 RUM 工具如 web-vitals 库）
- 对于 Web Worker 中的执行，需在 **Select JavaScript VM instance** 中选择对应 Worker

---

## 二、Memory 面板 — 内存快照与分析

### 用途

Memory 面板用于分析 JS 堆内存分布，检测内存泄漏（Memory Leak）、内存膨胀（Memory Bloat）和频繁垃圾回收（Frequent GC）。

### 核心工具

#### 1. Heap Snapshot（堆快照）

**用途**：拍摄某一时刻的 JS 堆内存快照，展示 JS 对象和 DOM 节点的内存分布。

**操作步骤**：

1. 打开 Memory 面板，选择 **Heap snapshot**
2. 选择目标 JavaScript VM 实例
3. 点击 **Take snapshot**（快捷键 `Cmd+E` / `Ctrl+E` 可快速重复拍摄）
4. 快照拍摄前会自动执行一次 GC

**四种查看模式**：

| 模式 | 内容 | 用途 |
|------|------|------|
| **Summary** | 按构造函数名和源码位置分组 | 按类型查找对象，追踪 DOM 泄漏 |
| **Comparison** | 两个快照之间的差异 | 对比操作前后的内存变化，确认泄漏 |
| **Containment** | 堆内容的对象结构 | 分析闭包、全局命名空间引用 |
| **Statistics** | 内存分配的饼图 | 查看 code、strings、JS arrays、typed arrays 的占比 |

**Summary 视图关键列**：

| 列 | 含义 |
|----|------|
| **Distance** | 到 GC Root 的最短属性引用路径长度 |
| **Shallow Size** | 对象自身直接占用的内存（字节） |
| **Retained Size** | 对象被删除后（连同其依赖且不可达的对象）能释放的总内存 |

**常用过滤器**（Summary 视图右侧下拉菜单）：

- **All objects**：所有对象（默认）
- **Objects allocated before snapshot 1**：第一个快照前已分配的对象
- **Objects allocated between Snapshots 1 and 2**：两个快照之间新增的对象
- **Duplicated strings**：重复存储的字符串
- **Objects retained by detached nodes**：被 detached DOM 节点保留的对象
- **Objects retained by the DevTools console**：被 DevTools 控制台保留的对象

**特殊条目说明**：

| 条目 | 说明 |
|------|------|
| `(array)` | V8 内部数组对象，如 `(object elements)[]`、`(object properties)[]` |
| `(compiled code)` | V8 为函数优化分配的代码缓存数据 |
| `(concatenated string)` | V8 的 Rope 结构，延迟拼接的字符串对 |
| `InternalNode` | Blink C++ 对象（非 V8 堆对象） |
| `(object shape)` | V8 隐藏类（Hidden Class / Map） |
| `(sliced string)` | V8 切片字符串，指向原字符串的偏移引用 |
| `system / Context` | 闭包的局部变量上下文 |

**检测 Detached DOM Nodes 的标准流程**：

1. 拍摄快照 1（操作前）
2. 执行可疑操作（如打开/关闭弹窗、切换路由）
3. 执行反向操作并重复几次
4. 拍摄快照 2
5. 切换到 **Comparison** 视图，对比 Snapshot 1
6. 在 Class filter 中输入 `Detached` 筛选 detached DOM 树
7. 展开树结构，查看保留引用链（Retainers 面板）

#### 2. Allocation Timeline（分配时间线）

**用途**：实时记录内存分配的时间线，追踪新内存分配的来源。

**操作步骤**：

1. 选择 **Allocations on timeline**
2. 点击 **Record** 开始录制
3. 执行操作
4. 点击 **Stop** 停止
5. 蓝色竖条表示新内存分配，点击可放大查看该时间段分配的对象

**适用场景**：定位持续增长的内存分配源头，适合追踪间歇性泄漏。

#### 3. Allocation Sampling（分配采样）

**用途**：按函数采样内存分配，低开销，找出分配内存最多的函数。

**操作步骤**：

1. 选择 **Allocation sampling**
2. 点击 **Start**
3. 执行操作
4. 点击 **Stop**
5. 默认 **Heavy (Bottom Up)** 视图，按内存分配量降序排列函数

**适用场景**：生产环境或长时间运行的页面，比 Allocation Timeline 开销更低。

#### 4. Detached Elements（游离元素分析）

**用途**：专门展示因 JavaScript 引用而未被 GC 回收的 detached DOM 元素。

**操作步骤**：

1. 选择 **Detached elements**
2. 点击 **Start**
3. 执行操作
4. 查看具体的 HTML 节点和节点数量

### 内存术语

| 术语 | 说明 |
|------|------|
| **Shallow Size** | 对象自身直接占用的内存大小。通常只有数组和字符串有显著的 shallow size |
| **Retained Size** | 对象被删除后（连同其依赖且不可达的对象）能释放的总内存 |
| **GC Roots** | 垃圾回收的起始引用点，包括 window 全局对象、DOM 树、调试器上下文等 |
| **Distance** | 堆快照中对象到 GC Root 的最短属性引用路径长度 |
| **Dominator** | 支配者对象。若从 GC Root 到对象 A 的所有路径都经过对象 B，则 B 支配 A |
| **Cons String** | 拼接字符串的中间表示，由字符串对组成，按需执行实际拼接 |
| **Map（Hidden Class）** | 描述对象类型和布局的内部结构，用于快速属性访问 |
| **Wrapper Object** | JS 侧包装对象，用于访问不在 V8 堆中的原生对象（如 DOM 节点） |

### 最佳实践

- **录制前后各执行一次强制 GC**（点击垃圾桶图标），以便对比 GC 前后的内存基线
- **使用无痕模式**录制，避免扩展程序干扰
- **命名闭包函数**，便于在快照中区分不同闭包
- **忽略已知 Retainer**：右键 Retainer 选择 "Ignore this retainer" 可临时隐藏已知引用，快速排查其他保留路径

---

## 三、Memory Inspector — 二进制内存检查

> 自 Chrome 91 起可用。WebAssembly.Memory 检查需 Chrome 107+。

### 用途

检查 `ArrayBuffer`、`TypedArray`、`DataView` 和 `WebAssembly.Memory` 的原始字节内容。这些二进制数据缓冲区**不存储在 V8 JS 堆中**，因此 Memory 面板的 Heap Snapshot 无法直接查看其内容。

### 打开方式

1. **菜单打开**：More Options > More tools > Memory inspector
2. **调试时打开**：在 Sources 面板的 Scope 面板中，点击 `ArrayBuffer` 属性旁的图标，或右键选择 "Reveal in Memory Inspector panel"

### 界面结构

| 区域 | 功能 |
|------|------|
| **Navigation bar** | 地址输入（十六进制）、前进/后退、跳转导航、刷新 |
| **Memory buffer** | 左侧地址（十六进制）、中间内存字节（十六进制）、右侧 ASCII 表示 |
| **Value inspector** | 多种值解释（Int8/16/32/64, Uint8/16/32/64, Float32/64, Pointer 等），支持大小端切换 |

### 适用场景

- 调试 WebAssembly 应用的 C++ 内存
- 检查二进制数据解析是否正确（如图片解码、协议解析）
- 分析 TypedArray 数据布局

### 限制

- 仅支持 `ArrayBuffer` 类型的对象，普通 JS 对象无法在此面板查看
- WebAssembly.Memory 检查建议安装 [C/C++ DevTools Support (DWARF)](https://chrome.google.com/webstore/detail/cc%20%20-devtools-support-dwa/pdcpmagijalfljmkmjngeonclgbbannb) 扩展以获得 C++ 类名显示

---

## 四、其他定位问题的工具

### 1. Chrome Task Manager（任务管理器）

**打开方式**：`Shift+Esc` 或 Chrome 菜单 > 更多工具 > 任务管理器

**关键列**：

| 列 | 含义 |
|----|------|
| **Memory footprint** | OS 内存（含 DOM 节点等原生内存）。值增长说明 DOM 节点在增加 |
| **JavaScript memory** | JS 堆内存。括号内的 **live** 值表示可达对象使用的内存 |

**用途**：实时监控标签页内存，作为内存问题排查的起点。频繁升降的 Memory 值表示频繁 GC。

### 2. Coverage 面板（代码覆盖率）

**打开方式**：More Options > More tools > Coverage

**用途**：分析页面加载和执行过程中，有多少 JavaScript 和 CSS 代码被实际使用。

- 红色条：未使用的字节
- 绿色条：已使用的字节
- 点击具体文件可跳转到 Sources 面板查看未使用代码的具体行

**适用场景**：识别未使用的代码，优化 bundle 大小，做代码分割（Code Splitting）决策。

### 3. Rendering 面板（渲染调试）

**打开方式**：More Options > More tools > Rendering

**关键选项**：

| 选项 | 用途 |
|------|------|
| **Paint flashing** | 高亮显示重绘区域 |
| **Layer borders** | 显示合成层边界 |
| **Show Rendering stats** | 实时显示 FPS 和渲染统计 |
| **FPS meter** | 实时帧率叠加层 |
| **Layout Shift Regions** | 高亮显示布局偏移区域 |

**适用场景**：调试渲染性能问题，识别不必要的重排/重绘，分析合成层。

### 4. Performance Monitor 面板（性能监控器）

**打开方式**：More Options > More tools > Performance Monitor

**实时监控指标**：

| 指标 | 说明 |
|------|------|
| **JS Heap Size** | JS 堆内存大小 |
| **DOM Nodes** | DOM 节点数量 |
| **JS Event Listeners** | JS 事件监听器数量 |
| **Documents** | 文档（iframe）数量 |
| **Layouts / sec** | 每秒布局次数 |
| **Style Recalcs / sec** | 每秒样式重算次数 |

**适用场景**：实时观察页面操作对性能指标的影响，无需录制即可发现异常趋势。

### 5. Lighthouse 面板

**用途**：自动化性能审计，生成包含 Performance、Accessibility、Best Practices、SEO 等维度的评分报告。

**关键性能指标**：

| 指标 | 说明 |
|------|------|
| **FCP** | First Contentful Paint，首次内容绘制 |
| **LCP** | Largest Contentful Paint，最大内容绘制 |
| **TBT** | Total Blocking Time，总阻塞时间（实验室指标，INP 的代理） |
| **CLS** | Cumulative Layout Shift，累积布局偏移 |
| **Speed Index** | 速度指数 |

**适用场景**：页面加载性能审计，CI/CD 集成，生成可分享的性能报告。

### 6. Network 面板

**用途**：分析网络请求的时序、大小、缓存状态。

**关键功能**：

- **Waterfall 视图**：识别阻塞性请求
- **Cache 列**：检查资源是否命中缓存
- **Priority 列**：查看请求优先级
- **Throttling**：模拟慢速网络（Fast 3G / Slow 3G / Offline）

### 7. Long Tasks API

**用途**：通过 JavaScript API 检测超过 50ms 的长任务。

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log('Long task detected:', entry.duration, 'ms');
  }
});
observer.observe({ type: 'longtask', buffered: true });
```

**适用场景**：生产环境监控长任务，结合 RUM 数据收集。

---

## 五、工具选择指南

| 问题类型 | 推荐工具 | 原因 |
|----------|----------|------|
| 页面加载慢 | Lighthouse + Network | 审计加载链路，识别阻塞资源 |
| 动画卡顿 / 交互延迟 | Performance 面板（火焰图） | 定位主线程瓶颈，识别长任务 |
| 内存持续增长 | Memory 面板（Heap Snapshot Comparison） | 对比快照，找出未释放的对象 |
| 页面偶尔暂停 | Performance 面板（启用 Memory）或 Task Manager | 观察 GC 频率 |
| 不确定哪些代码未使用 | Coverage 面板 | 查看 JS/CSS 使用率 |
| 渲染闪烁 / 布局偏移 | Rendering 面板 | 可视化重绘和布局区域 |
| 实时观察指标变化 | Performance Monitor | 无需录制即可监控 |
| 二进制数据调试 | Memory Inspector | 查看 ArrayBuffer 原始字节 |
| 生产环境监控 | Long Tasks API + web-vitals 库 | 收集真实用户数据 |

---

## 六、常见问题

### Q: Heap Snapshot 拍摄很慢怎么办？

- 快照大小与堆内存中的对象数量成正比
- 拍摄前可先执行一次强制 GC 减少对象数量
- 考虑使用 Allocation Sampling（开销更低）替代

### Q: 火焰图中看不到源码函数名？

- 确保已加载 Source Map
- 保存 trace 时开启 "Include script source maps"
- 生产环境需部署 source map 或使用 `//# sourceMappingURL` 注释

### Q: 如何判断是否存在内存泄漏？

标准流程（来自 Chrome 官方文档）：

1. 在 Task Manager 中观察 JavaScript memory 的 live 值是否持续增长
2. 在 Performance 面板录制，观察 HEAP 曲线是否在 GC 后仍高于基线
3. 拍摄两个 Heap Snapshot（操作前后），用 Comparison 视图查看 delta
4. 若某类对象在操作后数量只增不减，且 Retainer 链指向业务代码，则存在泄漏

### Q: Memory Inspector 和 Memory 面板的 Heap Snapshot 有什么区别？

- **Heap Snapshot**：分析 V8 JS 堆上的对象（普通 JS 对象、DOM wrapper 等）
- **Memory Inspector**：分析 `ArrayBuffer` / `TypedArray` / `DataView` / `WebAssembly.Memory` 的原始字节（不在 JS 堆上）

两者互补，覆盖不同的内存区域。

---

## 七、相关概念

| 概念 | 说明 | 关联笔记 |
|------|------|----------|
| **V8 Garbage Collector** | V8 的分代、精确垃圾回收器 | [[Chrome 的内存限制]] |
| **Orinoco** | V8 的并行和并发 GC 实现 | [[Chrome 的内存限制]] |
| **Blink** | Chromium 渲染引擎，管理 DOM 节点 | [[Chrome 多进程模型]] |
| **RAIL 模型** | Response / Animation / Idle / Load 性能模型 | [[前端性能优化]] |
| **Core Web Vitals** | LCP / INP / CLS 核心体验指标 | [[前端性能优化]] |
| **Site Isolation** | 站点隔离，影响总内存消耗 | [[Chrome 多进程模型]] |

---

## 参考资料

- [Fix memory problems - Chrome DevTools](https://developer.chrome.com/docs/devtools/memory-problems) — Chrome 官方内存问题排查指南
- [Memory terminology - Chrome DevTools](https://developer.chrome.com/docs/devtools/memory-problems/get-started) — Chrome 官方内存术语表
- [Record heap snapshots - Chrome DevTools](https://developer.chrome.com/docs/devtools/memory-problems/heap-snapshots) — 堆快照录制与分析指南
- [Allocation Profiler tool - Chrome DevTools](https://developer.chrome.com/docs/devtools/memory-problems/allocation-profiler) — 分配分析器工具
- [Analyze runtime performance - Chrome DevTools](https://developer.chrome.com/docs/devtools/performance) — 运行时性能分析教程
- [Save and share performance traces - Chrome DevTools](https://developer.chrome.com/docs/devtools/performance/save-trace) — 保存与分享 trace 文件
- [Memory Inspector - Chrome DevTools](https://developer.chrome.com/docs/devtools/memory-inspector) — 二进制内存检查面板
- [Coverage panel - Chrome DevTools](https://developer.chrome.com/docs/devtools/coverage) — 代码覆盖率分析
- [Rendering panel - Chrome DevTools](https://developer.chrome.com/docs/devtools/rendering/performance) — 渲染性能调试
- [Performance Monitor - Chrome DevTools](https://developer.chrome.com/docs/devtools/performance-monitor) — 实时性能监控
- [Optimize long tasks - web.dev](https://web.dev/articles/optimize-long-tasks) — 长任务优化指南
- [RAIL Model - web.dev](https://web.dev/rail/) — RAIL 性能模型
