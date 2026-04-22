---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - HTTP
  - Network
  - Web
  - RESTful
  - API设计
aliases:
  - HTTP Methods
  - HTTP请求方法
  - HTTP动词
source_type: spec
source_urls:
  - 'https://datatracker.ietf.org/doc/html/rfc9110#section-9'
  - 'https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods'
status: verified
---
## 概述

HTTP Request Methods（请求方法）定义了客户端对服务器资源的操作意图。每个方法具有特定的语义，并对应不同的操作类型：获取数据、提交数据、修改数据、删除数据等。

HTTP 方法是 RESTful API 设计的核心要素，正确理解和使用方法是构建规范 Web API 的基础。

---

## 方法属性

### 安全性（Safe）

**定义**：安全方法不会对服务器资源状态产生副作用，即不会导致资源修改。

> RFC 9110 §9.2.1：安全方法定义为只读操作，调用该方法不会导致服务器承担额外义务（如修改数据）。

**安全方法列表**：`GET`、`HEAD`、`OPTIONS`、`TRACE`

**意义**：
- 安全方法可以被缓存、预取，无需用户确认
- 安全请求不会改变资源状态，适合自动化调用
- 爬虫、预加载器可放心调用安全方法

> 注意："安全"仅指对资源状态的副作用，不代表请求本身无害（如日志记录、统计计数等非资源副作用）。

### 幂等性（Idempotent）

**定义**：幂等方法多次执行同一请求，结果与单次执行相同。

> RFC 9110 §9.2.2：幂等方法的多次请求产生的副作用与单次请求相同。适用于网络故障时的安全重试。

**幂等方法列表**：`GET`、`HEAD`、`PUT`、`DELETE`、`OPTIONS`、`TRACE`

**非幂等方法**：`POST`、`CONNECT`

**意义**：
- 网络故障时可安全重试幂等请求
- 幂等请求可被缓存代理安全转发
- 自动重试机制可应用于幂等方法

**示例对比**：

```http
PUT /users/123 HTTP/1.1
Content-Type: application/json

{"name": "Alice"}

无论执行 1 次还是 N 次，用户 123 的状态始终为 {"name": "Alice"}

POST /orders HTTP/1.1
Content-Type: application/json

{"item": "book"}

执行 1 次：创建 1 个订单
执行 N 次：创建 N 个订单（非幂等）
```

### 可缓存性（Cacheable）

**定义**：可缓存方法的响应可被客户端或代理缓存以加速后续请求。

> RFC 9110 §9.2.3：GET 和 HEAD 是主要可缓存方法。POST 在特定条件下也可缓存（需响应头明确指示）。

**默认可缓存方法**：`GET`、`HEAD`

**条件可缓存**：`POST`（需 `Content-Location` 或 `Location` 头配合，且响应头明确缓存策略）

---

## 标准方法详解

### GET

**语义**：请求获取目标资源的表示（representation）。

> RFC 9110 §9.3.1：GET 方法请求传输目标资源的选定表示。

| 属性 | 值 |
|------|-----|
| 安全 | ✅ |
| 幂等 | ✅ |
| 可缓存 | ✅ |
| 请求体 | 不推荐使用（语义未定义，部分服务器会拒绝） |

**典型用途**：
- 获取网页、图片、JSON 数据
- 搜索查询（参数放 URL query string）
- 列表资源（如 `/users`）

**请求示例**：

```http
GET /users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
```

**注意事项**：
- GET 不应用于数据修改操作
- 请求参数应在 URL 中传递，不应使用请求体
- 敏感数据不应放 URL（可能被日志记录）

---

### HEAD

**语义**：与 GET 相同，但服务器只返回响应头，不返回响应体。

> RFC 9110 §9.3.2：HEAD 方法与 GET 相同，但服务器响应不得包含消息体。

| 属性 | 值 |
|------|-----|
| 安全 | ✅ |
| 幂等 | ✅ |
| 可缓存 | ✅ |
| 请求体 | 不使用 |

**典型用途**：
- 检查资源是否存在（通过状态码）
- 检查资源是否修改（通过 `Last-Modified` 或 `ETag`）
- 获取资源元数据（如 `Content-Length`）而不下载完整内容

**请求示例**：

```http
HEAD /files/document.pdf HTTP/1.1
Host: example.com
```

**响应示例**：

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Length: 102400
Last-Modified: Mon, 20 Apr 2026 10:00:00 GMT
（无响应体）
```

---

### POST

**语义**：请求服务器根据请求体内容执行特定处理，通常用于创建资源或提交数据。

> RFC 9110 §9.3.3：POST 方法请求目标资源处理请求体中包含的表示。

| 属性 | 值 |
|------|-----|
| 安全 | ❌ |
| 幂等 | ❌ |
| 可缓存 | 条件可缓存（需响应头指示） |
| 请求体 | ✅ 必须包含 |

**典型用途**：
- 创建新资源（如新增用户、发布文章）
- 提交表单数据
- 上传文件
- 执行非幂等的操作（如处理支付）

**请求示例**：

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Bob",
  "email": "bob@example.com"
}
```

**成功响应**：

```http
HTTP/1.1 201 Created
Location: /users/124
Content-Type: application/json

{
  "id": 124,
  "name": "Bob",
  "email": "bob@example.com"
}
```

**POST vs PUT 对比**：

| 场景 | 使用方法 |
|------|----------|
| 创建资源，URL 由服务器决定 | `POST /users` → 返回 `/users/124` |
| 创建资源，URL 由客户端指定 | `PUT /users/124` |
| 更新已有资源 | `PUT /users/124` 或 `PATCH /users/124` |

---

### PUT

**语义**：请求用请求体内容替换目标资源的全部内容。若目标资源不存在，可创建新资源。

> RFC 9110 §9.3.4：PUT 方法请求创建或替换目标资源的表示。

| 属性 | 值 |
|------|-----|
| 安全 | ❌ |
| 幂等 | ✅ |
| 可缓存 | ❌ |
| 请求体 | ✅ 必须包含完整资源表示 |

**典型用途**：
- 创建资源（URL 由客户端指定）
- 完整替换已有资源

**请求示例**：

```http
PUT /users/124 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Bob Updated",
  "email": "bob.new@example.com",
  "age": 30
}
```

**关键特性**：
- **全量替换**：PUT 会完全替换目标资源，未提供的字段会被删除或重置为默认值
- **幂等性**：多次执行同一 PUT 请求，资源状态始终相同

**PUT vs PATCH 对比**：

| 操作类型 | 方法 | 说明 |
|----------|------|------|
| 全量更新 | PUT | 必须提供完整资源表示 |
| 部分更新 | PATCH | 只需提供修改字段（非标准方法，但广泛使用） |

---

### DELETE

**语义**：请求删除目标资源。

> RFC 9110 §9.3.5：DELETE 方法请求删除目标资源。

| 属性 | 值 |
|------|-----|
| 安全 | ❌ |
| 幂等 | ✅ |
| 可缓存 | ❌ |
| 请求体 | 可包含（语义未严格定义） |

**典型用途**：
- 删除指定资源

**请求示例**：

```http
DELETE /users/124 HTTP/1.1
Host: api.example.com
```

**成功响应**：

```http
HTTP/1.1 204 No Content
（或 200 OK 并返回已删除资源信息）
```

**幂等性说明**：
- 删除已存在的资源：第一次返回 204/200，后续返回 404 或 204（资源已不存在）
- 结果一致：资源最终均处于"不存在"状态

---

### CONNECT

**语义**：请求建立到目标 URI 所标识服务器的隧道连接（通常用于代理服务器）。

> RFC 9110 §9.3.6：CONNECT 方法请求与目标资源建立隧道连接。

| 属性 | 值 |
|------|-----|
| 安全 | ❌ |
| 幂等 | ❌ |
| 可缓存 | ❌ |
| 请求体 | 不使用 |

**典型用途**：
- HTTP 代理建立 SSL 隧道（HTTPS 代理）
- WebSocket 连接升级

**请求示例**：

```http
CONNECT server.example.com:443 HTTP/1.1
Host: server.example.com
Proxy-Authorization: Basic abc123
```

**响应示例**：

```http
HTTP/1.1 200 Connection Established
（此后代理转发原始 TCP 数据流）
```

---

### OPTIONS

**语义**：请求获取目标资源支持的通信选项（如支持的方法列表）。

> RFC 9110 §9.3.7：OPTIONS 方法请求获取目标资源支持的通信选项。

| 属性 | 值 |
|------|-----|
| 安全 | ✅ |
| 幂等 | ✅ |
| 可缓存 | ❌ |
| 请求体 | 不使用 |

**典型用途**：
- CORS 预检请求（跨域请求前的权限检查）
- 发现 API 能力（查询支持的方法）

**请求示例**：

```http
OPTIONS /users HTTP/1.1
Host: api.example.com
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
```

**响应示例**：

```http
HTTP/1.1 204 No Content
Allow: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

---

### TRACE

**语义**：请求服务器返回收到请求的副本，用于诊断请求在传输链路中的变化。

> RFC 9110 §9.3.8：TRACE 方法请求服务器在响应体中返回收到请求的副本。

| 属性 | 值 |
|------|-----|
| 安全 | ✅ |
| 幂等 | ✅ |
| 可缓存 | ❌ |
| 请求体 | 不使用 |

**典型用途**：
- 诊断请求链路（检测代理修改请求的情况）
- 测试服务器连通性

**安全风险**：
- 可能泄露敏感信息（如 Authorization 头）
- 多数生产环境服务器禁用 TRACE
- 存在 XST（Cross-Site Tracing）攻击风险

**建议**：生产环境应禁用 TRACE 方法。

---

## 方法属性汇总表

| 方法 | 安全 | 幂等 | 可缓存 | 请求体 | 典型用途 |
|------|------|------|--------|--------|----------|
| GET | ✅ | ✅ | ✅ | 不推荐 | 获取资源 |
| HEAD | ✅ | ✅ | ✅ | ❌ | 获取元数据 |
| POST | ❌ | ❌ | 条件 | ✅ | 创建资源、提交数据 |
| PUT | ❌ | ✅ | ❌ | ✅ | 创建/替换资源 |
| DELETE | ❌ | ✅ | ❌ | 可选 | 删除资源 |
| CONNECT | ❌ | ❌ | ❌ | ❌ | 建立隧道 |
| OPTIONS | ✅ | ✅ | ❌ | ❌ | 获取通信选项 |
| TRACE | ✅ | ✅ | ❌ | ❌ | 诊断请求链路 |

---

## RESTful API 设计中的方法使用

### 资源操作映射

| 资源操作 | HTTP 方法 | URI 示例 |
|----------|-----------|----------|
| 列表资源 | GET | `/users` |
| 获取单个资源 | GET | `/users/123` |
| 创建资源（服务器生成 ID） | POST | `/users` |
| 创建资源（客户端指定 ID） | PUT | `/users/123` |
| 全量更新资源 | PUT | `/users/123` |
| 部分更新资源 | PATCH | `/users/123` |
| 删除资源 | DELETE | `/users/123` |

### 设计原则

1. **URI 表示资源，方法表示操作**
   - URI `/users/123` 表示"用户 123"这个资源
   - 方法 GET/PUT/DELETE 表示对资源的操作类型

2. **避免动词化 URI**
   ```http
   ❌ GET /getUser/123
   ❌ POST /createUser
   ❌ POST /deleteUser/123
   
   ✅ GET /users/123
   ✅ POST /users
   ✅ DELETE /users/123
   ```

3. **幂等操作优先使用幂等方法**
   - 可重复执行且不产生额外副作用时，使用 PUT/DELETE
   - 需避免重复执行时，使用 POST

4. **GET 不应修改资源**
   - GET 请求不应触发资源修改（违反安全性语义）
   - 需修改资源时使用 POST/PUT/PATCH/DELETE

---

## 常见误区

### 误区 1：POST 总是用于创建资源

POST 可用于多种场景：
- 创建资源
- 提交表单
- 执行复杂操作（如批量处理、异步任务）
- 更新资源（当更新操作非幂等时）

POST 的语义是"请求服务器处理请求体内容"，不局限于创建。

### 误区 2：PUT 和 POST 可互换

关键区别：
- **幂等性**：PUT 幂等，POST 非幂等
- **URL 含义**：PUT 指定具体资源 URL，POST 通常指向集合 URL
- **用途**：PUT 用于已知 URL 的创建/替换，POST 用于新增或处理

### 误区 3：GET 请求可以带请求体

RFC 9110 未明文禁止 GET 请求体，但：
- 语义未定义，行为不可预期
- 多数 HTTP 库、服务器、代理会忽略或拒绝 GET 请求体
- 数据应通过 URL query string 或请求头传递

### 误区 4：所有 HTTP 方法都被所有服务器支持

实际情况：
- TRACE 通常被禁用（安全风险）
- PATCH 不是 RFC 9110 标准方法（但广泛支持）
- 某些服务器可能限制 PUT/DELETE（需 CORS 配置）

---

## 最佳实践

1. **严格遵循方法语义**
   - GET：只读获取
   - POST：非幂等操作、资源创建
   - PUT：幂等创建/替换
   - DELETE：幂等删除

2. **幂等方法支持重试**
   - 网络故障时可自动重试 GET/PUT/DELETE
   - POST 重试需谨慎（可能产生重复资源）

3. **GET 参数放 URL，POST/PUT 数据放请求体**
   - GET 参数通过 query string 传递
   - POST/PUT 数据通过请求体传递
   - 敏感数据不应放 URL（可能被日志记录）

4. **返回正确的状态码**
   - GET 成功：200 OK
   - POST 创建成功：201 Created
   - PUT 成功：200 OK 或 204 No Content
   - DELETE 成功：204 No Content 或 200 OK

5. **CORS 跨域请求配置**
   - 非简单请求（PUT/DELETE、自定义头）需 OPTIONS 预检
   - 服务器需正确响应 `Access-Control-Allow-Methods`

6. **禁用危险的 TRACE 方法**
   - 生产环境应在服务器配置中禁用 TRACE
   - 防止 XST 攻击和信息泄露

---

## 参考资料

- RFC 9110 §9 — Methods：https://datatracker.ietf.org/doc/html/rfc9110#section-9
- RFC 9110 §9.2 — Common Method Properties：https://datatracker.ietf.org/doc/html/rfc9110#section-9.2
- MDN Web Docs — HTTP request methods：https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
- RFC 5789 — PATCH Method for HTTP：https://datatracker.ietf.org/doc/html/rfc5789
