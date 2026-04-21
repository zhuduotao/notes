---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - AI
  - LangChain
  - LangGraph
  - Agent
  - Orchestration
  - Stateful
aliases:
  - LangGraph
  - langgraph
  - LangChain Graph
source_type: official-doc
source_urls:
  - 'https://docs.langchain.com/oss/python/langgraph/overview'
  - 'https://github.com/langchain-ai/langgraph'
  - 'https://docs.langchain.com/oss/python/langgraph/quickstart'
status: verified
---
LangGraph 是 LangChain 官方推出的**低级编排框架和运行时**，用于构建、管理和部署**长期运行、有状态**的 agent。它专注于 agent 编排，不抽象提示或架构——开发者可以完全控制 agent 的行为和流程设计。

核心定位：为任何长期运行、有状态的工作流或 agent 提供底层基础设施支持。

---

## 核心作用

### 持久执行（Durable Execution）

Agent 可以在故障后恢复执行，长期运行，并从上次中断点自动恢复。适用于：
- 长时间任务（如复杂研究、代码生成）
- 需要跨多轮对话保持状态
- 在中断后恢复工作流

机制：通过 checkpointing 保存执行状态，支持从任意节点恢复。

### Human-in-the-loop

在执行任意节点时暂停，允许人工：
- 检查当前 agent 状态
- 修改状态值
- 批准或拒绝下一步操作

适用场景：
- 敏感操作审批（如文件删除、外部 API 调用）
- 复杂决策的监督
- 调试和验证 agent 行为

实现：基于 LangGraph interrupt 机制，在指定节点插入暂停点。

### 综合记忆能力（Comprehensive Memory）

两种记忆类型：
- **短期工作记忆**：当前对话/任务的推理上下文（如 messages history）
- **长期持久记忆**：跨会话、跨线程的记忆，通过 Memory Store 实现

应用：
- 多轮对话中保持上下文
- 用户偏好、历史交互的持久化
- Agent 学习和知识积累

### 可视化调试（LangSmith 集成）

LangSmith 提供：
- 执行路径可视化
- 状态转换追踪
- 详细运行时指标
- 节点级调试

配置方式：设置 `LANGSMITH_TRACING=true` 和 API key 即可启用。

### 生产级部署

LangSmith Deployment 提供：
- 长期运行、有状态工作流的专用部署平台
- 可扩展基础设施
- 跨团队的 agent 发现、复用、配置、共享
- LangSmith Studio 可视化原型开发

---

## 核心概念

### StateGraph（状态图）

LangGraph 的核心抽象。将 agent 工作流建模为**有状态图**：
- **节点（Nodes）**：执行具体逻辑的函数
- **边（Edges）**：节点间的连接，定义执行流向
- **状态（State）**：在节点间传递和更新的数据结构

```python
from langgraph.graph import StateGraph, MessagesState, START, END

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()
```

### State 定义

State 是 TypedDict 或类似结构，定义图的状态类型：

```python
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    context: str
    decision: str
```

State 在节点间传递，每个节点可读取和更新 State。

### START / END

特殊节点：
- `START`：图的入口点，连接第一个执行节点
- `END`：图的终点，执行到此节点时结束

所有图必须从 START 开始，到 END 结束。

### compile()

将 StateGraph 编译为可执行的 CompiledGraph：

```python
compiled_graph = graph.compile()
```

编译后的图支持：
- `invoke()`：同步执行
- `stream()`：流式执行
- `ainvoke()` / `astream()`：异步执行

### invoke() / stream()

执行编译后的图：

```python
# 同步执行
result = graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})

# 流式执行（逐步返回节点输出）
for step in graph.stream({"messages": [{"role": "user", "content": "hi!"}]}):
    print(step)
```

---

## 与 LangChain 的关系

架构层级：

| 层级 | 产物 | 作用 |
|------|------|------|
| Framework | LangChain | 提供核心构建块：模型接口、工具定义、提示管理、chain 组装 |
| Runtime | LangGraph | 提供运行时：持久执行、流式输出、checkpointing、human-in-the-loop |
| Harness | Deep Agents | 在 LangChain + LangGraph 之上，提供完整 agent harness（工具集 + 提示 + 上下文管理） |

关键关系：
- **LangGraph 可独立使用**：不需要 LangChain 也可构建 agent
- **与 LangChain 无缝集成**：可使用 LangChain 的模型、工具、组件
- **LangChain agents 基于 LangGraph 构建**：LangChain 的 `create_agent` 内部使用 LangGraph

---

## 设计灵感

LangGraph 受以下系统启发：
- **Pregel**：Google 的图计算框架，强调分布式、容错的图执行
- **Apache Beam**：批处理/流处理统一框架，强调可扩展、持久执行
- **NetworkX**：Python 图库，公共接口设计受其影响

核心设计理念：将 agent 工作流视为**有状态的图执行过程**，而非简单的 chain 或 pipeline。

---

## 适用场景

**推荐使用 LangGraph 的场景**：

- 长期运行、有状态的 agent 工作流
- 需要 human-in-the-loop 的人工监督
- 复杂多步骤任务，需要分支、循环、条件路由
- 需要持久化执行状态，支持中断恢复
- 生产级部署，需要可扩展基础设施
- 需要详细调试和可视化追踪

**更简单的场景建议**：

- 简单 chain：直接使用 LangChain 的 Chain
- 简单 agent：使用 LangChain 的 `create_agent`（内部基于 LangGraph）
- 单步任务：无需图编排

---

## 典型使用流程

1. 定义 State（状态结构）
2. 创建节点函数（每个节点读取/更新 State）
3. 构建 StateGraph，添加节点和边
4. 编译图：`graph.compile()`
5. 执行：`invoke()` / `stream()`
6. 配置持久化：添加 checkpointer
7. 启用 human-in-the-loop：配置 interrupt
8. 部署：使用 LangSmith Deployment 或自托管

---

## 核心扩展能力

### Checkpointer（状态持久化）

配置 checkpointer 使图可持久化执行：

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
compiled_graph = graph.compile(checkpointer=checkpointer)
```

支持的 checkpointer：
- MemorySaver：内存持久化（适合开发）
- SqliteSaver / PostgresSaver：数据库持久化（适合生产）

### Interrupt（暂停点）

在指定节点插入暂停点，实现 human-in-the-loop：

```python
compiled_graph = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["sensitive_node"]  # 执行前暂停
)
```

执行后可通过 `update_state()` 人工修改状态，再继续执行。

### Subgraph（子图）

在节点中嵌入另一个图，实现模块化：

```python
subgraph = StateGraph(SubState)
# ... 定义子图

parent_graph.add_node("subgraph_node", subgraph.compile())
```

---

## 与 Deep Agents 的关系

Deep Agents 是 LangGraph 之上的 agent harness：
- Deep Agents 内部使用 LangGraph 作为运行时
- `create_deep_agent()` 返回一个编译好的 LangGraph 图
- Deep Agents 提供开箱即用的工具集（planning、filesystem、subagent）
- LangGraph 提供底层编排能力（持久执行、human-in-the-loop）

选择建议：
- 需要**完整 agent harness**：使用 Deep Agents
- 需要**精细控制图结构**：直接使用 LangGraph

---

## 参考资料

- [LangGraph 官方文档](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph GitHub 仓库](https://github.com/langchain-ai/langgraph)
- [LangGraph Quickstart](https://docs.langchain.com/oss/python/langgraph/quickstart)
- [LangSmith（调试与追踪）](https://www.langchain.com/langsmith)
- [LangSmith Deployment](https://docs.langchain.com/langsmith/deployments)
- [Deep Agents（LangGraph 之上的 harness）](https://github.com/langchain-ai/deepagents)
- [Frameworks, runtimes, and harnesses 概念说明](https://docs.langchain.com/oss/python/concepts/products)
- [Pregel 论文](https://research.google/pubs/pub37252/)
- [Apache Beam](https://beam.apache.org/)
- [NetworkX](https://networkx.org/documentation/latest/)
