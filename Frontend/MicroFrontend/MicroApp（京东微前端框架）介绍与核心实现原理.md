---
created: 2026-04-27
updated: 2026-04-27
tags:
  - frontend
  - micro-frontend
  - micro-app
  - jd-opensource
  - web-components
aliases:
  - micro-app
  - 京东微前端
  - MicroApp 框架
  - "@micro-zoe/micro-app"
source_type: official-doc
source_urls:
  - https://jd-opensource.github.io/micro-app/
  - https://github.com/jd-opensource/micro-app
status: verified
---

## 概述

micro-app 是由**京东零售**开源的一款微前端框架，核心理念是**从组件化的思维实现微前端**。它基于类 WebComponent 自定义元素（`<micro-app>`）进行渲染，接入成本极低，官方宣称是"接入微前端成本最低的框架"。

- **官方仓库**: [jd-opensource/micro-app](https://github.com/jd-opensource/micro-app)
- **最新版本**: 1.0.0-rc.x（截至 2026 年 4 月）
- **NPM 包名**: `@micro-zoe/micro-app`
- **开源协议**: MIT

## 核心特性

| 特性 | 说明 |
|------|------|
| **一行代码接入** | 通过 `<micro-app>` 自定义元素加载子应用，使用方式类似 iframe |
| **框架无关** | 主/子应用可使用任意前端框架（React、Vue、Angular、Vite 等） |
| **JS 沙箱** | 提供 with 沙箱和 iframe 沙箱两种模式，隔离子应用运行环境 |
| **样式隔离** | 基于 `<micro-app>` 标签的 name 属性为 CSS 添加作用域前缀 |
| **元素隔离** | 模拟 Shadow DOM 行为，子应用操作元素时不会逃离 `<micro-app>` 边界 |
| **虚拟路由系统** | 自定义 location/history，子应用路由与主应用隔离 |
| **数据通信** | 提供主/子应用双向通信和全局通信机制 |
| **插件系统** | 可拦截和处理子应用的 JS/HTML 资源，修复兼容性问题 |
| **预加载** | 支持浏览器空闲时预加载子应用资源 |
| **资源路径补全** | 自动补全子应用相对路径的静态资源地址 |

## 核心实现原理

### 1. 基于 WebComponent 的渲染

micro-app 的核心思路是**将微前端应用视为一个自定义元素**。通过 `CustomElements API` 注册 `<micro-app>` 标签，用户只需在页面中放置该标签并传入 `name` 和 `url` 属性，框架便会自动完成子应用的加载和渲染。

```html
<micro-app name='my-app' url='http://localhost:3000/'></micro-app>
```

**工作流程**：
1. 主应用通过 `fetch` 获取子应用的 `index.html`
2. 解析 HTML，提取其中的 `<script>` 和 `<link>` 资源
3. 在沙箱环境中执行 JS，将 DOM 元素插入 `<micro-app>` 容器内
4. 子应用渲染完成

这种设计使得接入微前端就像引入一个组件一样简单，无需修改子应用的生命周期或打包配置。

### 2. JS 沙箱机制

micro-app 提供**两种沙箱模式**，可自由切换：

#### with 沙箱（默认）

基于 `with` 语句和 `Proxy` 实现。通过自定义的 `window` 和 `document` 代理，拦截子应用对全局变量的操作，创建一个相对独立的运行空间。

- **原理**：子应用的 JS 代码在 `with (proxyWindow) { ... }` 环境中执行，所有对 `window` 的访问都会被 Proxy 拦截
- **自定义 document**：代理了 `document` 的大部分操作（事件监听、元素增删改查），部分属性兜底到原生 document（如 `document.body`、`document.head`）
- **获取真实 window**：子应用可通过 `window.rawWindow` 和 `window.rawDocument` 获取最外层主应用的真实对象

#### iframe 沙箱

当 with 沙箱无法正常工作时（如 Vite 项目），可切换到 iframe 沙箱。

- **原理**：子应用的 JS 在 iframe 中运行，利用 iframe 天然的全局隔离
- **注意事项**：由于 iframe 的 src 必须指向主应用域名，初始化时可能加载主应用资源。可通过创建空的 `empty.html` 或使用 `window.stop()` 解决

**沙箱依赖**：
- `CustomElements API`：不支持的浏览器可通过 [webcomponents/polyfills](https://github.com/webcomponents/polyfills/tree/master/packages/custom-elements) 兼容
- `Proxy API`：无法 polyfill，不支持 Proxy 的浏览器无法运行（需 iOS 10+、Android 5+，不支持 IE）

### 3. 样式隔离

micro-app 的样式隔离默认开启，实现方式如下：

```css
/* 子应用原始 CSS */
.test {
  color: red;
}

/* 转换后 */
micro-app[name=xxx] .test {
  color: red;
}
```

**实现机制**：利用 `<micro-app>` 标签的 `name` 属性作为样式作用域，为每个 CSS 选择器添加前缀，将子应用的样式影响限制在标签区域内。

**灵活控制**：支持多层级的样式隔离禁用：
- 全局禁用：`microApp.start({ disableScopecss: true })`
- 单个应用禁用：`<micro-app disable-scopecss>`
- 文件级禁用：CSS 注释 `/*! scopecss-disable */`
- 行级禁用：CSS 注释 `/*! scopecss-disable-next-line */`

> **注意**：主应用的样式仍可能影响子应用，推荐通过约定前缀或 CSS Modules 解决。

### 4. 元素隔离

概念来自 Shadow DOM，micro-app 模拟实现了类似功能：

- 子应用通过 `document.querySelector()` 获取的元素仅限自身内部
- 主应用仍可获取子应用的元素（未做限制，因为主应用需要统筹全局）
- 可通过 `removeDomScope()` 方法临时解除元素绑定

### 5. 虚拟路由系统

micro-app 通过拦截浏览器路由事件和自定义 `location`/`history`，实现了一套虚拟路由系统，使子应用路由与主应用隔离。

**路由模式**：

| 模式 | 说明 |
|------|------|
| `search`（默认） | 子应用路由作为 query 参数同步到浏览器地址 |
| `native` | 放开路由隔离，主/子应用共同基于浏览器路由渲染 |
| `native-scope` | 同 native，但子应用域名指向自身 |
| `pure` | 子应用独立于浏览器路由渲染，类似组件（测试中） |
| `state` | 基于 `history.state` 模拟路由，不修改浏览器地址（测试中） |

**路由 API**：
- `microApp.router.push()` / `replace()` — 控制子应用跳转
- `microApp.router.go()` / `back()` / `forward()` — 历史记录导航
- `microApp.router.beforeEach()` / `afterEach()` — 导航守卫
- `microApp.router.setBaseAppRouter()` — 子应用控制主应用跳转

### 6. 数据通信

提供灵活的数据通信机制，支持主/子应用双向通信和全局通信：

**主 → 子**：
- 通过 `<micro-app data={...}>` 属性传递
- `microApp.setData(name, data)` 手动发送

**子 → 主**：
- `window.microApp.dispatch(data)` 发送数据
- `window.microApp.addDataListener(callback)` 监听数据

**全局通信**：
- `microApp.setGlobalData(data)` / `window.microApp.setGlobalData(data)`
- 向主应用和所有子应用广播数据

**特点**：
- 数据变更检测（新旧值合并，相同数据不重复发送）
- 异步批量执行（多个调用在下一帧合并）
- 数据缓存（子应用卸载后保留，可通过 `clear-data` 或 `destroy` 清空）

### 7. 资源系统

**资源路径自动补全**：
- 对子应用的相对路径资源（`link`、`script`、`img`、CSS 中的 `background-image` 等）自动补全为绝对地址
- 失效场景：某些框架创建的动态元素无法被拦截，或关闭沙箱/样式隔离时

**运行时 publicPath 方案**：
```js
// 子应用 public-path.js
if (window.__MICRO_APP_ENVIRONMENT__) {
  __webpack_public_path__ = window.__MICRO_APP_PUBLIC_PATH__
}
```

**资源共享**：
- `globalAssets` 配置：预加载公共资源到缓存
- `global` 属性：`<script global>` 标记资源为公共资源

### 8. 插件系统

插件系统允许开发者拦截和处理子应用的 JS/HTML 资源，主要用于修复沙箱环境下的兼容性问题。

**使用场景**：沙箱中顶层变量无法泄漏为全局变量（如 `var xx = ` 定义的变量），可通过插件将其转换为 `window.xx = `。

```js
microApp.start({
  plugins: {
    global: [/* 全局插件 */],
    modules: {
      'appName': [/* 特定应用插件 */]
    }
  }
})
```

**官方地图插件**：`@micro-zoe/micro-plugin-map`，适配高德、百度、腾讯地图。

## 快速开始

### 主应用

```bash
npm i @micro-zoe/micro-app --save
```

```js
// main.js
import microApp from '@micro-zoe/micro-app'
microApp.start()
```

```jsx
// React 组件
<micro-app name='my-app' url='http://localhost:3000/'></micro-app>
```

### 子应用

只需配置跨域支持（开发环境）：

```js
// webpack devServer
devServer: {
  headers: {
    'Access-Control-Allow-Origin': '*',
  }
}
```

子应用几乎无需改造，这是 micro-app 相比其他框架的显著优势。

## 与其他微前端框架对比

| 特性 | micro-app | qiankun | wujie | single-spa |
|------|:---------:|:-------:|:-----:|:----------:|
| **核心原理** | WebComponent 自定义元素 | single-spa + HTML Entry + 沙箱 | Web Components + iframe | 路由劫持 + 生命周期 |
| **接入成本** | 极低（类似组件） | 低（HTML Entry） | 极低 | 中（需改生命周期） |
| **JS 沙箱** | with / iframe | Proxy / Snapshot | iframe | 无 |
| **样式隔离** | CSS 前缀 | Shadow DOM / ScopedCSS | Shadow DOM | 无 |
| **子应用改造** | 几乎无需改造 | 几乎无需改造 | 几乎无需改造 | 需适配生命周期 |
| **框架支持** | 任意框架 | 任意框架 | 任意框架 | 任意框架 |
| **vite 支持** | 原生支持（iframe 沙箱） | 需适配 | 原生支持 | 需适配 |

## 限制与注意事项

### 浏览器兼容性
- 依赖 `CustomElements` 和 `Proxy` 两个 API
- 不支持 IE 浏览器
- 移动端需 iOS 10+、Android 5+

### 跨域要求
- 子应用**必须支持跨域**，因为主应用通过 fetch 加载子应用资源
- 生产环境需配置 Nginx 或其他服务端跨域支持

### 沙箱常见问题
- **顶层变量泄漏问题**：沙箱中 `var name` 或 `function name() {}` 定义的顶层变量无法泄漏为全局变量，需修改为 `window.name = xx` 或通过插件系统处理
- **Module Federation**：需将 `library.type` 设置为 `window`
- **DllPlugin**：需将 `output.library.type` 设置为 `window`

### 内存管理
- 子应用卸载后默认保留缓存（提升二次渲染速度），内存不会持续增长
- 频繁渲染/卸载场景下，避免使用 `destroy` 属性（反而增加内存消耗）
- 推荐一个子应用对应一个 `name`，通过路由控制页面切换
- 内存泄漏排查：切换 UMD 模式、避免设置 `destroy`、避免频繁创建新 `name`

### 性能考量
- 开启 `fiber` 模式可异步执行子应用 JS，减少对主应用的影响，但会降低子应用渲染速度
- 关闭样式隔离（`disable-scopecss`）可提升渲染速度，但需确保应用间样式不冲突
- 预加载（`prefetch`）可在浏览器空闲时提前加载子应用资源

## 适用场景

- **快速接入微前端**：希望以最低成本、最少代码改造实现微前端
- **多框架共存**：主/子应用使用不同框架，需要无缝集成
- **遗留系统改造**：子应用几乎无需改造即可接入
- **组件化思维**：希望将微前端应用当作普通组件使用

## 参考资料

- [micro-app 官方文档](https://jd-opensource.github.io/micro-app/)
- [GitHub 仓库](https://github.com/jd-opensource/micro-app)
- [快速开始文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/start)
- [JS 沙箱文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/sandbox)
- [虚拟路由系统文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/router)
- [数据通信文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/data)
- [插件系统文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/plugins)
- [资源系统文档](https://jd-opensource.github.io/micro-app/docs.html#/zh-cn/static-source)
