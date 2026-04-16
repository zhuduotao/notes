---
created: 2026-04-15
updated: 2026-04-15
tags:
  - network
  - security
  - tls
  - ssl
  - https
  - certificate
  - pki
  - cryptography
aliases:
  - HTTPS 证书
  - SSL 证书
  - TLS 证书
  - X.509 证书
  - 数字证书
  - DV OV EV 证书
  - 证书算法
  - 证书提供商
source_type: mixed
source_urls:
  - 'https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Transport_Layer_Security'
  - 'https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/TLS'
  - 'https://datatracker.ietf.org/doc/html/rfc8446'
  - 'https://datatracker.ietf.org/doc/html/rfc5280'
  - 'https://cabforum.org/baseline-requirements-documents/'
  - 'https://letsencrypt.org/how-it-works/'
  - 'https://www.digicert.com/difference-between-dv-ov-and-ev-ssl-certificates'
status: verified
---

## 概述

HTTPS 证书（通常称为 SSL/TLS 证书）是数字证书的一种，基于 X.509 标准（[RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)），用于在 TLS 握手过程中验证服务器身份并建立加密通信通道。证书由受信任的证书颁发机构（Certificate Authority, CA）签发，将服务器的公钥与其域名绑定，使浏览器能够确认"正在连接的确实是想连接的服务器"。

HTTPS = HTTP + TLS，其中 TLS（Transport Layer Security）是 SSL（Secure Sockets Layer）的继任者。SSL 3.0 之后被 IETF 标准化为 TLS，当前最新版本为 TLS 1.3（[RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)，2018 年发布）。

## 证书的核心作用

TLS 证书在安全通信中提供三个关键保障：

- **加密（Encryption）**：客户端与服务器之间传输的数据在传输过程中被加密，防止第三方窃听
- **完整性（Integrity）**：传输中的数据无法被攻击者秘密修改而不被发现
- **身份认证（Authentication）**：服务器（可选客户端）能够向对方证明自己的身份

这三项保障共同防御中间人攻击（MITM），是现代 Web 安全的基础。

## 证书类型（按验证级别分类）

根据 CA/Browser Forum 基线要求，证书按验证严格程度分为三个级别：

### DV（Domain Validated）— 域名验证证书

| 维度 | 说明 |
|------|------|
| **验证内容** | 仅验证申请者对域名的控制权 |
| **验证方式** | DNS 记录验证、HTTP 文件验证或邮箱验证 |
| **签发速度** | 通常几分钟内自动完成 |
| **身份展示** | 证书中不包含组织信息 |
| **适用场景** | 个人博客、测试环境、内部系统、不收集敏感信息的网站 |
| **安全级别** | 最低 — 匿名实体也可获取 |

DV 证书是加密网站的"最低可行产品"，仅提供传输加密，不验证网站背后的实体身份。钓鱼网站也可以轻松获取 DV 证书。

### OV（Organization Validated）— 组织验证证书

| 维度 | 说明 |
|------|------|
| **验证内容** | 验证域名控制权 + 组织合法性（约 9 项检查） |
| **验证方式** | 除 DV 验证外，还需验证企业名称、类型、状态、物理地址等 |
| **签发速度** | 通常 1-3 个工作日 |
| **身份展示** | 证书中包含组织名称，可在证书详情中查看 |
| **适用场景** | 企业官网、登录页面、业务系统 |
| **安全级别** | 中等 — 提供组织身份背书 |

OV 证书在 DV 基础上增加了组织身份验证，CA 会核实企业注册信息和运营状态。

### EV（Extended Validation）— 扩展验证证书

| 维度 | 说明 |
|------|------|
| **验证内容** | DV + OV 全部验证 + 额外 9 项严格检查（共约 16 项） |
| **验证方式** | 包括企业运营存在性、物理地址、请求者 employment 状态电话核实、域名欺诈检查、联系人黑名单检查等 |
| **签发速度** | 通常 3-7 个工作日 |
| **身份展示** | 证书中包含详细的组织和法律实体信息 |
| **适用场景** | 银行、金融服务、电商平台、Fortune 500 企业、政府机构 |
| **安全级别** | 最高 — 最严格的身份验证 |

EV 证书提供最高级别的身份保障。据统计，DigiCert 验证团队每年拒绝约 3,750 个 EV 证书申请（部分因欺诈请求）。全球 2000 强企业中 81%、财富 500 强中 89%、全球前 100 大银行中 97 家使用 EV/OV 证书。

### 三种证书对比总结

| 对比维度 | DV | OV | EV |
|----------|-----|-----|-----|
| 域名控制验证 | ✅ | ✅ | ✅ |
| 组织合法性验证 | ❌ | ✅ | ✅ |
| 法律实体验证 | ❌ | ❌ | ✅ |
| 运营存在性验证 | ❌ | ❌ | ✅ |
| 电话核实请求者身份 | ❌ | ❌ | ✅ |
| 证书中显示组织信息 | ❌ | ✅ | ✅ |
| 签发速度 | 分钟级 | 1-3 天 | 3-7 天 |
| 价格 | 免费 ~ 低价 | 中等 | 较高 |
| 防钓鱼能力 | 弱 | 中 | 强 |

## 证书类型（按覆盖域名范围分类）

| 类型 | 说明 | 示例 |
|------|------|------|
| **单域名证书** | 保护一个特定域名 | `example.com` |
| **多域名证书（SAN/UCC）** | 保护多个不同域名，通过 Subject Alternative Name 字段指定 | `example.com`、`example.org`、`api.example.com` |
| **通配符证书** | 保护一个域名及其所有一级子域名 | `*.example.com` 可保护 `mail.example.com`、`shop.example.com` 等 |
| **多域名通配符证书** | 结合多域名和通配符能力 | `*.example.com` + `*.example.org` |

## TLS 加密算法体系

TLS 加密通过**密码套件（Cipher Suite）**定义，密码套件是一组算法的组合，包含：

- **密钥交换算法**：协商共享密钥
- **身份认证算法**：验证通信方身份
- **对称加密算法**：加密实际传输数据
- **消息认证码（MAC）/ AEAD**：保证数据完整性

### 密钥交换算法

| 算法 | 全称 | 说明 | 前向保密 | 现状 |
|------|------|------|----------|------|
| **RSA** | Rivest-Shamir-Adleman | 使用服务器 RSA 公钥加密预主密钥 | ❌ 无 | TLS 1.3 已移除 |
| **DHE** | Diffie-Hellman Ephemeral | 基于有限域的临时 Diffie-Hellman | ✅ 有 | 可用，但性能较差 |
| **ECDHE** | Elliptic Curve DHE | 基于椭圆曲线的临时 Diffie-Hellman | ✅ 有 | **推荐**，性能好 |
| **PSK** | Pre-Shared Key | 预共享密钥（TLS 1.3） | 取决于是否结合 ECDHE | TLS 1.3 支持 |

**前向保密（Forward Secrecy）**：即使服务器私钥在未来被泄露，也无法解密过去捕获的通信数据。ECDHE 和 DHE 提供前向保密，RSA 密钥交换不提供。

### 身份认证（签名）算法

| 算法 | 全称 | 密钥长度 | 性能 | 安全性 | 现状 |
|------|------|----------|------|--------|------|
| **RSA** | RSA 签名 | 2048-4096 位 | 验证快，签名慢 | 成熟，需足够密钥长度 | 广泛使用 |
| **ECDSA** | Elliptic Curve DSA | 256-384 位 | 快 | 高，同等安全下密钥更短 | **推荐** |
| **EdDSA** | Edwards-curve DSA | 256 位（Ed25519） | 最快 | 最高，抗侧信道攻击 | TLS 1.3 新增，推荐 |
| **DSA** | Digital Signature Algorithm | 1024-3072 位 | 慢 | 已不推荐 | TLS 1.3 已移除 |

### 对称加密算法

| 算法 | 模式 | 说明 | 现状 |
|------|------|------|------|
| **AES-128-GCM** | AEAD | 128 位密钥，GCM 模式，硬件加速好 | **推荐** |
| **AES-256-GCM** | AEAD | 256 位密钥，GCM 模式 | **推荐**，更高安全需求 |
| **ChaCha20-Poly1305** | AEAD | 流加密，无硬件 AES 加速时性能优于 AES | **推荐**，移动端友好 |
| **AES-128-CBC** | CBC | 旧模式，易受填充预言攻击 | TLS 1.3 已移除 |
| **AES-256-CBC** | CBC | 旧模式 | TLS 1.3 已移除 |
| **3DES** | CBC | 三重 DES，已过时 | 已废弃 |
| **RC4** | 流加密 | 存在严重漏洞 | 已废弃 |

**AEAD（Authenticated Encryption with Associated Data）**：同时提供加密和认证的算法模式，TLS 1.3 仅保留 AEAD 算法。

### 哈希算法

| 算法 | 输出长度 | 用途 | 现状 |
|------|----------|------|------|
| **SHA-256** | 256 位 | 证书签名、TLS 握手 HMAC | **推荐** |
| **SHA-384** | 384 位 | 高安全场景 | **推荐** |
| **SHA-1** | 160 位 | 旧证书签名 | **已废弃**，浏览器不再信任 |
| **MD5** | 128 位 | 极旧证书 | **已废弃**，存在碰撞攻击 |

### TLS 1.3 推荐密码套件

TLS 1.3 简化了密码套件设计，仅保留安全算法。以下是常见的 TLS 1.3 密码套件：

```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

### 哪个算法更好？

**综合推荐配置（2026 年最佳实践）**：

| 组件 | 推荐算法 | 理由 |
|------|----------|------|
| 密钥交换 | **ECDHE**（x25519 或 secp256r1 曲线） | 提供前向保密，性能好 |
| 签名算法 | **EdDSA（Ed25519）** 或 **ECDSA（P-256）** | 密钥短、速度快、安全性高 |
| 对称加密 | **AES-256-GCM** 或 **ChaCha20-Poly1305** | AEAD 模式，抗篡改 |
| 哈希算法 | **SHA-256** 或 **SHA-384** | 无已知碰撞攻击 |
| TLS 版本 | **TLS 1.3** | 最安全、最快、强制前向保密 |

**不推荐/已废弃的算法**：
- RSA 密钥交换（无前向保密）
- CBC 模式对称加密（填充预言攻击风险）
- SHA-1 / MD5 哈希（碰撞攻击）
- RC4 / 3DES / DES（已知漏洞）
- DSA 签名（性能差，TLS 1.3 已移除）
- TLS 1.0 / 1.1（已废弃，存在已知漏洞）
- TLS 1.2 中的静态 RSA 和 DHE 套件

## 证书链与信任模型

HTTPS 证书不是孤立存在的，而是通过**证书链（Certificate Chain）**建立信任：

```
根证书（Root CA）
  └── 中间证书（Intermediate CA）
        └── 服务器证书（Leaf/End-entity Certificate）
```

### 根证书（Root CA）

- 自签名证书，是信任链的起点
- 预装在操作系统和浏览器的信任存储中
- 通常离线保存，极少使用
- 例如：DigiCert Global Root G2、Let's Encrypt ISRG Root X1

### 中间证书（Intermediate CA）

- 由根证书签发，用于签发终端实体证书
- 可以有多个层级
- 服务器在 TLS 握手时必须提供中间证书（证书链补全）
- 如果根证书泄露，只需吊销对应的中间证书，不影响其他分支

### 服务器证书（Leaf Certificate）

- 直接保护网站域名的证书
- 由中间证书签发
- 包含域名、公钥、有效期、颁发者等信息

### 证书验证流程

1. 浏览器收到服务器证书
2. 检查证书是否在有效期内
3. 检查域名是否匹配（CN 和 SAN 字段）
4. 沿着证书链向上验证签名，直到到达受信任的根证书
5. 检查证书是否被吊销（通过 CRL 或 OCSP）
6. 检查证书透明度（CT）日志

## 主流证书提供商（CA）

### 免费 CA

| 提供商 | 证书类型 | 有效期 | 特点 |
|--------|----------|--------|------|
| **Let's Encrypt** | DV | 90 天 | 全球最大免费 CA，支持 ACME 自动签发和续期，由 ISRG 非营利组织运营 |
| **ZeroSSL** | DV | 90 天 | 支持 ACME 协议，提供免费和付费计划 |
| **Google Trust Services** | DV | 90 天 | Google 运营的公共 CA，部分服务免费 |
| **Buypass** | DV | 180 天 | 挪威 CA，提供免费 DV 证书 |

### 商业 CA

| 提供商 | 证书类型 | 特点 |
|--------|----------|------|
| **DigiCert** | DV / OV / EV | 全球最大商业 CA，收购了 Symantec、GeoTrust、Thawte、RapidSSL，根证书覆盖率 99.9% |
| **Sectigo**（原 Comodo CA） | DV / OV / EV | 市场份额最大的商业 CA 之一，性价比高 |
| **GlobalSign** | DV / OV / EV | 日本 CA，企业级服务，IoT 证书能力强 |
| **GoDaddy** | DV / OV / EV | 域名注册商旗下 CA，适合小型企业 |
| **Entrust** | DV / OV / EV | 老牌 CA，金融和政府行业使用广泛（注：2024 年被主要浏览器逐步取消信任） |

### Let's Encrypt 工作原理

Let's Encrypt 通过 ACME 协议（[RFC 8555](https://tools.ietf.org/html/rfc8555)）实现自动化证书签发：

1. **域名验证**：ACME 客户端向 CA 证明对域名的控制权（DNS 记录或 HTTP 文件）
2. **多视角验证**：Let's Encrypt 从多个网络视角并行验证，防止中间人攻击
3. **证书签发**：验证通过后，客户端提交 CSR（Certificate Signing Request），CA 签发证书
4. **证书透明性**：签发的证书自动提交到公共 CT 日志
5. **自动续期**：证书 90 天有效期，建议通过 cron 定时任务自动续期

## 证书签发流程

1. **生成密钥对**：在服务器上生成私钥和公钥
2. **创建 CSR**：生成 Certificate Signing Request，包含域名、组织信息和公钥
3. **提交给 CA**：将 CSR 提交给证书颁发机构
4. **域名/组织验证**：CA 按证书类型执行相应验证
5. **签发证书**：CA 用其私钥对证书签名并返回
6. **安装证书**：将证书和中间证书链安装到服务器
7. **配置服务器**：配置 Web 服务器使用证书和私钥
8. **测试验证**：使用 SSL Labs 等工具测试配置

## 证书透明度（Certificate Transparency）

证书透明度（CT）是一个开放框架，用于监控和审计 SSL 证书的签发：

- **目的**：防止 CA 错误或恶意签发证书
- **机制**：所有公开信任的证书必须记录到公共 CT 日志中
- **验证**：浏览器会检查证书是否包含有效的 SCT（Signed Certificate Timestamp）
- **查询**：可通过 [crt.sh](https://crt.sh/) 等工具查询域名下的所有证书

## 证书吊销机制

当证书私钥泄露或证书信息变更时，需要吊销证书：

| 机制 | 说明 | 优缺点 |
|------|------|--------|
| **CRL**（Certificate Revocation List） | CA 定期发布吊销列表 | 简单但列表可能很大，更新有延迟 |
| **OCSP**（Online Certificate Status Protocol） | 实时查询证书状态 | 实时性好，但增加延迟和隐私问题 |
| **OCSP Stapling** | 服务器代为查询 OCSP 响应并附加到 TLS 握手 | 解决隐私和性能问题，**推荐** |
| **CRLSets / OneCRL** | 浏览器内置的压缩吊销列表 | 快速但覆盖有限 |

## 常见误区

### 误区 1：SSL 和 TLS 是两种不同的东西

SSL 是 TLS 的前身。SSL 1.0 从未公开发布，SSL 2.0/3.0 存在严重漏洞。2015 年 IETF 正式废弃 SSL 3.0。现在所谓的"SSL 证书"实际上都是 TLS 证书，只是习惯上沿用了旧称。

### 误区 2：有 HTTPS 锁标志就是安全的网站

DV 证书任何人都可以获取，包括钓鱼网站。锁标志只表示连接是加密的，不表示网站本身可信。需要查看证书详情中的组织信息（OV/EV）来判断网站身份。

### 误区 3：证书有效期越长越好

自 2020 年 9 月起，Apple 要求公开信任的 TLS 证书有效期不超过 398 天（约 13 个月）。Let's Encrypt 固定为 90 天。短有效期可以降低私钥泄露风险，并推动自动化管理。

### 误区 4：通配符证书可以保护所有子域名

通配符证书 `*.example.com` 只保护一级子域名，不保护多级子域名。例如 `*.example.com` 可以保护 `mail.example.com`，但不能保护 `sub.mail.example.com`。

### 误区 5：TLS 1.3 比 TLS 1.2 慢

恰恰相反。TLS 1.3 握手只需 1-RTT（首次连接），支持 0-RTT 恢复连接，比 TLS 1.2 的 2-RTT 更快。同时移除了不安全的旧算法，减少了协商开销。

## 最佳实践

### 服务器配置

- 使用 Mozilla [SSL Configuration Generator](https://ssl-config.mozilla.org/) 生成推荐配置
- 优先启用 TLS 1.3，兼容 TLS 1.2
- 禁用 TLS 1.0、1.1 和 SSL 所有版本
- 启用 OCSP Stapling
- 配置 HSTS（HTTP Strict Transport Security）响应头
- 将 HTTP 301 重定向到 HTTPS

### 证书管理

- 使用 ACME 客户端（如 Certbot）自动化证书签发和续期
- 监控证书过期时间，设置告警
- 定期轮换密钥对
- 使用证书透明度日志监控异常签发
- 私钥妥善保管，不要提交到版本控制系统

### HSTS 配置示例

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

- `max-age=63072000`：两年内强制使用 HTTPS
- `includeSubDomains`：所有子域名同样强制 HTTPS
- `preload`：加入浏览器 HSTS 预加载列表，首次访问也安全

提交到 [HSTS Preload List](https://hstspreload.org/) 后，浏览器会在首次访问前就强制使用 HTTPS，防止 SSL stripping 攻击。

## 相关概念

- **PKI（Public Key Infrastructure）**：公钥基础设施，管理密钥和证书的系统
- **CSR（Certificate Signing Request）**：证书签名请求，包含申请者的公钥和身份信息
- **ACME（Automatic Certificate Management Environment）**：自动化证书管理协议
- **mTLS（Mutual TLS）**：双向 TLS，客户端和服务器都验证证书
- **SNI（Server Name Indication）**：TLS 扩展，允许同一 IP 托管多个 HTTPS 域名

## 参考资料

- [MDN - Transport Layer Security (TLS)](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Transport_Layer_Security)
- [MDN - TLS Configuration Guide](https://developer.mozilla.org/en-US/docs/Web/Security/Practical_implementation_guides/TLS)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5280 - X.509 Certificate Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [RFC 8555 - ACME Protocol](https://tools.ietf.org/html/rfc8555)
- [CA/Browser Forum Baseline Requirements](https://cabforum.org/working-groups/server/baseline-requirements/documents/)
- [Let's Encrypt - How It Works](https://letsencrypt.org/how-it-works/)
- [DigiCert - DV vs OV vs EV Certificates](https://www.digicert.com/difference-between-dv-ov-and-ev-ssl-certificates)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [SSL Labs - Server Test](https://www.ssllabs.com/ssltest/)
- [HSTS Preload List](https://hstspreload.org/)
- [Certificate Transparency - crt.sh](https://crt.sh/)
