---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - browser
  - iframe
  - postMessage
  - MessageChannel
  - BroadcastChannel
  - cross-origin
  - web-security
aliases:
  - iframe通信
  - iframe postMessage
  - 跨域iframe通信
source_type: official-doc
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage'
  - 'https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy
  - 'https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel'
  - 'https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel'
  - >-
    https://html.spec.whatwg.org/multipage/web-messaging.html#dom-window-postmessage-options-dev
status: verified
---
iframe 作为嵌套浏览上下文（nested browsing context），其内部页面与父页面之间存在天然隔离。通信方案的选择取决于是否同源、是否需要双向通信、以及性能和安全要求。

## 同源 vs 跨域

「源」（origin）由协议（scheme）、主机（host）、端口（port）三部分组成。只有三者完全一致才为同源。同源策略（Same-origin policy）限制跨源脚本对 Window 和 DOM 的直接访问。

同源 iframe 可直接访问父页面的 Window 对象和 DOM；跨域 iframe 则必须使用 `postMessage` 等受限机制。

## 方案一：window.postMessage

`postMessage` 是跨域 iframe 通信的核心 API，支持在任意两个 Window 对象之间安全传递消息。

### 基本语法

```javascript
targetWindow.postMessage(message, targetOrigin, [transfer])
```

| 参数 | 说明 |
|------|------|
| `message` | 要传递的数据，使用结构化克隆算法序列化。支持对象、数组、Map、Set、Blob、ArrayBuffer 等类型 |
| `targetOrigin` | 目标窗口的源。必须完全匹配（协议、主机、端口）才会投递消息。`*` 表示任意源（不推荐） |
| `transfer` | 可转移对象数组（如 MessagePort），转移后发送方不可再用 |

### 父页面向 iframe 发送

```javascript
const iframe = document.querySelector('iframe')
iframe.contentWindow.postMessage({ type: 'init', data: 'hello' }, 'https://child.example.com')
```

### iframe 向父页面发送

```javascript
window.parent.postMessage({ type: 'response', result: 'ok' }, 'https://parent.example.com')
```

### 接收消息

双方均通过 `message` 事件监听：

```javascript
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://trusted.example.com') return
  console.log(event.data, event.source)
})
```

事件对象包含三个关键属性：

| 属性 | 说明 |
|------|------|
| `data` | 发送方传递的数据 |
| `origin` | 发送方调用 `postMessage` 时的源 |
| `source` | 发送方 Window 对象引用，可用于双向通信 |

### 安全要点

MDN 明确强调两点：

1. **必须验证 `origin`**：任何窗口都可以向当前窗口发送消息，不验证源会导致 XSS 风险
2. **必须指定精确的 `targetOrigin`**：使用 `*` 可能被恶意站点拦截数据。只有在发送到 `data:` URL 时才必须用 `*`（因为 `data:` URL 是不透明源）

此外，`origin` 值不受 `document.domain` 影响；对 IDN 域名需同时检查 Unicode 和 Punycode 形式以兼容各浏览器。

## 方案二：MessageChannel

`MessageChannel` 提供两个相互连接的 `MessagePort`，适合需要建立专用通信通道的场景。可通过 `postMessage` 的 `transfer` 参数将一个端口转移到 iframe，随后双方通过各自端口直接通信。

### 使用示例

父页面：

```javascript
const channel = new MessageChannel()
const iframe = document.querySelector('iframe')

iframe.addEventListener('load', () => {
  channel.port1.onmessage = (e) => console.log('来自 iframe:', e.data)
  iframe.contentWindow.postMessage('port', '*', [channel.port2])
})
```

iframe 内：

```javascript
window.addEventListener('message', (e) => {
  if (e.data === 'port') {
    const port = e.ports[0]
    port.onmessage = (e) => console.log('来自父页面:', e.data)
    port.postMessage('hello from iframe')
  }
})
```

### 特点

- 通道一旦建立，后续通信无需再指定 `targetOrigin`
- 支持同源和跨域场景
- 可通过 `port.start()` 和 `port.close()` 控制生命周期
- 适用于 Web Worker 与主线程、iframe 与父页面的点对点通信

## 方案三：BroadcastChannel

`BroadcastChannel` 用于同源下的广播通信。同一源下的多个浏览上下文（窗口、标签页、iframe、Worker）可订阅同名频道，消息会广播到所有订阅者（发送方除外）。

### 使用示例

父页面：

```javascript
const bc = new BroadcastChannel('sync-channel')
bc.postMessage({ action: 'update', payload: { id: 1 } })
bc.onmessage = (e) => console.log('收到:', e.data)
```

iframe 内：

```javascript
const bc = new BroadcastChannel('sync-channel')
bc.onmessage = (e) => {
  if (e.data.action === 'update') {
    console.log('同步更新:', e.data.payload)
  }
}
```

### 特点

- 仅限同源使用，跨域无法订阅同一频道
- 消息自动广播到所有同名频道的订阅者
- 适合多窗口/多 iframe 间的状态同步、事件通知
- 不适合点对点定向通信（因为发送方自己也会收到）

## 方案四：同源直接访问

若 iframe 与父页面同源，可直接通过 Window 引用访问对方：

| 方向 | 获取引用的方式 |
|------|----------------|
| 父 → iframe | `iframe.contentWindow` / `iframe.contentDocument` |
| iframe → 父 | `window.parent` / `window.top`（`top` 指向最顶层窗口） |

同源下可访问对方 DOM、调用函数、读写变量。但需注意：

- 即使同源，若 iframe 设置了 `sandbox` 且不含 `allow-same-origin`，则被视为特殊源，无法直接访问
- `document.domain` 方式已被废弃，会破坏安全模型，不建议使用

## 方案对比

| 方案 | 同源 | 跨域 | 双向 | 多接收者 | 安全验证 | 适用场景 |
|------|------|------|------|----------|----------|----------|
| `postMessage` | ✓ | ✓ | ✓ | 需自行管理 | 必须验证 `origin` | 通用跨域通信 |
| `MessageChannel` | ✓ | ✓ | ✓ | 点对点 | 端口转移后无需验证 | 专用通道、Worker 通信 |
| `BroadcastChannel` | ✓ | ✗ | ✓ | ✓ | 自动（同源限制） | 同源多窗口/iframe 状态同步 |
| 直接 DOM 访问 | ✓ | ✗ | ✓ | - | 无（同源本身是信任边界） | 同源下深度集成 |

## 跨域 iframe 的额外限制

除了通信受限，跨域 iframe 还存在以下访问限制（依据 HTML Living Standard § Cross-origin objects）：

### Window 属性访问

跨域可访问的 Window 属性仅限：

- 方法：`blur()`、`close()`、`focus()`、`postMessage()`
- 属性（只读）：`closed`、`frames`、`length`、`opener`、`parent`、`self`、`top`、`window`
- 属性（读写）：`location`（写时会触发导航）

访问其他属性（如 `document`、`localStorage`）会抛出 SecurityError。

### Location 属性访问

跨域可访问的 Location 属性仅限：

- 方法：`replace()`
- 属性：`href`（仅写）

### 顶层导航限制

跨域 iframe 导航顶层页面（`window.top.location`）受到浏览器「framebusting intervention」限制：

- 无用户交互（sticky activation）时，跨域 iframe 无法直接导航顶层页面
- 浏览器可能弹出权限提示或静默拒绝
- 若 iframe 有 `sandbox`，需设置 `allow-top-navigation` 或 `allow-top-navigation-by-user-activation` 才能导航顶层

## sandbox 属性对通信的影响

`<iframe sandbox>` 会限制内部页面的能力。关键标志位：

| 标志位 | 影响 |
|--------|------|
| `allow-same-origin` | 缺失时 iframe 被视为「不透明源」，即使 URL 同源也无法访问父页面 DOM/Storage |
| `allow-scripts` | 缺失时 iframe 内无法执行脚本，也就无法监听 `message` 事件 |
| `allow-top-navigation` | 允许 iframe 导航顶层窗口 |
| `allow-top-navigation-by-user-activation` | 仅在用户交互后允许导航顶层 |

同时设置 `allow-scripts` 和 `allow-same-origin` 时，iframe 内脚本可移除 `sandbox` 属性，使沙箱失效。因此同源嵌入时应避免同时启用这两个标志。

## 浏览器兼容性

| API | 最低版本 | Baseline 状态 |
|-----|----------|---------------|
| `postMessage` | IE8+, 所有现代浏览器 | Widely available（2015.07 起） |
| `MessageChannel` | IE10+, Chrome 4+, Firefox 41+ | Widely available（2015.09 起） |
| `BroadcastChannel` | Chrome 54+, Firefox 38+, Safari 15.4+ | Widely available（2022.03 起） |
| `iframe.contentWindow` | 所有浏览器 | Widely available |

IE8-9 的 `postMessage` 仅支持字符串消息；结构化克隆算法支持始于 IE10。

## 常见问题

### 消息未触发

可能原因：

- `targetOrigin` 不匹配，浏览器静默丢弃消息
- iframe 未加载完成，脚本未执行或监听器未注册
- 跨域 iframe 缺少 `allow-scripts`

### 消息被恶意站点接收

若发送方使用 `targetOrigin: '*'`，任何嵌入了该 iframe 的页面都可接收消息。必须指定精确目标源。

### SharedArrayBuffer 通信失败

使用 `SharedArrayBuffer` 进行共享内存通信需要站点配置 COOP/COEP 头：

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

可通过 `window.crossOriginIsolated` 检查是否满足条件。

## 参考资料

- [Window.postMessage() - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
- [iframe 元素 - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe)
- [同源策略 - MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)
- [MessageChannel - MDN](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)
- [BroadcastChannel - MDN](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)
- [HTML Living Standard - Web Messaging](https://html.spec.whatwg.org/multipage/web-messaging.html#dom-window-postmessage-options-dev)
- [HTML Living Standard - Cross-origin objects](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects)
