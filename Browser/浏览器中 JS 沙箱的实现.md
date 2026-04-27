## 概述

JS 沙箱（JavaScript Sandbox）是浏览器中用于隔离不可信代码执行环境的机制。它的核心目标是让代码在受限的上下文中运行，防止恶意或错误代码访问敏感 API、污染全局状态、或影响宿主页面。

浏览器本身提供了多层沙箱机制，从进程级隔离到 API 级限制，再到开发者可在应用层实现的运行时沙箱。理解这些机制对微前端架构、第三方脚本加载、在线代码编辑器、插件系统等场景至关重要。

## 为什么需要 JS 沙箱

| 场景 | 风险 | 沙箱作用 |
|------|------|----------|
| 第三方脚本（广告、分析、SDK） | 可能读取用户数据、篡改页面、注入恶意代码 | 限制 API 访问、隔离全局变量 |
| 微前端子应用 | 子应用之间全局变量冲突、样式污染 | 运行时隔离，确保独立部署 |
| 在线代码编辑器（CodePen、JSFiddle） | 用户输入的代码可能包含恶意逻辑 | 阻止危险 API、限制网络请求 |
| 浏览器扩展 | 恶意扩展可能窃取页面数据 | 权限模型 + 内容脚本隔离 |
| 插件系统 | 插件代码可能影响宿主应用稳定性 | 限制执行能力、控制资源访问 |

## 浏览器原生沙箱机制

### 1. 多进程模型（进程级沙箱）

现代浏览器（Chrome、Firefox、Edge、Safari）采用多进程架构，将不同来源的页面分配到独立的渲染进程中：

- **每个渲染进程拥有独立的 V8/SpiderMonkey 实例**，进程间内存完全隔离
- 一个标签页崩溃不会影响其他标签页（Site Isolation）
- 渲染进程运行在沙箱化的操作系统权限下（Chrome 使用 sandbox 机制限制系统调用）

Chrome 的 Site Isolation 确保不同 origin 的页面运行在不同进程中，即使它们嵌入在同一页面中（如跨域 iframe）。这是浏览器最底层的安全沙箱 [[Chrome 多进程模型]]。

**限制**：进程级沙箱由浏览器内部控制，开发者无法直接利用它来隔离同域下的不同脚本。

### 2. iframe sandbox 属性（文档级沙箱）

`<iframe>` 的 `sandbox` 属性是 HTML5 引入的最强大的原生沙箱机制。当设置 `sandbox` 属性时，iframe 内的内容被视为来自一个**不透明的唯一源（unique opaque origin）**，即使 URL 与父页面同源 [[HTML Standard § iframe sandbox]](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-sandbox)。

#### 默认限制（仅设置 `sandbox` 不加任何值）

- 脚本无法执行（`<script>` 不运行）
- 表单无法提交
- 弹窗（`alert`、`confirm`、`prompt`）被禁用
- 指针锁定（Pointer Lock）被禁用
- 无法导航顶层窗口或其他框架
- 无法自动下载文件
- 无法使用 Presentation API
- 插件（如 Flash）被禁用

#### 可选权限标志

通过添加空格分隔的关键字，可以有选择地放宽限制：

| 标志 | 作用 |
|------|------|
| `allow-scripts` | 允许执行脚本 |
| `allow-same-origin` | 将内容视为真实源（而非不透明源），允许访问同源资源 |
| `allow-forms` | 允许提交表单 |
| `allow-modals` | 允许弹窗（`alert`、`confirm`、`prompt`） |
| `allow-popups` | 允许打开新窗口（`window.open`） |
| `allow-popups-to-escape-sandbox` | 新窗口脱离沙箱 |
| `allow-top-navigation` | 允许导航顶层窗口 |
| `allow-top-navigation-by-user-activation` | 仅在用户交互后允许导航顶层 |
| `allow-presentation` | 允许使用 Presentation API |
| `allow-pointer-lock` | 允许 Pointer Lock API |
| `allow-downloads` | 允许下载文件 |
| `allow-top-navigation-to-custom-protocols` | 允许导航到自定义协议（如 `weixin://`） |

#### 使用示例

```html
<!-- 最严格的沙箱：仅允许渲染静态 HTML -->
<iframe sandbox src="https://untrusted.example.com/content"></iframe>

<!-- 允许脚本执行，但仍保持源隔离 -->
<iframe sandbox="allow-scripts" src="https://untrusted.example.com/widget"></iframe>

<!-- 允许脚本且视为同源（注意：同源 + 脚本 = 可自行移除 sandbox 属性） -->
<iframe sandbox="allow-scripts allow-same-origin" src="/trusted-widget.html"></iframe>
```

> **安全警告**：同时设置 `allow-scripts` 和 `allow-same-origin` 时，如果嵌入的页面与父页面同源，沙箱内的脚本可以移除 `sandbox` 属性并重新加载自身，从而完全逃逸沙箱 [[HTML Standard]](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-sandbox)。因此同源嵌入时应避免同时启用这两个标志。

#### srcdoc 属性配合

`srcdoc` 允许直接在属性中指定 iframe 的 HTML 内容，常与 `sandbox` 配合使用来安全渲染用户生成的内容：

```html
<iframe sandbox="allow-scripts" srcdoc="<p>用户输入的内容</p>"></iframe>
```

`srcdoc` 创建的文档的源为 `about:srcdoc`，继承父页面的源，但仍受 `sandbox` 属性限制 [[HTML Standard]](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-srcdoc)。

### 3. Content Security Policy（CSP，资源级沙箱）

CSP 通过 HTTP 响应头或 `<meta>` 标签，控制页面允许加载和执行哪些资源，是防御 XSS 的核心机制 [[MDN CSP]](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)。

#### 与 JS 沙箱相关的核心指令

| 指令 | 作用 |
|------|------|
| `script-src` | 控制允许加载的脚本来源 |
| `script-src-elem` | 控制 `<script>` 元素的来源 |
| `script-src-attr` | 控制内联事件处理器（如 `onclick`） |
| `default-src` | 其他指令未设置时的回退策略 |

#### 关键行为

**禁用内联脚本**：当设置了 `script-src` 或 `default-src` 后，以下行为默认被阻止：
- `<script>` 标签中的内联代码
- 内联事件处理器（`onclick`、`onload` 等）
- `javascript:` URL
- `eval()` 和 `Function()` 构造函数
- `setTimeout("string")` 和 `setInterval("string")`

**放行方式**：

```http
# 方式一：Nonce（推荐，每次请求生成随机值）
Content-Security-Policy: script-src 'nonce-abc123'

# 页面中脚本需匹配 nonce
<script nonce="abc123" src="app.js"></script>

# 方式二：Hash（适合静态内容）
Content-Security-Policy: script-src 'sha256-Base64Hash...'

# 方式三：strict-dynamic（信任已验证脚本加载的子脚本）
Content-Security-Policy: script-src 'nonce-abc123' 'strict-dynamic'
```

**禁用 eval**：`unsafe-eval` 关键字控制是否允许 `eval()` 等动态代码执行：

```http
# 禁止 eval（默认行为，推荐）
Content-Security-Policy: script-src 'self'

# 允许 eval（不推荐，仅在必要时使用）
Content-Security-Policy: script-src 'self' 'unsafe-eval'
```

#### Strict CSP 最佳实践

```http
Content-Security-Policy:
  script-src 'nonce-{RANDOM}' 'strict-dynamic';
  object-src 'none';
  base-uri 'none';
```

此策略：
- 仅允许带 nonce 的脚本执行
- 信任已验证脚本动态加载的子脚本（`strict-dynamic`）
- 禁止 `<object>` 和 `<embed>` 元素
- 禁止 `<base>` 元素篡改基础 URL

### 4. 同源策略（Same-Origin Policy，访问级沙箱）

同源策略是浏览器最基础的安全机制，限制不同 origin 的文档之间的交互 [[MDN Same-Origin Policy]](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)。

**源（origin）**由协议（scheme）、主机（host）、端口（port）三部分组成。只有三者完全一致才为同源。

#### 跨域脚本访问限制

跨域时，通过 `iframe.contentWindow`、`window.parent`、`window.open` 等 API 获取的 Window 引用仅暴露有限属性：

**可访问的 Window 方法**：
- `blur()`、`close()`、`focus()`、`postMessage()`

**可访问的 Window 属性（只读）**：
- `closed`、`frames`、`length`、`opener`、`parent`、`self`、`top`、`window`

**可访问的 Window 属性（读写）**：
- `location`（写时会触发导航）

**可访问的 Location 属性**：
- `replace()` 方法
- `href`（仅写）

访问其他属性（如 `document`、`localStorage`）会抛出 `SecurityError` [[HTML Living Standard § Cross-origin objects]](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects)。

#### 数据存储隔离

- `localStorage`、`sessionStorage`、`IndexedDB` 按源隔离
- Cookie 使用独立的源定义（可设置 `Domain` 属性限定作用域）

### 5. Permissions Policy（权限级沙箱）

Permissions Policy（原 Feature Policy）允许页面控制哪些浏览器功能可以被自身或 iframe 使用 [[MDN Permissions Policy]](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy)。

```http
# HTTP 响应头
Permissions-Policy: geolocation=(), camera=(), microphone=()

# iframe 级别的精细控制
<iframe src="https://example.com" allow="geolocation; camera"></iframe>
```

常用可控制的功能：
- `geolocation`、`camera`、`microphone`
- `accelerometer`、`gyroscope`、`magnetometer`
- `payment`、`usb`、`screen-wake-lock`
- `fullscreen`、`autoplay`

### 6. Trusted Types（注入防护）

Trusted Types API 通过强制要求对注入点（如 `innerHTML`、`eval`）的输入进行安全转换，防止 DOM-based XSS [[MDN Trusted Types]](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP#requiring_trusted_types)。

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types myPolicy
```

```javascript
// 定义策略
const policy = trustedTypes.createPolicy('myPolicy', {
  createHTML: (input) => DOMPurify.sanitize(input),
  createScript: (input) => { /* 自定义安全逻辑 */ },
});

// 必须使用策略创建的受信任类型
el.innerHTML = policy.createHTML(userInput);
```

## 应用层 JS 沙箱（开发者实现）

浏览器原生机制之外，开发者可在 JavaScript 运行时层面实现沙箱，主要用于微前端框架和在线代码执行环境。

### 1. Proxy 沙箱

利用 ES6 `Proxy` 拦截对 `window` 对象的访问，将子应用的全局变量读写重定向到独立的 `fakeWindow`：

```javascript
class ProxySandbox {
  constructor() {
    this.fakeWindow = Object.create(null);
    this.proxy = new Proxy(window, {
      get(target, key) {
        if (key in this.fakeWindow) {
          return this.fakeWindow[key];
        }
        return target[key];
      },
      set(target, key, value) {
        this.fakeWindow[key] = value;
        return true;
      },
      has(target, key) {
        return key in this.fakeWindow || key in target;
      },
    });
  }
}
```

**特点**：
- 每个子应用拥有独立的 `fakeWindow`，互不干扰
- 子应用卸载时，丢弃 `fakeWindow`，不留痕迹
- 需要浏览器支持 `Proxy`（iOS 10+、Android 5+，不支持 IE）

**限制**：
- 无法拦截通过 `with` 语句或 `eval` 执行时的某些隐式全局访问
- 对 `window` 上已有属性的修改可能仍会影响全局（如 `window.location`）
- 无法完全阻止对原生构造函数的篡改（如 `Array.prototype.push`）

qiankun 框架的 `ProxySandbox` 即基于此原理 [[qiankun 核心设计]](../Frontend/MicroFrontend/qiankun（乾坤）核心设计与原理.md)。

### 2. Snapshot 沙箱（快照沙箱）

在不支持 `Proxy` 的环境中，通过记录 `window` 的快照实现隔离：

```javascript
class SnapshotSandbox {
  constructor() {
    this.windowSnapshot = {};
  }

  active() {
    // 记录当前 window 状态
    for (const key in window) {
      this.windowSnapshot[key] = window[key];
    }
  }

  inactive() {
    // 恢复快照，撤销子应用的修改
    for (const key in window) {
      if (this.windowSnapshot.hasOwnProperty(key)) {
        window[key] = this.windowSnapshot[key];
      } else {
        delete window[key];
      }
    }
  }
}
```

**特点**：
- 兼容性好，不依赖 `Proxy`
- 只能单实例运行（多个子应用共享同一 `window`）

**限制**：
- 性能差：每次激活/卸载需要遍历整个 `window`
- 只能单实例模式，不支持多子应用同时运行
- 无法精确还原被删除的原生属性

qiankun 的 `SnapshotSandbox` 用于 IE 兼容模式 [[qiankun FAQ]](https://qiankun.umijs.org/faq#does-qiankun-compatible-with-ie)。

### 3. with 沙箱

利用 `with` 语句改变作用域链，配合自定义的代理对象：

```javascript
const fakeWindow = new Proxy({}, {
  get(target, key) {
    return target[key] || window[key];
  },
  set(target, key, value) {
    target[key] = value;
    return true;
  },
});

with (fakeWindow) {
  // 子应用代码在此执行
  // 对全局变量的访问优先从 fakeWindow 读取
}
```

micro-app 框架默认使用 with 沙箱模式 [[MicroApp 介绍]](../Frontend/MicroFrontend/MicroApp（京东微前端框架）介绍与核心实现原理.md)。

**限制**：
- `with` 在严格模式（`"use strict"`）下被禁用
- 无法拦截所有全局变量访问场景
- 性能开销较大

### 4. iframe 沙箱（应用层）

将子应用的 JS 代码放在 iframe 中执行，利用 iframe 天然的全局隔离：

```javascript
const iframe = document.createElement('iframe');
iframe.style.display = 'none';
document.body.appendChild(iframe);

// 在 iframe 中执行代码
const iframeWindow = iframe.contentWindow;
iframeWindow.eval(subAppCode);
```

**特点**：
- 完全的全局变量隔离（独立的 `window`、`document`）
- 天然支持多实例

**限制**：
- 通信必须通过 `postMessage`，开销大
- URL 管理复杂（路由不同步）
- 弹窗、遮罩等组件限制在 iframe 内部
- 性能较差（每个 iframe 需要独立的 JS 引擎实例）

wujie 框架和 micro-app 的 iframe 模式采用此方案。

### 5. Web Worker 沙箱

Web Worker 在独立线程中运行，无法直接访问 DOM 和 `window`，天然适合沙箱化执行纯逻辑代码：

```javascript
// 主线程
const worker = new Worker('sandbox.js');
worker.postMessage({ code: userCode });
worker.onmessage = (e) => console.log(e.data);

// sandbox.js
self.onmessage = (e) => {
  // 在受限环境中执行
  const result = eval(e.data.code);
  self.postMessage(result);
};
```

**特点**：
- 无法访问 DOM、`window`、`document`
- 无法阻塞主线程
- 独立的内存空间

**限制**：
- 仅适合纯计算逻辑，无法执行需要 DOM 的代码
- 部分 API 不可用（如 `localStorage`）

### 6. 沙箱模式对比

| 模式 | 隔离级别 | 多实例 | 性能 | 兼容性 | 典型场景 |
|------|----------|--------|------|--------|----------|
| iframe sandbox | 最高（进程/文档级） | ✓ | 低 | 全浏览器 | 第三方内容渲染 |
| Proxy 沙箱 | 中（全局变量级） | ✓ | 高 | 需 Proxy 支持 | 微前端（qiankun） |
| Snapshot 沙箱 | 低（全局变量级） | ✗ | 低 | 全浏览器 | IE 兼容模式 |
| with 沙箱 | 中（作用域级） | ✓ | 中 | 不支持严格模式 | 微前端（micro-app） |
| iframe 应用层 | 高（全局隔离） | ✓ | 低 | 全浏览器 | 微前端（wujie） |
| Web Worker | 高（线程级） | ✓ | 高 | 全浏览器 | 纯逻辑执行 |
| CSP | 资源加载级 | - | 无影响 | 全浏览器 | XSS 防御 |

## 限制与注意事项

### Proxy 沙箱的已知问题

1. **无法拦截原型链上的修改**：子应用修改 `Array.prototype` 仍会影响全局
2. **无法完全阻止 `eval` 逃逸**：通过 `eval` 执行的代码可能绕过 Proxy
3. **某些原生 API 无法代理**：如 `window.location`、`window.document` 的部分属性
4. **性能开销**：每次全局变量访问都经过 Proxy handler

### iframe 沙箱的已知问题

1. **通信成本高**：`postMessage` 是异步的，结构化克隆有性能开销
2. **路由管理复杂**：浏览器地址栏不反映 iframe 内部路由
3. **弹窗限制**：`alert`、Modal 等组件被限制在 iframe 边界内
4. **样式隔离过度**：第三方 UI 库可能需要额外适配

### CSP 的已知问题

1. **维护成本高**：第三方脚本变更时需要更新 nonce/hash
2. **不兼容静态站点**：nonce 需要服务端动态生成
3. **`unsafe-inline` 和 `unsafe-eval` 削弱保护**：很多项目为了兼容性被迫启用
4. **不阻止已加载脚本的行为**：CSP 只控制加载，不控制执行时的行为

### 通用限制

- **没有任何沙箱能完全阻止所有攻击**：沙箱是纵深防御（defense in depth）的一层，不是银弹
- **沙箱逃逸漏洞**：浏览器和框架的沙箱实现都可能存在可被利用的漏洞
- **性能与安全的权衡**：隔离越严格，性能开销越大

## 最佳实践

### 加载第三方脚本

```html
<!-- 使用 iframe sandbox 渲染不可信内容 -->
<iframe sandbox="allow-scripts" src="https://ads.example.com/widget"></iframe>

<!-- 使用 CSP 限制脚本来源 -->
<meta http-equiv="Content-Security-Policy" content="script-src 'self' https://trusted-cdn.example.com">
```

### 微前端架构

- 优先使用 Proxy 沙箱（性能好、支持多实例）
- 对不可信子应用使用 iframe 模式
- 配合样式隔离（Scoped CSS 或 Shadow DOM）
- 限制子应用对全局原型的修改

### 在线代码编辑器

```html
<!-- 使用 iframe + sandbox + CSP 多层防护 -->
<iframe
  sandbox="allow-scripts allow-same-origin"
  src="sandbox-runner.html"
  csp="script-src 'self' 'unsafe-eval'; object-src 'none'"
></iframe>
```

- 使用独立的域名托管用户代码
- 禁用 `allow-top-navigation` 防止导航逃逸
- 对 `eval` 执行设置超时限制

### 防御 XSS

- 部署 Strict CSP（nonce 或 hash 模式）
- 启用 Trusted Types API
- 对用户输入进行 HTML 转义
- 避免使用 `innerHTML`、`document.write`、`eval`

## 相关概念

- [[浏览器 Same Origin 与 Same Site 区别详解]] — 同源策略的详细解释
- [[iframe 与父页面通信方案详解]] — iframe 沙箱中的通信机制
- [[Chrome 多进程模型]] — 浏览器进程级沙箱的基础
- [qiankun（乾坤）核心设计与原理](../Frontend/MicroFrontend/qiankun（乾坤）核心设计与原理.md) — 微前端 Proxy/Snapshot 沙箱实现
- [MicroApp（京东微前端框架）介绍与核心实现原理](../Frontend/MicroFrontend/MicroApp（京东微前端框架）介绍与核心实现原理.md) — with/iframe 沙箱实现
- Web Components / Shadow DOM — 浏览器原生组件化与隔离标准

## 参考资料

- [HTML Living Standard — The iframe element (sandbox attribute)](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-sandbox)
- [MDN — Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)
- [MDN — Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- [MDN — Permissions Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy)
- [MDN — Trusted Types API](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)
- [HTML Living Standard — Cross-origin objects](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects)
- [CSP Is Dead, Long Live CSP!](https://dl.acm.org/doi/pdf/10.1145/2976749.2978363) — 关于白名单 CSP 不安全性的研究
- [qiankun 官方文档](https://qiankun.umijs.org/)
- [micro-app 官方文档](https://jd-opensource.github.io/micro-app/)
- [wujie 官方文档](https://www.wujie-micro.com/)
