---
created: '2026-04-27'
updated: '2026-04-27'
tags:
  - frontend
  - micro-frontend
  - single-spa
  - javascript
  - architecture
aliases:
  - single-spa
  - single-spa 微前端
  - single-spa 框架
source_type: official-doc
source_urls:
  - 'https://single-spa.js.org/docs/getting-started-overview/'
  - 'https://single-spa.js.org/docs/api/'
  - 'https://single-spa.js.org/docs/recommended-setup/'
  - 'https://single-spa.js.org/docs/building-applications/'
  - 'https://single-spa.js.org/docs/configuration/'
  - 'https://single-spa.js.org/docs/faq/'
  - 'https://github.com/single-spa/single-spa'
status: verified
---

## 概述

single-spa 是一个用于在前端应用中组合多个 JavaScript 微前端（microfrontends）的框架。它通过将现代框架的**组件生命周期抽象为应用级别的生命周期**，使多个独立开发、独立部署的微应用能够在同一页面共存，且无需刷新页面。

- **当前版本**：6.x（最新稳定版 6.0.3，发布于 2024-09-29）[^1]
- **GitHub Stars**：13.9k+
- **开源方**：最初由 Canopy 公司开发并开源
- **定位**：微前端领域的**基础抽象层**，提供最核心的路由劫持与生命周期管理能力，不内置沙箱或样式隔离

> single-spa 本质上是一个**顶层路由器（top-level router）**。当路由激活时，它会下载并执行对应微应用的代码。[^2]

---

## 为什么需要 single-spa

### 解决的痛点

- **技术栈迁移困难**：遗留系统（如 AngularJS）无法快速重写，需要与新框架（如 React）共存并渐进式迁移
- **前端单体困境**：代码库庞大、构建时间长、多团队协作冲突频繁、发布周期长
- **独立部署需求**：不同业务域需要独立的 CI/CD 流水线，互不影响

### 核心优势

| 优势 | 说明 |
|------|------|
| **多框架共存** | 同一页面可同时运行 React、Angular、Vue、Ember 等不同框架[^3] |
| **独立部署** | 每个微应用有自己的仓库、CI 流程和部署周期 |
| **按需加载** | 通过 loading function 实现路由级懒加载，优化首屏性能 |
| **技术栈无关** | 不强制使用特定构建工具，支持 Webpack、Rollup、SystemJS 等 |
| **渐进式迁移** | 使用"绞杀者模式"逐步替换旧代码，无需全量重写[^2] |

---

## 核心概念

### 架构组成

single-spa 应用由以下部分组成：

1. **Root Config（根配置）**
   - 负责渲染共享的 HTML 页面
   - 注册所有微应用（`registerApplication`）
   - **官方建议**：根配置应使用纯 JavaScript 模块，不要使用 React/Angular/Vue 等 UI 框架[^2]

2. **Application（微应用）**
   - 可视为一个没有独立 HTML 页面的 SPA
   - 必须实现 `bootstrap`、`mount`、`unmount` 三个生命周期函数
   - 每个微应用可以有自己的客户端路由和框架

3. **Parcel（包裹）**
   - 可复用的组件单元，可在任何框架中使用
   - 与 Application 不同，Parcel 不由路由控制，而是由父组件手动挂载/卸载

### 应用生命周期

每个微应用在其主入口文件中必须导出以下生命周期函数：

| 生命周期 | 调用时机 | 说明 |
|----------|----------|------|
| `bootstrap` | 首次挂载前调用一次 | 一次性初始化逻辑 |
| `mount` | 应用激活时调用 | 创建 DOM 元素、绑定事件监听器、渲染内容 |
| `unmount` | 应用失活时调用 | 清理 DOM 元素、事件监听器、泄漏的内存、全局变量等 |
| `unload` | 可选，调用 `unloadApplication` 时触发 | 用于热重载或重新 bootstrap 前的清理逻辑 |

```js
// 微应用入口文件示例
export async function bootstrap(props) {
  // 一次性初始化
  return Promise.resolve();
}

export async function mount(props) {
  // 渲染 UI
  return Promise.resolve();
}

export async function unmount(props) {
  // 清理资源
  return Promise.resolve();
}
```

> 所有生命周期函数必须返回 `Promise` 或使用 `async function`。[^4]

### 应用状态机

微应用在生命周期中会经历以下状态：

```
NOT_LOADED → LOADING_SOURCE_CODE → NOT_BOOTSTRAPPED → BOOTSTRAPPING 
→ NOT_MOUNTED → MOUNTING → MOUNTED → UNMOUNTING → UNLOADING → NOT_LOADED
```

异常状态：
- `LOAD_ERROR`：加载失败（如网络错误），可配置重试逻辑
- `SKIP_BECAUSE_BROKEN`：生命周期函数抛出错误且 `dieOnTimeout=true` 时被隔离

---

## 快速开始

### 1. 创建根配置

```bash
npx create-single-spa --moduleType root-config
```

官方脚手架会生成包含 import map 的 `index.ejs` 和注册逻辑的 `root-config.js`。

### 2. 创建微应用

```bash
npx create-single-spa --moduleType app-parcel
```

选择目标框架（React、Vue、Angular 等），脚手架会自动配置生命周期适配。

### 3. 注册微应用

```js
// root-config.js
import { registerApplication, start } from 'single-spa';

registerApplication({
  name: 'app1',
  app: () => import('app1/main.js'),
  activeWhen: ['/app1'],
  customProps: {
    authToken: 'xxx',
  },
});

start();
```

### 4. 配置共享依赖（Import Maps）

在 `index.ejs` 中声明 import map，使多个微应用共享同一份框架代码：

```html
<script type="systemjs-importmap">
{
  "imports": {
    "react": "https://cdn.jsdelivr.net/npm/react@19.0.0/+esm",
    "react-dom": "https://cdn.jsdelivr.net/npm/react-dom@19.0.0/+esm"
  }
}
</script>
```

---

## 核心 API

### registerApplication

注册微应用，支持两种调用方式：

```js
// 方式一：简单参数
registerApplication(
  'appName',
  () => System.import('appName'),
  (location) => location.pathname.startsWith('/app1'),
  { customValue: 'xxx' }
);

// 方式二：配置对象（推荐）
registerApplication({
  name: 'appName',
  app: () => import('appName/main.js'),
  activeWhen: ['/app1', '/app2'],  // 支持路径前缀、活动函数或数组
  customProps: (name, location) => ({ token: 'xxx' }),  // 支持动态 props
});
```

**activeWhen 路径匹配规则**[^5]：
- `/app1`：匹配所有以 `/app1` 开头的路径
- `/users/:userId/profile`：支持动态路由参数
- `/pathname/#/hash`：支持 hash 路由

### start

必须在根配置中调用，否则微应用只会加载而不会 bootstrap/mount/unmount。

```js
start({
  urlRerouteOnly: true,  // 默认 true，仅当路由变化时才触发重路由
});
```

### 应用管理 API

| API | 说明 |
|-----|------|
| `getAppNames()` | 获取所有已注册的应用名称 |
| `getMountedApps()` | 获取当前已挂载的应用名称 |
| `getAppStatus(appName)` | 获取应用状态 |
| `unloadApplication(appName, { waitForUnmount })` | 卸载应用（重置为 NOT_LOADED） |
| `unregisterApplication(appName)` | 取消注册（v5.8.0+） |
| `navigateToUrl(url)` | 路由导航工具函数 |
| `checkActivityFunctions(location)` | 检查指定 URL 下哪些应用应该激活 |

### 错误处理

```js
singleSpa.addErrorHandler((err) => {
  console.log(err.appOrParcelName);
  console.log(singleSpa.getAppStatus(err.appOrParcelName));
  
  // 处理 LOAD_ERROR 状态，允许重试
  if (singleSpa.getAppStatus(err.appOrParcelName) === singleSpa.LOAD_ERROR) {
    System.delete(System.resolve(err.appOrParcelName));
  }
});
```

### 超时配置

```js
// 全局配置
singleSpa.setBootstrapMaxTime(3000, false);  // 3秒超时，仅警告
singleSpa.setMountMaxTime(3000, true);       // 3秒超时，隔离应用

// 应用级覆盖（在微应用入口文件中导出）
export const timeouts = {
  bootstrap: { millis: 5000, dieOnTimeout: true },
  mount: { millis: 5000, dieOnTimeout: false },
};
```

---

## 推荐架构

single-spa 核心团队推荐的架构实践[^6]：

### 模块类型

| 类型 | 用途 | 建议数量 |
|------|------|----------|
| **Application** | 路由级微应用 | 多个，按业务域划分 |
| **Parcel** | 跨框架可复用组件 | 极少，优先使用框架原生组件 |
| **Utility Module** | 工具库（样式库、API 封装、认证等） | 按需创建 |

> **原则**：优先按路由拆分 Application，而非按组件拆分。路由切换通常涉及大量 UI 状态销毁重建，不同路由的 Application 不需要共享 UI 状态。[^6]

### 共享依赖策略

- **Import Maps（推荐）**：通过浏览器原生 ES 模块 + import map 共享依赖，任何构建工具都支持
- **Module Federation**：webpack 5 专属特性，可与 single-spa 配合使用
- **不建议混用**：选择一种方案管理第三方共享依赖[^6]

### 跨微前端通信

| 通信内容 | 推荐方案 |
|----------|----------|
| 函数/组件/逻辑 | 跨微前端 import（通过 import map 暴露公共接口） |
| API 数据 | 共享 API 工具模块 + 内存缓存 |
| UI 状态 | 尽量避免共享；必要时使用 CustomEvent 或 RxJS Observable |

> **官方建议**：如果两个微前端需要频繁共享 UI 状态，考虑将它们合并为一个微前端。[^6]

### 本地开发

使用 [import-map-overrides](https://github.com/joeldenning/import-map-overrides) 工具，将 import map 中某个微应用的 URL 指向 localhost，其他微应用使用已部署版本。这样只需启动当前开发的微应用，无需启动全部。[^6]

---

## 限制与注意事项

### 不内置的能力

single-spa 是**最小化抽象**，以下能力需要自行实现或借助生态库：

| 能力 | 说明 |
|------|------|
| **JS 沙箱** | 不提供，需自行实现或使用 qiankun 等上层框架 |
| **样式隔离** | 不提供，推荐使用 CSS Modules、BEM 命名规范或 Shadow DOM |
| **应用通信** | 无内置方案，通过 customProps 或跨模块 import 实现 |

### 性能影响

- 按推荐方式配置时，性能和包体积与代码分割的单体应用几乎一致[^2]
- 额外开销主要来自 single-spa 库本身（约 6KB gzipped）和 SystemJS（如使用）
- 共享依赖配置不当会导致重复加载

### 与 Create React App 的兼容性

CRA 默认输出"单体"应用，不能直接作为 single-spa 微应用使用。需要：
1. eject 后修改 webpack 配置
2. 使用 craco / react-app-rewired 等第三方工具
3. 迁移到 `create-single-spa` 脚手架[^2]

### 状态管理

官方**不建议**使用 Redux/MobX 等全局状态管理库跨微前端共享状态。推荐每个微前端维护自己的状态，或使用独立的状态库。[^6]

---

## 生态系统

single-spa 提供了丰富的框架适配库：

| 框架 | 适配库 |
|------|--------|
| React | [single-spa-react](https://single-spa.js.org/docs/ecosystem-react/) |
| Vue | [single-spa-vue](https://single-spa.js.org/docs/ecosystem-vue/) |
| Angular | [single-spa-angular](https://single-spa.js.org/docs/ecosystem-angular/) |
| AngularJS | [single-spa-angularjs](https://single-spa.js.org/docs/ecosystem-angularjs/) |
| Svelte | [single-spa-svelte](https://single-spa.js.org/docs/ecosystem-svelte/) |
| Preact | [single-spa-preact](https://single-spa.js.org/docs/ecosystem-preact/) |

其他工具：
- [create-single-spa](https://single-spa.js.org/docs/create-single-spa/)：脚手架工具
- [import-map-deployer](https://github.com/single-spa/import-map-deployer)：CI/CD 中安全更新 import map
- [single-spa-inspector](https://github.com/single-spa/single-spa-inspector)：浏览器调试扩展

---

## 与其他微前端框架的关系

| 框架 | 与 single-spa 的关系 |
|------|---------------------|
| **qiankun** | 基于 single-spa 封装，增加了 HTML Entry、JS 沙箱、样式隔离 |
| **wujie** | 独立架构（Web Components + iframe），不依赖 single-spa |
| **icestark** | 独立架构，兼容 single-spa 生命周期规范 |
| **Module Federation** | 互补关系：single-spa 负责路由，Module Federation 负责模块共享 |

---

## 常见问题

### 两个微应用可以同时激活吗？

可以。需要为每个微应用分配独立的 DOM 容器，容器 ID 必须以 `single-spa-application:` 为前缀：

```html
<div id="single-spa-application:app-name"></div>
<div id="single-spa-application:other-app"></div>
```

### 微应用可以独立部署吗？

可以。每个微应用有独立的构建和部署流程：
1. 将 JavaScript 产物上传到 CDN
2. 更新 import map 指向新版本的 URL[^2]

### 能否只加载一个框架版本？

可以且强烈推荐。通过 import map 将 React/Vue/Angular 等框架定义为共享依赖，各微应用通过 `externals` 配置不打包框架代码。[^2]

---

## 参考资料

- [single-spa 官方文档](https://single-spa.js.org/docs/getting-started-overview/) — 官方 Getting Started
- [single-spa API 文档](https://single-spa.js.org/docs/api/) — 完整 API 参考（v6.x）
- [Recommended Setup](https://single-spa.js.org/docs/recommended-setup/) — 官方推荐架构实践
- [Building Applications](https://single-spa.js.org/docs/building-applications/) — 微应用生命周期详解
- [Configuring single-spa](https://single-spa.js.org/docs/configuration/) — 根配置指南
- [FAQ](https://single-spa.js.org/docs/faq/) — 常见问题解答
- [GitHub 仓库](https://github.com/single-spa/single-spa) — single-spa/single-spa（v6.0.3）
- [Micro Frontends](https://micro-frontends.org/) — Michael Geers，微前端概念与实践指南

---

[^1]: GitHub Releases: https://github.com/single-spa/single-spa/releases/tag/v6.0.3
[^2]: single-spa FAQ: https://single-spa.js.org/docs/faq/
[^3]: single-spa Ecosystem: https://single-spa.js.org/docs/ecosystem/
[^4]: Building Applications: https://single-spa.js.org/docs/building-applications/
[^5]: Configuring single-spa: https://single-spa.js.org/docs/configuration/
[^6]: Recommended Setup: https://single-spa.js.org/docs/recommended-setup/
