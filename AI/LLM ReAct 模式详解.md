---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - LLM
  - Agent
  - Reasoning
  - Prompting
  - AI-Agent
aliases:
  - ReAct
  - Reasoning and Acting
source_type: official-paper
source_urls:
  - 'https://arxiv.org/abs/2210.03629'
  - 'https://react-lm.github.io'
  - 'https://github.com/ysymyth/ReAct'
status: verified
---
## 概述

ReAct（Reasoning and Acting）是一种让大语言模型（LLM）同时进行推理和行动的范式。模型在解决问题的过程中交替生成"思考轨迹"（reasoning traces）和"具体行动"（task-specific actions），使推理与行动相互协同。由 Google Research 于 2022 年提出，论文发表于 ICLR 2023。

## 核心思想

ReAct 将 LLM 的两大能力——推理（如 Chain-of-Thought 提示）和行动（如调用工具、与环境交互）——统一到一个框架中：

- **Reasoning（推理）**：模型生成思考过程，帮助诱导、追踪、更新行动计划，处理异常情况
- **Acting（行动）**：模型执行具体操作（如调用 API、查询知识库），获取外部信息

两者交织执行，形成"思考 → 行动 → 观察 → 再思考"的循环。

## 与 Chain-of-Thought 的对比

| 方法 | 特点 | 优势 | 问题 |
|------|------|------|------|
| **Chain-of-Thought (CoT)** | 仅推理，不行动 | 展示推理过程，提升复杂问题求解 | 依赖内部知识，易产生幻觉；无法获取外部信息 |
| **Act-only** | 仅行动，不推理 | 可获取真实信息 | 缺乏规划能力，难以综合信息得出结论 |
| **ReAct** | 推理+行动交织 | 推理指导行动，行动提供事实依据；轨迹可解释 | 需要外部工具支持；推理步骤可能冗余 |

CoT 的典型问题：推理过程中使用错误的事实知识（幻觉），且无法修正。ReAct 通过行动调用 Wikipedia API 等外部源，获取真实信息并更新推理。

## 执行流程

ReAct 的典型执行轨迹包含三种元素：

```
Thought: 我需要先查找苹果公司的成立时间
Action: Search[苹果公司 成立时间]
Observation: 苹果公司成立于1976年4月1日
Thought: 现在我需要查找史蒂夫·乔布斯去世的时间
Action: Search[史蒂夫·乔布斯 死亡时间]
Observation: 史蒂夫·乔布斯于2011年10月5日去世
Thought: 我已经获得两个关键时间点，可以计算差值
Answer: 苹果公司成立35年后，史蒂夫·乔布斯去世
```

1. **Thought**：模型的推理步骤，规划下一步行动或分析观察结果
2. **Action**：具体的行动指令（如搜索、查询、操作）
3. **Observation**：环境返回的结果（外部工具的输出）

## Prompting 实现

ReAct 通过 few-shot prompting 实现：在提示中包含若干人工编写的任务解决轨迹示例，每个示例展示完整的 Thought-Action-Observation 循环。论文附录提供了针对不同任务的 prompt 模板。

关键设计要点：
- 示例轨迹需包含真实的外部交互（如 Wikipedia API 调用）
- Thought 内容应简洁、有针对性
- 可根据任务类型调整推理密度（决策类任务推理可稀疏）

## 适用场景

| 任务类型 | 基准测试 | 效果 |
|----------|----------|------|
| 问答系统 | HotpotQA | 通过 Wikipedia API 获取事实，减少幻觉 |
| 事实验证 | Fever | 验证陈述真实性，优于纯推理方法 |
| 交互决策 | ALFWorld | 家庭任务执行，成功率比模仿学习高 34% |
| 在线购物 | WebShop | 网站交互购物，成功率比强化学习高 10% |

典型应用：
- 需要外部知识验证的问答
- 多步骤工具调用
- 与环境交互的决策任务
- 可解释性要求高的场景

## 限制与注意事项

1. **依赖外部工具**：需要可靠的工具/API 支持，工具失败会导致任务中断
2. **上下文窗口限制**：长轨迹会消耗大量 token，可通过微调缓解
3. **推理冗余**：不必要的 Thought 会增加成本，需要精心设计 prompt
4. **错误传播**：早期行动错误可能导致后续推理偏离（但可通过人工干预修正）
5. **冷启动问题**：复杂任务可能需要较多示例才能稳定

## 微调效果

论文初步实验表明：
- ReAct 格式是各模型规模下的最佳微调格式
- 微调的小模型可超越 prompting 的大模型

## 框架实现

主流 Agent 框架已内置 ReAct：

- **LangChain**：`create_react_agent` 构建 ReAct agent，支持自定义工具
- **AutoGPT / BabyAGI**：基于 ReAct 思想的自主 agent
- **LangGraph**：更灵活的 ReAct 流程编排

## 相关概念

- **Chain-of-Thought (CoT)**：纯推理提示方法
- **Tool Learning**：工具使用能力
- **Agent**：具有自主决策能力的 LLM 系统
- **Grounding**：将 LLM 输出锚定到外部事实

## 参考资料

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) - 原始论文
- [ReAct Project Site](https://react-lm.github.io) - 官方项目页，含示例
- [ReAct GitHub](https://github.com/ysymyth/ReAct) - 官方代码仓库
- [Google AI Blog: ReAct](https://ai.googleblog.com/2022/11/react-synergizing-reasoning-and-acting.html) - 官方博客介绍
