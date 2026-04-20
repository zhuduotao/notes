---
created: '2026-04-18'
updated: '2026-04-18'
tags:
  - network/http
  - network/proxy
  - security/ip
  - backend/nginx
  - architecture/load-balancer
aliases:
  - 获取用户真实IP
  - 真实IP获取方案
  - X-Forwarded-For
  - Forwarded Header
  - PROXY Protocol
  - 多层代理IP传递
source_type: mixed
source_urls:
  - 'https://datatracker.ietf.org/doc/html/rfc7239'
  - >-
    https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Forwarded-For
  - 'https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/'
  - 'https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt'
status: verified
---

## 为什么需要获取用户真实 IP

当客户端请求经过代理、负载均衡器、CDN 或多层反向代理后，服务端直接看到的 TCP 连接来源 IP 通常是**最后一跳代理的 IP**，而非客户端真实 IP。真实 IP 在以下场景至关重要：

- **访问控制**：IP 白名单/黑名单、地域限制
- **限流与反滥用**：基于真实 IP 的速率限制、防刷策略
- **审计与日志**：合规要求、故障排查、用户行为分析
- **个性化服务**：地域化内容、语言选择、风控评分

## 核心机制概览

| 方案 | 层级 | 标准化程度 | 适用场景 |
|------|------|-----------|---------|
| `X-Forwarded-For` | HTTP 头部 | 事实标准（非 RFC） | 最广泛使用，兼容性好 |
| `Forwarded` | HTTP 头部 | RFC 7239 标准 | 标准化替代方案 |
| `X-Real-IP` | HTTP 头部 | 非标准（Nginx 推广） | 单层代理场景 |
| PROXY Protocol | TCP 层 | HAProxy 规范（v1/v2） | TCP/SSL/非 HTTP 协议 |
| `CF-Connecting-IP` 等 | HTTP 头部 | 厂商私有 | 特定 CDN/云厂商 |

## 方案一：X-Forwarded-For（XFF）

### 定义与语法

`X-Forwarded-For` 是事实标准的请求头，用于标识通过代理服务器连接的客户端原始 IP 地址[^mdn-xff]。

```http
X-Forwarded-For: <client>, <proxy1>, <proxy2>, ..., <proxyN>
```

- **最左侧**：发起请求的原始客户端 IP
- **中间**：依次经过的每个代理的 IP
- **最右侧**：最近一次代理的 IP

### 工作原理

每经过一层代理，该代理会将**与它直连的上游 IP**追加到 XFF 末尾：

```
客户端 (203.0.113.195)
  → 代理1 (2001:db8::1) 添加: X-Forwarded-For: 203.0.113.195
    → 代理2 (198.51.100.178) 追加: X-Forwarded-For: 203.0.113.195, 2001:db8::1
      → 服务端看到: X-Forwarded-For: 203.0.113.195, 2001:db8::1, 198.51.100.178
```

### 安全与解析要点

**⚠️ 核心安全原则**：XFF 中只有**受信任代理添加的部分**可信赖，其余部分可被客户端伪造[^mdn-xff]。

**解析规则**：

1. **多个 XFF 头部**：必须合并处理，不能只取其中一个
   - 方法：将所有 XFF 值用逗号连接后分割，或分别分割后合并
2. **选择不可信 IP**（仅用于统计等非安全场景）：
   - 从左侧开始，选择第一个**有效且非私有/内部**的 IP
3. **选择可信 IP**（用于限流、访问控制等安全场景）：
   - **可信代理计数法**：配置已知代理数量，从右侧向左数 `代理数 - 1` 个位置
   - **可信代理列表法**：配置可信代理 IP/CIDR，从右侧向左跳过所有匹配项，第一个不匹配的即为目标 IP

### Nginx 配置示例

```nginx
# 从可信代理获取真实 IP
http {
    # 定义可信代理地址范围
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from 172.16.0.0/12;
    set_real_ip_from 192.168.0.0/16;

    # 告诉 Nginx 从 X-Forwarded-For 中提取真实 IP
    real_ip_header X-Forwarded-For;

    # 替换 $remote_addr 为真实 IP
    real_ip_recursive on;
}
```

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 获取到内网 IP | 客户端或内部代理添加了私有地址 | 使用可信代理列表法，跳过已知内网段 |
| 多个 XFF 头解析错误 | 只读取了其中一个头部 | 合并所有 XFF 头部值后统一解析 |
| 限流被绕过 | 使用了不可信的 XFF 值 | 仅使用可信代理添加的 IP 进行安全决策 |

## 方案二：Forwarded 头部（RFC 7239）

### 定义

`Forwarded` 是 IETF RFC 7239 定义的标准 HTTP 头部，旨在替代 `X-Forwarded-*` 系列非标准头部[^rfc7239]。

### 语法

```http
Forwarded: for=<client>;proto=<protocol>;by=<proxy>;host=<host>
Forwarded: for=192.0.2.43, for="[2001:db8:cafe::17]", for=unknown
```

### 参数说明

| 参数 | 含义 | 示例 |
|------|------|------|
| `for` | 发起请求的客户端或上游代理 | `for=192.0.2.60` |
| `by` | 接收请求的代理接口 | `by=203.0.113.43` |
| `proto` | 使用的协议 | `proto=http` 或 `proto=https` |
| `host` | 原始 Host 头部值 | `host=example.com` |

### 节点标识符格式

- **IPv4**：`for=192.0.2.43`
- **IPv6**：必须用方括号和引号包裹 `for="[2001:db8:cafe::17]:4711"`
- **未知**：`for=unknown`
- **混淆标识**：`for=_gazonk`（以 `_` 开头，用于隐私保护）

### 与 X-Forwarded-For 的关系

- `Forwarded` 将 `X-Forwarded-For`、`X-Forwarded-Proto`、`X-Forwarded-Host` 等信息整合到单一头部
- 支持将信息按代理分组，可明确知道哪些信息属于同一跳
- RFC 7239 建议：如果代理收到 `X-Forwarded-*` 头部，应尝试转换为 `Forwarded` 格式[^rfc7239]

### 转换示例

```
# X-Forwarded-For 原始值
X-Forwarded-For: 192.0.2.43, 2001:db8:cafe::17

# 转换为 Forwarded
Forwarded: for=192.0.2.43, for="[2001:db8:cafe::17]"
```

### 安全考虑

RFC 7239 明确指出[^rfc7239]：

- `Forwarded` 头部**不能被信赖为正确**，可能被路径上任何节点（包括客户端）修改
- 确保正确性的方法：验证代理并将其加入白名单
- **不应复制到响应消息中**，否则会向客户端暴露整个代理链
- 默认配置应使用**混淆标识符**保护隐私

## 方案三：PROXY Protocol

### 定义

PROXY Protocol 由 HAProxy 提出，在 **TCP 连接建立时**传递客户端连接信息，不依赖 HTTP 头部[^haproxy-proxy]。

### 适用场景

- TCP/UDP 负载均衡（非 HTTP 协议）
- SSL/TLS 终止后需要传递真实 IP
- WebSocket、SPDY、HTTP/2 等场景
- 需要比 HTTP 头部更可靠的 IP 传递机制

### 工作原理

负载均衡器在 TCP 连接建立后、应用数据之前，发送一行 PROXY Protocol 头部：

```
# v1 格式（人类可读）
PROXY TCP4 203.0.113.195 198.51.100.10 12345 443\r\n

# v2 格式（二进制，更高效）
\x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A\x21...
```

包含信息：协议族、客户端 IP、代理 IP、客户端端口、代理端口。

### Nginx 配置

```nginx
# 启用 PROXY Protocol 接收
http {
    server {
        listen 80   proxy_protocol;
        listen 443  ssl proxy_protocol;

        # 使用 RealIP 模块替换 $remote_addr
        set_real_ip_from 10.0.0.0/8;
        real_ip_header proxy_protocol;
    }
}

# 传递给上游服务器
http {
    proxy_set_header X-Real-IP       $proxy_protocol_addr;
    proxy_set_header X-Forwarded-For $proxy_protocol_addr;
}

# Stream (TCP) 场景
stream {
    server {
        listen 12345 ssl proxy_protocol;
        proxy_pass   backend.example.com:12345;
        proxy_protocol on;  # 向后传递 PROXY Protocol
    }
}
```

### 版本要求

| 功能 | Nginx Open Source | Nginx Plus |
|------|------------------|------------|
| PROXY Protocol v2 | 1.13.11+ | R16+ |
| HTTP 支持 | 1.5.12+ | R3+ |
| TCP 客户端侧 | 1.9.3+ | R7+ |
| TCP 完整支持 | 1.11.4+ | R11+ |

> RealIP 模块在 Nginx Open Source 中默认不包含，需要编译时添加 `--with-http_realip_module` 和 `--with-stream_realip_module`。Nginx Plus 无需额外步骤。

## 方案四：X-Real-IP

### 定义

`X-Real-IP` 是由 Nginx 推广的非标准头部，通常用于**单层代理**场景，只传递直接客户端 IP[^nginx-forwarded]。

### 与 X-Forwarded-For 的区别

| 特性 | X-Real-IP | X-Forwarded-For |
|------|-----------|-----------------|
| 传递内容 | 仅直接客户端 IP | 完整代理链 IP 列表 |
| 多层代理 | 每层可能覆盖 | 每层追加，保留完整链 |
| 标准化 | 非标准 | 事实标准 |
| 适用场景 | 简单单层代理 | 复杂多层架构 |

### Nginx 配置

```nginx
# 作为代理时设置
location / {
    proxy_pass http://backend;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

## 方案五：云厂商/CDN 私有头部

各大云厂商和 CDN 服务通常提供自己的头部来传递真实 IP：

| 厂商 | 头部名称 | 说明 |
|------|---------|------|
| Cloudflare | `CF-Connecting-IP` | Cloudflare 客户端 IP |
| Cloudflare | `CF-Connecting-IPv6` | Cloudflare 客户端 IPv6 |
| AWS CloudFront | `CloudFront-Viewer-Address` | 查看者 IP:端口 |
| AWS ELB/ALB | `X-Forwarded-For` | 标准 XFF 行为 |
| 阿里云 CDN | `Cdn-Src-Ip` | 回源客户端 IP |
| 腾讯云 CDN | `X-Forwarded-For` | 标准 XFF 行为 |

### Cloudflare 示例

```nginx
# 信任 Cloudflare 并获取真实 IP
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
# ... 更多 Cloudflare IP 段
real_ip_header CF-Connecting-IP;
```

> Cloudflare IP 段列表：https://www.cloudflare.com/ips/

## 多层路由架构中的最佳实践

### 典型架构

```
客户端 → CDN → WAF → 负载均衡器 → API Gateway → 应用服务器
```

### 推荐方案

1. **最外层（CDN/WAF）**：
   - 使用厂商私有头部（如 `CF-Connecting-IP`）或设置初始 XFF
   - 清理客户端可能伪造的 XFF 头部

2. **中间层（负载均衡器/API Gateway）**：
   - 追加自身 IP 到 XFF 或使用 `Forwarded` 头部
   - 或使用 PROXY Protocol 向后传递

3. **应用层**：
   - 配置可信代理列表
   - 从可信位置提取真实 IP
   - 记录 `$realip_remote_addr`（最后直连代理）用于调试

### Nginx 多层配置完整示例

```nginx
http {
    # 定义所有可信代理的 CIDR
    set_real_ip_from 10.0.0.0/8;
    set_real_ip_from 172.16.0.0/12;
    set_real_ip_from 192.168.0.0/16;
    set_real_ip_from 100.64.0.0/10;  # CGNAT

    # 使用 X-Forwarded-For
    real_ip_header X-Forwarded-For;

    # 递归解析，跳过所有可信代理 IP
    real_ip_recursive on;

    # 日志格式同时记录真实 IP 和直连代理 IP
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'real_ip=$remote_addr proxy=$realip_remote_addr';

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            # 向后传递真实 IP
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;
        }
    }
}
```

### 应用层代码示例

**Node.js (Express)**：

```javascript
// 启用 trust proxy，Express 会自动处理 X-Forwarded-For
app.set('trust proxy', true);

// 或使用可信代理列表
app.set('trust proxy', '10.0.0.0/8, 172.16.0.0/12');

// 获取真实 IP
const clientIp = req.ip;  // Express 已处理
```

**Go (net/http)**：

```go
func GetRealIP(r *http.Request, trustedProxies []string) string {
    xff := r.Header.Get("X-Forwarded-For")
    if xff == "" {
        return r.RemoteAddr
    }

    ips := strings.Split(xff, ",")
    // 从右向左查找第一个不在可信列表中的 IP
    for i := len(ips) - 1; i >= 0; i-- {
        ip := strings.TrimSpace(ips[i])
        if !isTrustedProxy(ip, trustedProxies) {
            return ip
        }
    }
    return strings.TrimSpace(ips[0])
}
```

## 安全注意事项

### 1. 永远不要信任未经验证的 XFF 值

- 客户端可以伪造任意 `X-Forwarded-For` 值
- 如果服务器可被直接访问（即使同时有反向代理），**XFF 中没有任何部分可信赖**
- 用于安全决策（限流、访问控制）的 IP 必须来自可信代理添加的部分

### 2. 防止 IP 欺骗攻击

- 配置网络层只允许负载均衡器/代理访问应用服务器
- 使用安全组/防火墙规则限制来源
- 在云环境中使用私有子网隔离应用层

### 3. 隐私合规

- IP 地址属于个人数据（GDPR 等法规）
- 记录日志时考虑数据最小化原则
- 考虑使用混淆标识符（RFC 7239 的 `_obfuscated` 格式）

### 4. IPv6 注意事项

- IPv6 地址在 XFF 中不带引号，在 `Forwarded` 中必须用 `"[...]"` 包裹
- 解析时注意 IPv6 地址可能包含冒号，不能简单用冒号分割
- 客户端可能通过 IPv6 绕过基于 IPv4 的限制

## 常见问题排查

| 问题 | 可能原因 | 排查方法 |
|------|---------|---------|
| 获取到负载均衡器 IP | 未配置 real_ip_header 或 trust proxy | 检查代理配置和应用层信任设置 |
| 获取到内网 IP | 可信代理范围配置不全 | 扩大 set_real_ip_from 范围 |
| XFF 为空 | 代理未配置传递 XFF | 检查代理的 proxy_set_header 配置 |
| 多个 IP 不知取哪个 | 多层代理未正确配置可信列表 | 使用可信代理列表法从右向左解析 |
| PROXY Protocol 报错 | 发送方和接收方版本不匹配 | 检查 v1/v2 配置一致性 |

## 相关概念

- **[[HTTP协议]]**：HTTP 请求/响应模型
- **[[NAT打洞（NAT穿透）详解]]**：NAT 环境下的 IP 共享问题
- **[[TCP-IP 协议]]**：TCP 连接与 IP 层基础

## 参考资料

- RFC 7239 - Forwarded HTTP Extension: https://datatracker.ietf.org/doc/html/rfc7239
- MDN - X-Forwarded-For Header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Forwarded-For
- Nginx - Accepting the PROXY Protocol: https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
- Nginx - Forwarded Header Examples: https://www.nginx.com/resources/wiki/start/topics/examples/forwarded/
- HAProxy PROXY Protocol Specification: https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt
- Cloudflare IP Ranges: https://www.cloudflare.com/ips/

[^mdn-xff]: MDN Web Docs, "X-Forwarded-For header", https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Forwarded-For
[^rfc7239]: IETF RFC 7239, "Forwarded HTTP Extension", June 2014, https://datatracker.ietf.org/doc/html/rfc7239
[^haproxy-proxy]: HAProxy Technologies, "PROXY Protocol Specification", https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt
[^nginx-forwarded]: F5 NGINX, "Forwarded Header Examples", https://www.nginx.com/resources/wiki/start/topics/examples/forwarded/
