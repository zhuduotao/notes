---
created: 2026-04-27
updated: 2026-04-27
tags:
  - testing
  - contract-testing
  - micro-frontend
  - architecture
  - integration
aliases:
  - Contract Testing
  - 契约测试
  - Consumer Driven Contracts
  - CDC
  - Pact
source_type: mixed
source_urls:
  - https://docs.pact.io/
  - https://martinfowler.com/bliki/ContractTest.html
  - https://martinfowler.com/articles/consumerDrivenContracts.html
  - https://micro-frontends.org/
status: verified
---

## 是什么

契约测试（Contract Testing）是一种**针对集成点**的测试技术，通过检查通信双方各自发送或接收的消息是否符合一份共享的"契约"（Contract）来保证集成的正确性。这里的"消息"对于 HTTP 通信指请求与响应，对于消息队列指入队/出队的消息体。

Martin Fowler 的定义：契约测试持续使用 Test Double（测试替身）运行本地测试，同时**定期运行一组独立的契约测试**，验证测试替身的行为与真实外部服务返回的结果一致[[ContractTest|https://martinfowler.com/bliki/ContractTest.html]]。

契约 ≠ 接口文档 / Schema。契约是**通过执行一组具体测试用例**来强制执行的"按例契约"（Contract by Example），每条契约描述一个具体的请求/响应对，而非描述资源的所有可能状态[[Pact Intro|https://docs.pact.io/]]。

## 为什么重要

在微服务或微前端架构中，多个团队独立开发、独立部署各自的服务/模块。集成失败通常发生在**边界处**：

- 提供方修改了 API 字段名、类型或删除了字段
- 消费方期望的字段不存在或格式变化
- 双方对同一接口的理解不一致

传统端到端集成测试（E2E）的痛点：

| 痛点 | 说明 |
|------|------|
| 昂贵 | 需要部署完整环境（所有服务/模块） |
| 脆弱 | 环境波动、网络问题导致测试不稳定 |
| 缓慢 | 全量部署 + 全量测试耗时长 |
| 定位困难 | 失败时难以判断是哪一方的问题 |

契约测试将集成验证**左移**到开发阶段，在不需要部署对方的情况下即可发现不兼容变更。

## 核心概念

### Consumer 与 Provider

- **Consumer（消费方）**：发起请求、期望获取数据的一方
- **Provider（提供方）**：接收请求、返回数据的一方

在微服务架构中，传统的 Client/Server 术语不够准确（例如通过消息队列通信时），因此统一使用 Consumer/Provider[[Pact Terminology|https://docs.pact.io/]]。

### Consumer-Driven Contracts（CDC，消费者驱动契约）

契约由消费方定义并驱动。消费方在自动化测试中生成契约，只测试**实际使用的通信部分**。提供方未使用的行为可以自由变更而不破坏测试[[Consumer Driven Contracts|https://martinfowler.com/articles/consumerDrivenContracts.html]]。

CDC 的核心优势：

1. **最小化测试范围**：只覆盖消费方真正使用的字段和路径
2. **提供方自由演进**：未被使用的字段/端点可以安全修改
3. **变更可见性**：提供方在合并前就知道哪些消费方会受影响

### 契约测试工作流程（以 Pact 为例）

1. **Consumer 端**：编写测试，使用 Pact 的 mock server 代替真实 Provider
2. **生成契约**：测试运行时，Pact 记录所有请求/响应对，生成 JSON 契约文件
3. **发布契约**：契约文件上传到 Pact Broker（契约管理平台）
4. **Provider 端**：从 Broker 拉取契约，在真实 Provider 上回放请求，验证响应是否匹配
5. **验证结果**：通过/失败状态回写 Broker，双方均可查看兼容性状态

## 在微前端项目中如何落地

微前端将微服务理念延伸到前端，一个应用由多个独立团队拥有的功能模块组成[[Micro Frontends|https://micro-frontends.org/]]。契约测试在微前端中的适用场景和方式如下：

### 适用场景

| 场景 | 契约类型 | 说明 |
|------|----------|------|
| 微前端 ↔ 后端 API | HTTP 契约 | 各微前端模块独立调用后端接口，用 Pact 等工具验证 |
| 微前端 ↔ 微前端（DOM 级） | DOM 契约 | 通过 Custom Element 的 tag-name、attributes、events 作为公共契约 |
| 微前端 ↔ 微前端（事件级） | 事件契约 | 通过 CustomEvent 的名称、payload 结构定义契约 |
| 微前端 ↔ 共享组件库 | API 契约 | 组件的 props、回调函数签名作为契约 |

### 微前端间的契约定义

micro-frontends.org 明确提出：**DOM 就是 API**[[The DOM is the API|https://micro-frontends.org/]]。

```
<!-- 团队 Product 使用团队 Checkout 提供的 Custom Element -->
<blue-buy sku="t_porsche"></blue-buy>
```

这里的契约包括：

- **Tag Name**：`blue-buy`（命名规范 `[team_color]-[feature]`）
- **Attributes**：`sku`（字符串，表示商品编码）
- **Events**：如 `blue:basket:changed`（自定义事件，通知购物车更新）

这些就是两个团队之间的"契约"。如果团队 Checkout 修改了事件名或删除了某个属性，团队 Product 的页面就会出问题。

### 微前端契约测试实践

#### 1. DOM/Custom Element 契约测试

```javascript
// 消费方测试：验证 Custom Element 是否正确渲染
describe('blue-buy element', () => {
  it('renders with correct sku attribute', () => {
    const el = document.createElement('blue-buy');
    el.setAttribute('sku', 't_porsche');
    document.body.appendChild(el);

    // 验证契约：按钮存在且显示正确价格
    const button = el.shadowRoot.querySelector('button');
    expect(button).toBeTruthy();
    expect(button.textContent).toContain('66,00 €');
  });
});
```

#### 2. 事件契约测试

```javascript
// 提供方测试：验证事件正确触发
describe('blue-buy events', () => {
  it('dispatches blue:basket:changed on click', () => {
    const el = document.createElement('blue-buy');
    el.setAttribute('sku', 't_porsche');
    document.body.appendChild(el);

    const handler = jest.fn();
    window.addEventListener('blue:basket:changed', handler);

    el.shadowRoot.querySelector('button').click();

    expect(handler).toHaveBeenCalledTimes(1);
    expect(handler.mock.calls[0][0].type).toBe('blue:basket:changed');
    expect(handler.mock.calls[0][0].bubbles).toBe(true);
  });
});
```

#### 3. 后端 API 契约测试（Pact）

各微前端模块作为 Consumer，后端服务作为 Provider：

```javascript
// 微前端模块的 Pact 测试
import { Pact } from '@pact-foundation/pact';

const provider = new Pact({
  consumer: 'CheckoutMicroApp',
  provider: 'ProductAPI',
});

describe('Product API', () => {
  it('accepts a valid product request', () => {
    return provider.addInteraction({
      state: 'product t_porsche exists',
      uponReceiving: 'a request for product details',
      withRequest: {
        method: 'GET',
        path: '/api/products/t_porsche',
      },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          sku: 't_porsche',
          name: 'Porsche Tractor',
          price: 66.00,
        },
      },
    });
  });
});
```

### 微前端契约测试推荐策略

```
┌─────────────────────────────────────────────────┐
│                  契约测试分层                      │
├─────────────────────────────────────────────────┤
│  L1: 单元契约 - Custom Element props/events 验证  │
│  L2: HTTP 契约 - Pact 验证微前端 ↔ 后端 API        │
│  L3: 集成契约 - 定期运行，验证所有微前端组合兼容性    │
└─────────────────────────────────────────────────┘
```

- **L1** 在每次提交时运行，快速反馈
- **L2** 在 CI 中运行，生成/验证契约文件
- **L3** 每日或每周运行，覆盖全量集成场景

## 限制与注意事项

### 契约测试**不**做什么

- **不替代 E2E 测试**：契约测试验证接口兼容性，不验证完整的用户流程
- **不验证业务逻辑正确性**：只验证消息格式，不验证业务规则
- **不验证性能**：响应时间、吞吐量不在契约测试范围内
- **不验证非 HTTP 协议的所有行为**：Pact 主要支持 HTTP 和消息队列，WebSocket 等协议支持有限

### 常见误区

| 误区 | 正确理解 |
|------|----------|
| "有了契约就不需要集成测试" | 契约测试是集成测试的补充，不是替代 |
| "契约就是 OpenAPI/Swagger" | OpenAPI 描述所有可能的接口状态，契约只覆盖实际使用的路径 |
| "契约测试很慢" | 契约测试比 E2E 快得多，因为不需要部署完整环境 |
| "只有微服务需要契约测试" | 微前端、前后端分离、任何有集成点的场景都适用 |

### 版本兼容性

- 契约变更分为**向后兼容**（新增可选字段）和**破坏性变更**（删除/重命名字段）
- 使用 Pact Broker 的**兼容性矩阵**（Can I Deploy）可以判断某次变更是否安全
- 对于破坏性变更，推荐**先扩展后废弃**的策略：新增字段 → 双写 → 通知消费方迁移 → 移除旧字段

## 相关概念

- **Consumer-Driven Contracts（CDC）**：由消费方驱动契约定义的模式[[CDC|https://martinfowler.com/articles/consumerDrivenContracts.html]]
- **Test Double**：测试替身，包括 Stub、Mock、Fake 等[[TestDouble|https://martinfowler.com/bliki/TestDouble.html]]
- **Schema Testing**：基于 Schema（如 JSON Schema、OpenAPI）的验证，与契约测试互补但不同
- **Pact**：最流行的契约测试工具，支持多种语言[[Pact|https://docs.pact.io/]]
- **Pact Broker**：契约管理和兼容性验证平台
- **微前端（Micro Frontends）**：将微服务理念延伸到前端的架构风格[[Micro Frontends|https://micro-frontends.org/]]

## 参考资料

- Martin Fowler - Contract Test: <https://martinfowler.com/bliki/ContractTest.html>
- Martin Fowler - Consumer-Driven Contracts: <https://martinfowler.com/articles/consumerDrivenContracts.html>
- Pact 官方文档: <https://docs.pact.io/>
- Micro Frontends 官方网站: <https://micro-frontends.org/>
- ThoughtWorks Technology Radar - Micro Frontends: <https://www.thoughtworks.com/radar/techniques/micro-frontends>
