---
created: 2026-04-27
updated: 2026-04-27
tags:
  - frontend
  - micro-frontend
  - qiankun
  - javascript
  - architecture
  - ant-group
aliases:
  - qiankun
  - 乾坤
  - qiankunjs
source_type: official-doc
source_urls:
  - https://qiankun.umijs.org/
  - https://github.com/umijs/qiankun
  - https://qiankun.umijs.org/api
  - https://qiankun.umijs.org/faq
  - https://github.com/umijs/qiankun/discussions/1378
status: verified
---

## 概述

qiankun（乾坤）是由蚂蚁集团（Ant Financial）开源的微前端框架，基于 [single-spa](https://github.com/single-spa/single-spa) 封装，自称"最完整的微前端解决方案"。名称取自《易经》——"乾"代表天，"坤"代表地，乾坤即宇宙，寓意其能容纳万物。

截至 2026 年，qiankun 在 GitHub 上拥有 16.6k+ Stars，稳定版本为 **2.x**（最新 2.10.16），已在蚂蚁集团内部服务超过 2000 个线上应用。3.0 版本自 2021 年起持续开发中，采用 Monorepo 架构，目前处于预览阶段 [[Roadmap]](https://github.com/umijs/qiankun/discussions/1378)。

## 核心设计理念

qiankun 的设计围绕两个核心原则展开 [[Guide]](https://qiankun.umijs.org/guide)：

### 🥄 Simple（简单）

对使用者而言，qiankun 就像一个 jQuery 风格的库——只需调用几个 API 即可完成微前端改造。其 **HTML Entry** 设计让接入子应用的体验类似于使用 iframe，但底层并非 iframe：

```js
import { registerMicroApps, start } from 'qiankun';

registerMicroApps([
  {
    name: 'reactApp',
    entry: '//localhost:7100',
    container: '#container',
    activeRule: '/react',
  },
]);

start();
```

### 🍡 Decoupling / Technology Agnostic（解耦 / 技术栈无关）

qiankun 的所有设计都服务于一个目标：**确保子应用具备真正的独立开发和运行能力**。因此：

- 主应用不限制子应用的技术栈（React / Vue / Angular / jQuery 均可）
- 子应用仓库独立，前后端可独立开发部署
- 运行时状态隔离，子应用之间不共享运行时状态
- 通信机制设计避免重新引入耦合

## 核心架构

qiankun 的架构可概括为以下层次：

```
┌─────────────────────────────────────────┐
│              主应用（Main App）            │
│  ┌───────────────────────────────────┐  │
│  │         registerMicroApps         │  │
│  │         loadMicroApp              │  │
│  └──────────────┬────────────────────┘  │
│                 │                       │
│  ┌──────────────▼────────────────────┐  │
│  │         HTML Entry Loader         │  │
│  │    (import-html-entry 模块)        │  │
│  └──────────────┬────────────────────┘  │
│                 │                       │
│  ┌──────────────▼────────────────────┐  │
│  │           JS Sandbox              │  │
│  │   (ProxySandbox / SnapshotSandbox)│  │
│  └──────────────┬────────────────────┘  │
│                 │                       │
│  ┌──────────────▼────────────────────┐  │
│  │        Style Isolation            │  │
│  │  (Shadow DOM / Scoped CSS)        │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## 核心机制详解

### 1. HTML Entry

HTML Entry 是 qiankun 区别于 single-spa 最显著的特征。子应用通过 URL 加载，qiankun 会：

1. 通过 `fetch` 获取子应用的 HTML 内容（需支持 CORS）
2. 解析 HTML，提取 `<script>` 和 `<link>` 资源
3. 将 HTML 内容挂载到主应用指定的容器中
4. 在沙箱环境中执行提取的脚本

**优势**：
- 子应用几乎无需改造，保持独立的 HTML 入口
- 接入成本低，类似 iframe 的使用体验
- 自动处理资源路径（`publicPath` 注入）

**entry 配置形式**：

```js
// 字符串形式：访问地址
entry: '//localhost:7100'

// 对象形式：直接传入 HTML 内容（不推荐）
entry: {
  html: '<div id="root"></div>',
  scripts: ['//cdn.example.com/app.js'],
  styles: ['//cdn.example.com/app.css'],
}
```

> **注意**：子应用的 HTML 入口必须支持跨域（CORS），因为 qiankun 通过 `fetch` 获取资源 [[FAQ]](https://qiankun.umijs.org/faq#must-a-sub-app-asset-support-cors)。

### 2. JS 沙箱（JavaScript Sandbox）

沙箱是 qiankun 实现运行时隔离的核心，确保子应用的全局变量和事件不会相互污染。

#### ProxySandbox（现代浏览器）

在支持 `Proxy` 的浏览器中，qiankun 使用 Proxy 代理 `window` 对象：

- 子应用对全局变量的读写被拦截并重定向到沙箱的 `fakeWindow`
- 每个子应用拥有独立的 `fakeWindow`，互不干扰
- 子应用卸载时，沙箱状态被丢弃，不留痕迹

```js
// 简化示意
const sandbox = new Proxy(window, {
  get(target, key) {
    return fakeWindow[key] || target[key];
  },
  set(target, key, value) {
    fakeWindow[key] = value;
    return true;
  },
});
```

#### SnapshotSandbox（兼容模式 / IE）

在不支持 `Proxy` 的浏览器（如 IE）中，使用快照沙箱：

- 子应用激活前，记录当前 `window` 的快照
- 子应用卸载时，对比快照，还原被修改的全局变量
- 同时恢复上一个子应用的快照

> **限制**：IE 环境只能使用单实例模式（`singular: true`），qiankun 检测到 IE 会自动开启 [[FAQ]](https://qiankun.umijs.org/faq#does-qiankun-compatible-with-ie)。

#### 沙箱配置

```js
start({
  sandbox: true, // 默认开启
});

// 关闭沙箱（不推荐）
start({
  sandbox: false,
});
```

### 3. 样式隔离

样式隔离是微前端中最常见的问题之一。qiankun 提供两种方案：

#### StrictStyleIsolation（Shadow DOM）

将每个子应用的容器节点转换为 Shadow DOM，利用浏览器原生的样式隔离：

```js
start({
  sandbox: {
    strictStyleIsolation: true,
  },
});
```

**优点**：完全隔离，子应用样式不会泄漏到全局。

**缺点**：
- 对第三方 UI 库兼容性要求高（如 React 需要 17+）
- 弹窗、Modal 等挂载到 `document.body` 的组件会脱离 Shadow DOM，样式丢失
- 全局样式（如 reset.css）不会生效

#### ExperimentalStyleIsolation（Scoped CSS，推荐）

通过动态添加选择器约束，将子应用样式的作用域限制在其容器内：

```js
start({
  sandbox: {
    experimentalStyleIsolation: true,
  },
});
```

转换效果示例（子应用名为 `react16`）：

```css
/* 原始样式 */
.app-main {
  font-size: 14px;
}

/* 转换后 */
div[data-qiankun-react16] .app-main {
  font-size: 14px;
}
```

**不支持的 CSS 规则**：`@keyframes`、`@font-face`、`@import`、`@page` 不会被重写 [[API]](https://qiankun.umijs.org/api#startopts)。

> **3.0 规划**：qiankun 3.0 计划将 Scoped CSS 作为默认推荐方案，Shadow DOM 变为遗留策略。原因是 Shadow DOM 对第三方库的适配成本远高于 Scoped CSS [[Roadmap]](https://github.com/umijs/qiankun/discussions/1378)。

### 4. 预加载（Prefetch）

qiankun 利用浏览器的空闲时间（`requestIdleCallback`）预加载未打开的子应用资源，提升切换速度。

```js
start({
  prefetch: true,       // 默认值，首个子应用挂载后预加载其余
  prefetch: 'all',      // start 后立即预加载所有
  prefetch: ['app1'],   // 仅预加载指定的
  prefetch: (apps) => ({ // 自定义策略
    criticalAppNames: ['app1'],
    minorAppsName: ['app2', 'app3'],
  }),
});
```

### 5. 应用通信

qiankun 提供 `initGlobalState` 用于主应用与子应用之间的通信：

```js
// 主应用
import { initGlobalState } from 'qiankun';

const actions = initGlobalState({ user: 'admin' });

actions.onGlobalStateChange((state, prev) => {
  console.log('状态变化', state, prev);
});

actions.setGlobalState({ user: 'guest' });
```

```js
// 子应用（通过 mount props 获取）
export function mount(props) {
  props.onGlobalStateChange((state, prev) => {
    console.log('收到主应用状态', state);
  });

  props.setGlobalState({ theme: 'dark' });
}
```

**通信方式总结**：

| 方式 | 适用场景 | 耦合度 |
|------|----------|--------|
| URL 参数 | 跨应用跳转、状态传递 | 最低（推荐） |
| Props 传递 | 主应用向子应用下发数据 | 低 |
| `initGlobalState` | 需要响应式状态同步 | 中 |
| CustomEvent | 发布/订阅模式的事件通知 | 低 |

> **最佳实践**：微应用之间应**尽量减少直接通信**，避免重新引入耦合。判断两个微应用是否应该合并的标准：是否存在频繁的通信需求 [[API]](https://qiankun.umijs.org/api#manually-load-micro-applications)。

## API 概览

### 路由模式（基于 URL 激活）

| API | 用途 |
|-----|------|
| `registerMicroApps(apps, lifeCycles?)` | 注册子应用配置 |
| `start(opts?)` | 启动 qiankun |
| `setDefaultMountApp(appLink)` | 设置默认挂载的子应用 |
| `runAfterFirstMounted(effect)` | 首个子应用挂载后执行 |

### 手动加载模式

| API | 用途 |
|-----|------|
| `loadMicroApp(app, configuration?)` | 手动加载单个微应用，返回可控制的实例 |
| `prefetchApps(apps, importEntryOpts?)` | 手动预加载指定子应用资源 |

### 全局错误处理

| API | 用途 |
|-----|------|
| `addGlobalUncaughtErrorHandler(handler)` | 添加全局未捕获错误处理器 |
| `removeGlobalUncaughtErrorHandler(handler)` | 移除全局未捕获错误处理器 |

### 生命周期

子应用需导出以下生命周期函数：

```js
// 子应用入口
export async function bootstrap() {
  // 初始化，只执行一次
}

export async function mount(props) {
  // 挂载，每次激活时调用
  renderApp(props);
}

export async function unmount(props) {
  // 卸载，每次失活时调用
  unmountApp(props);
}

// 可选
export async function update(props) {
  // 更新，用于 loadMicroApp 的 update 调用
}
```

## 子应用接入要求

子应用需要进行以下改造：

### 1. 导出生命周期

```js
export async function bootstrap() { /* ... */ }
export async function mount(props) { /* ... */ }
export async function unmount(props) { /* ... */ }
```

### 2. 配置构建工具

**webpack 配置要点**：

```js
const packageName = require('./package.json').name;

module.exports = {
  output: {
    library: `${packageName}-[name]`,
    libraryTarget: 'umd',
    jsonpFunction: `webpackJsonp_${packageName}`,
  },
};
```

### 3. 运行时 publicPath

在子应用入口文件顶部添加：

```js
__webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
```

这解决了子应用动态加载资源（懒加载组件、图片等）路径错误的问题 [[FAQ]](https://qiankun.umijs.org/faq#why-dynamic-imported-assets-missing)。

### 4. 独立运行判断

```js
if (!window.__POWERED_BY_QIANKUN__) {
  render(); // 独立运行时直接渲染
}

export const mount = async () => render();
```

## 为什么不用 iframe

qiankun 官方专门撰文解释 [[Why Not Iframe]](https://www.yuque.com/kuitos/gky7yw/gesexv)，核心原因包括：

| 问题 | iframe | qiankun |
|------|--------|---------|
| URL 管理 | 浏览器地址栏不反映子应用路由 | 共享主应用 URL，路由统一 |
| 弹窗/遮罩 | 限制在 iframe 内部，无法全屏 | 正常挂载到 document.body |
| 性能 | 每次加载需要完整的 DOM/JS 环境 | 共享主应用运行时，按需加载 |
| 通信 | postMessage，繁琐且异步 | Props 传递 / 全局状态，同步响应式 |
| 用户体验 | 加载时白屏明显 | 可预加载，切换流畅 |
| SEO | 搜索引擎不索引 iframe 内容 | 内容在主应用 DOM 中 |

## 限制与注意事项

### 子应用必须支持 CORS

子应用的静态资源（HTML、JS、CSS）必须支持跨域访问，因为 qiankun 通过 `fetch` 获取 [[FAQ]](https://qiankun.umijs.org/faq#must-a-sub-app-asset-support-cors)。

### document.write 不兼容

子应用中不能使用 `document.write`（如高德地图 1.x、腾讯地图 2.x），会导致容器被销毁。解决方案：
- 升级到兼容版本（如高德地图 2.x）
- 将地图脚本放在主应用加载

### 子应用跳转

子应用不能直接使用自身的路由实例跳转到其他微应用或主应用页面（如 React Router 的 `<Link>`），因为路由是基于 `base` 的。应使用：
- `history.pushState()`
- 完整的 `<a href="http://localhost:8080/app1">` 标签
- `window.location.href`

### 样式隔离的局限

- Shadow DOM 模式下，第三方 UI 库可能需要额外适配
- Scoped CSS 模式下，`@keyframes`、`@font-face` 等不会被重写
- 主应用样式需要自行隔离（如添加前缀、使用 CSS Modules）

### IE 兼容

- 仅支持单实例模式（`singular: true`）
- 需要引入 polyfill：`whatwg-fetch`、`custom-event-polyfill`、`core-js` 等
- 推荐使用 `@babel/preset-env` 自动 polyfill

### 缓存问题

子应用的 `index.html` 应设置 `Cache-Control: no-cache`，确保每次请求检查更新：

```nginx
location = /index.html {
  add_header Cache-Control no-cache;
}
```

## 3.0 版本规划

qiankun 3.0 正在开发中，主要变化包括 [[Roadmap]](https://github.com/umijs/qiankun/discussions/1378)：

### 核心改进

- **独立的 App Loader 模块**：使用 `DOMParser` 替代正则解析，支持流式加载
- **独立的沙箱模块**：插件化设计，API 参考 TC39 Realms Proposal
- **Scoped CSS 作为默认方案**：Shadow DOM 变为遗留策略
- **支持 Bundless（Vite）**：原生支持 Vite 构建的子应用
- **支持 Webpack 5 Module Federation**
- **支持 CSP 安全策略**

### 开发者体验

- 官方 Webpack 插件
- CLI 脚手架
- React / Vue / Angular 的 UI 绑定组件

### 破坏性变更

- 移除废弃的 `render` 配置
- 弃用 Shadow DOM 方案
- 移除 `globalState` API（提供替代方案）
- 移除 `addGlobalUncaughtErrorHandler` 等与 qiankun 核心无关的 API

> **当前状态**：3.0 仍处于预览阶段，生产环境建议使用 2.x 稳定版。

## 与 single-spa 的关系

qiankun 基于 single-spa，在以下方面做了增强：

| 特性 | single-spa | qiankun |
|------|:----------:|:-------:|
| 路由劫持 + 生命周期 | ✅ | ✅（继承） |
| HTML Entry | ❌ | ✅ |
| JS 沙箱 | ❌ | ✅ |
| 样式隔离 | ❌ | ✅ |
| 预加载 | ❌ | ✅ |
| 开箱即用 | 需自行实现较多 | 是 |
| 接入成本 | 中（需改生命周期） | 低（HTML Entry） |

## 相关概念

- [[常见的微前端框架介绍]] — qiankun 与其他微前端框架的对比
- [[契约测试（Contract Testing）]] — 微前端架构下的跨应用测试策略
- single-spa — qiankun 的基础框架
- Web Components / Shadow DOM — 浏览器原生组件化与隔离标准
- Module Federation — webpack 5 的模块联邦方案

## 参考资料

- [qiankun 官方文档](https://qiankun.umijs.org/)
- [qiankun GitHub 仓库](https://github.com/umijs/qiankun)
- [qiankun API 文档](https://qiankun.umijs.org/api)
- [qiankun FAQ](https://qiankun.umijs.org/faq)
- [qiankun 3.0 Roadmap](https://github.com/umijs/qiankun/discussions/1378)
- [single-spa 官方文档](https://single-spa.js.org/)
- [Micro Frontends](https://micro-frontends.org/) — 微前端概念与实践
- [Why Not Iframe](https://www.yuque.com/kuitos/gky7yw/gesexv) — qiankun 作者解释为何不使用 iframe
