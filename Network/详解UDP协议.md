---
created: 2026-04-17
updated: 2026-04-17
tags:
  - network/protocol
  - network/transport-layer
  - UDP
  - internet
aliases:
  - UDP
  - User Datagram Protocol
  - 用户数据报协议
source_type: spec
source_urls:
  - https://www.rfc-editor.org/rfc/rfc768
  - https://www.rfc-editor.org/rfc/rfc1122
  - https://developer.mozilla.org/en-US/docs/Glossary/UDP
  - https://www.cloudflare.com/learning/ddos/glossary/user-datagram-protocol-udp/
status: verified
---

UDP（User Datagram Protocol，用户数据报协议）是互联网协议族（TCP/IP）中的核心**传输层协议**，提供无连接、不可靠但低开销的数据报传输服务。其规范由 Postel 于 1980 年 8 月发布在 **RFC 768** 中，是互联网上最早定义的传输层协议之一。

## 为什么需要 UDP

在 TCP/IP 协议族中，UDP 填补了与 TCP 互补的定位：

- **低延迟优先**：无需建立连接、无确认重传机制，数据传输延迟极低
- **轻量级**：头部仅 8 字节，协议处理开销极小
- **应用层可控**：将可靠性、排序、流量控制等决策权交给应用层，适合有特殊需求的场景
- **无状态**：服务器无需维护连接状态，可支持海量并发客户端

## UDP 头部格式

UDP 头部极其简洁，固定为 **8 字节**：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Source Port           |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                          Data (Payload)                       |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| **Source Port** | 16 bits | 源端口号，可选；若不使用则置 0 |
| **Destination Port** | 16 bits | 目标端口号，标识接收应用 |
| **Length** | 16 bits | UDP 头部 + 数据的总长度（字节），最小值 8（仅头部） |
| **Checksum** | 16 bits | 校验和，覆盖伪头部 + UDP 头部 + 数据（IPv4 可选，IPv6 强制） |

### 伪头部（Pseudo Header）

校验和计算时使用伪头部，用于验证数据报是否送达正确目的地：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Source Address                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Zero    |   Protocol    |          UDP Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Source/Destination Address**：IPv4 为 32 位，IPv6 为 128 位
- **Protocol**：UDP 的协议号 `17`（0x11）
- **UDP Length**：与 UDP 头部中的 Length 字段相同

> **注意**：IPv4 中校验和是可选的（RFC 1122 §4.1.3），若不使用则置全 0；但 **IPv6 中校验和是强制的**（RFC 2460 §8.1），因为 IPv6 头部本身不包含校验和，需要传输层承担此职责。

## 核心特性

### 无连接（Connectionless）

UDP 在发送数据前**不需要建立连接**，也不需要在通信结束后释放连接。每个数据报独立处理，发送方直接调用 `sendto()` 将数据报发出，接收方通过 `recvfrom()` 接收。

### 不可靠（Unreliable）

UDP 不提供以下保证：

- **不保证送达**：数据报可能丢失，UDP 不会重传
- **不保证顺序**：后发送的数据报可能先到达
- **不保证不重复**：网络层可能产生重复数据报，UDP 不会去重
- **无拥塞控制**：发送速率不受网络拥塞状态影响

### 面向报文（Message-oriented）

UDP 保留**消息边界**：

- 发送方调用一次 `send()` 发送一个报文，接收方调用一次 `recv()` 接收一个完整报文
- 与 TCP 的字节流模型不同，UDP 不会将多个报文合并或拆分
- 如果接收缓冲区小于报文大小，**超出部分会被静默丢弃**

### 支持单播、组播和广播

| 模式 | 说明 | 典型用途 |
|------|------|----------|
| **单播（Unicast）** | 一对一通信，目标为特定 IP | DNS 查询、视频通话 |
| **组播（Multicast）** | 一对多通信，目标为组播地址（224.0.0.0 - 239.255.255.255） | 视频会议、实时行情分发 |
| **广播（Broadcast）** | 一对所有通信，目标为广播地址（如 255.255.255.255） | 局域网服务发现、DHCP |

> **注意**：IPv6 不再支持广播，组播取代了广播的功能。

## 与 TCP 的对比

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接性 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（确认、重传、排序） | 不可靠 |
| 有序性 | 保证按序到达 | 不保证顺序 |
| 消息边界 | 无（字节流） | 有（数据报） |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有（慢启动、拥塞避免等） | 无 |
| 头部开销 | 20-60 字节 | 8 字节（固定） |
| 传输模式 | 仅单播 | 单播、组播、广播 |
| 延迟 | 较高（握手 + 确认） | 极低 |
| 适用场景 | 文件传输、网页浏览、邮件 | 实时音视频、DNS、游戏、IoT |

## 典型应用场景

### DNS（域名系统）

- DNS 查询默认使用 **UDP 端口 53**
- 单次查询/响应通常很小（< 512 字节），UDP 的低延迟优势明显
- 若响应超过 512 字节，服务器设置 TC（Truncated）标志，客户端改用 TCP 重试（RFC 1035 §4.2.1）
- 现代 DNS 扩展（EDNS0，RFC 6891）支持 UDP 承载更大响应（通常 4096 字节）

### 实时音视频传输

- **RTP/RTCP**（Real-time Transport Protocol）：基于 UDP 传输实时音频/视频流（RFC 3550）
- **WebRTC**：浏览器实时通信框架，底层使用 UDP（通过 ICE/STUN/TURN 穿透 NAT）
- 音视频应用对延迟敏感，少量丢包可通过编解码器（如 Opus、VP8）的丢包隐藏（PLC）机制补偿

### 在线游戏

- 游戏状态同步需要极低延迟，UDP 是首选
- 游戏引擎（如 Unity、Unreal）通常在 UDP 之上实现自定义可靠性层：
  - 关键指令（如玩家加入/退出）使用可靠传输
  - 状态更新（如位置、朝向）使用不可靠传输，允许丢弃过期数据

### DHCP（动态主机配置协议）

- DHCP 使用 **UDP 端口 67（服务器）和 68（客户端）**
- 客户端在获取 IP 前没有网络层地址，只能使用广播，UDP 天然支持

### SNMP（简单网络管理协议）

- SNMP 使用 **UDP 端口 161（代理）和 162（管理器）**
- 网络管理查询通常短小频繁，UDP 开销低

### QUIC / HTTP/3

- **QUIC**（Quick UDP Internet Connections）：Google 开发的传输层协议，基于 UDP 构建（RFC 9000）
- **HTTP/3**：基于 QUIC 的 HTTP 版本，解决 TCP 队头阻塞问题
- 在 UDP 之上重新实现可靠性、拥塞控制、多路复用，同时整合 TLS 1.3 加密

## 限制与注意事项

### 无拥塞控制的风险

UDP 本身不具备拥塞控制机制，发送方可以以任意速率发送数据。这可能导致：

- **网络拥塞加剧**：大量 UDP 流量可能占满带宽，影响 TCP 流（TCP 会在拥塞时降速，UDP 不会）
- **不公平竞争**：UDP 流可能"饿死"TCP 流

> **最佳实践**：基于 UDP 的应用应在应用层实现适当的速率控制或拥塞避免机制。RFC 5405 提供了 UDP 应用使用建议。

### 分片问题

当 UDP 数据报大小超过路径 MTU 时，IP 层会进行分片：

- **分片丢失风险**：任一分片丢失，整个 UDP 数据报无法重组
- **IPv4**：IP 层自动分片，但效率低
- **IPv6**：中间路由器不分片，源主机需使用路径 MTU 发现（PMTUD）

> **最佳实践**：应用层应控制 UDP 数据报大小，避免 IP 分片。通常建议不超过 **1200 字节**（考虑 IPv6 最小 MTU 1280 字节和头部开销）。

### NAT 穿透

UDP 的无状态特性使其在 NAT 环境下需要特殊处理：

- **NAT 映射**：NAT 设备为 UDP 流创建临时映射，通常有超时时间（常见 30-180 秒）
- **NAT 保持（Keepalive）**：定期发送心跳包维持映射
- **穿透技术**：STUN（RFC 5389）、TURN（RFC 5766）、ICE（RFC 8445）用于 P2P 通信

### 校验和的局限性

- UDP 校验和仅检测数据在传输过程中的**意外损坏**（如比特翻转）
- **不提供完整性保护**：无法抵御恶意篡改
- **不提供认证**：无法验证发送方身份
- 需要安全性时应在上层使用 DTLS（RFC 6347）或其他加密协议

## 常见误区

| 误区 | 事实 |
|------|------|
| "UDP 比 TCP 快" | UDP 延迟更低，但"快"取决于场景；TCP 在稳定大文件传输中吞吐量更高 |
| "UDP 不需要错误检测" | UDP 有校验和（Checksum），只是不提供重传 |
| "UDP 数据报不会丢失" | UDP 不保证送达，数据报可能因网络拥塞、缓冲区满等原因丢失 |
| "UDP 适合所有实时应用" | 实时应用仍需考虑丢包率、抖动、延迟预算，并非所有场景都适合 UDP |
| "IPv4 中 UDP 校验和可以忽略" | 虽然 RFC 允许置 0，但强烈建议启用；IPv6 中校验和是强制的 |

## 编程示例

### C 语言 UDP Socket

```c
// 服务端
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in serv_addr = {0};
serv_addr.sin_family = AF_INET;
serv_addr.sin_port = htons(8080);
serv_addr.sin_addr.s_addr = INADDR_ANY;
bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));

char buffer[1024];
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
recvfrom(sockfd, buffer, sizeof(buffer), 0, (struct sockaddr*)&client_addr, &len);

// 客户端
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in serv_addr = {0};
serv_addr.sin_family = AF_INET;
serv_addr.sin_port = htons(8080);
inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);
sendto(sockfd, "Hello", 5, 0, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
```

### Node.js UDP Socket

```javascript
const dgram = require('dgram');

// 服务端
const server = dgram.createSocket('udp4');
server.on('message', (msg, rinfo) => {
  console.log(`Received: ${msg} from ${rinfo.address}:${rinfo.port}`);
});
server.bind(8080);

// 客户端
const client = dgram.createSocket('udp4');
client.send('Hello', 8080, '127.0.0.1', (err) => {
  if (err) console.error(err);
  client.close();
});
```

## 相关概念

- **TCP**：面向连接的可靠传输层协议，与 UDP 互补
- **DTLS**（Datagram TLS）：基于 UDP 的 TLS 变体，提供加密和认证（RFC 6347）
- **QUIC**：基于 UDP 的现代传输层协议，整合 TLS 1.3，用于 HTTP/3
- **RTP/RTCP**：基于 UDP 的实时传输协议，用于音视频流
- **SCTP**：流控制传输协议，介于 TCP 和 UDP 之间，支持多流和多宿
- **ICMP**：网际控制报文协议，用于错误报告和诊断（如 ping、traceroute）

## 参考资料

- [RFC 768 - User Datagram Protocol (UDP)](https://www.rfc-editor.org/rfc/rfc768) — UDP 协议原始规范（1980 年）
- [RFC 1122 - Requirements for Internet Hosts](https://www.rfc-editor.org/rfc/rfc1122) — 主机要求，包含 UDP 实现细节（§4.1）
- [RFC 2460 - Internet Protocol, Version 6 (IPv6)](https://www.rfc-editor.org/rfc/rfc2460) — IPv6 规范，规定 UDP 校验和强制（§8.1）
- [RFC 5405 - Unicast UDP Usage Guidelines](https://www.rfc-editor.org/rfc/rfc5405) — UDP 单播使用建议
- [RFC 6891 - Extension Mechanisms for DNS (EDNS0)](https://www.rfc-editor.org/rfc/rfc6891) — DNS UDP 扩展
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000) — QUIC 协议规范
- [RFC 6347 - Datagram Transport Layer Security Version 1.2](https://www.rfc-editor.org/rfc/rfc6347) — DTLS 规范
- [MDN - UDP Glossary](https://developer.mozilla.org/en-US/docs/Glossary/UDP) — MDN UDP 词条
- [Cloudflare - What is UDP?](https://www.cloudflare.com/learning/ddos/glossary/user-datagram-protocol-udp/) — Cloudflare 技术文档
