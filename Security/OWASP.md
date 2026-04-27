---
created: '2026-04-23'
updated: '2026-04-23'
tags:
  - security
  - web-security
  - application-security
  - organization
  - open-source
aliases:
  - 开放式Web应用安全项目
  - Open Web Application Security Project
  - Open Worldwide Application Security Project
source_type: official-doc
source_urls:
  - 'https://owasp.org/'
  - 'https://owasp.org/about/'
  - 'https://owasp.org/projects/'
  - 'https://owasp.org/www-project-top-ten/'
status: verified
---
OWASP（Open Worldwide Application Security Project，开放式全球应用安全项目）是一个致力于提升软件安全性的非营利基金会，通过社区驱动的开源项目、教育资源和全球协作推动应用安全实践。

## 组织概况

OWASP 于 2001 年 12 月 1 日启动，2004 年 4 月 21 日在美国注册为非营利慈善机构（EIN #20-0963503）。截至 2026 年，OWASP 拥有：

- 250+ 个全球本地分会
- 数万名成员
- 418+ 个活跃项目
- 年度全球 AppSec 会议和培训活动

### 愿景与使命

- **愿景**：No more insecure software（不再有不安全的软件）
- **使命**：通过教育、工具和协作成为推动安全软件的全球开放社区

### 核心价值观

| 价值观 | 说明 |
|--------|------|
| Open（开放） | 从财务到代码，一切都保持极度透明 |
| Innovative（创新） | 鼓励和支持针对软件安全挑战的创新与实验 |
| Global（全球） | 鼓励世界各地任何人参与 OWASP 社区 |
| Integrity（诚信） | 社区保持尊重、支持、诚实和 vendor neutral（供应商中立） |

### 法律实体

- **OWASP Foundation Inc.**：300 Delaware Ave Ste 210 #384, Wilmington, DE 19801, USA
- **OWASP Europe VZW**：Steenvoordestraat 184, 9070 Destelbergen, Belgium（VAT: BE 0836743279）

## 为什么重要

OWASP 是全球最大的关注软件安全的非营利组织，其产出被广泛采纳为行业标准：

1. **行业标准参考**：OWASP Top 10 被全球开发者视为安全编码的第一步，许多合规框架（如 PCI DSS）引用 OWASP 标准
2. **免费开放资源**：所有项目、工具、文档均免费开放，采用 Creative Commons 或 OSI-approved Open Source License
3. **供应商中立**：不背书或推荐商业产品，保持社区中立性
4. **实践导向**：产出可直接用于开发、测试、审计和安全培训

## 核心项目分类

OWASP 项目按成熟度分为 Flagship（旗舰）、Production（生产）、Lab/Incubator（实验室/孵化器）三类，覆盖代码、文档和工具。

### Flagship Projects（旗舰项目）

| 项目 | 类型 | 用途 |
|------|------|------|
| [OWASP Top 10](https://owasp.org/www-project-top-ten/) | 文档 | Web 应用最关键安全风险的参考标准 |
| [ASVS](https://owasp.org/www-project-application-security-verification-standard/) | 标准 | 应用安全验证标准框架，定义设计/开发/测试阶段的安全控制要求 |
| [OWASP Cheat Sheet Series](https://owasp.org/www-project-cheat-sheets/) | 文档 | 针对开发者和防御者的简洁最佳实践指南集合 |
| [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) | 代码 | 现代、复杂的不安全 Web 应用，用于安全培训、演示和 CTF |
| [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) | 工具 | SCA（软件组成分析）工具，识别依赖中的已知漏洞 |
| [OWASP Dependency-Track](https://owasp.org/www-project-dependency-track/) | 工具 | 组件分析平台，识别和降低软件供应链风险 |
| [OWASP SAMM](https://owasp.org/www-project-samm/) | 文档 | 软件保障成熟度模型，帮助组织分析和改进软件安全态势 |
| [OWASP CRS](https://owasp.org/www-project-modsecurity-core-rule-set/) | 工具 | 用于 ModSecurity 或兼容 WAF 的通用攻击检测规则集 |
| [OWASP Web Security Testing Guide (WSTG)](https://owasp.org/www-project-web-security-testing-guide/) | 文档 | Web 应用安全测试的权威资源 |

### Production Projects（生产项目）

代表性项目包括：

- [OWASP API Security Project](https://owasp.org/www-project-api-security/)：API 安全策略和解决方案
- [OWASP Coraza WAF](https://owasp.org/www-project-coraza-web-application-firewall/)：Go 语言企业级 WAF 框架
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)：HTTP 安全头技术信息

### 其他 Top 10 系列

OWASP 针对不同领域发布了多款 Top 10 风险清单：

- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)：大语言模型应用安全风险
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)：移动应用安全风险
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)：API 安全风险（隶属于 API Security Project）
- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)：CI/CD 安全风险
- [OWASP IoT Top 10](https://owasp.org/www-project-internet-of-things/)：物联网安全风险（Embedded Application Security 项目）

## OWASP Top 10 简介

OWASP Top 10 是最广为人知的项目，定期更新以反映当前最关键的 Web 应用安全风险：

- 最新版本：OWASP Top 10 2025
- 前序版本：2021、2017、2013、2010
- 全球开发者将其作为安全编码入门的参考标准
- 已翻译为中文、日文、法文、西班牙文等多种语言

采用 Top 10 是改变软件开发文化、产出更安全代码的最有效第一步。

## 如何参与

- **参与项目**：通过 [OWASP Projects](https://owasp.org/projects/) 提交新项目或贡献现有项目
- **加入分会**：全球 250+ 个本地分会，通过 [OWASP Chapters](https://owasp.org/chapters/) 查找
- **参加会议**：年度 Global AppSec 会议（欧盟、美国）、BASC、LASCON 等
- **成为会员**：[OWASP Membership](https://owasp.glueup.com/organization/6727/memberships)
- **社区沟通**：[Slack Channel](https://owasp.slack.com/)、[Google Groups](https://groups.google.com/a/owasp.com/)

## 常见误区

1. **OWASP 是商业公司**：OWASP 是非营利基金会，不售卖商业产品或服务
2. **OWASP Top 10 足够覆盖所有风险**：Top 10 仅列出最常见风险，实际安全测试需参考 WSTG、ASVS 等更全面的资源
3. **OWASP 内容收费**：所有文档和工具均免费开放，采用开源或 Creative Commons 许可
4. **OWASP 推荐特定工具**：OWASP 保持供应商中立，不背书商业产品

## 相关概念

- **ASVS（Application Security Verification Standard）**：用于定义和验证应用安全控制要求的标准框架
- **WSTG（Web Security Testing Guide）**：Web 应用渗透测试的全面指南
- **SCA（Software Composition Analysis）**：软件组成分析，识别第三方依赖中的漏洞
- **DevSecOps**：将安全集成到 DevOps 流程中，OWASP 有专门的 [DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)
- **SBOM（Software Bill of Materials）**：软件物料清单，OWASP CycloneDX 是相关标准（ECMA-424）

## 参考资料

- [OWASP 官网](https://owasp.org/)
- [OWASP About](https://owasp.org/about/)
- [OWASP Projects](https://owasp.org/projects/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Top 10 2025](https://owasp.org/Top10/2025/)
- [OWASP Impact Report 2025](https://owasp.org/assets/files/OWASP_Impact_Report_2025.pdf)
- [OWASP Policy & Procedures](https://policy.owasp.org/)
- [OWASP GitHub](https://github.com/OWASP)
