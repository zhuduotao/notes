---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - browser
  - web-security
  - same-origin
  - same-site
  - cookie
  - origin
  - site
  - cors
  - csrf
aliases:
  - 同源同站
  - same-origin same-site
  - origin site 区别
source_type: official-doc
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy'
  - 'https://web.dev/articles/same-site-same-origin'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
  - 'https://publicsuffix.org/list/'
  - 'https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects'
status: verified
---
Same origin 与 same site 是 Web 安全中两个基础但常被混淆的概念。它们定义了不同的边界，应用于不同的安全机制。

## Origin（源）

Origin 由三个部分组成：

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| Scheme | 协议/方案 | `https`、`http`、`ftp`、`ws`、`wss` |
| Host | 主机名（域名或 IP） | `www.example.com`、`127.0.0.1` |
| Port | 端口号（若未显式指定则使用默认值） | `443`（HTTPS 默认）、`80`（HTTP 默认） |

例如 URL `https://www.example.com:443/path` 的 origin 是 `https://www.example.com:443`。

### Same-origin 判断规则

两个 URL 只有在 **scheme、host、port 三者完全一致** 时才为 same-origin。

以 `https://www.example.com:443` 为基准的判断示例：

| 对比 URL | 结果 | 原因 |
|---------|------|------|
| `https://www.example.com:443/other` | Same-origin | 仅路径不同 |
| `https://**login**.example.com:443` | Cross-origin | 子域名不同 |
| `https://example.com:443` | Cross-origin | 子域名不同 |
| `**http**://www.example.com:443` | Cross-origin | 协议不同 |
| `https://www.example.com:**80**` | Cross-origin | 端口不同 |
| `https://www.example.com` | Same-origin | 隐式端口 443 匹配 |

### Origin 的安全作用

同源策略（Same-origin policy）是浏览器核心安全机制，限制不同 origin 之间的交互：

- **DOM 访问**：跨 origin 无法直接访问 `document`、读写 DOM
- **JavaScript API**：跨 origin 对 `Window` 和 `Location` 的访问受限（仅 `postMessage`、`blur`、`close`、`focus` 等可用）
- **数据存储**：`localStorage`、`IndexedDB`、`sessionStorage` 按 origin 隔离
- **网络请求**：跨 origin 的 `fetch()`、XHR 受 CORS 限制

详见 [HTML Living Standard § Cross-origin objects](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects)。

## Site（站）

Site 的定义涉及 **eTLD（effective TLD）** 和 **eTLD+1** 的概念。

### TLD 与 eTLD

传统 TLD（Top-Level Domain）如 `.com`、`.org` 由 IANA Root Zone Database 维护。但仅靠 TLD 无法准确识别"站点边界"，因为存在如 `.co.jp`、`.github.io` 这样的复合结构。

Public Suffix List 定义了 **eTLD（有效顶级域名）**，用于标识"不可注册"的域名层级。例如：

| 域名部分 | eTLD | 说明 |
|---------|------|------|
| `.com` | `.com` | 标准 TLD |
| `.co.uk` | `.co.uk` | 英国商业域名（不可单独注册） |
| `.github.io` | `.github.io` | GitHub Pages 域名（不可单独注册） |
| `.edu.cn` | `.edu.cn` | 中国教育机构域名 |

eTLD 列表由 Mozilla 维护，见 [publicsuffix.org/list](https://publicsuffix.org/list/)。

### eTLD+1

**eTLD+1** = eTLD + 其前面的一个域名部分（可注册域名）。这是确定"站点"的关键：

| 完整域名 | eTLD | eTLD+1 | 含义 |
|---------|------|--------|------|
| `www.example.com` | `.com` | `example.com` | 可注册域名 |
| `blog.example.co.uk` | `.co.uk` | `example.co.uk` | 可注册域名 |
| `user.project.github.io` | `.github.io` | `project.github.io` | GitHub 项目站点 |

### Same-site 判断规则（Schemeful Same-site）

自 2019 年起，[WHATWG URL 规范](https://github.com/whatwg/url/issues/448) 将 **scheme** 纳入 site 定义。当前 same-site 判断规则：

两个 URL 只有在 **scheme 相同** 且 **eTLD+1 相同** 时才为 same-site。

以 `https://www.example.com:443` 为基准的判断示例：

| 对比 URL | 结果 | 原因 |
|---------|------|------|
| `https://**login**.example.com:443` | Same-site | 子域名不影响 |
| `https://example.com:443` | Same-site | 子域名不影响 |
| `https://www.example.com:**80**` | Same-site | 端口不影响 |
| `**http**://www.example.com:443` | Cross-site | 协议不同 |
| `https://**evil**.com` | Cross-site | eTLD+1 不同 |

### Schemeless Same-site（旧定义）

早期 same-site 定义不含 scheme。`http://example.com` 和 `https://example.com` 在旧定义下为 same-site。

因 HTTP 作为弱通道存在安全风险，新规范已将 scheme 纳入。旧形式现称 **schemeless same-site**，仅用于历史兼容参考。

## Same-origin vs Same-site 对比

| 特性 | Same-origin | Same-site |
|------|-------------|-----------|
| **边界粒度** | 细（子域名、端口均区分） | 粗（忽略子域名、端口） |
| **组成要素** | Scheme + Host + Port | Scheme + eTLD+1 |
| **子域名关系** | `a.example.com` ≠ `b.example.com` | `a.example.com` = `b.example.com` |
| **端口关系** | `example.com:443` ≠ `example.com:80` | 端口不影响 |
| **协议关系** | `http` ≠ `https` | `http` ≠ `https`（新规范） |
| **主要应用** | DOM 访问、Storage、CORS | Cookie SameSite、Site Isolation |
| **规范来源** | HTML Living Standard | WHATWG URL / RFC 6265 |

### 关系图示

```
Origin 粒度：精细
├─ https://www.example.com:443  ── Origin A
├─ https://login.example.com:443  ── Origin B（与 A cross-origin）
└─ https://www.example.com:80    ── Origin C（与 A cross-origin）

Site 粒度：粗
├─ https://www.example.com:443  ── Site X
├─ https://login.example.com:443  ── Site X（与上面 same-site）
└─ https://www.example.com:80    ── Site X（与上面 same-site）

跨 Site：
└─ http://www.example.com      ── Site Y（cross-site，因协议不同）
└─ https://evil.com            ── Site Z（cross-site）
```

## 主要应用场景

### Same-origin 的应用

| 场景 | 说明 |
|------|------|
| DOM/Window 访问 | iframe 跨 origin 无法直接访问父页面 DOM |
| Web Storage | `localStorage`、`sessionStorage` 按 origin 隔离 |
| IndexedDB | 数据库按 origin 隔离 |
| Service Worker | 注册范围受 origin 限制 |
| BroadcastChannel | 仅 same-origin 上下文可订阅同一频道 |
| CORS | 跨 origin 资源请求需服务器明确授权 |

### Same-site 的应用

| 场景 | 说明 |
|------|------|
| Cookie SameSite 属性 | 控制跨站请求是否携带 cookie |
| Site Isolation | Chrome 进程隔离以 site 为边界，防范 Spectre 攻击 |
| Sec-Fetch-Site Header | 浏览器自动附加，标识请求的 site 关系 |

## Cookie SameSite 属性

Cookie 的 SameSite 属性基于 **site** 概念而非 origin。这是防范 CSRF 攻击的关键机制。

### 属性值说明

| 值 | 行为 | 适用场景 |
|----|------|---------|
| `Strict` | 仅 same-site 请求携带 | 高安全 cookie（如认证） |
| `Lax` | same-site 请求 + 顶级导航的安全方法（GET）携带 | 平衡安全与可用性（多数 cookie 默认值） |
| `None` | 跨站请求也携带 | 需跨站使用的 cookie（如第三方登录），必须配合 `Secure` |

**Lax 的"安全方法"限制**：仅 `GET`、`HEAD`、`OPTIONS`、`TRACE`。`POST`、`PUT`、`DELETE` 跨站导航不携带。

**Lax 的"顶级导航"限制**：用户点击链接跳转、表单 GET 提交可携带；`fetch()`、`<img>`、`<script>`、iframe 内导航不携带。

### 默认值演变

- Chrome 80+（2020 年 2 月）：默认 `Lax`
- Safari 13+：默认 `Lax`
- Firefox 69+：默认 `Lax`（实验性）

若未显式设置 SameSite，现代浏览器默认应用 `Lax`。详见 [MDN Set-Cookie SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#samesitesamesite-value)。

### 示例配置

```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
Set-Cookie: tracking=xyz; SameSite=None; Secure
Set-Cookie: preferences=defaults; SameSite=Lax
```

## Sec-Fetch-Site HTTP Header

现代浏览器（Chrome 76+、Firefox 90+、Safari 16.4+）在请求中自动附加 `Sec-Fetch-Site` header，标识请求与目标资源的 site 关系：

| 值 | 含义 |
|----|------|
| `same-origin` | 请求与目标 same-origin |
| `same-site` | 请求与目标 same-site 但 cross-origin（子域名或端口不同） |
| `cross-site` | 请求与目标 cross-site |
| `none` | 请求由用户直接发起（如地址栏输入、书签） |

服务器可据此判断请求来源进行安全校验。Header 以 `Sec-` 开头，JavaScript 无法修改，浏览器强制设置，可信度高。

详见 [MDN Sec-Fetch-Site](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site)。

## 特殊 Origin 类型

### Opaque Origin（不透明源）

以下 URL 产生 opaque origin：

| 类型 | 说明 |
|------|------|
| `data:` URL | 每个 `data:` URL 获得独立的空安全上下文 |
| `about:blank` | 继承创建者的 origin |
| `javascript:` URL | 继承执行脚本的 origin |
| `file:///` | 现代浏览器通常视为 opaque origin，同目录文件不一定同源 |

Opaque origin 与任何其他 origin 比较，结果均为 cross-origin。

### Sandbox iframe

设置了 `sandbox` 属性的 iframe：

- 若无 `allow-same-origin`：iframe 内页面被视为独特 opaque origin
- 即使 URL 与父页面同源，也无法访问父页面 DOM/Storage

同时设置 `allow-scripts` 和 `allow-same-origin` 时，iframe 内脚本可移除 sandbox，导致沙箱失效。

## 常见误区

| 误区 | 正确理解 |
|------|---------|
| "同站就是同源" | Same-site 不关心子域名和端口，比 same-origin 更宽松 |
| "`http` 和 `https` 同站" | 新规范下不同协议为 cross-site（schemeless same-site 下才同站） |
| "Cookie 按 origin 隔离" | Cookie 按 domain/path 隔离，SameSite 属性基于 site 判断 |
| "`document.domain` 可改变 origin" | 该方式已废弃，会破坏安全模型，不建议使用 |
| "端口不影响同源判断" | 端口是 origin 组成部分，不同端口为 cross-origin |

## 参考资料

- [Same-origin policy - MDN](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
- ["Same-site" and "same-origin" - web.dev](https://web.dev/articles/same-site-same-origin)
- [Set-Cookie SameSite - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#samesitesamesite-value)
- [Public Suffix List](https://publicsuffix.org/list/)
- [HTML Living Standard § Cross-origin objects](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects)
- [WHATWG URL 规范 - Site 定义变更](https://github.com/whatwg/url/issues/448)
- [RFC 6265 - HTTP State Management Mechanism](https://datatracker.ietf.org/doc/html/rfc6265)
