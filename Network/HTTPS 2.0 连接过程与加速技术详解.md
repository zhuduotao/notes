---
created: 2026-04-15
updated: 2026-04-15
tags:
  - network
  - http2
  - https
  - performance
  - tls
  - web-optimization
  - protocol
aliases:
  - HTTP/2 连接
  - HTTPS 2.0 连接过程
  - HTTP/2 加速
  - HTTP/2 多路复用
  - HTTP/2 使用率
  - h2
source_type: mixed
source_urls:
  - 'https://datatracker.ietf.org/doc/html/rfc9113'
  - 'https://datatracker.ietf.org/doc/html/rfc8446'
  - 'https://datatracker.ietf.org/doc/html/rfc7301'
  - 'https://datatracker.ietf.org/doc/html/rfc7541'
  - 'https://developers.google.com/web/fundamentals/performance/http2'
  - 'https://w3techs.com/technologies/details/ce-http2'
  - 'https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP'
status: verified
---

## 概述

HTTPS 2.0（即 HTTP/2 over TLS）是 HTTP 协议的第二代主要版本，于 2015 年 5 月首次发布为 [RFC 7540](https://datatracker.ietf.org/doc/html/rfc7540)，2022 年 6 月更新为 [RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113)。HTTP/2 基于 Google 的 SPDY 协议演进而来，核心目标是**降低延迟、提高网络资源利用率**，同时保持与 HTTP/1.1 应用语义的完全兼容。

HTTPS 2.0 = HTTP/2 + TLS，其中 TLS 是强制性的传输层加密。主流浏览器（Chrome、Firefox、Safari、Edge）仅支持通过 TLS 协商的 HTTP/2（标识符为 `h2`）。

## HTTPS 2.0 连接建立过程

HTTPS 2.0 的连接建立是一个多阶段过程，涉及 TCP 握手、TLS 协商和 HTTP/2 连接前缀交换。

### 完整连接流程（TLS 1.2）

```
客户端                                          服务器
  |                                                |
  | 1. TCP 三次握手（SYN, SYN-ACK, ACK）            |
  |------------------------------------------------>|
  |                                                |
  | 2. TLS 握手（ClientHello + ALPN: "h2"）         |
  |------------------------------------------------>|
  |                    ServerHello + ALPN: "h2"      |
  |                    Certificate                 |
  |                    ServerKeyExchange           |
  |                    ServerHelloDone             |
  |<------------------------------------------------|
  |                                                |
  | 3. 客户端密钥交换、验证证书、完成 TLS             |
  |------------------------------------------------>|
  |                    ChangeCipherSpec            |
  |                    Finished                    |
  |<------------------------------------------------|
  |                                                |
  | 4. HTTP/2 连接前缀（Connection Preface）         |
  |------------------------------------------------>|
  |  PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n              |
  |  SETTINGS frame                                |
  |                                                |
  |                    SETTINGS frame              |
  |<------------------------------------------------|
  |                                                |
  | 5. SETTINGS ACK                                |
  |------------------------------------------------>|
  |                    SETTINGS ACK                |
  |<------------------------------------------------|
  |                                                |
  | 6. 开始交换 HTTP/2 帧（多路复用）                |
  |<================================================>|
```

### 各阶段详解

#### 阶段 1：TCP 三次握手

- 客户端向服务器 443 端口发起 TCP 连接
- 消耗 **1 个 RTT**（Round Trip Time）

#### 阶段 2-3：TLS 握手与 ALPN 协商

这是 HTTPS 2.0 连接的核心安全阶段：

1. **ClientHello**：客户端发送支持的 TLS 版本、密码套件列表，并在 **ALPN（Application-Layer Protocol Negotiation）扩展**中声明支持 `h2`（HTTP/2 over TLS）和 `http/1.1`
2. **ServerHello**：服务器选择 TLS 参数，并在 ALPN 扩展中返回选定的协议（`h2` 表示使用 HTTP/2）
3. **证书交换**：服务器发送 X.509 证书链，客户端验证证书有效性
4. **密钥交换**：双方协商共享密钥（ECDHE 等），建立加密通道

TLS 1.2 握手消耗 **2 个 RTT**。

#### 阶段 4：HTTP/2 连接前缀（Connection Preface）

TLS 握手完成后，双方必须交换 HTTP/2 连接前缀以确认协议：

**客户端前缀**：
```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```
这是一个 24 字节的魔数序列（`0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a`），后跟一个 `SETTINGS` 帧。该序列专门设计为使不支持 HTTP/2 的 HTTP/1.1 服务器不会尝试处理后续数据。

**服务器前缀**：
- 服务器必须发送一个 `SETTINGS` 帧（可以为空）作为其连接前缀

#### 阶段 5：SETTINGS 确认

双方收到对方的 `SETTINGS` 帧后，必须发送 `SETTINGS ACK` 确认。至此，HTTP/2 连接正式建立，可以开始多路复用通信。

### TLS 1.3 优化后的连接流程

使用 TLS 1.3（[RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)）时，握手过程显著简化：

```
客户端                                          服务器
  | 1. TCP 握手（1 RTT）                          |
  | 2. TLS 1.3 握手（ClientHello + key_share）    |
  |------------------------------------------------>|
  |                    ServerHello + key_share     |
  |                    EncryptedExtensions         |
  |                    Certificate                 |
  |                    CertificateVerify           |
  |                    Finished                    |
  |<------------------------------------------------|
  | 3. 客户端 Finished                              |
  |------------------------------------------------>|
  | 4. HTTP/2 连接前缀 + 数据（可立即发送）           |
  |<================================================>|
```

TLS 1.3 握手仅需 **1 个 RTT**（TLS 1.2 为 2 个 RTT），且支持 **0-RTT 恢复连接**（会话恢复时零往返延迟）。

### 连接延迟总结

| 场景 | TCP 握手 | TLS 握手 | HTTP/2 前缀 | 总 RTT |
|------|----------|----------|-------------|--------|
| TLS 1.2 首次连接 | 1 RTT | 2 RTT | 0 RTT（与应用数据并行） | **3 RTT** |
| TLS 1.3 首次连接 | 1 RTT | 1 RTT | 0 RTT（与应用数据并行） | **2 RTT** |
| TLS 1.3 会话恢复 | 1 RTT | 0 RTT（0-RTT） | 0 RTT | **1 RTT** |

## 加速连接的核心技术

HTTP/2 引入了多项加速技术，从不同层面优化连接性能。

### 1. 多路复用（Multiplexing）

**原理**：通过二进制分帧层，多个 HTTP 请求/响应可以在**同一个 TCP 连接上并行传输**，每个请求/响应分配独立的流（Stream），帧可以交错发送和接收。

**解决的问题**：
- HTTP/1.1 的队头阻塞（Head-of-Line Blocking）：同一连接上的请求必须按顺序响应
- 浏览器并发连接数限制（通常对同一域名最多 6 个连接）

**性能影响**：
- 单连接承载所有请求，减少 TCP 连接建立开销
- 消除应用层队头阻塞，请求不再互相等待
- 根据 Mozilla 测试，HTTP/2 连接的 74% 仅承载单个 HTTP 事务（HTTP/1.1 下这一比例为 25%），说明连接复用效率大幅提升

**限制**：
- **TCP 层队头阻塞仍然存在**：底层 TCP 丢包重传会阻塞该连接上的所有流
- 高丢包率网络环境下，性能可能不如 HTTP/1.1 的多连接策略

### 2. HPACK 头部压缩

**原理**：使用 HPACK 算法（[RFC 7541](https://datatracker.ietf.org/doc/html/rfc7541)）压缩 HTTP 请求头和响应头，包含两种技术：

1. **Huffman 编码**：对传输的头部字段值进行熵编码压缩
2. **静态/动态表**：客户端和服务器维护共享的索引表
   - 静态表：规范预定义的常见头部（如 `:method`、`:path`、`content-type` 等）
   - 动态表：连接运行过程中累积的自定义头部

**解决的问题**：
- HTTP/1.1 每次请求都明文传输完整头部，通常带来 500-800 字节开销，使用 Cookie 时可达数 KB
- 在低带宽 DSL 连接上，头部压缩可减少 45-1142ms 的页面加载时间

**压缩效果**：早期 SPDY 使用 zlib 压缩可减少 85%-88% 的头部传输大小。HPACK 在保持安全性的同时实现了类似的压缩率。

**安全考虑**：早期的 zlib 压缩方案受到 CRIME 攻击影响，HPACK 专门设计来避免此类安全问题。

### 3. 流优先级（Stream Prioritization）

**原理**：客户端可以为每个流分配：
- **权重（Weight）**：1-256 的整数值
- **依赖关系（Dependency）**：指定父流，形成优先级树

**工作方式**：
- 共享同一父流的子流按权重比例分配资源
- 有依赖关系的流，父流优先获得资源

**示例**：
- 浏览器可以为 HTML 文档分配最高优先级
- CSS 和阻塞型 JavaScript 次之
- 图片和非关键资源最低

**注意**：RFC 9113 已**弃用**了 RFC 7540 中的优先级信令方案，因为实际部署中发现浏览器和服务器的优先级实现差异较大，效果不如预期。

### 4. 服务器推送（Server Push）

**原理**：服务器可以在客户端请求之前，主动推送它预期客户端需要的资源。通过 `PUSH_PROMISE` 帧告知客户端即将推送的资源。

**典型场景**：
- 服务器返回 HTML 文档时，同时推送关联的 CSS、JS、字体文件
- 替代 HTTP/1.1 时代的资源内联（Data URI）做法

**优势**：
- 推送的资源可以被客户端缓存
- 推送的资源可以跨页面复用
- 推送的资源可以独立多路复用和设置优先级
- 客户端可以拒绝不需要的推送（通过 `RST_STREAM` 帧）

**现状**：
- 由于缓存失效、带宽浪费等问题，**Chrome 已默认禁用服务器推送**（Chrome 106+）
- 现代替代方案：`<link rel="preload">`、`Early Hints (103)` 响应
- 服务器推送在 HTTP/3 中已被完全移除

### 5. 单连接持久化

**原理**：HTTP/2 默认使用持久连接，一个 TCP 连接服务于同一域名的所有请求。

**优势**：
- 减少 TCP 连接建立和 TLS 握手次数
- 更好的 TCP 拥塞控制（单个长连接比多个短连接更高效）
- 减少客户端和服务器的内存和处理开销
- 更好的 HPACK 压缩效果（共享压缩上下文）

### 6. 二进制分帧层

**原理**：HTTP/2 将所有通信拆分为小的二进制帧，每种帧类型有特定用途：

| 帧类型 | 用途 |
|--------|------|
| `DATA` | 传输 HTTP 消息体 |
| `HEADERS` | 传输请求/响应头部 |
| `SETTINGS` | 交换连接配置参数 |
| `WINDOW_UPDATE` | 流控信用更新 |
| `PING` | 连接保活和延迟测量 |
| `GOAWAY` | 优雅关闭连接 |
| `PUSH_PROMISE` | 服务器推送预告 |
| `RST_STREAM` | 终止单个流 |
| `CONTINUATION` |  continuation 头部块片段 |

**优势**：
- 固定 9 字节帧头，解析高效
- 二进制格式比文本格式更紧凑、更健壮
- 支持帧级别的流控和错误处理

### 7. 流控（Flow Control）

**原理**：HTTP/2 在应用层实现了流级别的流量控制，接收方通过 `WINDOW_UPDATE` 帧告知发送方可以发送多少数据。

**与 TCP 流控的区别**：
- TCP 流控是连接级别的，不够精细
- HTTP/2 流控是流级别的，可以针对单个请求/响应独立控制
- 支持 hop-by-hop（逐跳），中间代理可以独立控制

**应用场景**：
- 用户暂停视频播放时，客户端可以减少该流的窗口大小，暂停数据传输
- 浏览器可以先获取图片预览，暂停完整下载，优先加载关键资源

## 支持情况

### 浏览器支持

所有主流浏览器均已支持 HTTP/2 over TLS（`h2`）：

| 浏览器 | 支持版本 | 备注 |
|--------|----------|------|
| Chrome | 41+ (2015) | 仅支持 TLS 的 HTTP/2 |
| Firefox | 36+ (2015) | 仅支持 TLS 的 HTTP/2 |
| Safari | 9+ (2015) | macOS 10.11+ / iOS 9+ |
| Edge | 12+ (2015) | 全版本支持 |
| Opera | 28+ (2015) | 与 Chrome 同步支持 |

**注意**：
- 所有主流浏览器**仅通过 TLS 启用 HTTP/2**，不支持明文 HTTP/2（`h2c`）
- SPDY 协议已于 2016 年在 Chrome 中移除，被 HTTP/2 完全取代

### 服务器支持

| 服务器 | 支持情况 |
|--------|----------|
| Nginx | 1.9.5+ 支持 HTTP/2 |
| Apache | 2.4.17+ 支持 HTTP/2（mod_http2） |
| Caddy | 默认启用 HTTP/2 |
| IIS | 10+ 支持 HTTP/2 |
| Node.js | 8.4.0+ 内置 HTTP/2 支持 |
| Cloudflare | 默认启用 HTTP/2 |
| AWS ALB/CloudFront | 支持 HTTP/2 |

### 协议标识符

| 标识符 | 含义 | 使用场景 |
|--------|------|----------|
| `h2` | HTTP/2 over TLS | 通过 TLS ALPN 协商，**主流使用方式** |
| `h2c` | HTTP/2 over TCP（明文） | 通过 HTTP Upgrade 机制，**已被 RFC 9113 弃用** |

## 使用率统计

根据 [w3techs.com](https://w3techs.com/technologies/details/ce-http2) 2026 年 4 月的数据：

- **HTTP/2 使用率**：约 **34.8%** 的网站使用 HTTP/2
- **历史趋势**：2022 年 1 月达到峰值约 46.9%，此后部分网站迁移至 HTTP/3，使用率有所下降
- **典型用户**：Google.com、Microsoft.com、YouTube.com、Apple.com、Cloudflare.com、Amazon.com、LinkedIn.com 等

**使用率下降原因**：
- 部分大型网站已迁移至 HTTP/3（基于 QUIC）
- HTTP/3 提供了更好的性能和抗丢包能力
- 但 HTTP/2 仍然是当前 Web 的主力协议，HTTP/3 截至 2022 年 10 月仅约 26% 的网站支持

## 适用场景

### 最适合使用 HTTP/2 的场景

| 场景 | 原因 |
|------|------|
| **资源密集型网站** | 多路复用允许并行加载 CSS、JS、图片等资源，无需域名分片 |
| **高延迟网络** | 单连接减少握手开销，HPACK 压缩减少传输量 |
| **API 服务** | 多个 API 调用可复用同一连接，降低延迟 |
| **移动端应用** | 移动网络延迟高、带宽不稳定，HTTP/2 的流控和多路复用优势明显 |
| **HTTPS 站点** | TLS 连接建立成本高，单连接复用最大化 TLS 会话价值 |
| **微服务间通信** | gRPC 默认使用 HTTP/2，利用其多路复用和流式传输能力 |

### 不太适合或需要注意的场景

| 场景 | 注意事项 |
|------|----------|
| **高丢包率网络** | TCP 层队头阻塞可能导致性能不如 HTTP/1.1，建议考虑 HTTP/3 |
| **大量小文件传输** | 二进制帧的 9 字节头部开销在极小资源上占比明显 |
| **需要服务器推送** | Chrome 已默认禁用，建议使用 `<link rel="preload">` 替代 |
| **CDN 边缘节点不支持** | 部分老旧 CDN 可能不支持 HTTP/2 回源 |

### HTTP/2 vs HTTP/1.1 选型建议

| 维度 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 资源数量少的简单页面 | 差异不大 | 略优 |
| 资源数量多的复杂页面 | 需要域名分片、雪碧图等优化 | **显著优势**，无需额外优化 |
| 高延迟网络（移动端） | 多连接开销大 | **显著优势** |
| 高丢包率网络 | 多连接有一定容错 | 可能更差（TCP 队头阻塞） |
| API 服务 | 连接复用有限 | **显著优势**（gRPC 默认使用） |

### HTTP/2 vs HTTP/3 选型建议

| 维度 | HTTP/2 | HTTP/3 |
|------|--------|--------|
| 网络稳定性好 | 足够使用 | 优势不明显 |
| 高丢包率/移动网络 | TCP 队头阻塞 | **显著优势**（QUIC 流级隔离） |
| 网络切换频繁（WiFi ↔ 蜂窝） | 需重建连接 | **支持连接迁移** |
| 首次连接延迟 | TLS 1.3 下 2 RTT | 1 RTT（支持 0-RTT） |
| 企业防火墙环境 | 广泛兼容（TCP 443） | 可能被阻止（UDP） |
| 部署复杂度 | 低 | 中（需 UDP 支持） |

## 最佳实践

### 服务器配置

```nginx
# Nginx HTTP/2 配置示例
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # 优先启用 TLS 1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # 推荐密码套件
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # 启用 OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
}
```

### 迁移 HTTP/2 时的优化调整

**可以移除的 HTTP/1.1 优化**：
- ❌ 域名分片（Domain Sharding）— HTTP/2 单连接多路复用已解决
- ❌ 资源合并/雪碧图（CSS/JS 合并、Image Sprites）— 反而不利于缓存和优先级
- ❌ 资源内联（Data URI）— 使用服务器推送或 `<link rel="preload">` 替代

**应保留的优化**：
- ✅ 资源压缩（Gzip/Brotli）
- ✅ 缓存策略（Cache-Control、ETag）
- ✅ CDN 分发
- ✅ 关键渲染路径优化

### 性能监控

- 使用浏览器 DevTools 的 Network 面板查看协议版本（Protocol 列显示 `h2`）
- 使用 [WebPageTest](https://www.webpagetest.org/) 分析 HTTP/2 性能
- 使用 `curl --http2 -v https://example.com` 测试服务器 HTTP/2 支持

## 常见误区

### 误区 1：HTTP/2 一定比 HTTP/1.1 快

- 在低丢包率、稳定网络下确实更快
- 在高丢包率网络下，TCP 层队头阻塞可能导致性能下降
- 对于资源很少的简单页面，差异不明显

### 误区 2：HTTP/2 需要修改网站代码

- HTTP/2 对应用层完全透明
- 所有 HTTP 语义（方法、状态码、URI、头部）保持不变
- 只需升级服务器和客户端支持

### 误区 3：服务器推送是银弹

- 推送不需要的资源会浪费带宽
- 客户端已有缓存的资源被重复推送
- Chrome 已默认禁用，现代替代方案更可控

### 误区 4：HTTP/2 不需要 HTTPS

- RFC 9113 允许明文 HTTP/2（`h2c`），但已被弃用
- 所有主流浏览器仅支持 TLS 的 HTTP/2
- 实际部署中 HTTPS 是 HTTP/2 的前提

### 误区 5：HTTP/2 的优先级能保证资源按顺序传输

- 优先级只是**建议**，不是强制要求
- 服务器可以自行决定如何处理优先级
- 不同服务器实现的优先级策略差异很大

## 相关概念

- **HTTP/3**：基于 QUIC（UDP）的下一代 HTTP 协议，解决 TCP 层队头阻塞
- **gRPC**：基于 HTTP/2 的 RPC 框架，充分利用多路复用和流式传输
- **ALPN**：TLS 扩展，用于在 TLS 握手期间协商应用层协议
- **HPACK**：HTTP/2 头部压缩算法
- **QUIC**：基于 UDP 的传输协议，HTTP/3 的底层传输

## 参考资料

- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 7301 - TLS ALPN Extension](https://datatracker.ietf.org/doc/html/rfc7301)
- [RFC 7541 - HPACK Compression](https://datatracker.ietf.org/doc/html/rfc7541)
- [High Performance Browser Networking - HTTP/2](https://developers.google.com/web/fundamentals/performance/http2)
- [w3techs - HTTP/2 Usage Statistics](https://w3techs.com/technologies/details/ce-http2)
- [MDN - Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP)
