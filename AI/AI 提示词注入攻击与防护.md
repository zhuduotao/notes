---
created: '2025-04-23'
updated: '2025-04-23'
tags:
  - AI安全
  - LLM
  - PromptInjection
  - OWASP
  - 安全防护
aliases:
  - Prompt Injection
  - 提示词注入攻击
  - LLM注入
source_type: official-doc
source_urls:
  - 'https://genai.owasp.org/llmrisk/llm01-prompt-injection/'
  - 'https://owasp.org/www-project-top-10-for-large-language-model-applications/'
  - 'https://atlas.mitre.org/techniques/AML.T0051.000'
  - 'https://atlas.mitre.org/techniques/AML.T0051.001'
status: verified
---
提示词注入（Prompt Injection）是 OWASP LLM Top 10 安全风险榜单的首位风险（LLM01:2025），指用户输入的提示词以非预期方式改变 LLM 的行为或输出。攻击者可利用该漏洞绕过安全措施、泄露敏感信息、执行未授权操作或操控关键决策。

## 为什么重要

LLM 将用户输入与系统提示词合并处理，无法可靠区分"指令"与"数据"。这一设计特性导致攻击者可通过精心构造的输入覆盖、修改或绕过系统设定的行为约束。随着 LLM 应用集成更多外部数据源（RAG）和功能调用（Function Calling），提示词注入的攻击面持续扩大。

## 攻击类型

### 直接注入（Direct Prompt Injection）

用户直接在输入中嵌入恶意指令，改变模型行为。可以是：
- **有意攻击**：恶意用户刻意构造输入以利用模型漏洞
- **无意触发**：用户不了解系统约束，输入触发了非预期行为

示例：攻击者向客服机器人注入指令，要求其忽略原有规则、查询私有数据并发送邮件。

### 间接注入（Indirect Prompt Injection）

LLM 从外部来源（网页、文件、数据库、API 响应）获取数据时，外部内容中嵌入的恶意指令被模型解析执行。这类攻击更隐蔽，因为用户可能完全 unaware。

示例：用户用 LLM 总结网页，网页中隐藏指令导致 LLM 插入指向恶意 URL 的图片，泄露对话内容。

### 多模态注入

攻击者将恶意提示词嵌入图像、音频或其他模态数据中。多模态模型在同时处理文本和非文本数据时，跨模态交互可能触发隐藏指令。这类攻击难以用传统文本过滤手段检测。

## 与 Jailbreak 的关系

提示词注入与 Jailbreak 常被混用，但存在区别：

| 概念 | 定义 | 关系 |
|------|------|------|
| Prompt Injection | 通过特定输入操控模型响应，改变其行为 | 更广泛概念，包含多种攻击形式 |
| Jailbreak | 提供输入使模型完全忽略安全协议 | Prompt Injection 的一种子类型 |

Jailbreak 的有效防护需要模型层面的安全机制更新，而常规提示词注入可通过应用层设计缓解。

## 攻击场景示例

### 职位描述触发

公司职位描述中包含识别 AI 生成简历的指令。求职者使用 LLM 优化简历时，无意中触发该指令导致简历被标记。

### RAG 数据投毒

攻击者修改 RAG 系统检索的文档库内容。用户查询返回被篡改内容时，嵌入的恶意指令操控 LLM 输出误导性结果。

### Payload Splitting

攻击者将恶意提示词拆分后嵌入简历不同部分。LLM 评估候选人时，拆分片段组合执行，输出正面评价与实际内容不符。

### 对抗性后缀（Adversarial Suffix）

攻击者在提示词末尾附加看似无意义的字符串，实际影响模型输出方向，绕过安全过滤。

### 多语言/编码混淆

使用多语言混合或 Base64、Emoji 等编码方式隐藏恶意指令，规避基于关键词的检测。

## 防护策略

OWASP 明确指出：由于生成式 AI 的随机性本质，**不存在完全消除提示词注入的方法**[^1]。以下措施可降低风险：

### 1. 约束模型行为

在系统提示词中明确：
- 模型的角色、能力和限制
- 严格限定任务范围和响应主题
- 指令要求忽略任何修改核心规则的尝试

### 2. 定义并验证输出格式

- 指定清晰的输出结构（JSON、表格等）
- 要求提供推理过程和来源引用
- 使用确定性代码验证输出格式合规性

### 3. 实施输入/输出过滤

- 定义敏感内容类别和过滤规则
- 应用语义过滤和字符串检测扫描不允许内容
- 使用 RAG Triad 评估响应：上下文相关性、事实依据、问答相关性

### 4. 强制权限控制与最小权限原则

- 为应用功能使用独立的 API Token，而非将 Token 传递给模型
- 在代码层面处理敏感操作，而非委托给 LLM
- 限制模型仅拥有执行预期操作所需的最低权限

### 5. 高风险操作需人工审批

对涉及敏感数据访问、外部系统调用等高风险行为，实施 Human-in-the-Loop 控制，要求人工确认后方可执行。

### 6. 分隔并标识外部内容

将来自外部来源的内容与用户提示词明确分隔，使用分隔符或标记清晰标识不可信内容，限制其对用户提示词的影响。

### 7. 执行对抗性测试

定期进行渗透测试和攻击模拟，将模型视为不可信用户，测试信任边界和访问控制的有效性。

## 限制与注意事项

### RAG 和微调无法消除风险

研究表明，RAG 和微调可以提高输出相关性，但**不能完全消除提示词注入漏洞**[^1]。外部检索数据本身可能包含恶意内容。

### 无完美解决方案

模型处理提示词的机制决定了无法可靠区分指令与数据。任何声称"完全防止"的方案都应审慎评估。

### 多模态风险持续增长

多模态 AI 扩大了攻击面，跨模态攻击难以检测。该领域需要持续研究新的防护手段。

## 相关概念

- **LLM02:2025 敏感信息泄露** — 提示词注入可能导致该风险
- **LLM05:2025 不当输出处理** — 未验证 LLM 输出的下游风险
- **LLM06:2025 过度权限** — 赋予 LLM 过多操作权限加剧注入危害
- **MITRE ATLAS AML.T0051.000** — 直接提示词注入技术分类
- **MITRE ATLAS AML.T0051.001** — 间接提示词注入技术分类

## 参考资料

[^1]: OWASP GenAI Security Project - LLM01:2025 Prompt Injection: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
[^2]: OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
[^3]: MITRE ATLAS - LLM Prompt Injection: Direct: https://atlas.mitre.org/techniques/AML.T0051.000
[^4]: MITRE ATLAS - LLM Prompt Injection: Indirect: https://atlas.mitre.org/techniques/AML.T0051.001
[^5]: Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection: https://arxiv.org/pdf/2302.12173.pdf
[^6]: Prompt Injection attack against LLM-integrated Applications: https://arxiv.org/abs/2306.05499
[^7]: Reducing The Impact of Prompt Injection Attacks Through Design: https://research.kudelskisecurity.com/2023/05/25/reducing-the-impact-of-prompt-injection-attacks-through-design/
