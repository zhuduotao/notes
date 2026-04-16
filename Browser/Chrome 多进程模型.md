
## 是什么

Chrome 多进程模型（Multi-process Architecture）是 Chromium 浏览器的核心架构设计，将浏览器拆分为多个独立进程，每个进程承担不同职责，通过进程间通信（IPC）协同工作。该架构将操作系统级别的进程隔离、内存保护和访问控制引入浏览器，显著提升了浏览器的稳定性、安全性和性能。

## 为什么重要

在 2006 年前后，浏览器普遍采用单进程模型，所有标签页、插件和浏览器 UI 运行在同一进程中。这种设计存在三个致命问题：

- **稳定性差**：一个标签页的渲染引擎崩溃或插件异常会导致整个浏览器崩溃
- **安全风险高**：恶意网页可利用渲染引擎漏洞访问文件系统、设备或其他标签页数据
- **性能受限**：单进程无法充分利用多核 CPU，标签页之间相互阻塞

Chrome 多进程架构借鉴了现代操作系统的进程隔离思想，将"一个应用崩溃不影响其他应用"的理念引入浏览器。

## 核心进程类型

### Browser Process（浏览器进程）

主进程，负责：

- 运行浏览器 UI（地址栏、书签栏、菜单等）
- 管理所有其他进程的创建、销毁和生命周期
- 处理网络请求、文件访问、Cookie 管理等特权操作
- 维护 `RenderProcessHost` 对象，作为与每个渲染进程通信的浏览器端代理
- 执行安全策略和站点隔离决策

每个 Chrome 实例有且仅有一个浏览器进程。

### Renderer Process（渲染进程）

负责：

- 使用 Blink 渲染引擎解析 HTML、CSS，执行 JavaScript
- 布局和渲染网页内容
- 处理用户交互（点击、输入等）
- 维护 `RenderProcess` 对象（每个渲染进程一个）和 `RenderFrame` 对象（每个文档/帧一个）

渲染进程运行在沙箱中，对文件系统、网络和设备的访问受到严格限制。

### GPU Process（GPU 进程）

- 负责 GPU 加速的图形渲染
- 将渲染进程的绘图命令转换为 GPU 指令
- 隔离 GPU 驱动崩溃对整个浏览器的影响

### Network Service（网络服务进程）

- 统一处理所有网络请求（HTTP、HTTPS、WebSocket、FTP、data URI 等）
- 渲染进程只能通过该服务访问网络，不能直接发起网络请求
- 集中管理连接池、HTTP 缓存、Cookie、代理配置等网络资源
- 详见下方 [Network Service 详解](#network-service-详解) 小节

### Storage Service（存储服务进程）

- 管理本地存储（LocalStorage、IndexedDB、Cache Storage 等）
- 渲染进程通过该服务读写持久化数据

### Sandboxed Utility Processes（沙箱工具进程）

- 用于小型或高风险任务（如音频解码、PDF 渲染等）
- 遵循 Chromium 的 [Rule of Two](https://chromium.googlesource.com/chromium/src/+/master/docs/security/rule-of-2.md) 安全原则

## Network Service 详解

### 架构演进：从 Browser Process IO Thread 到独立进程

Chrome 的网络栈经历了三个主要阶段的演进：

#### 阶段一：网络栈运行在 Browser Process 的 IO Thread（Chrome 73 之前）

在早期架构中，Chromium 的 `net` 库（网络栈）运行在浏览器进程的一个专用 IO 线程上：

- 所有网络请求通过 `ResourceDispatcherHost` 从渲染进程路由到 Browser Process
- `ResourceDispatcherHost` 为每个请求创建 `URLRequest` 对象，交由对应的 `URLRequestJob` 处理
- 多个 `URLRequest` 共享同一个 `URLRequestContext`，包含 Cookie、HostResolver、ProxyService、HttpCache 等上下文
- 网络栈是单线程的（运行在 IO Thread），所有操作必须是非阻塞的异步回调

**存在的问题：**

- Browser Process 承担了过多职责，一旦网络栈崩溃会影响整个浏览器
- 渲染进程虽然被沙箱化，但仍能通过 IPC 间接访问网络栈的某些能力
- 难以对网络访问实施细粒度的安全策略和站点隔离

#### 阶段二：Network Service 作为 Browser Process 内的独立服务（过渡阶段）

Chromium 引入了 Services 架构（基于 Mojo IPC），将网络栈封装为 `network::mojom::NetworkService` 接口：

- 网络栈逻辑被封装为独立的服务模块，通过 Mojo 接口暴露能力
- 此时 Network Service 仍运行在 Browser Process 内，但已实现接口隔离
- 为后续迁移到独立进程奠定了接口基础

#### 阶段三：Network Service 运行在独立进程中（Chrome 73+ 默认启用）

从 Chrome 73 开始，Network Service 默认运行在独立的 `--type=network` 进程中：

- 网络栈完全脱离 Browser Process，拥有独立的进程空间和内存
- 通过 Mojo IPC 与 Browser Process、Renderer Process 通信
- 可施加独立的沙箱限制，进一步缩小攻击面

### 为什么 Chrome 要将网络请求注册为系统服务（Service-izing）

将网络栈从 Browser Process 中剥离并注册为独立服务进程，主要出于以下核心动机：

#### 1. 强化渲染进程沙箱（Sandboxing）

这是 Network Service 最核心的安全动机：

- **渲染进程不应拥有直接网络访问能力**：沙箱化的渲染进程如果被攻破，攻击者不应能直接发起任意网络请求
- **网络访问成为特权操作**：只有 Network Service 进程拥有打开 TCP/UDP socket、DNS 解析、TLS 握手等能力
- **渲染进程只能通过受控的 Mojo 接口请求网络资源**：Browser Process 和 Network Service 可对请求进行安全检查、来源验证和策略执行

在沙箱设计中，这遵循了 Chromium 的 **Rule of Two** 原则：任何处理不可信数据的代码最多只能缺少两个安全机制。将网络访问从渲染进程剥离，是构建深度防御（Defense in Depth）的关键一环。

#### 2. 支持 Site Isolation（站点隔离）

Site Isolation 要求不同站点的页面运行在不同进程中，并阻止跨站点数据泄漏：

- **Cross-Origin Read Blocking (CORB)**：Network Service 在数据送达渲染进程前，可在浏览器进程/网络服务进程层面拦截跨站点的 HTML、XML、JSON 等敏感数据
- **Cookie 和认证信息隔离**：Network Service 可根据请求来源的站点，控制哪些 Cookie 和认证信息可附带在请求中
- **浏览器进程强制执行**：安全检查从渲染进程移至 Browser Process / Network Service，即使渲染进程被完全攻破，攻击者也无法绕过这些检查

#### 3. 崩溃隔离（Crash Isolation）

网络栈是浏览器中最复杂的子系统之一，涉及：

- HTTP/1.1、HTTP/2、HTTP/3 (QUIC) 协议实现
- TLS/SSL 握手和证书验证（通过 BoringSSL）
- DNS 解析、代理配置（PAC 脚本执行）、连接池管理
- HTTP 缓存、磁盘缓存
- WebSocket、FTP、data URI 等协议

将如此复杂的代码运行在独立进程中：

- 网络栈崩溃（如协议解析 bug、内存错误）不会导致整个浏览器崩溃
- Network Service 崩溃后可被自动重启，受影响标签页可重试请求
- Browser Process 和其他渲染进程不受影响

#### 4. 统一网络状态管理

HTTP/1.1 规范要求浏览器作为整体不能对同一主机开启过多连接（默认每主机 6 个并发连接）：

- **连接池共享**：所有标签页、所有进程的网络请求共享同一个连接池，避免连接数超标
- **Cookie 一致性**：Cookie 需要在所有标签页间保持一致，集中管理避免状态不一致
- **HTTP 缓存共享**：多个标签页可共享同一资源的缓存，减少重复请求
- **代理配置统一**：系统代理、PAC 脚本等配置只需在一处维护和执行

如果每个渲染进程独立管理网络，这些全局状态将难以保持一致。

#### 5. 性能优化

- **网络预测和预连接**：Network Service 可分析页面引用模式，对可能发生的请求提前进行 DNS 预解析或 TCP 预连接
- **连接复用**：HTTP/2 和 HTTP/3 的多路复用能力在集中管理的 Network Service 中更易实现
- **缓存命中优化**：集中管理的 HTTP 缓存和磁盘缓存可提高跨标签页的缓存命中率

#### 6. 架构可维护性

- **职责分离**：Browser Process 专注于 UI 和进程管理，Network Service 专注于网络协议
- **独立测试和迭代**：网络栈可在独立进程中测试、调试和升级，不影响其他组件
- **跨平台抽象**：`net` 库本身是跨平台的，独立为服务后更易在不同平台上施加不同的沙箱策略

### Network Service 内部工作机制

Network Service 进程内部包含以下核心组件：

| 组件 | 职责 |
|------|------|
| **URLLoaderFactory** | 创建 `URLLoader` 实例处理具体请求，是 Network Service 的主要 Mojo 接口 |
| **URLLoader** | 表示单个网络请求，负责协议分发、请求执行和响应返回 |
| **URLRequestContext** | 包含请求的上下文：Cookie、缓存、代理、HostResolver 等 |
| **HttpCache** | HTTP 缓存层，先检查缓存再决定是否发起网络请求 |
| **HttpNetworkTransaction** | 实际执行 HTTP 事务，包括请求发送和响应接收 |
| **HttpStreamFactory** | 创建 HTTP 流（HttpBasicStream 或 SpdyHttpStream），处理连接建立 |
| **ClientSocketPoolManager** | 管理所有 socket 连接池，实施每主机、每代理、每进程的连接数限制 |
| **HostResolver** | DNS 解析，在非阻塞的工作线程上执行 `getaddrinfo()` 等阻塞调用 |
| **ProxyService** | 代理配置管理和 PAC 脚本执行 |
| **CookieManager** | Cookie 的存储、查询和设置 |
| **SSLClientSocket** | TLS/SSL 连接，使用 BoringSSL 处理加密和解密 |

**请求流程简述：**

1. 渲染进程通过 Mojo 调用 `URLLoaderFactory::CreateLoaderAndStart()`
2. Network Service 创建 `URLLoader`，根据 URL scheme 选择对应的 `URLRequestJob`
3. 对于 HTTP 请求，先经过 `HttpCache` 检查缓存
4. 缓存未命中时，创建 `HttpNetworkTransaction`
5. `HttpStreamFactory` 通过 `ClientSocketPoolManager` 获取或建立 socket 连接
6. 连接建立后，通过 `HttpBasicStream`（HTTP/1.1）或 `SpdyHttpStream`（HTTP/2）发送请求
7. 响应数据通过 Mojo 管道流式返回给渲染进程

### Network Service 的沙箱限制

作为独立进程，Network Service 也运行在沙箱中，但权限比渲染进程更宽松：

- **允许**：打开 TCP/UDP socket、DNS 解析、TLS 连接
- **限制**：文件系统访问受限（仅缓存目录）、不能执行任意代码、不能访问用户敏感数据
- **平台差异**：不同操作系统上的沙箱策略不同，Windows 使用 Job Object + Integrity Level，macOS 使用 Seatbelt sandbox，Linux 使用 seccomp-bpf

### 启用时间线

| 版本 | 变更 |
|------|------|
| Chrome 60+ | Network Service 作为 Browser Process 内的服务引入（Mojo 接口） |
| Chrome 70+ | 支持将 Network Service 运行在独立进程中（实验性） |
| **Chrome 73** | **桌面平台默认启用独立 Network Service 进程** |
| Chrome 75+ | Android 平台启用 |
| Chrome 80+ | 所有平台默认启用，旧代码路径移除 |

### 如何验证 Network Service 是否运行

- 打开 `chrome://process-internals` 可查看进程类型
- 打开 Chrome 任务管理器（Shift+Esc），可看到独立的 "Network Service" 进程
- 访问 `chrome://net-internals` 可查看网络请求的内部日志

## 进程间通信（IPC）

浏览器进程与渲染进程之间通过以下机制通信：

| 机制 | 说明 |
|------|------|
| **Mojo** | Chromium 当前的主要 IPC 框架，提供类型安全的跨进程通信 |
| **Legacy IPC** | Chromium 早期的 IPC 系统，已逐步被 Mojo 取代 |

通信模型：

- `RenderProcessHost`（浏览器端）↔ `RenderProcess`（渲染端）：进程级通信
- `RenderFrameHost`（浏览器端）↔ `RenderFrame`（渲染端）：文档/帧级通信
- 每个 `RenderFrame` 有唯一的 routing ID，用于在同一渲染进程内区分多个文档

## 进程分配策略

### 默认行为

- 每个新标签页/窗口默认在新进程中打开
- 同一站点（site）的页面可能共享同一渲染进程
- `window.open` 创建的窗口若属于同一 origin，则共享进程

### Site 的定义

Site Isolation 中 **site** 的精确定义：

- 包含 scheme 和注册域名（含 public suffix）
- **忽略**子域名、端口和路径
- 例如：`https://mail.google.com` 和 `https://docs.google.com` 属于同一 site（`google.com`）

使用 site 而非 origin 是为了兼容通过 `document.domain` 进行跨子域名通信的现有网页。

### SiteInstance

**SiteInstance** 是 Chromium 中"不能被拆分到多个进程的最小单元"：

- 同一 SiteInstance 内的文档可以互相脚本访问
- 可能跨越多个标签页或帧
- 可能来自同一 site 的多个子域名
- HTML 规范中称为 "unit of related similar-origin browsing contexts"

### 进程复用

当渲染进程数量过多时，Chrome 会采取策略复用现有进程：

- 同一 site 的页面优先复用已有进程
- 不同 site 的页面不会复用进程（Site Isolation 启用后）
- Android 端为控制内存开销，仅对登录站点启用独立进程

## Site Isolation（站点隔离）

### 背景

在 Chrome 67 之前，Chrome 虽然采用了多进程架构，但出于兼容性考虑，许多情况下不同站点的页面仍共享同一进程：

- 跨站点 iframe 通常与父文档在同一进程
- 渲染进程发起的导航（链接点击、表单提交等）即使跨站点也保持在当前进程
- 进程数量过多时会复用现有进程

这意味着浏览器依赖渲染进程内部的 Same Origin Policy 来隔离不同站点，一旦渲染进程被攻破，攻击者可访问同一进程内所有站点的数据。

### Site Isolation 核心机制

| 机制 | 说明 | 状态 |
|------|------|------|
| **Cross-Process Navigations** | 跨站点导航时切换渲染进程 | 已完成 |
| **Cross-Process JavaScript** | 跨进程的 `postMessage`、`close`、`focus` 等交互通过异步消息实现 | 已完成 |
| **Out-of-Process iframes (OOPIF)** | 跨站点 iframe 在独立进程中渲染 | 已完成 |
| **Cross-Origin Read Blocking (CORB)** | 阻止渲染进程接收跨站点的 HTML、XML、JSON 数据 | 已完成 |
| **Browser Process Enforcements** | 浏览器进程直接执行安全检查，而非依赖渲染进程 | 大部分完成 |

### 启用时间线

| 版本 | 平台 | 范围 |
|------|------|------|
| Chrome 56 | 桌面 | 扩展页面与 Web 内容隔离 |
| Chrome 63 | 桌面 | 可作为实验性功能启用 |
| **Chrome 67** | **桌面（Win/Mac/Linux/ChromeOS）** | **默认启用，所有站点** |
| Chrome 77 | Android（≥2GB RAM） | 仅用户登录的站点 |

### 防御的威胁

Site Isolation 启用后，可防御以下攻击：

- 窃取跨站点的 Cookie 和 HTML5 存储数据
- 窃取跨站点的 HTML、XML、JSON 数据
- 窃取已保存的密码
- 滥用其他站点获得的权限（如地理位置）
- 利用 UXSS（Universal XSS）漏洞访问跨站点 DOM
- Spectre 等推测执行侧信道攻击

**不防御**的攻击：传统 XSS、CSRF、XSSI、Clickjacking（这些发生在页面内部，不涉及跨进程）。

## 沙箱（Sandboxing）

渲染进程运行在操作系统级别的沙箱中：

- **网络访问**：只能通过 Network Service，不能直接发起网络请求
- **文件系统**：通过操作系统权限限制，只能访问授权目录
- **设备访问**：不能直接访问摄像头、麦克风、USB 等设备
- **显示和输入**：访问受到限制，防止屏幕捕获或键盘记录

沙箱显著限制了被攻破的渲染进程能够造成的损害。

## 内存优化

Chrome 多进程架构带来了内存管理优势：

- **隐藏标签页降权**：无顶层标签页的渲染进程会被标记为低优先级
- **Working Set 释放**：Windows 等系统会在内存紧张时优先将低优先级进程的内存交换到磁盘
- **渐进式释放**：为避免频繁切换标签页时的性能损失，内存释放是渐进的
- **对比单进程**：单进程浏览器中所有标签页的数据随机分布在内存中，无法区分活跃和非活跃数据

## 崩溃处理

每个 IPC 连接监控渲染进程的句柄：

- 当句柄被信号化时，浏览器进程检测到渲染进程崩溃
- 受影响的标签页/帧显示 "Sad Tab" 或 "Sad Frame" 提示
- 用户可按刷新按钮重新加载，浏览器会创建新的渲染进程
- 其他标签页不受影响，可继续使用

## 限制与注意事项

### 内存开销

- 每个进程有独立的内存开销（V8 引擎实例、Blink 实例等）
- Site Isolation 启用后，包含多个跨站点 iframe 的页面会创建更多进程
- Chrome 通过进程复用、进程合并策略控制内存使用

### 性能影响

- **正面**：不同站点的页面可并行渲染，互不阻塞
- **负面**：跨进程导航有额外延迟，进程间通信有开销
- 实际性能影响通过真实浏览数据持续监控和优化

### 开发者影响

- `window.open` 和跨窗口交互在跨站点场景下行为可能变化
- 依赖 `document.domain` 的跨子域名通信仍可用（site 定义已考虑）
- 调试时可观察多个渲染进程，需理解进程边界

## 相关概念

| 概念 | 说明 |
|------|------|
| **Blink** | Chromium 使用的开源渲染引擎，负责 HTML/CSS 解析和布局 |
| **V8** | Chromium 使用的 JavaScript 引擎，运行在渲染进程中 |
| **Mojo** | Chromium 的 IPC 框架，用于进程间通信 |
| **Same Origin Policy** | 浏览器的核心安全模型，限制不同源之间的交互 |
| **OOPIF** | Out-of-Process iframes，跨站点 iframe 在独立进程中渲染 |
| **CORB** | Cross-Origin Read Blocking，阻止跨站点敏感数据泄漏到渲染进程 |
| **Rule of Two** | Chromium 安全原则：任何处理不可信数据的代码最多只能缺少两个安全机制 |
| **BoringSSL** | Chromium 使用的 TLS/SSL 库，基于 OpenSSL 的分支 |
| **QUIC/HTTP/3** | Google 开发的基于 UDP 的传输协议，Chrome 网络栈原生支持 |

## 参考资料

- [Chromium Multi-process Architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture/) - Chromium 官方多进程架构设计文档
- [Site Isolation Design Document](https://www.chromium.org/developers/design-documents/site-isolation/) - Chromium 官方站点隔离设计文档
- [Process Models](https://chromium.googlesource.com/chromium/src/+/main/docs/process_model_and_site_isolation.md) - Chromium 源码中的进程模型文档
- [Chromium Sandbox Design](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/design/sandbox.md) - Chromium 沙箱设计文档
- [Rule of Two](https://chromium.googlesource.com/chromium/src/+/master/docs/security/rule-of-2.md) - Chromium 安全原则
- [Site Isolation: Process Separation for Web Sites within the Browser](https://www.usenix.org/conference/usenixsecurity19/presentation/reis) - USENIX Security 2019 论文
- [Chromium Network Stack](https://www.chromium.org/developers/design-documents/network-stack/) - Chromium 官方网络栈设计文档
- [Multi-process Resource Loading](https://www.chromium.org/developers/design-documents/multi-process-resource-loading/) - Chromium 多进程资源加载设计文档
- [Network Stack Use in Chromium](https://www.chromium.org/developers/design-documents/network-stack/network-stack-use-in-chromium/) - Chromium 中网络栈的使用方式
