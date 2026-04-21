---
created: '2026-04-21T00:00:00.000Z'
updated: '2026-04-21T00:00:00.000Z'
tags:
  - browser/chrome
  - network/webrtc
  - network/p2p
  - api/webrtc
  - realtime-communication
aliases:
  - Chrome WebRTC
  - Chrome P2P
  - WebRTC 实现
  - RTCPeerConnection
source_type: official-doc
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API'
  - 'https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection'
  - 'https://webrtc.org/getting-started/overview'
  - 'https://webrtc.org/getting-started/peer-connections'
  - 'https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity'
  - 'https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Session_lifetime'
status: verified
---
Chrome 通过 WebRTC（Web Real-Time Communication）API 实现浏览器间的 P2P 连接，支持音视频流和任意数据的点对点传输，无需插件或第三方软件。

## 核心架构

WebRTC 在 Chrome 中包含两大技术模块：

| 模块 | 核心接口 | 功能 |
|------|----------|------|
| **媒体捕获** | `navigator.mediaDevices.getUserMedia()` | 摄取摄像头、麦克风、屏幕共享的 `MediaStream` |
| **P2P 连接** | `RTCPeerConnection` | 建立和管理两个 peer 之间的连接通道 |

`RTCPeerConnection` 是 P2P 连接的核心接口，负责高效流式传输数据、维护连接状态、处理 ICE 协商、安全加密等。

## P2P 连接建立流程

WebRTC 无法在没有中介服务器的情况下直接建立连接，必须依赖**信令通道**（signaling channel）交换连接信息。信令协议不由 WebRTC 规范定义，开发者可自行选择 WebSocket、HTTP API、XMPP 等方式。

### 完整流程时序

```
Peer A（Caller）                    信令服务器                    Peer B（Receiver）
    |                                  |                              |
    |--- 创建 RTCPeerConnection ------|                              |
    |                                  |                              |
    |--- createOffer() -------------->|                              |
    |    生成 SDP Offer                |                              |
    |                                  |                              |
    |--- setLocalDescription(offer) -->|                              |
    |    触发 ICE 候选收集              |                              |
    |                                  |                              |
    |--- 发送 Offer 到信令服务器 ======>|                              |
    |                                  |====== 转发 Offer =======>    |
    |                                  |                              |
    |                                  |                              |--- setRemoteDescription(offer)
    |                                  |                              |--- createAnswer()
    |                                  |                              |--- setLocalDescription(answer)
    |                                  |                              |    触发 ICE 候选收集
    |                                  |                              |
    |                                  |<===== 发送 Answer ==========|
    |<=== 转发 Answer =================|                              |
    |                                  |                              |
    |--- setRemoteDescription(answer) |                              |
    |                                  |                              |
    |<=== 交换 ICE Candidates =========|======== 交换 ICE Candidates=>|
    |                                  |                              |
    |<========== P2P 连接建立 ========================= P2P 连接建立>|
```

### 关键步骤详解

1. **创建 Offer**：Caller 调用 `createOffer()` 生成包含媒体格式、编解码器、ICE 候选信息的 SDP（Session Description Protocol）描述

2. **设置本地描述**：调用 `setLocalDescription()` 将 Offer 设为本地描述，同时触发 ICE 候选收集

3. **信令交换**：通过信令服务器将 Offer/Answer 和 ICE 候选发送给对方

4. **创建 Answer**：Receiver 收到 Offer 后调用 `setRemoteDescription()`，再调用 `createAnswer()` 生成响应

5. **ICE 候选交换**：双方通过 Trickle ICE 技术逐个发送候选，而非等待收集完成

## ICE 框架与候选类型

ICE（Interactive Connectivity Establishment）框架用于发现和选择最优连接路径，综合 STUN 和 TURN 协议。

### ICE 候选类型

| 类型 | 代码 | 来源 | 特点 |
|------|------|------|------|
| **Host** | `host` | 本机网卡地址 | 最高优先级，无 NAT 穿透开销 |
| **Server Reflexive** | `srflx` | STUN 服务器反射 | 穿透一层 NAT 后的公网地址 |
| **Peer Reflexive** | `prflx` | 对端 NAT 反射 | Symmetric NAT 场景中动态发现 |
| **Relay** | `relay` | TURN 服务器中继 | 最后兜底方案，所有流量经服务器转发 |

### 候选优先级

ICE 控制端（controlling agent）按优先级选择候选对：

1. **Host 候选**：同局域网可直接连接，延迟最低
2. **srflx 候选**：STUN 打洞成功，P2P 直连
3. **relay 候选**：TURN 中继回退，延迟和带宽成本最高

### Trickle ICE

传统方式需等待所有候选收集完毕后再交换，现代实践采用 Trickle ICE：每个候选发现后立即通过信令发送，显著减少连接建立时间。

```javascript
peerConnection.addEventListener('icecandidate', event => {
    if (event.candidate) {
        signalingChannel.send({ 'new-ice-candidate': event.candidate });
    }
});

signalingChannel.addEventListener('message', async message => {
    if (message.iceCandidate) {
        await peerConnection.addIceCandidate(message.iceCandidate);
    }
});
```

## SDP 会话描述

SDP（Session Description Protocol）描述连接端点的配置信息，包括媒体格式、编解码器、传输协议、IP 地址端口等。WebRTC 通过 Offer/Answer 模型交换 SDP。

### Offer/Answer 模型

- **Offer**：Caller 发起的连接提案，描述本地媒体配置
- **Answer**：Receiver 对 Offer 的响应，描述己方配置
- **Local Description**：本端 SDP（`localDescription`）
- **Remote Description**：远端 SDP（`remoteDescription`）

### Pending vs Current Description

 renegotiation 场景下，WebRTC 区分：

- **Current Description**：当前实际生效的配置（`currentLocalDescription`、`currentRemoteDescription`）
- **Pending Description**：正在协商中待确认的新配置（`pendingLocalDescription`、`pendingRemoteDescription`）

这种设计确保协商失败时可回退到稳定状态，不影响已有连接。

## 安全机制

WebRTC 强制使用加密传输，Chrome 实现以下安全层：

| 安全层 | 协议 | 作用 |
|--------|------|------|
| **DTLS** | Datagram TLS | 加密 SCTP 数据通道（类似 UDP 的 TLS） |
| **SRTP** | Secure RTP | 加密音视频媒体流 |
| **ICE 认证** | ufrag/password | 防止第三方注入伪造候选 |

DTLS 在 ICE 连接建立后协商，使用自签名证书或通过 `RTCPeerConnection.generateCertificate()` 生成的证书。

## Chrome 实现细节

### 默认 STUN 服务器

Chrome 内置 Google 公共 STUN 服务器：

```javascript
const configuration = {
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
};
const pc = new RTCPeerConnection(configuration);
```

实际应用建议配置自建 STUN/TURN 服务器以避免依赖公共服务。

### ICE 状态监控

Chrome 支持以下 ICE 状态属性和事件：

| 属性/事件 | 含义 |
|-----------|------|
| `iceGatheringState` | `new` → `gathering` → `complete` |
| `iceConnectionState` | `new` → `checking` → `connected` → `completed` → `failed` → `disconnected` → `closed` |
| `connectionState` | `new` → `connecting` → `connected` → `disconnected` → `failed` → `closed` |
| `icecandidate` 事件 | 新候选发现时触发 |
| `iceconnectionstatechange` 事件 | ICE 连接状态变化时触发 |

### ICE 重启

网络条件变化时（如切换 Wi-Fi/蜂窝网络），可触发 ICE Restart 重新收集候选：

```javascript
pc.addEventListener('iceconnectionstatechange', () => {
    if (pc.iceConnectionState === 'failed') {
        pc.setConfiguration(newConfig);
        pc.restartIce();
    }
});
```

`restartIce()` 自动在下次 `createOffer()` 中添加 `iceRestart` 标志，生成新的 ufrag 和 password。

### 数据通道

Chrome 通过 `RTCDataChannel` 支持 P2P 数据传输：

```javascript
const channel = pc.createDataChannel('chat', { ordered: true });
channel.addEventListener('open', () => {
    channel.send('Hello!');
});
channel.addEventListener('message', event => {
    console.log('Received:', event.data);
});

pc.addEventListener('datachannel', event => {
    const remoteChannel = event.channel;
    remoteChannel.addEventListener('message', event => {
        console.log('Received:', event.data);
    });
});
```

数据通道底层使用 SCTP over DTLS，支持可靠或不可靠传输模式。

## 连接生命周期

### 状态机

`RTCPeerConnection` 的 `signalingState` 反映协商状态：

```
stable → have-local-offer → stable（收到 Answer）
stable → have-remote-offer → stable（发送 Answer）
stable → closed
```

 renegotiation 时通过 rollback 可回退到上一个 `stable` 状态。

### Keepalive 维持

- NAT 映射有超时时间（通常 30-180 秒）
- WebRTC 自动发送 STUN binding requests 作为心跳
- 建议应用层也实现自定义心跳检测连接活性

## 常见问题与排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| ICE 连接 `failed` | Symmetric NAT、防火墙拦截、TURN 未配置 | 配置 TURN 中继服务器 |
| 媒体单向传输 | 对方未添加 track 或权限未授予 | 检查 `getUserMedia()` 权限和 `addTrack()` |
| 连接延迟高 | 使用 relay 候选而非 host/srflx | 检查 ICE 候选类型，优化网络配置 |
| renegotiation 失败 | 状态不在 `stable` 时触发协商 | 使用 Perfect Negotiation 模式 |

### Chrome DevTools 调试

在 Chrome DevTools 中可查看：

- `chrome://webrtc-internals/`：WebRTC 内部状态和统计
- `getStats()` API：获取连接详细统计信息（延迟、丢包率、编解码器等）

```javascript
const stats = await pc.getStats();
stats.forEach(report => {
    console.log(report.type, report);
});
```

## 与其他浏览器差异

| 特性 | Chrome | Firefox | Safari |
|------|--------|---------|--------|
| SCTP 数据通道 | 支持 | 支持 | 支持 |
| Simulcast | 支持 | 支持 | 部分支持 |
| Unified Plan | 默认 | 默认 | 默认 |
| ICE TCP 候选 | 支持 | 不支持 | 不支持 |
| `addStream` API | 已废弃 | 已废弃 | 已废弃 |

建议使用 `adapter.js` shim 库处理浏览器间兼容性差异。

## 完整示例代码

```javascript
async function makeCall(signalingChannel) {
    const config = {
        iceServers: [
            { urls: 'stun:stun.l.google.com:19302' },
            { urls: 'turn:turn.example.com', username: 'user', credential: 'pass' }
        ]
    };
    
    const pc = new RTCPeerConnection(config);
    
    pc.addEventListener('icecandidate', e => {
        if (e.candidate) signalingChannel.send({ candidate: e.candidate });
    });
    
    pc.addEventListener('connectionstatechange', () => {
        console.log('Connection state:', pc.connectionState);
    });
    
    const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    stream.getTracks().forEach(track => pc.addTrack(track, stream));
    
    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    signalingChannel.send({ offer: offer });
    
    signalingChannel.addEventListener('message', async msg => {
        if (msg.answer) {
            await pc.setRemoteDescription(new RTCSessionDescription(msg.answer));
        }
        if (msg.candidate) {
            await pc.addIceCandidate(new RTCIceCandidate(msg.candidate));
        }
    });
}

async function receiveCall(signalingChannel, offer) {
    const config = {
        iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    };
    
    const pc = new RTCPeerConnection(config);
    
    pc.addEventListener('icecandidate', e => {
        if (e.candidate) signalingChannel.send({ candidate: e.candidate });
    });
    
    pc.addEventListener('track', e => {
        const remoteVideo = document.getElementById('remoteVideo');
        remoteVideo.srcObject = e.streams[0];
    });
    
    await pc.setRemoteDescription(new RTCSessionDescription(offer));
    
    const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    stream.getTracks().forEach(track => pc.addTrack(track, stream));
    
    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);
    signalingChannel.send({ answer: answer });
}
```

## 相关概念

- [[NAT打洞（NAT穿透）详解]]：NAT 穿透原理与 STUN/TURN/ICE 框架
- [[详解UDP协议]]：WebRTC 底层传输协议
- **STUN**：发现公网地址的协议
- **TURN**：中继转发协议
- **SDP**：会话描述协议格式
- **DTLS/SRTP**：加密传输协议
- **RTP**：实时传输协议（音视频）
- **SCTP**：数据通道传输协议

## 参考资料

- [MDN WebRTC API 文档](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API) — WebRTC API 总览
- [MDN RTCPeerConnection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection) — 核心连接接口详细文档
- [WebRTC.org Getting Started](https://webrtc.org/getting-started/overview) — Google 官方 WebRTC 入门指南
- [WebRTC.org Peer Connections](https://webrtc.org/getting-started/peer-connections) — P2P 连接建立教程
- [MDN WebRTC Connectivity](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity) — 连接建立协议交互详解
- [MDN WebRTC Session Lifetime](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Session_lifetime) — 会话生命周期管理
- [RFC 8835 - Transports for RTCWEB](https://datatracker.ietf.org/doc/rfc8835/) — WebRTC 传输协议规范
- [RFC 8827 - WebRTC Security Architecture](https://datatracker.ietf.org/doc/rfc8827/) — WebRTC 安全架构
- [adapter.js GitHub](https://github.com/webrtcHacks/adapter) — WebRTC 浏览器兼容性 shim
