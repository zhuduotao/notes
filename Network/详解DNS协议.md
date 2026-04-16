---
created: 2026-04-15
updated: 2026-04-15
tags:
  - network/protocol
  - network/dns
  - internet/infrastructure
aliases:
  - DNS
  - Domain Name System
  - 域名系统
source_type: mixed
source_urls:
  - https://www.rfc-editor.org/rfc/rfc1034
  - https://www.rfc-editor.org/rfc/rfc1035
  - https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml
  - https://developer.mozilla.org/en-US/docs/Glossary/DNS
status: verified
---

DNS（Domain Name System，域名系统）是互联网上用于将人类可读的域名（如 `www.example.com`）映射为机器可识别的 IP 地址（如 `93.184.216.34`）的**分层分布式命名系统**。它运行在 OSI 模型的**应用层**，默认使用 **UDP/TCP 端口 53**，是互联网基础设施的核心组件之一。

## 为什么需要 DNS

在 DNS 出现之前，ARPANET 使用一个名为 `HOSTS.TXT` 的集中式文本文件来维护主机名到 IP 地址的映射，由 Stanford Research Institute 的 NIC 统一维护。随着网络规模的指数级增长，这种方案暴露出严重问题：

- **带宽开销**：每次更新需要所有主机重新下载文件，带宽消耗与主机数量的平方成正比
- **命名冲突**：所有主机名必须全局唯一，无法实现本地化管理
- **扩展性差**：无法支持邮件、服务等更复杂的命名需求

Paul Mockapetris 于 1983 年设计了 DNS，原始规范发布在 RFC 882/883 中，后由 **RFC 1034**（概念与设施）和 **RFC 1035**（实现与规范）于 1987 年 11 月取代并确立为 Internet Standard。

## 核心架构

DNS 由三个核心组件构成：

| 组件 | 角色 | 说明 |
|------|------|------|
| **域名空间（Domain Name Space）** | 数据结构 | 树状分层命名空间，每个节点关联资源记录（RR） |
| **名称服务器（Name Server）** | 服务端 | 存储域名空间的部分数据，响应查询请求 |
| **解析器（Resolver）** | 客户端 | 向名称服务器发起查询，将结果返回给应用程序 |

### 域名空间（Domain Name Space）

域名空间是一棵**倒置的树形结构**，根节点在最上方：

```
.                          ← 根（root），空标签
├── com.                   ← 顶级域（TLD）
│   └── example.           ← 二级域
│       └── www            ← 主机/子域
├── org.
│   └── wikipedia.
│       └── www
└── arpa.
    └── in-addr            ← 反向解析专用域
```

**关键规则**（RFC 1035 §2.3.4）：

- 每个标签（label）最长 **63 个字符**
- 完整域名（含标签长度字节）最长 **255 个字节**
- 标签仅允许字母、数字和连字符（**LDH 规则**），连字符不能出现在首尾
- 域名比较**不区分大小写**（`Example.COM` = `example.com`）
- 以 `.` 结尾的域名称为**绝对域名（FQDN）**，不以 `.` 结尾的为**相对域名**

### 区域（Zone）与委派（Delegation）

DNS 数据库按**区域（Zone）** 划分管理职责：

- **区域**：域名树中一段连续的子树，由单一管理机构负责
- **委派**：父区域将子域的管理权委托给另一组名称服务器
- **Glue Record（粘合记录）**：当子域的名称服务器本身位于该子域内时，父区域需要额外提供该名称服务器的 IP 地址，否则会产生循环依赖

例如 `example.com` 区域的管理员可以决定将 `dev.example.com` 委派给另一组名称服务器独立管理。

## 资源记录（Resource Records, RR）

每个域名节点关联一组资源记录，RR 是 DNS 数据的基本单位。

### RR 通用格式

```
                                    1  1  1  1  1  1
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    /                                               /
    /                      NAME                     /  ← 域名
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TYPE                     |  ← 记录类型（16 bit）
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     CLASS                     |  ← 协议族（16 bit）
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TTL                      |  ← 缓存生存时间（32 bit）
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                   RDLENGTH                    |  ← RDATA 长度（16 bit）
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--|
    /                     RDATA                     /  ← 记录数据（变长）
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

- **NAME**：该 RR 所属的域名
- **TYPE**：记录类型，决定 RDATA 的格式
- **CLASS**：协议族，Internet 使用 `IN`（值为 1）
- **TTL**：缓存时间（秒），0 表示禁止缓存
- **RDATA**：具体数据，格式因 TYPE 而异

### 常用记录类型

| 类型 | 值 | 名称 | 用途 | RDATA 格式 |
|------|-----|------|------|-----------|
| **A** | 1 | Address | IPv4 地址 | 32 位 IPv4 地址 |
| **AAAA** | 28 | IPv6 Address | IPv6 地址 | 128 位 IPv6 地址（RFC 3596） |
| **CNAME** | 5 | Canonical Name | 别名指向规范名称 | 一个域名 |
| **MX** | 15 | Mail Exchange | 邮件服务器 | 16 位优先级 + 域名（优先级越低越优先） |
| **NS** | 2 | Name Server | 区域的权威名称服务器 | 一个域名 |
| **SOA** | 6 | Start of Authority | 区域起始授权记录 | MNAME + RNAME + SERIAL + REFRESH + RETRY + EXPIRE + MINIMUM |
| **PTR** | 12 | Pointer | 反向解析指针 | 一个域名 |
| **TXT** | 16 | Text | 文本信息（SPF、DKIM 等） | 一个或多个字符串 |
| **SRV** | 33 | Service | 服务定位（RFC 2782） | 优先级 + 权重 + 端口 + 目标 |
| **CAA** | 257 | Certification Authority Authorization | 限制哪些 CA 可为该域名签发证书 |
| **DNSKEY** | 48 | DNS Key | DNSSEC 公钥（RFC 4034） |
| **DS** | 43 | Delegation Signer | DNSSEC 委派签名者 |

### SOA 记录详解

SOA（Start of Authority）记录标志一个区域的起始，每个区域有且仅有一条 SOA 记录：

| 字段 | 说明 |
|------|------|
| **MNAME** | 主名称服务器的域名 |
| **RNAME** | 管理员邮箱（`@` 替换为 `.`，如 `admin.example.com.` 表示 `admin@example.com`） |
| **SERIAL** | 区域版本号（32 位无符号整数），辅助服务器通过比较此值判断是否需要更新 |
| **REFRESH** | 辅助服务器检查主服务器更新的间隔（秒） |
| **RETRY** | 刷新失败后的重试间隔（秒） |
| **EXPIRE** | 无法联系主服务器时，区域数据过期时间（秒） |
| **MINIMUM** | 区域内所有 RR 的最小 TTL 下限 |

### CNAME 的重要限制

- 如果一个节点存在 CNAME 记录，则**该节点不能有其他任何类型的记录**（RFC 1034 §3.6.2）
- CNAME 可以链式指向另一个 CNAME，但应尽量避免（增加解析延迟）
- CNAME 不能出现在 MX 记录的交换器字段中

## DNS 消息格式

DNS 查询和响应使用统一的消息格式，包含 Header、Question、Answer、Authority 和 Additional 五个部分：

```
    +---------------------+
    |        Header       |  12 字节固定长度
    +---------------------+
    |       Question      |  查询问题（可变长）
    +---------------------+
    |        Answer       |  回答记录（可变长）
    +---------------------+
    |      Authority      |  权威记录（可变长）
    +---------------------+
    |      Additional     |  附加记录（可变长）
    +---------------------+
```

### Header 字段

```
    0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |                      ID                       |  16 bit：查询标识符
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |  各标志位
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |                    QDCOUNT                    |  16 bit：Question 数量
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |                    ANCOUNT                    |  16 bit：Answer 数量
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |                    NSCOUNT                    |  16 bit：Authority 数量
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
  |                    ARCOUNT                    |  16 bit：Additional 数量
  +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

**关键标志位**：

| 标志 | 名称 | 说明 |
|------|------|------|
| **QR** | Query/Response | 0 = 查询，1 = 响应 |
| **Opcode** | 操作码 | 0 = 标准查询，4 = NOTIFY，5 = UPDATE |
| **AA** | Authoritative Answer | 1 = 响应来自权威服务器 |
| **TC** | Truncated | 1 = 消息被截断（UDP 超过 512 字节时） |
| **RD** | Recursion Desired | 1 = 客户端请求递归查询 |
| **RA** | Recursion Available | 1 = 服务器支持递归 |
| **RCODE** | Response Code | 响应状态码（见下表） |

### RCODE（响应码）

| 值 | 名称 | 含义 |
|-----|------|------|
| 0 | NoError | 成功，无错误 |
| 1 | FormErr | 格式错误，服务器无法解析查询 |
| 2 | ServFail | 服务器内部错误 |
| 3 | NXDomain | 域名不存在 |
| 4 | NotImp | 服务器不支持该操作 |
| 5 | Refused | 服务器拒绝响应（如策略限制） |

### 名称压缩（Name Compression）

为减少消息大小，DNS 使用指针机制压缩重复出现的域名（RFC 1035 §4.1.4）：

- 指针以两个字节表示，最高两位为 `11`
- 剩余 14 位为指向消息中先前出现位置的偏移量
- 例如 `0xC00C` 表示指向消息偏移量 12 处的域名

## 域名解析流程

### 递归查询 vs 迭代查询

DNS 支持两种查询模式：

**递归查询（Recursive Query）**：
- 客户端向解析器发送一次请求，解析器负责完成所有后续查询
- 最终返回完整答案或错误
- 客户端设置 `RD = 1`

**迭代查询（Iterative Query）**：
- 服务器返回它能提供的最佳答案，可能是指向更接近目标的名称服务器的引用（referral）
- 客户端需要自行向引用的服务器继续查询
- RFC 1034 要求所有名称服务器必须支持迭代查询

### 完整解析示例

以解析 `www.wikipedia.org` 为例（假设解析器缓存为空）：

```
1. 客户端 → 本地递归解析器：查询 www.wikipedia.org 的 A 记录（RD=1）

2. 解析器 → 根服务器：查询 www.wikipedia.org（迭代）
   ← 根服务器返回：org 的 NS 记录 + glue

3. 解析器 → .org TLD 服务器：查询 www.wikipedia.org（迭代）
   ← TLD 服务器返回：wikipedia.org 的 NS 记录 + glue

4. 解析器 → wikipedia.org 权威服务器：查询 www.wikipedia.org（迭代）
   ← 权威服务器返回：www.wikipedia.org 的 A 记录（AA=1）

5. 解析器 → 客户端：返回 www.wikipedia.org 的 A 记录
```

### 缓存机制

- 解析器和递归服务器会缓存查询结果，缓存时长由 RR 的 **TTL** 决定
- TTL 为 0 的记录不应被缓存
- 缓存大幅减少了对根服务器的查询压力——实际中根服务器仅处理极小比例的请求
- SOA 记录的 MINIMUM 字段设定了区域内所有 RR 的最小 TTL 下限

## 传输协议

### 传统 DNS（Do53）

- **UDP 53**：默认查询传输方式，单次响应限制为 **512 字节**（RFC 1035 §4.2.1）
- **TCP 53**：用于区域传输（AXFR/IXFR）或响应超过 512 字节时（TC 标志置位后客户端重试）
- 如果 UDP 响应超过 512 字节，服务器会截断响应（TC=1），客户端应改用 TCP 重试

### 加密 DNS 协议

| 协议 | RFC | 端口 | 说明 |
|------|-----|------|------|
| **DoT**（DNS over TLS） | RFC 7858 | 853 | 在 TLS 连接上承载 DNS，端口固定 |
| **DoH**（DNS over HTTPS） | RFC 8484 | 443 | 通过 HTTPS 传输 DNS 查询，与 Web 流量混合 |
| **DoQ**（DNS over QUIC） | RFC 9250 | 853 | 基于 QUIC 协议，结合 TLS 1.3 和 UDP 特性 |
| **ODoH**（Oblivious DoH） | RFC 9230 | 443 | 通过代理层分离查询者身份与查询内容 |

## 反向解析（Reverse DNS）

反向解析将 IP 地址映射回域名，使用专用的 `in-addr.arpa`（IPv4）和 `ip6.arpa`（IPv6）域：

- **IPv4**：将 IP 地址的四个八位组**反转**后追加 `.in-addr.arpa.`
  - 例如 `192.0.2.1` → `1.2.0.192.in-addr.arpa.` 的 PTR 记录
- **IPv6**：将 128 位地址按**半字节（4 bit）反转**后追加 `.ip6.arpa.`
  - 例如 `2001:db8::1` → `1.0.0.0...8.b.d.0.1.0.0.2.ip6.arpa.` 的 PTR 记录

反向解析依赖 `in-addr.arpa` 域的委派结构，通常由 IP 地址的分配机构（如 ISP 或 RIR）管理。

## DNSSEC（DNS Security Extensions）

DNSSEC 通过数字签名解决 DNS 数据完整性和来源认证问题（RFC 4033/4034/4035）：

- **不加密**：DNSSEC 不提供机密性，数据仍为明文传输
- **签名链**：从根区域到叶子区域形成一条信任链（Chain of Trust）
- **关键记录类型**：
  - `DNSKEY`：区域签名公钥
  - `RRSIG`：资源记录的签名
  - `DS`：委派签名者，存储在父区域
  - `NSEC`/`NSEC3`：不存在性证明（证明某记录不存在）

## 常见误区

| 误区 | 事实 |
|------|------|
| "DNS 只负责域名到 IP 的映射" | DNS 还存储邮件路由（MX）、服务发现（SRV）、证书策略（CAA）等信息 |
| "DNS 查询总是走 UDP" | 区域传输（AXFR）和大响应强制使用 TCP；现代加密 DNS（DoT/DoH/DoQ）也使用 TCP 或 QUIC |
| "CNAME 和 A 记录可以共存" | 不可以，RFC 明确规定存在 CNAME 的节点不能有其他记录 |
| "DNS 是安全的" | 传统 DNS 无加密和认证，存在 DNS 劫持、缓存投毒等风险；需依赖 DNSSEC 或加密 DNS 协议 |
| "域名必须以字母开头" | 这是旧 HOSTS.TXT 规则；RFC 2181 放宽了限制，但为兼容性仍建议遵循 LDH 规则 |

## 与其他系统的关系

- **HTTP**：浏览器发起 HTTP 请求前必须先完成 DNS 解析，DNS 延迟直接影响页面加载时间
- **TLS/HTTPS**：TLS 握手需要目标 IP，因此 DNS 解析在 TLS 握手之前完成
- **CDN**：CDN 利用 DNS 的地理路由能力，为不同用户返回不同的 IP（就近接入）
- **邮件系统**：SMTP 通过查询 MX 记录确定邮件服务器，优先级决定投递顺序
- **操作系统**：`/etc/hosts`（Unix）或 `C:\Windows\System32\drivers\etc\hosts`（Windows）优先级高于 DNS

## 实用命令

```bash
# 查询 A 记录
dig example.com A

# 查询 MX 记录
dig example.com MX

# 查询所有记录
dig example.com ANY

# 跟踪完整解析链
dig example.com +trace

# 查询指定 DNS 服务器
dig @8.8.8.8 example.com

# 反向解析
dig -x 8.8.8.8

# 简短输出
dig example.com +short

# 查询 SOA 记录（查看区域信息）
dig example.com SOA
```

## 参考资料

- [RFC 1034 — Domain Names: Concepts and Facilities](https://www.rfc-editor.org/rfc/rfc1034)（DNS 概念与设施，Internet Standard）
- [RFC 1035 — Domain Names: Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)（DNS 实现与规范，Internet Standard）
- [RFC 2181 — Clarifications to the DNS Specification](https://www.rfc-editor.org/rfc/rfc2181)（DNS 规范澄清）
- [RFC 3596 — DNS Extensions to Support IP Version 6](https://www.rfc-editor.org/rfc/rfc3596)（AAAA 记录定义）
- [RFC 4033/4034/4035 — DNSSEC](https://www.rfc-editor.org/rfc/rfc4033)（DNS 安全扩展）
- [RFC 7858 — DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858)
- [RFC 8484 — DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484)
- [RFC 9250 — DNS over QUIC](https://www.rfc-editor.org/rfc/rfc9250)
- [IANA DNS Parameters Registry](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml)（记录类型、操作码、响应码等官方注册表）
- [MDN — DNS Glossary](https://developer.mozilla.org/en-US/docs/Glossary/DNS)
