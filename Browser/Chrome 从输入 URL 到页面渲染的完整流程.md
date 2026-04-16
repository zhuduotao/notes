---
created: 2026-04-15
updated: 2026-04-15
tags:
  - 浏览器原理
  - Chrome
  - 渲染机制
  - 前端基础
  - 导航流程
  - 性能优化
aliases:
  - URL to Rendering
  - Navigation Flow
  - Chrome Rendering Pipeline
  - 浏览器渲染流程
  - 从输入URL到页面渲染
source_type: official-doc
source_urls:
  - https://developers.google.com/web/updates/2018/09/inside-browser-part1
  - https://developers.google.com/web/updates/2018/09/inside-browser-part2
  - https://developers.google.com/web/updates/2018/09/inside-browser-part3
  - https://developers.google.com/web/updates/2018/09/inside-browser-part4
  - https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/Navigation_timing
  - https://html.spec.whatwg.org/multipage/parsing.html
status: verified
---

## 概述

当用户在浏览器地址栏输入 URL 并按下回车后，Chrome 需要经历一系列复杂的步骤才能将页面呈现到屏幕上。整个过程涉及**多进程架构协调**、**网络请求**、**安全校验**、**HTML/CSS/JavaScript 解析**、**布局计算**、**绘制**和**合成**等多个阶段。理解这一流程对于前端性能优化、问题排查和用户体验提升至关重要。

> **注意**：本文基于 Chrome 的多进程架构（Chrome 67+，Site Isolation 默认启用）进行描述。不同浏览器的实现细节可能有所不同，因为浏览器架构属于实现细节，没有统一的标准规范。

---

## 一、Chrome 多进程架构基础

在深入导航流程之前，需要先理解 Chrome 的多进程架构，因为整个渲染过程分布在不同的进程中协作完成。

### 核心进程类型

| 进程 | 职责 |
|------|------|
| **Browser Process**（浏览器进程） | 控制浏览器的"chrome"部分（地址栏、书签、前进后退按钮）；处理网络请求、文件访问等不可见的特权操作 |
| **Renderer Process**（渲染进程） | 控制标签页内网站显示的所有内容；每个标签页至少有一个渲染进程 |
| **GPU Process**（GPU 进程） | 独立处理 GPU 任务，与其他进程隔离 |
| **Plugin Process**（插件进程） | 控制网站使用的插件（如 Flash） |

### 为什么采用多进程架构

- **稳定性**：一个标签页崩溃不会影响其他标签页
- **安全性**：操作系统可以限制进程的权限，渲染进程被沙箱化，无法随意访问文件系统
- **Site Isolation**（站点隔离，Chrome 67+ 默认启用）：每个跨站 iframe 运行在独立的渲染进程中，有效防御 Meltdown 和 Spectre 等侧信道攻击

### 进程内线程

渲染进程内部包含多个线程：

- **Main Thread**（主线程）：处理大部分 HTML 解析、CSS 解析、JavaScript 执行、布局、绘制等核心工作
- **Worker Threads**（工作线程）：处理 Web Worker 或 Service Worker 中的 JavaScript
- **Compositor Thread**（合成线程）：高效平滑地合成页面，不依赖主线程
- **Raster Threads**（光栅线程）：将图层光栅化为位图，存储在 GPU 内存中

---

## 二、导航阶段（Navigation）

导航是指从用户输入 URL 到浏览器准备渲染页面的过程，主要由**浏览器进程**协调完成。

### Step 1：处理输入（Handling Input）

当用户开始在地址栏输入时，**UI 线程**（浏览器进程内）首先判断输入内容是搜索查询还是 URL。Chrome 的地址栏同时是搜索输入框，因此 UI 线程需要解析并决定是将用户发送到搜索引擎，还是导航到请求的网站。

### Step 2：启动导航（Start Navigation）

用户按下回车后，UI 线程发起网络请求获取网站内容。此时：

- 标签页角落显示加载旋转图标
- **Network 线程**（浏览器进程内）执行 DNS 查找和建立 TLS 连接
- 如果服务器返回 HTTP 301 等重定向响应，Network 线程会通知 UI 线程，然后发起新的 URL 请求

### Step 3：读取响应（Read Response）

当响应体（payload）开始到达时，Network 线程会检查数据流的前几个字节：

- **MIME Type 检查**：虽然响应头中的 `Content-Type` 应说明数据类型，但可能缺失或错误，因此需要执行 [MIME Type Sniffing](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
- **文件类型判断**：如果是 HTML 文件，数据将传递给渲染进程；如果是 zip 等文件，则传递给下载管理器
- **SafeBrowsing 检查**：如果域名和响应数据匹配已知的恶意网站，Network 线程会警告并显示警告页面
- **CORB（Cross Origin Read Blocking）检查**：确保敏感的跨站数据不会进入渲染进程

### Step 4：查找渲染进程（Find a Renderer Process）

所有检查完成后，Network 线程通知 UI 线程数据已就绪。UI 线程找到一个渲染进程来继续页面的渲染工作。

**性能优化**：在 Step 2 发送 URL 请求时，UI 线程已经知道要导航到哪个站点，因此会**主动并行查找或启动渲染进程**。这样当 Network 线程收到数据时，渲染进程通常已经处于待命状态。但如果导航发生跨站重定向，可能需要不同的进程。

### Step 5：提交导航（Commit Navigation）

数据和渲染进程准备就绪后，浏览器进程通过 **IPC（进程间通信）** 向渲染进程发送提交导航的请求，同时传递数据流以便渲染进程继续接收 HTML 数据。

收到渲染进程的确认响应后：

- 地址栏更新，安全指示器和站点设置 UI 反映新页面信息
- 标签页的会话历史更新，前进/后退按钮可以导航到刚访问的站点
- 会话历史存储到磁盘，便于标签页/窗口关闭后的恢复

### 额外步骤：初始加载完成（Initial Load Complete）

导航提交后，渲染进程继续加载资源并渲染页面。当渲染进程"完成"渲染后（所有 frame 的 `onload` 事件触发并执行完毕），会通过 IPC 通知浏览器进程，此时 UI 线程停止标签页上的加载旋转图标。

> **注意**：这里的"完成"是相对的，因为客户端 JavaScript 可能在此之后继续加载额外资源并渲染新视图。

---

## 三、特殊场景

### 导航到不同站点

当用户再次输入不同 URL 时，浏览器进程需要：

1. 检查当前渲染的站点是否关心 `beforeunload` 事件
2. 如果导航由渲染进程发起（如用户点击链接或执行 `window.location`），渲染进程首先检查 `beforeunload` 处理器，然后将导航请求从渲染进程发送到浏览器进程
3. 当导航到与当前渲染不同的站点时，会调用**单独的渲染进程**处理新导航，同时保留当前渲染进程处理 `unload` 事件

> **性能提示**：不要添加无条件的 `beforeunload` 处理器。这会增加延迟，因为导航开始前必须执行处理器。仅在需要警告用户可能丢失数据时才添加。

### Service Worker 场景

Service Worker 的引入改变了导航流程：

1. Service Worker 注册时，其作用域（scope）会被保留为引用
2. 导航发生时，Network 线程检查域名与已注册的 Service Worker 作用域
3. 如果该 URL 注册了 Service Worker，UI 线程找到渲染进程来执行 Service Worker 代码
4. Service Worker 可能从缓存加载数据，消除网络请求的需要，或从网络请求新资源

### Navigation Preload

Service Worker 启动可能导致延迟。**Navigation Preload** 机制通过在 Service Worker 启动的同时并行加载资源来加速这一过程。这些请求会被标记特殊的请求头，允许服务器决定发送不同的内容（例如仅发送更新的数据而非完整文档）。

---

## 四、渲染阶段（Rendering）

导航提交后，渲染进程开始将 HTML、CSS 和 JavaScript 转换为用户可交互的网页。

### 4.1 解析（Parsing）

#### 构建 DOM

渲染进程收到导航提交消息并开始接收 HTML 数据后，**主线程**开始将 HTML 文本字符串解析为 **DOM（Document Object Model）**。

- DOM 是浏览器对页面的内部表示，也是 Web 开发者通过 JavaScript 交互的数据结构和 API
- HTML 解析规范定义了容错机制，例如缺失闭合标签、标签交叉嵌套等都会被优雅处理
- 详见 [HTML Standard - Parsing](https://html.spec.whatwg.org/multipage/parsing.html)

#### 子资源加载（Subresource Loading）

网站通常使用图片、CSS、JavaScript 等外部资源。为了加速加载，**Preload Scanner**（预加载扫描器）与 HTML 解析器并发运行：

- 预加载扫描器查看 HTML 解析器生成的 token
- 如果发现 `<img>`、`<link>` 等标签，会提前向浏览器进程的 Network 线程发送请求

#### JavaScript 会阻塞解析

当 HTML 解析器遇到 `<script>` 标签时，会**暂停 HTML 解析**，必须先加载、解析并执行 JavaScript 代码。原因：

- JavaScript 可以通过 `document.write()` 等方式改变整个 DOM 结构
- 因此 HTML 解析器必须等待 JavaScript 执行完毕才能继续解析

**解决方案**：

- 如果 JavaScript 不使用 `document.write()`，可以添加 `async` 或 `defer` 属性，浏览器会异步加载和执行 JavaScript，不阻塞解析
- 使用 `<link rel="preload">` 告知浏览器该资源对当前导航必需，应尽快下载
- 使用 JavaScript Module（`<script type="module">`），默认具有 `defer` 行为

### 4.2 样式计算（Style Calculation）

仅有 DOM 不足以确定页面外观，因为 CSS 可以样式化页面元素。

主线程解析 CSS 并确定每个 DOM 节点的**计算样式（Computed Style）**：

- 这是基于 CSS 选择器应用于每个元素的样式信息
- 即使不提供任何 CSS，每个 DOM 节点也有计算样式，因为浏览器有默认样式表
- 可以在 DevTools 的 Computed 面板中查看这些信息

### 4.3 布局（Layout）

知道文档结构和每个节点的样式后，仍不足以渲染页面。布局是**查找元素几何信息**的过程。

主线程遍历 DOM 和计算样式，创建**布局树（Layout Tree）**，包含：

- x/y 坐标
- 边界框尺寸

**布局树与 DOM 树的区别**：

- 布局树只包含页面上可见的信息
- 应用 `display: none` 的元素不在布局树中（但 `visibility: hidden` 的元素在）
- 带有 `content` 的伪元素（如 `p::before{content:"Hi!"}`）会包含在布局树中，即使不在 DOM 中

布局是一个复杂的过程，需要考虑字体大小、换行位置、浮动、溢出遮罩、书写方向等。

### 4.4 绘制（Paint）

拥有 DOM、样式和布局后，仍不足以渲染页面。绘制阶段需要确定**元素的绘制顺序**。

例如，如果某些元素设置了 `z-index`，按照 HTML 中元素的顺序绘制会导致渲染错误。

在绘制阶段，主线程遍历布局树创建**绘制记录（Paint Records）**：

- 绘制记录是绘制过程的笔记，如"先背景，再文字，再矩形"
- 如果使用 JavaScript 在 `<canvas>` 上绘制，这个过程可能很熟悉

### 渲染管线的性能代价

渲染管线的每个步骤都使用前一步的结果创建新数据：

```
DOM + 样式 → 布局 → 绘制 → 合成
```

如果布局树发生变化，受影响部分的绘制顺序需要重新生成。

**动画场景**：大多数显示器每秒刷新 60 次（60 fps）。如果动画在帧之间错过，页面会出现"卡顿"（jank）。这些计算在主线程上运行，可能被 JavaScript 执行阻塞。

**优化建议**：

- 使用 `requestAnimationFrame()` 将 JavaScript 操作分成小块，在每帧中调度执行
- 在 Web Worker 中运行 JavaScript，避免阻塞主线程

### 4.5 合成（Compositing）

#### 什么是合成

合成是一种技术，将页面分成多个图层，分别光栅化，然后在**合成线程**中组合成页面。

**优势**：

- 滚动发生时，图层已经光栅化，只需组合新帧
- 动画可以通过移动图层并组合新帧来实现
- 合成不依赖主线程，不需要等待样式计算或 JavaScript 执行

#### 图层划分

主线程遍历布局树创建**图层树（Layer Tree）**（在 DevTools 性能面板中显示为"Update Layer Tree"）。

- 如果页面的某些部分应该是独立图层（如滑入式侧边菜单）但没有获得，可以使用 CSS 的 `will-change` 属性提示浏览器
- **警告**：给每个元素都分配图层可能导致比每帧光栅化页面小部分更慢的操作，必须测量应用的渲染性能

#### 光栅化和合成（脱离主线程）

图层树创建并确定绘制顺序后：

1. 主线程将信息提交给合成线程
2. 合成线程光栅化每个图层，将图层划分为多个**瓦片（Tiles）**
3. 将每个瓦片发送给**光栅线程**
4. 光栅线程光栅化每个瓦片并存储在 **GPU 内存**中
5. 合成线程收集瓦片信息（称为 **Draw Quads**）创建**合成帧（Compositor Frame）**
6. 合成帧通过 IPC 提交给浏览器进程
7. 浏览器进程可能从 UI 线程或其他渲染进程接收额外的合成帧
8. 这些合成帧发送给 GPU 显示在屏幕上

**Draw Quads**：包含瓦片在内存中的位置以及在页面中绘制瓦片的位置信息。

**Compositor Frame**：一组 Draw Quads，代表页面的一帧。

---

## 五、输入事件处理

### 输入事件路由

当用户在屏幕上触摸时，**浏览器进程**首先接收手势。但浏览器进程只知道手势发生的位置，因为标签页内的内容由渲染进程处理。因此浏览器进程将事件类型（如 `touchstart`）和坐标发送给渲染进程，渲染进程找到事件目标并运行附加的事件监听器。

### 合成线程接收输入事件

如果页面没有附加输入事件监听器，合成线程可以完全独立于主线程创建新的合成帧。但如果页面附加了事件监听器：

- 页面合成时，合成线程将附加了事件处理器的页面区域标记为 **"Non-Fast Scrollable Region"**（非快速可滚动区域）
- 如果输入事件发生在该区域内，合成线程必须将事件发送给主线程
- 如果输入事件发生在该区域外，合成线程继续合成新帧，无需等待主线程

### 事件委托的性能陷阱

事件委托是 Web 开发中的常见模式：

```javascript
// 整个页面被标记为非快速可滚动区域
document.body.addEventListener('touchstart', event => {
    if (event.target === area) {
        event.preventDefault();
    }
});
```

这会导致即使应用不关心页面某些部分的输入，合成线程也必须在每次输入事件发生时与主线程通信并等待，从而失去平滑滚动的能力。

**解决方案**：使用 `passive: true` 选项

```javascript
document.body.addEventListener('touchstart', event => {
    if (event.target === area) {
        event.preventDefault()
    }
}, { passive: true });
```

这提示浏览器仍然想在主线程中监听事件，但合成线程可以继续合成新帧。

### 检查事件是否可取消

如果只想限制某个区域的滚动方向为水平滚动：

```javascript
document.body.addEventListener('pointermove', event => {
    if (event.cancelable) {
        event.preventDefault(); // 阻止原生滚动
    }
}, { passive: true });
```

或者使用 CSS 规则 `touch-action` 完全消除事件处理器：

```css
#area {
  touch-action: pan-x;
}
```

### 事件合并（Event Coalescing）

触摸屏设备每秒传递触摸事件 60-120 次，鼠标每秒传递事件 100 次。输入事件的保真度高于屏幕刷新率。

为了减少对主线程的过度调用，Chrome 会**合并连续事件**（如 `wheel`、`mousewheel`、`mousemove`、`pointermove`、`touchmove`），并将分发延迟到下一个 `requestAnimationFrame` 之前。

**离散事件**（如 `keydown`、`keyup`、`mouseup`、`mousedown`、`touchstart`、`touchend`）会立即分发。

### 使用 `getCoalescedEvents` 获取帧内事件

对于绘图应用等需要平滑线条的场景，可以使用 `getCoalescedEvents` 方法获取被合并的事件信息：

```javascript
window.addEventListener('pointermove', event => {
    const events = event.getCoalescedEvents();
    for (let event of events) {
        const x = event.pageX;
        const y = event.pageY;
        // 使用 x 和 y 坐标绘制线条
    }
});
```

---

## 六、Navigation Timing API

Chrome 提供了 [Navigation Timing API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/Navigation_timing) 来测量导航过程中各个阶段的时间。

### 关键时间戳

| 时间戳 | 说明 |
|--------|------|
| `startTime` | 始终为 0 |
| `unloadEventStart` | 前一个文档的 `unload` 事件处理器开始前的时间戳 |
| `unloadEventEnd` | 前一个文档的 `unload` 事件处理器完成后的时间戳 |
| `domInteractive` | DOM 构建完成，可以与 JavaScript 交互的时间戳 |
| `domContentLoadedEventStart` | `DOMContentLoaded` 事件处理器开始前的时间戳 |
| `domContentLoadedEventEnd` | `DOMContentLoaded` 事件处理器完成后的时间戳 |
| `domComplete` | 文档和所有子资源加载完成的时间戳 |
| `loadEventStart` | `load` 事件处理器开始前的时间戳 |
| `loadEventEnd` | `load` 事件处理器完成后的时间戳 |

### 使用示例

```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach((entry) => {
    const domContentLoadedTime =
      entry.domContentLoadedEventEnd - entry.domContentLoadedEventStart;
    console.log(
      `${entry.name}: DOMContentLoaded processing time: ${domContentLoadedTime}ms`
    );
  });
});

observer.observe({ type: "navigation", buffered: true });
```

---

## 七、性能优化最佳实践

### 资源加载优化

- 对不使用 `document.write()` 的脚本使用 `async` 或 `defer` 属性
- 使用 `<link rel="preload">` 预加载关键资源
- 使用 JavaScript Module（`<script type="module">`）

### 渲染优化

- 动画优先使用合成器属性（`transform`、`opacity`），避免触发布局和绘制
- 使用 `will-change` 提示浏览器为特定元素创建独立图层，但避免过度使用
- 使用 `requestAnimationFrame()` 调度 JavaScript 执行
- 将计算密集型任务移至 Web Worker

### 输入事件优化

- 对不需要阻止默认行为的滚动/触摸事件使用 `{ passive: true }`
- 使用 `touch-action` CSS 属性替代 JavaScript 事件处理器
- 避免在 `document.body` 等大范围元素上附加事件处理器

### 测量工具

- 使用 [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview) 审计网站性能
- 使用 Chrome DevTools 的 [Performance 面板](https://developer.chrome.com/docs/devtools/performance/) 测量渲染性能
- 使用 [Layers 面板](https://developer.chrome.com/docs/devtools/layers/) 查看页面图层划分

---

## 八、常见误区

### 误区 1：浏览器是单进程的

现代 Chrome 采用多进程架构，每个标签页（甚至每个跨站 iframe）都有独立的渲染进程。

### 误区 2：JavaScript 不会影响页面加载

`<script>` 标签会阻塞 HTML 解析，因为 JavaScript 可以通过 `document.write()` 改变 DOM 结构。

### 误区 3：所有动画性能都一样

只有使用合成器属性（`transform`、`opacity`）的动画才能在合成线程上运行，不依赖主线程。触发布局或重绘的动画（如改变 `width`、`height`、`top`、`left`）会阻塞主线程。

### 误区 4：事件监听器越多越好

在 `document.body` 等大范围元素上附加事件处理器会将整个页面标记为"非快速可滚动区域"，导致合成线程必须等待主线程处理每个输入事件。

### 误区 5：Service Worker 总是加速加载

如果 Service Worker 最终决定从网络请求数据，浏览器进程和渲染进程之间的往返可能导致延迟。应使用 Navigation Preload 来优化。

---

## 九、相关概念

- **V8 引擎**：Chrome 的 JavaScript 引擎，负责 JavaScript 的解析、编译和执行
- **Blink**：Chrome 的渲染引擎，负责 HTML/CSS 解析、布局和绘制
- **事件循环（Event Loop）**：JavaScript 的并发模型，处理异步任务的调度
- **Critical Rendering Path**：浏览器将 HTML、CSS 和 JavaScript 转换为屏幕上像素的关键路径
- **Core Web Vitals**：Google 定义的核心网页指标（LCP、FID/INP、CLS），与渲染流程密切相关

---

## 参考资料

- [Chrome for Developers: Inside look at modern web browser (part 1) - CPU, GPU, Memory, and multi-process architecture](https://developers.google.com/web/updates/2018/09/inside-browser-part1)
- [Chrome for Developers: Inside look at modern web browser (part 2) - What happens in navigation](https://developers.google.com/web/updates/2018/09/inside-browser-part2)
- [Chrome for Developers: Inside look at modern web browser (part 3) - Inner workings of a Renderer Process](https://developers.google.com/web/updates/2018/09/inside-browser-part3)
- [Chrome for Developers: Inside look at modern web browser (part 4) - Input is coming to the Compositor](https://developers.google.com/web/updates/2018/09/inside-browser-part4)
- [MDN: Navigation Timing API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/Navigation_timing)
- [HTML Standard: Parsing](https://html.spec.whatwg.org/multipage/parsing.html)
- [Chromium: Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/)
- [Chromium: CORB for Developers](https://www.chromium.org/Home/chromium-security/corb-for-developers)
- [MDN: MIME types](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
- [MDN: beforeunload event](https://developer.mozilla.org/docs/Web/Events/beforeunload)
- [Google Web Fundamentals: Resource Prioritization](https://developers.google.com/web/fundamentals/performance/resource-prioritization)
- [Google Web Fundamentals: Optimize JavaScript Execution](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution)
- [Google Web Fundamentals: Stick to Compositor-Only Properties](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count)
