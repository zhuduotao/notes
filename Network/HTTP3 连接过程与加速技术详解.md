---
created: 2026-04-15
updated: 2026-04-15
tags:
  - network
  - http3
  - quic
  - https
  - performance
  - tls
  - protocol
  - web-optimization
aliases:
  - HTTP/3 连接
  - HTTP/3 加速技术
  - QUIC 连接过程
  - HTTPv3
  - h3
  - 0-RTT
  - 连接迁移
source_type: mixed
source_urls:
  - 'https://datatracker.ietf.org/doc/html/rfc9114'
  - 'https://datatracker.ietf.org/doc/html/rfc9000'
  - 'https://datatracker.ietf.org/doc/html/rfc9001'
  - 'https://datatracker.ietf.org/doc/html/rfc9204'
  - 'https://w3techs.com/technologies/details/ce-http3'
status: verified
---

## 概述

HTTP/3 是 HTTP 协议的第三代主要版本，于 2022 年 6 月正式发布为 [RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)。与 HTTP/1.1 和 HTTP/2 基于 TCP 不同，HTTP/3 基于 **QUIC 传输协议**（[RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000)），QUIC 构建在 UDP 之上，内置 TLS 1.3 加密（[RFC 9001](https://datatracker.ietf.org/doc/html/rfc9001)）。

HTTP/3 的核心设计目标是**彻底解决 TCP 层队头阻塞**，同时提供**更快的连接建立**和**网络切换无缝衔接**能力。

协议标识符：`h3`（通过 TLS ALPN 协商）。

**本文档覆盖内容**：
- HTTP/3 连接建立过程（1-RTT 和 0-RTT）
- QUIC 核心机制（Connection ID、地址验证、连接迁移）
- HTTP/3 加速技术（多路复用、QPACK、拥塞控制等）
- 常见问题与解决方案（证书过大、UDP 阻塞、0-RTT 重放等）
- 部署最佳实践与调试方法

## HTTP/3 连接建立过程

HTTP/3 的连接建立与 HTTP/2 有本质区别：QUIC 将传输层握手和 TLS 握手合二为一，大幅减少往返延迟。

### 完整连接流程（1-RTT 首次连接）

```
客户端（UDP）                                      服务器（UDP）
  |                                                    |
  | 1. Initial 包（ClientHello + 传输参数 + key_share） |
  |---------------------------------------------------->|
  |                                                    |
  | 2. Server Initial（ServerHello + 证书 + key_share）  |
  |    Handshake 包（EncryptedExtensions,               |
  |                Certificate, CertificateVerify,      |
  |                Finished）                           |
  |<----------------------------------------------------|
  |                                                    |
  | 3. Client Handshake（Finished）                      |
  |    1-RTT 加密的应用数据（可立即发送）                  |
  |---------------------------------------------------->|
  |                                                    |
  | 4. 服务器 1-RTT 响应                                |
  |<====================================================|
  |                                                    |
  | 5. HTTP/3 SETTINGS 帧（通过专用 Control Stream）     |
  |<====================================================|
```

### 各阶段详解

#### 阶段 1：QUIC Initial 包（客户端发起）

客户端发送 UDP 数据包，包含：
- **Initial 包**：携带 TLS ClientHello，其中包含：
  - ALPN 扩展：声明支持 `h3`
  - SNI 扩展：指定目标域名
  - key_share：客户端的 ECDHE 公钥（通常使用 x25519 曲线）
  - QUIC 传输参数扩展：`initial_max_data`、`initial_max_stream_data`、`initial_max_streams_bidi` 等
- **地址验证 Token**（可选）：如果服务器之前通过 `NEW_TOKEN` 帧发送过 token，客户端可附带以跳过地址验证

此阶段消耗 **0.5 个 RTT**（客户端发送后即开始等待）。

#### 阶段 2：服务器响应（Initial + Handshake 包）

服务器回复：
- **Initial 包**：携带 TLS ServerHello，包含：
  - 选定的 ALPN 协议（`h3`）
  - 服务器的 ECDHE 公钥
- **Handshake 包**（加密）：
  - `EncryptedExtensions`：QUIC 传输参数等
  - `Certificate`：服务器证书链
  - `CertificateVerify`：证书签名验证
  - `Finished`：握手完整性校验

此时双方已协商出 1-RTT 加密密钥。

此阶段消耗 **0.5 个 RTT**（服务器回复）。

#### 阶段 3：客户端完成握手 + 发送应用数据

客户端发送：
- **Handshake 包**：TLS `Finished` 消息，确认握手完成
- **1-RTT 加密的 HTTP/3 请求**：客户端可以在发送 `Finished` 的同一个 UDP 数据报中直接携带 HTTP/3 请求数据

**这是 HTTP/3 相比 HTTP/2 的关键优势**：HTTP/2 over TLS 1.3 需要 2 个 RTT 才能发送应用数据，而 HTTP/3 只需 1 个 RTT。

#### 阶段 4：服务器响应

服务器用 1-RTT 密钥加密返回 HTTP/3 响应。

#### 阶段 5：HTTP/3 SETTINGS 交换

QUIC 连接建立后，HTTP/3 端点必须通过专用的 **Control Stream**（单向流）交换 `SETTINGS` 帧，这是 HTTP/3 连接正式可用的标志：
- 每个端点必须在自己的 Control Stream 上首先发送 `SETTINGS` 帧
- `SETTINGS` 帧配置 `MAX_FIELD_SECTION_SIZE`、`QPACK` 参数等

### 0-RTT 快速连接（会话恢复）

如果客户端之前与该服务器建立过连接，可以使用 **0-RTT（Zero Round Trip Time）** 模式：

```
客户端                                          服务器
  | 1. Initial 包（ClientHello + early_data）    |
  |    0-RTT 加密的 HTTP/3 请求（立即发送）        |
  |------------------------------------------------>|
  |                                                |
  | 2. 服务器接受 0-RTT → 直接处理请求并响应         |
  |    或拒绝 0-RTT → 回退到 1-RTT 握手             |
  |<================================================|
```

**0-RTT 工作原理**：
1. 首次连接完成后，服务器发送 `NewSessionTicket` 帧（通过 CRYPTO 帧传输）
2. 客户端保存会话票证（Session Ticket）和之前的 QUIC 传输参数
3. 后续连接时，客户端使用之前的密钥材料直接加密并发送应用数据（0-RTT 数据）
4. 服务器验证票证后决定是否接受 0-RTT 数据

**0-RTT 限制**：
- **不防重放攻击**：0-RTT 数据可能被攻击者捕获并重放，因此**只能用于幂等操作**（GET、HEAD 等）
- 服务器可以随时拒绝 0-RTT，客户端必须准备好回退到 1-RTT
- TLS 规范限制会话票证有效期最长 7 天

### 连接延迟对比

| 场景 | HTTP/2 (TLS 1.2) | HTTP/2 (TLS 1.3) | HTTP/3 |
|------|------------------|------------------|--------|
| 首次连接 | 3 RTT | 2 RTT | **1 RTT** |
| 会话恢复 | 1-2 RTT | 1 RTT | **0 RTT** |
| 网络切换 | 需重建连接（2-3 RTT） | 需重建连接（2-3 RTT） | **0 RTT（连接迁移）** |

## QUIC 连接 ID 与地址验证

### Connection ID（连接标识符）

QUIC 使用 **Connection ID** 而非 IP:Port 来标识连接，这是实现连接迁移的基础：

- 每个端点在握手期间选择多个 Connection ID 并通过 `NEW_CONNECTION_ID` 帧分发给对端
- 后续数据包使用 Connection ID 而非源/目标地址来匹配连接
- 当网络路径变化时（如 WiFi 切换到蜂窝网络），只需更换数据包中的 Connection ID，连接本身不中断

### 地址验证

为防止放大攻击（Amplification Attack），QUIC 在握手初期实施地址验证：

1. **Retry 包机制**：服务器可发送 `Retry` 包要求客户端证明其能接收来自服务器地址的数据
2. **Token 机制**：服务器在 `NEW_TOKEN` 帧中发送 token，客户端在后续连接中携带以跳过验证
3. **放大限制**：在验证客户端地址前，服务器发送的数据量不能超过接收数据量的 3 倍

## HTTP/3 加速技术

### 1. 真正的流级多路复用（无队头阻塞）

**原理**：QUIC 在传输层实现了流级别的多路复用和独立的丢包恢复。每个 HTTP/3 请求/响应对应一个独立的 QUIC 流，流之间的数据传输和丢包重传完全隔离。

**与 HTTP/2 的本质区别**：

| 维度 | HTTP/2 (TCP) | HTTP/3 (QUIC) |
|------|-------------|---------------|
| 多路复用层 | 应用层（HTTP/2 帧） | 传输层（QUIC 流） |
| 丢包影响 | TCP 丢包重传阻塞**所有流** | 仅影响**丢包所在的流** |
| 拥塞控制 | 连接级别，所有流共享 | 连接级别，但流级独立重传 |
| 高丢包率表现 | 性能急剧下降 | 保持稳定 |

**实际影响**：在 1%-2% 丢包率的移动网络下，HTTP/2 的页面加载时间可能增加 2-5 倍，而 HTTP/3 几乎不受影响。

### 2. 0-RTT 快速连接

如上文连接过程所述，HTTP/3 支持 0-RTT 会话恢复，客户端可以在第一个数据包中就携带应用数据。

**加速效果**：
- 首次访问：1 RTT（比 HTTP/2 + TLS 1.3 快 1 个 RTT）
- 回访用户：0 RTT（浏览器可直接发送请求，无需等待握手）

**适用场景**：
- 静态资源请求（CSS、JS、图片）
- API GET 请求
- 页面导航请求

**不适用场景**：
- POST/PUT/DELETE 等非幂等操作
- 支付、下单等敏感操作

### 3. 连接迁移（Connection Migration）

**原理**：基于 Connection ID 而非 IP:Port 标识连接，当客户端网络路径变化时，无需重建连接。

**典型场景**：
- WiFi 切换到蜂窝网络（4G/5G）
- 蜂窝网络在不同基站间切换
- NAT 重新绑定（IP 地址变化）
- 从有线网络切换到 WiFi

**工作流程**：
1. 客户端检测到网络路径变化
2. 在新路径上发送 `PATH_CHALLENGE` 帧探测新路径
3. 服务器回复 `PATH_RESPONSE` 帧确认
4. 路径验证通过后，后续数据包使用新路径传输
5. 连接状态（流状态、流控窗口、拥塞窗口）保持不变

**性能影响**：
- HTTP/2 在网络切换时需要重建 TCP + TLS 连接（2-3 RTT）
- HTTP/3 仅需路径验证（通常 1 RTT），且已有数据流不中断

### 4. QPACK 头部压缩

**原理**：HTTP/3 使用 QPACK（[RFC 9204](https://datatracker.ietf.org/doc/html/rfc9204)）替代 HTTP/2 的 HPACK，解决 QUIC 乱序交付问题。

**QPACK vs HPACK 的关键区别**：

| 维度 | HPACK (HTTP/2) | QPACK (HTTP/3) |
|------|---------------|----------------|
| 动态表更新 | 直接在头部块中更新 | 通过**独立的单向流**更新 |
| 乱序处理 | 不支持（TCP 保证有序） | 支持（通过引用计数和阻塞机制） |
| 队头阻塞风险 | 无 | 编码器可控制（权衡压缩率与延迟） |

**QPACK 架构**：
- **Control Stream**：传输 `SETTINGS` 等连接级配置
- **Encoder Stream**（单向）：编码器发送动态表更新指令
- **Decoder Stream**（单向）：解码器发送确认和反馈
- **Request/Response Streams**：携带引用动态表状态的编码头部块

**压缩效果**：与 HPACK 相当，典型请求头部可减少 85%-90% 的传输大小。

### 5. 改进的拥塞控制

QUIC 在传输层实现了更精细的拥塞控制：

- **每个连接的拥塞控制**：与 TCP 类似，但实现更灵活
- **更快的丢包检测**：基于更精确的 RTT 测量和 ACK 反馈
- **可插拔的拥塞算法**：QUIC 规范不强制特定算法，允许实现选择最优策略
- **ECN（显式拥塞通知）支持**：通过网络设备标记拥塞状态，提前调整发送速率

### 6. 减少中间件干扰

**问题**：TCP 中间件（NAT、防火墙、代理）经常错误地修改或丢弃 TCP 包，导致连接问题。

**QUIC 的解决方案**：
- 基于 UDP，绕过大多数 TCP 特定的中间件问题
- 头部保护（Header Protection）：加密包头部关键字段，防止中间件检查和修改
- 版本协商机制：支持未来协议演进而不被中间件阻断

## 支持情况

### 浏览器支持

| 浏览器 | 支持版本 | 备注 |
|--------|----------|------|
| Chrome | 87+ (2020) | 默认启用 |
| Firefox | 88+ (2021) | 默认启用 |
| Safari | 14+ (2020) | macOS 11+ / iOS 14+ |
| Edge | 87+ (2020) | 与 Chrome 同步 |
| Opera | 73+ (2020) | 与 Chrome 同步 |

### 服务器/CDN 支持

| 服务 | 支持情况 |
|------|----------|
| Cloudflare | 默认启用 HTTP/3 |
| Fastly | 支持 HTTP/3 |
| Nginx | 1.25+ 实验性支持（`--with-http_v3_module`） |
| Caddy | 2.6+ 默认启用 |
| AWS CloudFront | 支持 HTTP/3 |
| Google Cloud CDN | 支持 HTTP/3 |
| Node.js | 22+ 实验性支持（`--experimental-quic`） |

### 使用率统计

根据 [w3techs.com](https://w3techs.com/technologies/details/ce-http3) 数据：

- **HTTP/3 使用率**：截至 2024 年约 **26%-30%** 的网站支持 HTTP/3
- **增长趋势**：持续上升，主要由 Cloudflare、Google 等大型 CDN 推动
- **典型用户**：Google.com、YouTube.com、Cloudflare.com、Facebook.com 等

## 适用场景

### 最适合使用 HTTP/3 的场景

| 场景 | 原因 |
|------|------|
| **移动网络环境** | 网络切换频繁，QUIC 连接迁移无缝衔接 |
| **高丢包率网络** | 流级独立重传，不受 TCP 队头阻塞影响 |
| **高延迟网络** | 0-RTT 快速连接，减少握手延迟 |
| **实时应用** | 流级多路复用保证关键数据不被阻塞 |
| **全球分发 CDN** | 跨国网络丢包和延迟波动大，HTTP/3 优势明显 |
| **IoT 设备通信** | 网络不稳定场景下连接保持能力更强 |

### 不太适合或需要注意的场景

| 场景 | 注意事项 |
|------|----------|
| **企业防火墙环境** | 部分企业网络阻止 UDP 流量（尤其是非标准端口） |
| **UDP 被限速的网络** | 某些 ISP 对 UDP 流量限速，HTTP/3 性能可能下降 |
| **老旧服务器基础设施** | 需要升级操作系统和 Web 服务器支持 UDP/QUIC |
| **需要深度包检测的环境** | QUIC 加密头部，传统 DPI 设备无法检查内容 |

### HTTP/2 vs HTTP/3 选型建议

| 维度 | HTTP/2 | HTTP/3 |
|------|--------|--------|
| 网络稳定性好 | 足够使用 | 优势不明显 |
| 高丢包率（>1%） | 性能下降明显 | **显著优势** |
| 移动网络/网络切换 | 需重建连接 | **连接迁移，无缝切换** |
| 首次连接延迟 | TLS 1.3 下 2 RTT | **1 RTT** |
| 回访用户延迟 | 1 RTT（TLS 会话恢复） | **0 RTT** |
| 企业防火墙兼容性 | 广泛兼容（TCP 443） | 可能被阻止（UDP） |
| 部署复杂度 | 低 | 中（需 UDP 支持、内核调优） |
| 中间件兼容性 | 成熟 | 部分老旧设备可能有问题 |

## 最佳实践

### 服务器配置

```nginx
# Nginx HTTP/3 配置示例（1.25+）
server {
    listen 443 ssl;
    listen 443 quic reuseport;  # 启用 QUIC
    http2 on;                    # 同时支持 HTTP/2

    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # HTTP/3 Alt-Svc 公告
    add_header Alt-Svc 'h3=":443"; ma=86400';

    # QUIC 相关配置
    quic_retry on;
    quic_gso on;  # Generic Segmentation Offload，提升性能
}
```

### 同时提供 HTTP/2 和 HTTP/3

推荐同时启用 HTTP/2 和 HTTP/3，通过 `Alt-Svc` 头部公告 HTTP/3 可用性：

```http
Alt-Svc: h3=":443"; ma=86400
```

- `h3`：协议标识符
- `:443`：HTTP/3 监听的端口
- `ma=86400`：缓存时间（秒），客户端在此时间内会尝试使用 HTTP/3

### 性能优化

- **启用 GSO（Generic Segmentation Offload）**：减少内核态到用户态的数据拷贝
- **优化 UDP 接收缓冲区**：`net.core.rmem_max` 和 `net.core.rmem_default`
- **启用 QUIC 连接迁移**：确保服务器配置多个 Connection ID
- **监控 UDP 丢包率**：使用 `netstat -su` 或 `ss -su` 监控

### 调试与监控

- Chrome DevTools Network 面板：Protocol 列显示 `h3` 表示使用 HTTP/3
- `curl --http3-only https://example.com`：强制使用 HTTP/3 测试
- `quic-go` 的 `quic_trace` 工具：分析 QUIC 连接细节
- Cloudflare `cf-ray` 头部：确认请求是否通过 HTTP/3 处理

## 常见问题 (FAQ)

### Q1: 证书过大导致握手延迟增加怎么办？

**问题描述**：QUIC 握手时，服务器证书通过 Handshake 包传输。如果证书链过大（如包含多级中间 CA、ECC + RSA 双证书、OCSP Stapling 响应等），会导致：
- Handshake 包超过 UDP MTU（通常 1200-1400 字节），需要分片
- 多个 Handshake 包增加 RTT 和丢包风险
- 客户端需要更多时间验证证书链

**解决方案**：
1. **精简证书链**：只发送必要的中间证书，移除冗余的根证书（客户端通常已内置）
2. **使用 ECC 证书**：ECDSA/P-256 证书比 RSA 2048 小约 50%（~200 bytes vs ~400 bytes）
3. **启用 OCSP Stapling**：将 OCSP 响应内联到证书中，避免客户端额外查询
4. **证书压缩**：部分实现支持证书压缩扩展（RFC 8879），可减少 30%-50% 大小
5. **会话复用**：通过 0-RTT/1-RTT 会话恢复避免重复传输证书

**典型证书大小对比**：
| 配置 | 证书链大小 | 传输包数 |
|------|-----------|---------|
| RSA 2048 + 2 级中间 CA | ~3.5 KB | 3-4 个 Handshake 包 |
| ECDSA P-256 + 1 级中间 CA | ~1.2 KB | 1-2 个 Handshake 包 |
| 上述 + 证书压缩 | ~0.6 KB | 1 个 Handshake 包 |

### Q2: UDP 被防火墙或 ISP 阻止怎么办？

**问题描述**：部分企业防火墙、公共 WiFi 或 ISP 会阻止或限速 UDP 流量（尤其是非标准端口），导致 HTTP/3 连接失败。

**解决方案**：
1. **使用标准端口 443**：大多数防火墙允许 UDP 443 端口
2. **启用 Happy Eyeballs 算法**：客户端同时尝试 HTTP/3 和 HTTP/2，选择先成功的
3. **Alt-Svc 回退机制**：通过 HTTP/2 响应 `Alt-Svc` 头部公告 HTTP/3，客户端自动尝试并回退
4. **连接迁移到 TCP**：部分实现支持 QUIC over TCP 回退（非标准，需自定义）
5. **监控回退率**：记录 HTTP/3 回退到 HTTP/2 的比例，评估网络环境问题

**检测 UDP 阻塞的方法**：
```bash
# 测试 UDP 443 连通性
curl -v --http3 https://example.com

# 检查是否回退到 HTTP/2
curl -s -o /dev/null -w "%{http_version}\n" --http3 https://example.com
# 返回 "3" 表示 HTTP/3，返回 "2" 表示回退
```

### Q3: 0-RTT 重放攻击如何防护？

**问题描述**：0-RTT 数据使用之前的密钥材料加密，攻击者可以捕获并重放这些数据，导致非幂等操作被重复执行。

**防护机制**：
1. **应用层限制**：0-RTT 只允许 GET、HEAD 等幂等方法，服务器拒绝其他方法
2. **时间窗口限制**：会话票证有效期最长 7 天（TLS 规范），过期后强制 1-RTT
3. **单用票证（Single-Use Tickets）**：服务器为每个票证记录是否已使用，拒绝重复使用
4. **客户端 Hello 绑定**：将 0-RTT 数据与新的 ClientHello 绑定，增加重放难度
5. **速率限制**：对 0-RTT 请求实施更严格的速率限制
6. **自定义重放保护**：添加时间戳、nonce 或签名到 0-RTT 数据中

**服务器配置示例（Nginx）**：
```nginx
# 限制 0-RTT 只读操作
if ($request_method !~ ^(GET|HEAD|OPTIONS)$) {
    set $reject_0rtt 1;
}
```

### Q4: 连接迁移失败或延迟高怎么办？

**问题描述**：网络切换时，连接迁移可能失败或需要较长时间验证新路径。

**常见原因**：
- 服务器未配置足够的 Connection ID
- 新路径 NAT 映射未建立
- 路径验证丢包

**解决方案**：
1. **预分配 Connection ID**：服务器在握手时发送至少 4-8 个 Connection ID
2. **主动路径探测**：客户端在网络切换前发送 `PATH_CHALLENGE` 预探测
3. **多路径 QUIC（MP-QUIC）**：同时维护多条路径，切换时零延迟（RFC 9369，实验性）
4. **调整 PATH_CHALLENGE 重试次数**：默认 3 次，可增加至 5-6 次提高成功率
5. **保持活跃探测**：定期发送 PING 帧保持 NAT 映射活跃

### Q5: QUIC 版本协商失败如何处理？

**问题描述**：客户端和服务器支持的 QUIC 版本不匹配，导致连接失败。

**版本协商流程**：
1. 客户端使用最高支持版本发送 Initial 包
2. 服务器如果不支持该版本，返回 `Version Negotiation` 包（包含支持的版本列表）
3. 客户端选择共同支持的最高版本重新连接

**解决方案**：
1. **服务器配置多版本支持**：同时支持 QUIC v1（RFC 9000）和 draft 版本
2. **版本锁定**：生产环境锁定稳定版本，避免自动升级导致不兼容
3. **监控版本分布**：记录客户端 QUIC 版本分布，指导升级策略
4. **灰度升级**：先在部分节点升级新版本，观察兼容性后再全量

### Q6: HTTP/3 内存和 CPU 开销比 HTTP/2 高吗？

**问题描述**：QUIC 在用户态实现（而非内核态），可能带来额外的资源开销。

**开销对比**：
| 资源 | HTTP/2 (TCP) | HTTP/3 (QUIC) | 差异 |
|------|-------------|---------------|------|
| 内存/连接 | ~50-100 KB | ~100-200 KB | +2x |
| CPU/连接 | 低（内核优化） | 中（用户态加解密） | +30%-50% |
| 上下文切换 | 少（内核态） | 多（用户态） | +20% |

**优化建议**：
1. **启用硬件加速**：使用 AES-NI、SHA 扩展加速 TLS 加解密
2. **连接复用**：减少连接数量，提高单连接利用率
3. **调整流限制**：合理设置 `initial_max_streams_bidi`，避免过多并发流
4. **使用 GSO/GRO**：启用 Generic Segmentation/Receive Offload 减少系统调用
5. **监控资源使用**：定期分析内存和 CPU 分布，识别瓶颈

### Q7: 中间件（NAT/防火墙/代理）干扰如何处理？

**问题描述**：部分中间件会修改或丢弃 UDP 包，导致 QUIC 连接异常。

**常见问题**：
- NAT 超时时间短（<30 秒），导致连接被意外关闭
- 防火墙修改 UDP 头部，破坏 QUIC 包完整性
- 代理服务器缓存或修改 Alt-Svc 头部

**解决方案**：
1. **保持活跃（Keep-Alive）**：定期发送 PING 帧（建议 15-20 秒间隔）
2. **头部保护**：QUIC 默认加密包头部关键字段，防止中间件篡改
3. **端口固定**：使用 443 端口，避免被识别为非标准 UDP 流量
4. **Fallback 机制**：检测到中间件干扰时自动回退到 HTTP/2
5. **与网络管理员协调**：在企业环境配置 UDP 443 白名单

### Q8: HTTP/3 与 HTTP/2 共存时如何选择？

**问题描述**：服务器同时支持 HTTP/2 和 HTTP/3，客户端如何选择最优协议？

**选择策略**：
1. **Alt-Svc 公告**：服务器通过 HTTP/2 响应 `Alt-Svc: h3=":443"; ma=86400` 公告 HTTP/3
2. **DNS HTTPS 记录**：通过 DNS `HTTPS` 记录（RFC 9113）提前公告 HTTP/3 支持
3. **浏览器策略**：Chrome/Firefox 优先尝试 HTTP/3，失败后回退到 HTTP/2
4. **服务器优先级**：部分服务器可配置协议优先级（如 Nginx `http3_priority`）

**推荐配置**：
```nginx
# 同时启用 HTTP/2 和 HTTP/3
listen 443 ssl;
listen 443 quic reuseport;
http2 on;

# Alt-Svc 公告
add_header Alt-Svc 'h3=":443"; ma=86400';

# DNS HTTPS 记录（在 DNS 服务器配置）
example.com. IN HTTPS 1 . alpn="h3,h2" ipv4hint=1.2.3.4
```

## 常见误区

### 误区 1：HTTP/3 一定比 HTTP/2 快

- 在稳定、低丢包率的有线网络下，差异不明显
- HTTP/3 的优势主要体现在高丢包率、移动网络和网络切换场景
- 首次连接时，如果 UDP 被防火墙阻止，回退到 TCP 反而增加延迟

### 误区 2：HTTP/3 不需要 HTTPS

- QUIC 内置 TLS 1.3 加密，**加密是强制性的**
- 不存在明文 HTTP/3 的概念
- 所有 HTTP/3 通信都是加密的

### 误区 3：0-RTT 是安全的

- 0-RTT 数据**不防重放攻击**
- 只能用于幂等操作（GET、HEAD 等）
- 服务器必须实现重放保护机制

### 误区 4：HTTP/3 已经全面普及

- 截至 2024 年，约 26%-30% 的网站支持 HTTP/3
- 企业防火墙和某些 ISP 仍可能阻止 UDP 流量
- 客户端需要同时支持 HTTP/2 和 HTTP/3 以保证兼容性

### 误区 5：QUIC 就是 UDP 版的 TCP

- QUIC 不仅仅是"UDP + 可靠性"，它是一个全新的传输协议
- QUIC 内置 TLS、流级多路复用、连接迁移等 TCP 不具备的特性
- QUIC 的拥塞控制和丢包恢复机制与 TCP 有本质区别

## 相关概念

- **QUIC**：基于 UDP 的多路复用安全传输协议（RFC 9000）
- **QPACK**：HTTP/3 的头部压缩算法（RFC 9204）
- **Alt-Svc**：HTTP 替代服务公告机制（RFC 7838）
- **0-RTT**：零往返时间快速连接建立
- **连接迁移**：QUIC 特有的网络路径切换能力
- **gRPC over HTTP/3**：正在演进中的 RPC 框架支持

## 参考资料

- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 9000 - QUIC Transport Protocol](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 9001 - Using TLS to Secure QUIC](https://datatracker.ietf.org/doc/html/rfc9001)
- [RFC 9204 - QPACK: Field Compression for HTTP/3](https://datatracker.ietf.org/doc/html/rfc9204)
- [w3techs - HTTP/3 Usage Statistics](https://w3techs.com/technologies/details/ce-http3)
- [Cloudflare - HTTP/3](https://www.cloudflare.com/learning/performance/what-is-http3/)
- [MDN - Evolution of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Evolution_of_HTTP)
