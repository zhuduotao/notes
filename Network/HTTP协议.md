---
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - network
  - http
  - protocol
  - web-fundamentals
aliases:
  - HTTP
  - HTTP协议
  - HTTP版本
  - HTTP/2
  - HTTP/3
source_type: mixed
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP'
  - 'https://datatracker.ietf.org/doc/html/rfc9110'
  - 'https://datatracker.ietf.org/doc/html/rfc9112'
  - 'https://datatracker.ietf.org/doc/html/rfc9113'
  - 'https://datatracker.ietf.org/doc/html/rfc9114'
status: verified
---

## 概述

HTTP（HyperText Transfer Protocol，超文本传输协议）是万维网的数据通信基础。它由 Tim Berners-Lee 及其团队于 1989-1991 年间在 CERN 开发，经历了从简单的文件传输协议到现代高性能、安全协议的演进。

HTTP 是一个**应用层协议**，默认基于 TCP 传输层（HTTP/3 基于 UDP），采用**请求-响应模型**，默认端口为 80（HTTPS 为 443）。

## HTTP/0.9 — 单行协议（1991）

HTTP/0.9 是最初的版本，后来为了区分后续版本而被命名为 0.9。

### 特点

- **极简设计**：请求仅由单行组成，只支持 `GET` 方法
- **无版本号**：请求行不包含协议版本信息
- **无请求头**：不能传递任何元数据
- **无响应头**：响应只包含 HTML 文件内容本身
- **无状态码**：错误通过返回特定 HTML 页面描述

### 请求示例

```http
GET /my-page.html
```

### 响应示例

```html
<html>
  A text-only web page
</html>
```

### 限制

- 只能传输 HTML 文件
- 无法传输图片、音频等其他类型资源
- 没有错误处理机制

## HTTP/1.0 — 构建扩展性（1996）

HTTP/1.0 在 1991-1995 年间通过"尝试-观察"的方式逐步演进，最终于 1996 年 11 月通过 [RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945) 正式定义。

### 新增特性

| 特性 | 说明 |
|------|------|
| 版本信息 | 请求行追加 `HTTP/1.0` 版本号 |
| 状态码 | 响应首行包含状态码（如 `200 OK`），浏览器可据此判断请求成败 |
| HTTP 头 | 引入请求头和响应头概念，支持元数据传输 |
| Content-Type | 通过 `Content-Type` 头支持传输 HTML 以外的文件类型（如图片） |

### 请求示例

```http
GET /my-page.html HTTP/1.0
User-Agent: NCSA_Mosaic/2.0 (Windows 3.1)
```

### 响应示例

```http
HTTP/1.0 200 OK
Date: Tue, 15 Nov 1994 08:12:31 GMT
Server: CERN/3.0 libwww/2.17
Content-Type: text/html

<HTML>
A page with an image
  <IMG SRC="/my-image.gif">
</HTML>
```

### 关键限制

- **短连接**：每个请求都需要新建一次 TCP 连接
- 加载包含多个资源的页面时，需要多次建立/断开连接，性能开销大

## HTTP/1.1 — 标准化协议（1997）

HTTP/1.1 于 1997 年 1 月首次发布为 [RFC 2068](https://datatracker.ietf.org/doc/html/rfc2068)，是 HTTP 的第一个标准化版本。后续经过多次修订：RFC 2616（1999）、RFC 7230-7235（2014）、RFC 9112（2022）。

### 核心改进

| 特性 | 说明 |
|------|------|
| **持久连接** | 默认启用 `Connection: keep-alive`，TCP 连接可复用，减少握手开销 |
| **管道化（Pipelining）** | 允许在收到前一个响应前发送下一个请求，降低延迟（实际部署中因队头阻塞问题较少使用） |
| **分块传输** | 支持 `Transfer-Encoding: chunked`，无需预先知道内容长度即可流式传输 |
| **缓存控制** | 引入更精细的缓存机制（`Cache-Control`、`ETag` 等） |
| **内容协商** | 支持语言、编码、类型的协商（`Accept-Language`、`Accept-Encoding` 等） |
| **Host 头** | 引入 `Host` 头，支持同一 IP 托管多个域名（虚拟主机） |
| **范围请求** | 支持 `Range` 头，可请求资源的部分内容（断点续传、视频seek） |

### 请求示例

```http
GET /en-US/docs/ HTTP/1.1
Host: developer.mozilla.org
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.5
Connection: keep-alive
```

### 性能优化与限制

- **TCP 慢启动**：长连接比新建连接更快
- **队头阻塞（Head-of-Line Blocking）**：同一连接上的请求必须按顺序响应，后续请求需等待前面响应完成
- **浏览器并发连接数**：为绕过队头阻塞，浏览器通常允许对同一域名建立最多 6 个并行 TCP 连接

### 规范演进

| RFC | 发布时间 | 说明 |
|-----|----------|------|
| RFC 2068 | 1997-01 | HTTP/1.1 首次发布 |
| RFC 2616 | 1999-06 | 第一次重大修订 |
| RFC 7230-7235 | 2014-06 | 拆分为多个文档，细化语义 |
| RFC 9112 | 2022-06 | HTTP/1.1 最新版本 |

## HTTP/2 — 性能优化协议（2015）

HTTP/2 基于 Google 的 SPDY 协议，于 2015 年 5 月正式发布为 [RFC 7540](https://datatracker.ietf.org/doc/html/rfc7540)，后更新为 [RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113)。

### 核心特性

| 特性 | 说明 |
|------|------|
| **二进制协议** | 从文本协议改为二进制分帧层，机器解析更高效，但不可人工读写 |
| **多路复用** | 多个请求/响应可在同一 TCP 连接上并行传输，彻底解决 HTTP/1.x 的队头阻塞 |
| **头部压缩（HPACK）** | 使用 HPACK 算法压缩请求头，消除重复头部带来的带宽浪费 |
| **服务器推送** | 服务器可主动向客户端推送资源（如 CSS、JS），减少往返延迟 |
| **流优先级** | 客户端可为请求设置优先级，服务器据此优化资源发送顺序 |

### 与 HTTP/1.1 的对比

| 对比维度 | HTTP/1.1 | HTTP/2 |
|----------|----------|--------|
| 协议格式 | 文本 | 二进制 |
| 连接模型 | 多连接并行（通常6个） | 单连接多路复用 |
| 头部传输 | 明文，每次重复传输 | HPACK 压缩 |
| 队头阻塞 | 存在（应用层） | 不存在（应用层） |
| 服务器推送 | 不支持 | 支持 |

### 部署情况

- 对网站和应用程序**无需修改代码**即可使用
- 只需升级服务器和浏览器支持
- 2022 年 1 月达到峰值，约 46.9% 的网站使用

### 已知问题

- **TCP 层队头阻塞**：虽然解决了应用层队头阻塞，但底层 TCP 的丢包重传仍会阻塞所有流
- 在高丢包率的网络环境下，性能可能不如 HTTP/1.1

## HTTP/3 — 基于 QUIC 的协议（2022）

HTTP/3 于 2022 年 6 月正式发布为 [RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)，是 HTTP 的最新主要版本。

### 核心变革

| 特性 | 说明 |
|------|------|
| **基于 QUIC** | 底层传输从 TCP 改为 QUIC（基于 UDP） |
| **真正的多路复用** | 每个流独立进行丢包检测和重传，单流丢包不影响其他流 |
| **0-RTT 连接建立** | 支持 0-RTT 握手，显著降低首次连接延迟 |
| **连接迁移** | 基于 Connection ID 而非 IP:Port，网络切换（如 WiFi 切蜂窝）不断连 |
| **内置 TLS 1.3** | QUIC 强制加密，安全性内建 |

### 传输层对比

| 对比维度 | HTTP/1.1 & HTTP/2 | HTTP/3 |
|----------|-------------------|--------|
| 传输层 | TCP | QUIC（基于 UDP） |
| 队头阻塞 | TCP 丢包会阻塞所有流 | 流级别隔离，单流丢包只影响该流 |
| 握手延迟 | TCP 握手 + TLS 握手（通常 2-3 RTT） | QUIC 握手（通常 1 RTT，支持 0-RTT） |
| 连接迁移 | 不支持（IP/Port 变化需重建连接） | 支持（基于 Connection ID） |
| 加密 | 可选（HTTPS） | 强制（TLS 1.3 内置） |

### 部署情况

- 截至 2022 年 10 月，约 26% 的网站使用 HTTP/3
- 主流浏览器均已支持：Chrome、Edge、Firefox、Safari

## 版本对比总结

| 特性 | HTTP/0.9 | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|----------|----------|--------|--------|
| 发布年份 | 1991 | 1996 | 1997 | 2015 | 2022 |
| 传输层 | TCP | TCP | TCP | TCP | QUIC (UDP) |
| 协议格式 | 文本 | 文本 | 文本 | 二进制 | 二进制 |
| 连接模型 | 短连接 | 短连接 | 持久连接 | 多路复用 | 多路复用 |
| 头部压缩 | 无 | 无 | 无 | HPACK | QPACK |
| 服务器推送 | 无 | 无 | 无 | 支持 | 支持 |
| 队头阻塞 | 无 | 无 | 应用层存在 | TCP 层存在 | 无 |
| 加密 | 无 | 可选 | 可选 | 可选 | 强制 |
| 连接迁移 | 不支持 | 不支持 | 不支持 | 不支持 | 支持 |

## 安全传输演进

### SSL/TLS 的发展

- **1994 年底**：Netscape 创建 SSL（Secure Sockets Layer）加密层
- **SSL 2.0/3.0**：支持电子商务网站，加密消息并保证真实性
- **TLS**：SSL 最终被标准化为 TLS（Transport Layer Security）
- **HTTPS**：HTTP over TLS，成为现代 Web 的标准

### 为什么需要加密

- Web 从学术网络变为商业和公共平台
- 广告商、个人和犯罪者竞争获取用户数据
- 现代 Web 应用需要访问地址簿、邮件、位置等隐私信息
- TLS 从电子商务扩展到所有场景

## 扩展与衍生协议

### WebDAV

- 1996 年左右扩展 HTTP 支持远程文档编辑
- 衍生出 CardDAV（通讯录）、CalDAV（日历）等标准
- 需要服务器端支持才能使用

### REST

- 2000 年提出 Representational State Transfer 架构风格
- 基于标准 HTTP 方法（GET、POST、PUT、DELETE）和 URI 设计 API
- 无需修改浏览器或服务器即可实现数据交互
- 2010 年代成为 API 设计的主流范式

### WebSocket

- 通过 HTTP `Upgrade` 头将 HTTP 连接升级为 WebSocket
- 支持全双工通信，适用于实时应用

### Server-Sent Events (SSE)

- 服务器可向浏览器推送事件消息
- 基于 HTTP 长连接，单向推送（服务器 → 客户端）

### CORS

- Cross-Origin Resource Sharing，跨域资源共享
- 通过 HTTP 头解除同源策略限制
- 现代 Web 应用跨域访问的基础

## 常见误区

### 误区 1：HTTP/2 一定比 HTTP/1.1 快

- 在低丢包率网络下确实更快
- 在高丢包率网络下，TCP 层队头阻塞可能导致性能下降
- 需要根据实际网络环境评估

### 误区 2：HTTP/2 需要修改网站代码

- HTTP/2 对应用层透明
- 只需升级服务器和客户端支持即可
- 网站代码无需任何修改

### 误区 3：HTTP/3 已经全面普及

- 截至 2022 年 10 月，仅约 26% 的网站支持
- 需要服务器和客户端同时支持
- 部分网络环境（如某些企业防火墙）可能阻止 UDP 流量

### 误区 4：HTTP 是无状态协议，所以不能保持登录状态

- HTTP 本身确实无状态
- 但通过 Cookie、Session、Token 等机制可实现状态保持
- 这是应用层的设计，不是协议本身的限制

## 参考资料

- [MDN - Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP)
- [RFC 9110 - HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [RFC 9112 - HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc9112)
- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 1945 - HTTP/1.0](https://datatracker.ietf.org/doc/html/rfc1945)
- [HTTP/2 网站使用统计](https://w3techs.com/technologies/details/ce-http2)
- [HTTP/3 网站使用统计](https://w3techs.com/technologies/details/ce-http3)
- [HTTP/3 浏览器兼容性](https://caniuse.com/http3)
