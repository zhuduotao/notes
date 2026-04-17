---
created: 2026-04-17
updated: 2026-04-17
tags:
  - network/nat
  - network/p2p
  - network/transport-layer
  - STUN
  - TURN
  - ICE
aliases:
  - NAT 打洞
  - NAT 穿透
  - NAT Traversal
  - Hole Punching
  - P2P 穿透
source_type: spec
source_urls:
  - https://www.rfc-editor.org/rfc/rfc4787
  - https://www.rfc-editor.org/rfc/rfc5389
  - https://www.rfc-editor.org/rfc/rfc5766
  - https://www.rfc-editor.org/rfc/rfc8445
  - https://www.rfc-editor.org/rfc/rfc5128
status: verified
---

NAT 打洞（NAT Traversal / Hole Punching）是一种在 NAT（Network Address Translation，网络地址转换）设备后方建立**端到端直接通信**的技术。它通过巧妙利用 NAT 的映射行为，使两个位于不同私有网络中的主机能够绕过 NAT 限制，直接交换数据。

## 为什么需要 NAT 打洞

### NAT 的固有局限

NAT 设计初衷是让多个内网主机共享一个公网 IP，但它破坏了互联网的**端到端通信模型**：

- **入站拦截**：NAT 仅允许已建立映射的入站流量通过，未知来源的数据包被丢弃
- **状态依赖**：映射由出站流量触发创建，内网主机无法主动接收外部发起的连接
- **地址隐藏**：内网主机不知道自己的公网地址和端口，无法告知对端如何联系自己

### 典型场景

| 场景 | 问题 | 打洞的作用 |
|------|------|------------|
| **P2P 文件共享** | 双方都在 NAT 后，无法直接连接 | 建立直接数据传输通道，避免中继服务器带宽成本 |
| **实时音视频（VoIP/视频会议）** | 需要低延迟点对点传输 | 绕过 TURN 中继，降低延迟和服务器负载 |
| **在线游戏** | 游戏状态同步需要低延迟 | 玩家之间直接通信，减少中继延迟 |
| **远程桌面/SSH** | 无公网 IP 无法从外部访问 | 通过打洞建立反向连接 |
| **IoT 设备管理** | 设备部署在私有网络中 | 云端服务与设备建立直接通道 |

## NAT 类型分类

NAT 打洞的成功率高度依赖于 NAT 设备的行为。RFC 4787 定义了 NAT 的两个关键维度：

### 映射行为（Mapping Behavior）

指 NAT 如何分配公网端口给内网主机的出站连接：

| 类型 | 名称 | 行为 | 打洞难度 |
|------|------|------|----------|
| **Endpoint-Independent Mapping (EIM)** | 端点无关映射 | 同一内网 IP:端口，无论目标地址是什么，始终映射到同一公网端口 | 低 |
| **Address-Dependent Mapping (ADM)** | 地址相关映射 | 映射随目标 IP 变化，同一内网端口访问不同目标获得不同公网端口 | 中 |
| **Address and Port-Dependent Mapping (APDM)** | 地址和端口相关映射 | 映射随目标 IP:端口变化，最严格 | 高 |

### 过滤行为（Filtering Behavior）

指 NAT 如何决定哪些入站数据包可以通过：

| 类型 | 名称 | 行为 | 打洞难度 |
|------|------|------|----------|
| **Endpoint-Independent Filtering (EIF)** | 端点无关过滤 | 一旦映射建立，任何外部地址均可通过该端口发送数据 | 低 |
| **Address-Dependent Filtering (ADF)** | 地址相关过滤 | 仅允许之前通信过的目标 IP 发送数据 | 中 |
| **Address and Port-Dependent Filtering (APDF)** | 地址和端口相关过滤 | 仅允许之前通信过的目标 IP:端口发送数据 | 高 |

> **传统分类**（RFC 3489，已过时但仍广泛引用）：
> - **Full Cone NAT**：EIM + EIF（最宽松，打洞最容易）
> - **Restricted Cone NAT**：EIM + ADF
> - **Port Restricted Cone NAT**：EIM + APDF
> - **Symmetric NAT**：ADM 或 APDM + APDF（最严格，UDP 打洞通常失败）

### NAT 类型检测

通过 STUN 协议（RFC 5389）可检测 NAT 类型：

```
客户端 → STUN 服务器：发送 Binding Request
STUN 服务器 → 客户端：返回客户端的公网 IP:端口

通过多次向不同 STUN 服务器/端口发送请求，
比较返回的映射地址，可推断 NAT 的映射和过滤行为。
```

## UDP 打洞原理

UDP 打洞是最常见的 NAT 穿透方式，核心思想是**让 NAT 设备"误以为"入站流量是对已建立映射的响应**。

### 基本流程

假设有两个客户端 A 和 B，都位于各自的 NAT 后方，需要建立直接通信：

```
客户端 A (NAT-A 后)          协调服务器 S          客户端 B (NAT-B 后)
      |                          |                          |
      |------ 注册 (A:port) ---->|                          |
      |<-- 确认，分配公网地址 ---|                          |
      |                          |                          |
      |                          |<----- 注册 (B:port) -----|
      |                          |---- 确认，分配公网地址 -->|
      |                          |                          |
      |<--- 请求 B 的公网地址 ----|                          |
      |--- 返回 B 的公网地址 --->|                          |
      |                          |                          |
      |=== 发送 UDP 包到 B 的公网地址 ===>|                  |
      |    (NAT-A 创建映射，但 NAT-B 丢弃)                    |
      |                          |                          |
      |                          |<--- 请求 A 的公网地址 ----|
      |<--- 返回 A 的公网地址 ----|                          |
      |                          |                          |
      |                          |<=== 发送 UDP 包到 A 的公网地址 =========|
      |                          |    (NAT-B 创建映射，但 NAT-A 丢弃)      |
      |                          |                          |
      |<====== 直接通信建立 ======|==========================|
      |   (双方 NAT 都已创建映射，后续包可通过)                |
```

### 关键步骤详解

1. **注册阶段**：A 和 B 分别向公共协调服务器 S 发送 UDP 包，NAT 设备为各自创建公网映射
2. **信息交换**：S 将 A 的公网地址告知 B，将 B 的公网地址告知 A
3. **同时打洞**：A 和 B **几乎同时**向对方的公网地址发送 UDP 包
   - A 发出的包在 NAT-A 上创建了到 B 的映射
   - B 发出的包在 NAT-B 上创建了到 A 的映射
4. **通信建立**：双方 NAT 都已建立映射，后续数据包可正常通过

### 时序要求

UDP 打洞对时序敏感：

- 双方需要在 NAT 映射超时前完成打洞（通常 30-180 秒）
- 首次打洞包可能被对方 NAT 丢弃，但已在己方 NAT 上创建了映射
- 后续从对方发来的包即可通过（因为映射已存在）

## TCP 打洞原理

TCP 打洞比 UDP 更复杂，因为 TCP 是面向连接的协议，需要完成三次握手。

### 同时打开（Simultaneous Open）

TCP 打洞利用 TCP 协议的**同时打开**特性（RFC 793 §3.4）：

```
客户端 A                              客户端 B
  |                                     |
  |--- SYN (seq=x) ------------------>|  同时发出
  |<-- SYN (seq=y) -------------------|  同时收到对方的 SYN
  |                                     |
  |--- ACK (ack=y+1) ---------------->|  双方进入 SYN-RCVD 后发送 ACK
  |<-- ACK (ack=x+1) -----------------|
  |                                     |
  |<======== 连接建立 (ESTABLISHED) ===>|
```

当两端几乎同时向对方发送 SYN 时，TCP 状态机会进入特殊路径，最终建立连接。

### NAT 对 TCP 同时打开的影响

| NAT 类型 | TCP 打洞成功率 | 说明 |
|----------|----------------|------|
| Full Cone | 高 | 映射独立于目标，打洞容易 |
| Restricted Cone | 中 | 需要目标 IP 已在映射中 |
| Symmetric | 极低 | 每个目标 IP 使用不同端口，无法预测对方端口 |

> **注意**：许多 NAT 设备不支持 TCP 同时打开，会丢弃非预期的 SYN 包。实际中 TCP 打洞成功率低于 UDP。

## STUN / TURN / ICE 框架

实际应用中，NAT 打洞通常通过标准化的协议框架实现：

### STUN（Session Traversal Utilities for NAT）

- **RFC 5389** 定义
- **用途**：发现 NAT 类型、获取公网地址和端口
- **工作方式**：客户端向 STUN 服务器发送 Binding Request，服务器返回客户端的公网地址
- **局限**：仅适用于 EIM 类型的 NAT，对 Symmetric NAT 无效

```
客户端 → STUN 服务器：Binding Request
STUN 服务器 → 客户端：Binding Response（含 XOR-MAPPED-ADDRESS）
```

### TURN（Traversal Using Relays around NAT）

- **RFC 5766** 定义
- **用途**：当打洞失败时，作为**中继服务器**转发数据
- **工作方式**：客户端与 TURN 服务器建立连接，所有数据通过 TURN 中转
- **代价**：增加延迟，消耗服务器带宽

```
客户端 A → TURN 服务器：Allocate Request
TURN 服务器 → 客户端 A：Allocation Response（分配中继地址）
客户端 A → TURN 服务器：Send（数据 + 目标地址）
TURN 服务器 → 客户端 B：Data（转发数据）
```

### ICE（Interactive Connectivity Establishment）

- **RFC 8445** 定义
- **用途**：综合 STUN 和 TURN，自动选择最优连接路径
- **工作方式**：
  1. 收集候选人（Candidates）：主机地址、STUN 反射地址、TURN 中继地址
  2. 通过信令服务器交换候选人列表
  3. 执行连通性检查（Connectivity Checks），按优先级测试每对候选人
  4. 选择首个成功的候选对作为通信路径

```
优先级顺序（通常）：
1. 主机候选人（同一局域网直连）
2. 服务器反射候选人（STUN 打洞成功）
3. 中继候选人（TURN  fallback）
```

### WebRTC 中的 ICE

WebRTC 是 ICE 框架的典型应用：

```javascript
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'turn:turn.example.com', username: 'user', credential: 'pass' }
  ]
});

// ICE 自动执行：收集候选人 → 交换 → 连通性检查 → 建立连接
pc.onicecandidate = (event) => {
  if (event.candidate) {
    // 通过信令服务器发送候选人给对方
    sendToRemote(event.candidate);
  }
};
```

## 限制与注意事项

### Symmetric NAT 的 UDP 打洞困境

Symmetric NAT（APDM）为每个目标地址分配不同的公网端口，导致：

- A 无法预测 B 的 NAT 会为 A 分配哪个端口
- 双方发送的打洞包使用的端口与对方期望的端口不匹配
- **UDP 打洞通常失败**，需回退到 TURN 中继

> **例外**：某些 Symmetric NAT 实现存在端口预测漏洞，可被利用，但不可依赖。

### NAT 映射超时

- NAT 映射有生命周期，超时后需重新打洞
- UDP 映射超时通常 30-180 秒（RFC 4787 建议至少 2 分钟）
- TCP 映射通常在连接关闭或超时后清除
- **保持活跃（Keepalive）**：定期发送心跳包维持映射

### 多层 NAT（Nested NAT）

设备可能位于多层 NAT 之后（如家庭路由器 + 运营商级 NAT CGNAT）：

- 需要穿透的每一层 NAT 都创建映射
- STUN 只能看到最外层 NAT 的映射
- 打洞成功率显著降低

### IPv6 的影响

- IPv6 理论上不需要 NAT（每个设备有全球唯一地址）
- 但防火墙仍可能阻止入站连接
- 过渡期间，IPv4/IPv6 双栈环境需要额外处理

## 常见误区

| 误区 | 事实 |
|------|------|
| "NAT 打洞适用于所有 NAT 类型" | Symmetric NAT 的 UDP 打洞通常失败，需 TURN 回退 |
| "打洞后连接永久有效" | NAT 映射有超时时间，需 Keepalive 维持 |
| "TCP 打洞和 UDP 打洞一样可靠" | TCP 打洞依赖同时打开，许多 NAT 不支持，成功率更低 |
| "STUN 可以解决所有穿透问题" | STUN 仅用于发现地址，对 Symmetric NAT 无效，需 TURN |
| "打洞是黑客技术" | NAT 打洞是标准网络技术，用于 P2P、VoIP 等合法场景 |

## 最佳实践

1. **始终实现 TURN 回退**：不要假设打洞一定成功，准备中继方案
2. **使用 ICE 框架**：不要手动实现打洞逻辑，使用标准化框架（如 WebRTC）
3. **合理设置 Keepalive**：根据 NAT 超时特性调整心跳间隔（建议 15-30 秒）
4. **优先 UDP 打洞**：UDP 打洞成功率高于 TCP，延迟更低
5. **监控连接质量**：实时检测丢包率、延迟，必要时切换路径

## 相关概念

- **NAT**：网络地址转换，打洞要解决的问题根源
- **STUN**：用于发现 NAT 类型和公网地址的协议
- **TURN**：打洞失败时的中继方案
- **ICE**：综合 STUN/TURN 的自动连接建立框架
- **WebRTC**：浏览器实时通信框架，内置 ICE 实现
- **CGNAT**：运营商级 NAT，多层 NAT 场景
- **UPnP IGD**：另一种 NAT 穿透方式，通过路由器协议自动映射端口

## 参考资料

- [RFC 4787 - Network Address Translation (NAT) Behavioral Requirements for Unicast UDP](https://www.rfc-editor.org/rfc/rfc4787) — NAT 行为要求规范，定义映射和过滤行为分类
- [RFC 5389 - Session Traversal Utilities for NAT (STUN)](https://www.rfc-editor.org/rfc/rfc5389) — STUN 协议规范
- [RFC 5766 - Traversal Using Relays around NAT (TURN)](https://www.rfc-editor.org/rfc/rfc5766) — TURN 中继协议规范
- [RFC 8445 - Interactive Connectivity Establishment (ICE)](https://www.rfc-editor.org/rfc/rfc8445) — ICE 框架规范
- [RFC 5128 - State of Peer-to-Peer (P2P) Communication across Network Address Translators](https://www.rfc-editor.org/rfc/rfc5128) — P2P NAT 穿透技术综述
- [RFC 3489 - STUN (Classic, Obsoleted)](https://www.rfc-editor.org/rfc/rfc3489) — 经典 STUN 规范（已过时，但 Cone/Symmetric 分类仍被引用）
- [RFC 793 - Transmission Control Protocol](https://www.rfc-editor.org/rfc/rfc793) — TCP 规范，包含同时打开描述（§3.4）
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/) — WebRTC 规范，包含 ICE 集成