---
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - network
  - protocol
  - TCP
  - IP
  - internet
  - computer-science
aliases:
  - TCP/IP
  - Transmission Control Protocol/Internet Protocol
  - TCP/IP 协议栈
  - TCP/IP 模型
source_type: spec
source_urls:
  - 'https://www.rfc-editor.org/rfc/rfc791'
  - 'https://www.rfc-editor.org/rfc/rfc793'
  - 'https://developer.mozilla.org/en-US/docs/Glossary/TCP'
status: verified
---

TCP/IP（Transmission Control Protocol / Internet Protocol，传输控制协议/网际协议）是一组用于互联网通信的协议集合，定义了电子设备如何连入互联网以及数据如何在它们之间传输。该协议族由 Vint Cerf 和 Bob Kahn 在 1970 年代设计，是互联网通信的基础。

## 协议概述

TCP/IP 不是单一协议，而是一个**协议族**（Protocol Suite），包含多个协同工作的协议。它采用分层架构，每一层负责不同的通信功能，上层依赖下层提供的服务。

### 为什么重要

- **互联网的基石**：所有现代互联网通信都基于 TCP/IP 协议族
- **跨平台兼容**：独立于底层硬件和操作系统，实现异构网络互联
- **可靠性保证**：TCP 提供面向连接、可靠的数据传输；IP 负责寻址和路由
- **可扩展性**：分层设计允许各层独立演进，不影响其他层

## 四层模型

TCP/IP 模型通常分为四层（也有五层或七层 OSI 对照模型的说法）：

| 层级 | 名称 | 主要协议 | 职责 |
|------|------|----------|------|
| 第 4 层 | 应用层（Application） | HTTP、FTP、SMTP、DNS、Telnet | 为应用程序提供网络服务接口 |
| 第 3 层 | 传输层（Transport） | TCP、UDP | 端到端的可靠/不可靠数据传输 |
| 第 2 层 | 网际层（Internet） | IP、ICMP、IGMP | 数据包的路由和转发 |
| 第 1 层 | 网络接口层（Network Interface） | Ethernet、Wi-Fi、ARP | 物理网络上的数据帧传输 |

### 与 OSI 七层模型的对照

| OSI 七层模型 | TCP/IP 四层模型 |
|-------------|----------------|
| 7. 应用层 | 应用层 |
| 6. 表示层 | 应用层 |
| 5. 会话层 | 应用层 |
| 4. 传输层 | 传输层 |
| 3. 网络层 | 网际层 |
| 2. 数据链路层 | 网络接口层 |
| 1. 物理层 | 网络接口层 |

## 核心协议详解

### IP（Internet Protocol，网际协议）

IP 是 TCP/IP 协议族的核心，负责将数据包从源主机路由到目标主机。当前广泛使用的是 **IPv4**（RFC 791，1981 年），正在逐步过渡到 **IPv6**。

#### IP 的核心特性

- **无连接**：每个数据包（datagram）独立处理，不维护连接状态
- **不可靠**：不保证数据包一定到达、不保证顺序、不做重传
- **尽力而为**（Best Effort）：尽最大努力交付，但不提供可靠性保证
- **寻址**：使用 IP 地址标识网络中的主机

#### IPv4 数据报头部格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段说明**：

| 字段 | 长度 | 说明 |
|------|------|------|
| Version | 4 bits | IP 版本，IPv4 为 `0100`（4） |
| IHL | 4 bits | 头部长度（32-bit 字为单位），最小值为 5（20 字节） |
| Total Length | 16 bits | 数据报总长度（含头部），最大 65,535 字节 |
| Identification | 16 bits | 用于分片重组的标识符 |
| Flags | 3 bits | DF（Don't Fragment）、MF（More Fragments） |
| Fragment Offset | 13 bits | 分片在原始数据报中的偏移（以 8 字节为单位） |
| TTL | 8 bits | 生存时间，每经过一个路由器减 1，为 0 时丢弃 |
| Protocol | 8 bits | 上层协议类型（6=TCP，17=UDP，1=ICMP） |
| Header Checksum | 16 bits | 头部校验和（仅校验头部，不校验数据） |
| Source Address | 32 bits | 源 IP 地址 |
| Destination Address | 32 bits | 目标 IP 地址 |

#### IP 地址分类（IPv4）

| 类别 | 高位比特 | 网络位 | 主机位 | 地址范围 | 适用场景 |
|------|----------|--------|--------|----------|----------|
| A 类 | `0` | 7 bits | 24 bits | 1.0.0.0 - 126.255.255.255 | 大型网络 |
| B 类 | `10` | 14 bits | 16 bits | 128.0.0.0 - 191.255.255.255 | 中型网络 |
| C 类 | `110` | 21 bits | 8 bits | 192.0.0.0 - 223.255.255.255 | 小型网络 |
| D 类 | `1110` | - | - | 224.0.0.0 - 239.255.255.255 | 组播 |
| E 类 | `1111` | - | - | 240.0.0.0 - 255.255.255.255 | 保留 |

> **注意**：现代网络已广泛采用 CIDR（无类别域间路由）替代传统的地址分类，使用可变长度子网掩码（VLSM）更灵活地分配地址。

#### IP 分片与重组

当数据报大小超过网络的 MTU（Maximum Transmission Unit，最大传输单元）时，IP 层会进行分片：

- **分片**：路由器将大数据报拆分为多个小分片，每个分片有独立的 IP 头部
- **重组**：仅在目标主机进行，中间路由器不重组
- **DF 标志**：若设置 `Don't Fragment`，路由器无法转发时会丢弃并返回 ICMP 错误
- **分片偏移**：以 8 字节为单位，标识分片在原始数据报中的位置

### TCP（Transmission Control Protocol，传输控制协议）

TCP 是面向连接的、可靠的、基于字节流的传输层协议（RFC 793，1981 年）。它在不可靠的 IP 层之上提供可靠的数据传输服务。

#### TCP 的核心特性

- **面向连接**：通信前需通过三次握手建立连接，通信结束后通过四次挥手释放连接
- **可靠传输**：通过序列号、确认应答、超时重传保证数据不丢失、不重复、按序到达
- **流量控制**：使用滑动窗口机制，防止发送方发送速度超过接收方处理能力
- **拥塞控制**：通过慢启动、拥塞避免、快速重传、快速恢复等算法避免网络拥塞
- **全双工通信**：连接建立后，双方可同时发送和接收数据
- **字节流服务**：TCP 将应用层数据视为无结构的字节流，不保留消息边界

#### TCP 头部格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段说明**：

| 字段 | 长度 | 说明 |
|------|------|------|
| Source Port | 16 bits | 源端口号 |
| Destination Port | 16 bits | 目标端口号 |
| Sequence Number | 32 bits | 序列号，标识本段数据第一个字节的序号 |
| Acknowledgment Number | 32 bits | 确认号，期望收到对方下一个字节的序号 |
| Data Offset | 4 bits | 头部长度（32-bit 字为单位），最小 5（20 字节） |
| URG | 1 bit | 紧急指针有效 |
| ACK | 1 bit | 确认号有效 |
| PSH | 1 bit | 推送标志，要求接收方立即将数据交给应用层 |
| RST | 1 bit | 重置连接 |
| SYN | 1 bit | 同步序列号，用于建立连接 |
| FIN | 1 bit | 结束标志，用于释放连接 |
| Window | 16 bits | 接收窗口大小，用于流量控制 |
| Checksum | 16 bits | 校验和（覆盖头部 + 数据 + 伪头部） |
| Urgent Pointer | 16 bits | 紧急指针，指向紧急数据末尾 |

#### 三次握手（Three-Way Handshake）

TCP 通过三次握手建立连接，同步双方的初始序列号（ISN）：

```
客户端                                    服务器
  |                                         |
  |------------ SYN (seq=x) -------------->|  1. 客户端发送 SYN，进入 SYN-SENT 状态
  |                                         |
  |<------- SYN-ACK (seq=y, ack=x+1) ------|  2. 服务器回应 SYN+ACK，进入 SYN-RCVD 状态
  |                                         |
  |------------ ACK (ack=y+1) ------------>|  3. 客户端发送 ACK，进入 ESTABLISHED 状态
  |                                         |  服务器收到 ACK 后也进入 ESTABLISHED 状态
```

**为什么需要三次？** 防止已失效的连接请求报文段突然传送到服务器，导致服务器错误地建立连接。两次握手无法防止这种历史报文干扰。

#### 四次挥手（Four-Way Wavehand）

TCP 通过四次挥手释放连接：

```
主动关闭方                                被动关闭方
  |                                         |
  |------------- FIN (seq=u) -------------->|  1. 主动方发送 FIN，进入 FIN-WAIT-1
  |                                         |
  |<-------------- ACK (ack=u+1) -----------|  2. 被动方确认，主动方进入 FIN-WAIT-2
  |                                         |     被动方进入 CLOSE-WAIT
  |                                         |     （被动方可能还有数据要发送）
  |                                         |
  |<------------- FIN (seq=v) --------------|  3. 被动方发送 FIN，进入 LAST-ACK
  |                                         |     主动方进入 TIME-WAIT
  |                                         |
  |-------------- ACK (ack=v+1) ----------->|  4. 主动方确认，被动方关闭
  |                                         |
  |     （等待 2MSL 后关闭）                  |
```

**TIME-WAIT 状态的作用**：

1. **保证最后一个 ACK 能到达被动关闭方**：若 ACK 丢失，被动方会重传 FIN，主动方需能响应
2. **防止历史报文干扰新连接**：等待 2MSL（Maximum Segment Lifetime，最大报文段生存时间）确保当前连接的所有报文段都从网络中消失

#### TCP 状态机

TCP 连接经历以下状态：

| 状态 | 说明 |
|------|------|
| CLOSED | 初始状态，无连接 |
| LISTEN | 服务器等待客户端连接请求 |
| SYN-SENT | 客户端已发送 SYN，等待服务器回应 |
| SYN-RCVD | 服务器已收到 SYN 并发送 SYN+ACK |
| ESTABLISHED | 连接已建立，可传输数据 |
| FIN-WAIT-1 | 已发送 FIN，等待对方确认或 FIN |
| FIN-WAIT-2 | 已收到对方对 FIN 的确认，等待对方 FIN |
| CLOSE-WAIT | 已收到对方 FIN，等待应用层关闭连接 |
| CLOSING | 双方同时关闭，已发送 FIN 但未收到确认 |
| LAST-ACK | 已发送 FIN，等待对方确认 |
| TIME-WAIT | 已收到对方 FIN 的确认，等待 2MSL |

#### 流量控制（滑动窗口）

TCP 使用滑动窗口机制进行流量控制：

- **接收窗口（rwnd）**：接收方通过 `Window` 字段告知发送方自己还能接收多少数据
- **发送窗口**：发送方根据接收窗口调整发送速率，避免淹没接收方
- **窗口大小**：16 位字段，最大 65,535 字节；通过窗口缩放选项（RFC 1323）可扩展至 1GB

#### 拥塞控制

TCP 拥塞控制通过以下算法动态调整发送速率：

| 算法 | 触发条件 | 行为 |
|------|----------|------|
| 慢启动（Slow Start） | 连接建立或超时重传 | 拥塞窗口（cwnd）每 RTT 翻倍增长 |
| 拥塞避免（Congestion Avoidance） | cwnd 达到慢启动阈值（ssthresh） | cwnd 每 RTT 线性增长（+1 MSS） |
| 快速重传（Fast Retransmit） | 收到 3 个重复 ACK | 立即重传丢失的报文段 |
| 快速恢复（Fast Recovery） | 快速重传后 | ssthresh 减半，cwnd 设为 ssthresh + 3，进入拥塞避免 |

### UDP（User Datagram Protocol，用户数据报协议）

UDP 是无连接的、不可靠的传输层协议（RFC 768）。

#### UDP 与 TCP 的对比

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接性 | 面向连接 | 无连接 |
| 可靠性 | 可靠（确认、重传、排序） | 不可靠 |
| 有序性 | 保证按序到达 | 不保证顺序 |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有 | 无 |
| 头部开销 | 20-60 字节 | 8 字节 |
| 传输速度 | 较慢 | 较快 |
| 适用场景 | 文件传输、网页浏览、邮件 | 视频流、DNS 查询、实时游戏 |

### ICMP（Internet Control Message Protocol，网际控制报文协议）

ICMP 用于在 IP 主机和路由器之间传递控制消息和错误报告：

- **目的不可达**（Destination Unreachable）：目标网络、主机、端口不可达
- **超时**（Time Exceeded）：TTL 减为 0，用于 traceroute
- **回显请求/应答**（Echo Request/Reply）：用于 ping 命令
- **重定向**（Redirect）：通知主机更优的路由路径

## 关键机制

### Socket（套接字）

Socket 是 TCP/IP 网络编程的接口抽象，由 **IP 地址 + 端口号** 组成：

- **三元组**：`{协议, 本地 IP:端口, 远程 IP:端口}` 唯一标识一条 TCP 连接
- **端口范围**：0-65535
  - 0-1023：知名端口（Well-known Ports），如 HTTP(80)、HTTPS(443)、SSH(22)
  - 1024-49151：注册端口（Registered Ports）
  - 49152-65535：动态/私有端口（Dynamic/Private Ports）

### NAT（Network Address Translation，网络地址转换）

NAT 允许多个内部主机共享一个公网 IP 地址：

- **静态 NAT**：一对一映射，内部 IP 固定映射到外部 IP
- **动态 NAT**：从地址池中动态分配外部 IP
- **PAT/NAPT**（端口地址转换）：多对一映射，通过端口号区分不同内部主机
- **限制**：NAT 破坏了端到端通信模型，对 P2P 应用、IPsec 等有影响

### MTU 与路径 MTU 发现

- **MTU**（Maximum Transmission Unit）：网络层能通过的最大数据包大小
- **以太网 MTU**：通常为 1500 字节
- **路径 MTU 发现**（PMTUD）：通过设置 DF 标志并探测 ICMP "Fragmentation Needed" 消息，确定源到目标路径上的最小 MTU

### DNS（Domain Name System，域名系统）

DNS 将人类可读的域名转换为 IP 地址：

- **递归查询**：DNS 服务器代替客户端完成完整查询
- **迭代查询**：DNS 服务器返回下一个应查询的服务器地址
- **缓存**：DNS 记录有 TTL，减少重复查询
- **记录类型**：A（IPv4）、AAAA（IPv6）、CNAME（别名）、MX（邮件交换）、NS（名称服务器）

## 常见问题与误区

### 常见误区

1. **"TCP 保证数据绝对安全"**：TCP 只保证传输层的可靠性，不提供加密。数据安全需要 TLS/SSL 等上层协议
2. **"IP 地址就是物理地址"**：IP 地址是逻辑地址，可变化；MAC 地址才是物理地址，烧录在网卡中
3. **"TCP 比 UDP 慢"**：TCP 建立连接有开销，但在稳定网络中，TCP 的拥塞控制可能比无控制的 UDP 更高效
4. **"关闭连接就是断开网线"**：TCP 关闭是协议层面的状态机转换，物理连接可能仍然存在

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| TIME_WAIT 过多 | 大量短连接快速关闭 | 调整内核参数、使用连接池、启用 SO_REUSEADDR |
| 半开连接 | 一方崩溃未通知对方 | 启用 TCP Keepalive 机制 |
| 连接被重置（RST） | 端口未监听、防火墙拦截、连接超时 | 检查服务状态、防火墙规则、超时配置 |
| 粘包/拆包 | TCP 是字节流，不保留消息边界 | 应用层定义消息长度或分隔符 |

## 相关概念

- **HTTP/HTTPS**：基于 TCP 的应用层协议，Web 通信基础
- **HTTP/2**：复用 TCP 连接，头部压缩，服务器推送
- **HTTP/3**：基于 QUIC（UDP 之上），解决 TCP 队头阻塞问题
- **TLS/SSL**：传输层安全协议，在 TCP 之上提供加密
- **WebSocket**：基于 TCP 的全双工通信协议
- **QUIC**：Google 开发的传输层协议，基于 UDP，整合 TLS 1.3

## 参考资料

- [RFC 791 - Internet Protocol (IP)](https://www.rfc-editor.org/rfc/rfc791) — IP 协议原始规范
- [RFC 793 - Transmission Control Protocol (TCP)](https://www.rfc-editor.org/rfc/rfc793) — TCP 协议原始规范
- [RFC 768 - User Datagram Protocol (UDP)](https://www.rfc-editor.org/rfc/rfc768) — UDP 协议规范
- [RFC 792 - Internet Control Message Protocol (ICMP)](https://www.rfc-editor.org/rfc/rfc792) — ICMP 协议规范
- [MDN - TCP Glossary](https://developer.mozilla.org/en-US/docs/Glossary/TCP) — MDN TCP 词条
- [Cloudflare - What is TCP/IP?](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/) — Cloudflare 技术文档
- [RFC 2460 - Internet Protocol, Version 6 (IPv6)](https://www.rfc-editor.org/rfc/rfc2460) — IPv6 规范
- [RFC 1323 - TCP Extensions for High Performance](https://www.rfc-editor.org/rfc/rfc1323) — TCP 高性能扩展（窗口缩放、时间戳）
