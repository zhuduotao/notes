---
created: 2026-04-15
updated: 2026-04-15
tags:
  - network
  - protocol
  - IPv4
  - IPv6
  - internet
  - addressing
aliases:
  - IPv4 vs IPv6
  - Internet Protocol Version 4 and 6
  - IP 协议对比
  - IPv4 IPv6 区别
source_type: spec
source_urls:
  - https://www.rfc-editor.org/rfc/rfc791
  - https://www.rfc-editor.org/rfc/rfc8200
  - https://www.rfc-editor.org/rfc/rfc4291
  - https://www.rfc-editor.org/rfc/rfc4861
  - https://www.apnic.net/about-apnic/corporate-documents/documents/resource-guidelines/address-policies/ipv6-deployment-statistics/
status: verified
---

IPv4（Internet Protocol version 4）和 IPv6（Internet Protocol version 6）是互联网协议（IP）的两个主要版本。IPv4 由 RFC 791 于 1981 年定义，是当前互联网的主力协议；IPv6 由 RFC 8200 于 2017 年正式标准化（取代 RFC 2460），是 IPv4 的继任者，旨在解决 IPv4 地址耗尽等根本性限制。

## 是什么

IP（Internet Protocol）是 TCP/IP 协议族中网际层的核心协议，负责将数据包从源主机路由到目标主机。IPv4 和 IPv6 是 IP 协议的两个版本，它们在地址格式、头部结构、功能特性上有显著差异，但核心职责——寻址和路由——保持一致。

## 为什么重要

- **IPv4 地址耗尽**：32 位地址空间仅能提供约 43 亿个地址，全球 IPv4 地址池已于 2011-2015 年间由五大 RIR（APNIC、RIPE NCC、ARIN、LACNIC、AfriNIC）陆续分配完毕
- **网络规模持续增长**：IoT 设备、移动终端、云计算等场景需要海量可路由地址
- **协议设计演进**：IPv6 在安全性、自动配置、路由效率等方面做了系统性改进

## 核心区别对比

### 地址空间

| 对比项 | IPv4 | IPv6 |
|--------|------|------|
| 地址长度 | 32 bits | 128 bits |
| 地址总数 | 约 4.3 × 10⁹（2³²） | 约 3.4 × 10³⁸（2¹²⁸） |
| 文本表示 | 点分十进制：`192.168.1.1` | 冒号分隔十六进制：`2001:0db8::1` |
| 地址分类 | A/B/C/D/E 类（已废弃，改用 CIDR） | 无类别，统一使用前缀长度（如 `/64`） |

> **关键结论**：IPv6 地址空间是 IPv4 的 2⁹⁶ 倍，约为地球上每粒沙子分配一个地址仍绰绰有余。

### 头部格式

**IPv4 头部**（RFC 791）：

- 固定 20 字节 + 可变长度 Options（最多 40 字节）
- 共 12 个字段：Version、IHL、Type of Service、Total Length、Identification、Flags、Fragment Offset、TTL、Protocol、Header Checksum、Source Address、Destination Address
- 包含 **Header Checksum**：每跳路由器需重新计算校验和
- 支持中间路由器 **分片**（Fragmentation）

**IPv6 头部**（RFC 8200）：

- 固定 40 字节，无 Options 字段
- 仅 8 个字段：Version、Traffic Class、Flow Label、Payload Length、Next Header、Hop Limit、Source Address、Destination Address
- **移除 Header Checksum**：依赖上层协议（TCP/UDP）和数据链路层校验，减少每跳处理开销
- 中间路由器 **不分片**：分片仅由源节点执行，通过 Fragment Extension Header 实现
- 可选功能通过 **Extension Headers**（扩展头部）实现：Hop-by-Hop Options、Routing、Fragment、Destination Options、Authentication Header (AH)、Encapsulating Security Payload (ESP)

### 地址类型

| 类型 | IPv4 | IPv6 |
|------|------|------|
| 单播（Unicast） | ✓ | ✓ |
| 组播（Multicast） | ✓ | ✓（增强，引入 Scope 字段） |
| 广播（Broadcast） | ✓（如 `255.255.255.255`） | **无**（由组播替代） |
| 任播（Anycast） | 非正式支持 | ✓（正式定义，语法与单播不可区分） |

**IPv6 特有地址类型**（RFC 4291）：

- **链路本地地址（Link-Local）**：`fe80::/10`，仅在同一链路上有效，用于邻居发现、无状态自动配置
- **唯一本地地址（ULA, Unique Local Address）**：`fc00::/7`，类似 IPv4 私有地址（`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`），但全局唯一概率极高
- **回环地址**：`::1/128`（对应 IPv4 `127.0.0.1`）
- **未指定地址**：`::/128`（对应 IPv4 `0.0.0.0`）

### 地址配置

| 机制 | IPv4 | IPv6 |
|------|------|------|
| 手动配置 | ✓ | ✓ |
| DHCP | DHCP（RFC 2131） | DHCPv6（RFC 8415） |
| 自动配置 | 无（需 DHCP 或手动） | **SLAAC**（无状态地址自动配置，RFC 4862） |
| 邻居发现 | ARP（Address Resolution Protocol） | **NDP**（Neighbor Discovery Protocol，RFC 4861） |

**SLAAC 工作原理**：

1. 主机启动后生成链路本地地址（`fe80::/64` + 接口 ID）
2. 发送 Router Solicitation（RS）消息
3. 路由器回复 Router Advertisement（RA），包含网络前缀
4. 主机组合前缀 + 接口 ID（通常基于 Modified EUI-64 或随机生成）形成全局单播地址

**NDP 替代 ARP**：

- IPv4 使用 ARP（基于广播）将 IP 地址解析为 MAC 地址
- IPv6 使用 NDP（基于 ICMPv6 和组播）完成地址解析、路由器发现、前缀发现、重定向、邻居不可达检测

### 分片机制

| 对比项 | IPv4 | IPv6 |
|--------|------|------|
| 分片执行者 | 源主机 **和** 中间路由器 | **仅源主机** |
| 分片信息位置 | IPv4 头部内（Identification、Flags、Fragment Offset） | Fragment Extension Header |
| MTU 发现 | 可选（通过 DF 标志 + ICMP） | **必须**实现路径 MTU 发现（PMTUD，RFC 8201） |
| 最小 MTU | 576 字节 | 1280 字节 |
| 重组位置 | 目标主机 | 目标主机 |
| 重组超时 | 未明确规定 | 60 秒 |

### 安全特性

| 对比项 | IPv4 | IPv6 |
|--------|------|------|
| IPsec 支持 | 可选扩展 | 协议设计时即考虑，但实现层面仍为可选（RFC 6434 不再强制要求所有节点实现） |
| 加密/认证 | 需额外部署 IPsec 或上层 TLS | 同上，IPsec 原生集成但非强制 |

> **注意**：早期 RFC 2460 曾要求所有 IPv6 实现必须支持 IPsec，但 RFC 6434 修正了这一要求，因为 IPsec 的密钥管理和部署复杂度使其难以强制执行。

## 应用场景

### IPv4 当前主要场景

- **现有互联网基础设施**：绝大多数服务器、路由器、终端设备仍运行 IPv4
- **企业内网**：大量使用 NAT + 私有地址（RFC 1918）的内部网络
- **遗留系统**：不支持 IPv6 的旧设备、旧软件
- **CDN 和云服务**：多数同时支持双栈，但 IPv4 仍是主要流量载体

### IPv6 主要应用场景

- **移动互联网**：LTE/5G 网络原生支持 IPv6，运营商广泛部署（如 Verizon、T-Mobile 已实现 IPv6 流量占比 >90%）
- **IoT（物联网）**：海量设备需要独立可路由地址，IPv6 的 SLAAC 和 6LoWPAN（RFC 4944）使其成为 IoT 首选
- **内容分发网络**：Google、Facebook、Cloudflare 等已大规模启用 IPv6
- **政府和新网络建设**：中国、印度、欧盟等推动 IPv6 规模部署，新建网络优先采用 IPv6
- **数据中心内部**：部分云服务商在数据中心内部使用 IPv6 简化管理

### 过渡技术

| 技术 | 说明 |
|------|------|
| **双栈（Dual Stack）** | 主机/路由器同时运行 IPv4 和 IPv6 协议栈，最推荐的过渡方案 |
| **NAT64/DNS64** | 允许 IPv6 -only 客户端访问 IPv4 服务器（RFC 6146、RFC 6147） |
| **464XLAT** | 在移动网络中实现 IPv4 访问的转换方案（RFC 6877） |
| **DS-Lite** | 双栈精简版，通过隧道将 IPv4 流量封装在 IPv6 中传输（RFC 6333） |
| **6rd** | IPv6 快速部署，通过 IPv4 网络隧道传输 IPv6（RFC 5969） |
| **Teredo** | 通过 UDP/IPv4 隧道传输 IPv6，适用于 NAT 后的主机（RFC 4380，已废弃） |
| **ISATAP** | 站内自动隧道地址协议，在企业内网中过渡使用 |

## 限制与注意事项

### IPv4 的限制

1. **地址耗尽**：公网 IPv4 地址已分配完毕，依赖 NAT 缓解但破坏端到端模型
2. **NAT 的副作用**：
   - 破坏端到端通信，P2P 应用需要 STUN/TURN/ICE 等穿透技术
   - 增加网络复杂度，故障排查困难
   - 某些应用层协议（如 FTP、SIP）需要 ALG（Application Layer Gateway）协助
3. **路由表膨胀**：BGP 全球路由表已超过 90 万条（2024 年数据），影响路由器性能
4. **头部处理开销**：Header Checksum 需每跳重新计算；Options 处理效率低
5. **缺乏原生安全**：IPsec 为可选扩展，实际部署率低
6. **无原生自动配置**：依赖 DHCP 服务器，增加运维成本

### IPv6 的限制

1. **部署惯性**：
   - 全球 IPv6 采用率约 40-45%（Google 统计数据，2024 年），仍有大量网络未启用
   - 许多企业防火墙、安全设备对 IPv6 支持不完善
2. **双栈运维复杂度**：
   - 同时维护 IPv4 和 IPv6 两套协议栈，配置和故障排查成本翻倍
   - 需要确保两套协议的安全策略一致
3. **扩展头部处理**：
   - 部分防火墙/IDS 无法正确解析 Extension Headers 链
   - 某些网络会丢弃携带 Extension Headers 的包（出于安全考虑）
4. **分片限制**：
   - 中间路由器不分片，若路径 MTU 发现失败（ICMPv6 Packet Too Big 被防火墙拦截），会导致连接中断
   - RFC 8200 要求最小 MTU 为 1280 字节，某些链路（如隧道）可能不满足
5. **隐私问题**：
   - 基于 MAC 地址生成的接口 ID（Modified EUI-64）可能暴露设备身份
   - 已通过临时地址（Privacy Extensions，RFC 4941）缓解，但并非所有系统默认启用
6. **兼容性问题**：
   - 部分老旧设备、嵌入式系统不支持 IPv6
   - 某些应用代码硬编码 IPv4 地址格式（如正则表达式验证），需改造
7. **任播路由扩展**：
   - 全球范围内的任播集合（如 DNS 根服务器）需要在全网维护独立路由条目，扩展性受限

### 常见误区

1. **"IPv6 比 IPv4 更安全"**：IPv6 协议设计时考虑了安全，但 IPsec 并非强制实现。安全性取决于具体部署，而非协议版本本身
2. **"IPv6 不需要防火墙"**：IPv6 地址空间大不等于不可扫描。攻击者可通过组播、NDP、已知前缀等方式发现主机，防火墙仍然必要
3. **"启用 IPv6 后 IPv4 可以关闭"**：目前全球仍有大量服务仅支持 IPv4，完全关闭 IPv4 会导致访问受限
4. **"NAT 是安全机制"**：NAT 的主要目的是地址复用，不是安全。IPv6 环境下应使用状态防火墙替代 NAT 的安全幻觉
5. **"IPv6 地址太长，人类无法记忆"**：IPv6 设计目标不是供人类记忆，而是通过 DNS 解析。实际使用中用户几乎不直接接触 IP 地址

## 相关概念

- **CIDR（无类别域间路由）**：IPv4/IPv6 通用的地址分配和路由聚合方法
- **NAT（网络地址转换）**：IPv4 地址耗尽的临时缓解方案
- **SLAAC（无状态地址自动配置）**：IPv6 特有的地址自动配置机制
- **NDP（邻居发现协议）**：IPv6 中替代 ARP 的协议族
- **IPsec**：网络层安全协议族，在 IPv4 和 IPv6 中均可使用
- **6LoWPAN**：在低功耗无线个人区域网（如 IEEE 802.15.4）上运行 IPv6 的适配层
- **QUIC/HTTP/3**：新一代传输协议，与 IP 版本无关，但受益于 IPv6 的端到端连接

## 参考资料

- [RFC 791 - Internet Protocol (IPv4)](https://www.rfc-editor.org/rfc/rfc791) — IPv4 协议原始规范（1981）
- [RFC 8200 - Internet Protocol, Version 6 (IPv6)](https://www.rfc-editor.org/rfc/rfc8200) — IPv6 协议规范（2017，取代 RFC 2460）
- [RFC 4291 - IP Version 6 Addressing Architecture](https://www.rfc-editor.org/rfc/rfc4291) — IPv6 地址架构
- [RFC 4861 - Neighbor Discovery for IP version 6](https://www.rfc-editor.org/rfc/rfc4861) — IPv6 邻居发现协议
- [RFC 4862 - IPv6 Stateless Address Autoconfiguration](https://www.rfc-editor.org/rfc/rfc4862) — IPv6 无状态地址自动配置
- [RFC 8201 - Path MTU Discovery for IP version 6](https://www.rfc-editor.org/rfc/rfc8201) — IPv6 路径 MTU 发现
- [RFC 6434 - IPv6 Node Requirements](https://www.rfc-editor.org/rfc/rfc6434) — IPv6 节点要求（修正 IPsec 强制性）
- [RFC 4941 - Privacy Extensions for SLAAC](https://www.rfc-editor.org/rfc/rfc4941) — IPv6 隐私扩展
- [RFC 6146 - NAT64](https://www.rfc-editor.org/rfc/rfc6146) — IPv6 到 IPv4 的网络地址和协议转换
- [Google IPv6 Statistics](https://www.google.com/intl/en/ipv6/statistics.html) — 全球 IPv6 采用率统计
- [APNIC IPv6 Deployment](https://www.apnic.net/about-apnic/corporate-documents/documents/resource-guidelines/address-policies/ipv6-deployment-statistics/) — APNIC IPv6 部署统计
