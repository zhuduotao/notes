---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - AI
  - LangChain
  - LangGraph
  - Agent
  - LLM
  - DeepAgents
  - 多模型
  - 动态模型
aliases:
  - Deep Agents
  - LangChain Agent Framework
  - deepagents
source_type: official-doc
source_urls:
  - 'https://docs.langchain.com/oss/python/deepagents/overview'
  - 'https://github.com/langchain-ai/deepagents'
  - 'https://docs.langchain.com/oss/python/deepagents/cli/overview'
  - 'https://docs.langchain.com/oss/python/deepagents/models'
  - 'https://docs.langchain.com/oss/python/langchain/agents#dynamic-model'
status: verified
---
Deep Agents 是 LangChain 官方推出的"电池包含型" agent 框架（agent harness），基于 LangChain 核心构建块和 LangGraph 运行时构建。它提供开箱即用的能力：任务规划、文件系统操作、Shell 执行、子 agent 派发、上下文管理、长期记忆、权限控制等——开发者无需手动组装工具链，即可获得一个可直接运行的 agent，并按需定制。

核心设计理念：将通用 agent 的基础设施（工具、提示、上下文管理）打包成可复用的 harness，开发者只需关注业务逻辑扩展。

---

## 核心能力

### 任务规划与分解

内置 `write_todos` 工具，agent 可将复杂任务拆解为离散步骤、追踪进度、并根据新信息动态调整计划。适用于需要多轮推理、分支决策的复杂任务。

### 文件系统工具

提供虚拟文件系统工具集：
- `read_file` / `write_file` / `edit_file`：读写与编辑文件
- `ls` / `glob` / `grep`：文件搜索与内容检索

文件系统后端可插拔：
- In-memory：内存状态，适合快速原型
- Local disk：本地磁盘，适合开发环境
- LangGraph Store：跨线程持久化
- Sandbox：隔离环境（Modal、Daytona、Deno 等），用于安全执行代码
- Composite：组合多个后端
- Custom：自定义后端实现

文件系统工具用于 offload 大量上下文，防止上下文窗口溢出，并处理变长工具输出。

### Shell 执行

当使用 sandbox 后端时，agent 获得 `execute` 工具，可在隔离环境中运行 Shell 命令（测试、构建、git 操作等），不影响宿主系统。

### 子 Agent 派发

内置 `task` 工具，agent 可派发专用子 agent 处理特定子任务，子 agent拥有独立的上下文窗口，避免污染主 agent 的上下文。适用于：
- 多领域协作（如一个 agent 做研究，另一个写代码）
- 深度子任务（需要大量上下文但不应影响主流程）

### 上下文管理

- **自动摘要**：当对话上下文过长时，自动压缩历史消息，保持 agent 有效
- **大输出转存**：工具返回的大输出自动保存到文件，避免上下文膨胀

### 长期记忆

通过 LangGraph Memory Store 实现跨线程持久化，agent 可保存和检索历史对话信息，实现"记忆"能力。

### 权限控制

声明式权限规则，控制 agent 可读/写的文件和目录范围。子 agent 可继承或覆盖父 agent 的权限规则。

### Human-in-the-loop

配置敏感工具操作需要人工审批（基于 LangGraph interrupt），如文件删除、Shell 命令执行等，在执行前暂停等待用户确认。

### Skills（技能）

通过 Skills 扩展 agent 的领域知识、工作流和自定义指令，类似"插件"机制。

### 智能默认提示

内置 opinionated system prompt，指导模型：
- 行动前先规划
- 验证工作结果
- 主动管理上下文

开发者可自定义或替换默认提示。

---

## 与 LangChain / LangGraph 的关系

架构层级：

| 层级 | 产物 | 作用 |
|------|------|------|
| Framework | LangChain | 提供核心构建块（模型接口、工具定义、提示管理等） |
| Runtime | LangGraph | 提供持久执行、流式输出、checkpointing、human-in-the-loop 等运行时能力 |
| Harness | Deep Agents | 在 LangChain + LangGraph 之上，提供完整的 agent harness（工具集 + 提示 + 上下文管理） |

`create_deep_agent` 返回一个编译好的 LangGraph 图，可直接用于：
- 流式响应
- LangGraph Studio
- Checkpointer（状态持久化）
- 其他 LangGraph 特性

---

## 适用场景

**推荐使用 Deep Agents 的场景**：

- 复杂多步骤任务，需要规划与分解
- 大量上下文管理（文件系统 + 自动摘要）
- 需要 shell 执行（配合 sandbox）
- 子 agent 上下文隔离
- 跨会话持久化记忆
- 需要人工监督敏感操作
- 使用任意支持 tool calling 的模型（跨提供商兼容）

**更简单的场景建议**：

- 简单 agent：使用 LangChain 的 `create_agent`
- 自定义复杂流：直接构建 LangGraph workflow

---

## 模型配置与多模型混合使用

Deep Agents 支持多种模型配置方式，并提供了灵活的多模型混合使用机制——可在运行时动态切换模型、为不同子 agent 配置不同模型、或根据任务复杂度自动路由。

### 基础模型配置

三种配置方式：

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| 字符串格式 | `"provider:model"`（如 `"openai:gpt-5"`） | 快速切换、原型开发 |
| `init_chat_model` | 通过 LangChain 初始化模型实例 | 需要配置特定参数（temperature、max_tokens 等） |
| Provider 类 | 直接使用 `ChatOpenAI`、`ChatAnthropic` 等类 | 完全控制模型配置、provider 特定功能 |

```python
# 方式一：字符串格式
agent = create_deep_agent(model="openai:gpt-5.4")

# 方式二：init_chat_model
from langchain.chat_models import init_chat_model
model = init_chat_model(
    model="openai:gpt-5.4",
    temperature=0.1,
    max_retries=10,
)
agent = create_deep_agent(model=model)

# 方式三：Provider 类
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-5.4", temperature=0.1)
agent = create_deep_agent(model=model)
```

### 运行时动态切换模型

使用 middleware 的 `@wrap_model_call` 装饰器，可在每次模型调用前根据条件切换模型。适用于：

- 用户 UI 下拉选择模型
- 根据对话复杂度自动路由（简单任务用小模型、复杂任务用大模型）
- 成本优化策略

```python
from dataclasses import dataclass
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from deepagents import create_deep_agent
from typing import Callable

@dataclass
class Context:
    model: str  # 用户选择的模型

@wrap_model_call
def configurable_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    model_name = request.runtime.context.model
    model = init_chat_model(model_name)
    return handler(request.override(model=model))

agent = create_deep_agent(
    model="google_genai:gemini-3.1-pro-preview",  # 默认模型
    middleware=[configurable_model],
    context_schema=Context,
)

# 用户选择使用 OpenAI
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello!"}]},
    context=Context(model="openai:gpt-5.4"),
)
```

### 根据对话复杂度自动路由

```python
from langchain_openai import ChatOpenAI
from langchain.agents.middleware import wrap_model_call

basic_model = ChatOpenAI(model="gpt-5.4-mini")
advanced_model = ChatOpenAI(model="gpt-5.4")

@wrap_model_call
def complexity_router(request, handler):
    """对话超过 10 轮时切换到大模型"""
    message_count = len(request.state["messages"])
    model = advanced_model if message_count > 10 else basic_model
    return handler(request.override(model=model))

agent = create_deep_agent(
    model=basic_model,
    middleware=[complexity_router],
)
```

### 子 Agent 使用不同模型

定义子 agent 时可指定独立的模型，实现任务级模型隔离：

```python
research_subagent = {
    "name": "research-agent",
    "description": "深度研究任务",
    "system_prompt": "You are a great researcher",
    "tools": [internet_search],
    "model": "openai:gpt-5.2",  # 子 agent 使用不同模型
}

coding_subagent = {
    "name": "coding-agent",
    "description": "代码生成任务",
    "model": "anthropic:claude-sonnet-4-6",  # 适合代码的模型
}

agent = create_deep_agent(
    model="claude-opus-4-6",  # 主 agent 模型
    subagents=[research_subagent, coding_subagent],
)
```

### 限制与注意事项

| 限制 | 说明 |
|------|------|
| 必须支持 tool calling | 所有模型必须具备工具调用能力，否则 agent 无法运行 |
| 结构化输出与预绑定模型冲突 | 已调用 `bind_tools` 的模型不支持与结构化输出（`response_format`）同时使用 |
| 参数因 provider 不同 | `temperature`、`max_tokens` 等参数在不同 provider 中行为可能不同，需查阅 provider 文档 |

### 支持的模型提供商

| Provider | 示例模型 |
|----------|----------|
| OpenAI | `gpt-5.4`, `gpt-4o`, `o3` |
| Anthropic | `claude-sonnet-4-6`, `claude-opus-4-6` |
| Google | `gemini-3.1-pro-preview` |
| Azure OpenAI | `gpt-5.4` |
| AWS Bedrock | `anthropic.claude-sonnet-4-6` |
| Ollama | `devstral-2` |
| OpenRouter | `anthropic/claude-sonnet-4-6` |
| Fireworks | `qwen3.5-397B-A17B` |
| Baseten | `GLM-5` |

---

## 三大组件

### Deep Agents SDK（Python 库）

核心 API：

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="openai:gpt-4o",  # 或 anthopic, google, ollama 等
    tools=[my_custom_tool],
    system_prompt="You are a helpful assistant",
)

agent.invoke({"messages": [{"role": "user", "content": "Research X and write a summary"}]})
```

特性：
- 100% 开源，MIT 许可
- 提供商无关（支持任何支持 tool calling 的模型）
- 完全可扩展（添加工具、替换模型、调整提示）
- MCP 支持通过 `langchain-mcp-adapters`

### Deep Agents CLI

终端编码 agent，类似 Claude Code / Cursor，无需编码即可使用：

安装：

```bash
curl -LsSf https://raw.githubusercontent.com/langchain-ai/deepagents/main/libs/cli/scripts/install.sh | bash
# 或
uv tool install 'deepagents-cli'
```

CLI 附加能力：
- Interactive TUI：富终端界面 + 流式响应
- Conversation resume：跨会话恢复对话
- Web search：实时信息检索
- Remote sandboxes：LangSmith、AgentCore、Daytona、Modal、Runloop 等
- Persistent memory：跨对话持久记忆
- Custom skills：自定义 slash 命令
- Headless mode：非交互模式，适合 CI/脚本化
- Human-in-the-loop：工具调用审批

### ACP Integration

Agent Client Protocol connector，用于在代码编辑器（如 Zed）中使用 Deep Agents。

---

## 设计灵感与安全模型

**灵感来源**：主要受 Claude Code 启发，初始目标是探索 Claude Code 的通用性要素并进一步扩展。

**安全模型**："信任 LLM"模式——agent 可以做其工具允许的任何事情。边界控制在工具/沙箱层面，而非期望模型自我约束。

关键原则：
- 不要期望 LLM 自我限制敏感操作
- 通过工具权限、sandbox 隔离、human-in-the-loop 等机制约束行为
- 在 tool/sandbox 层面强制边界

---

## 典型使用流程

1. 安装：`pip install deepagents` 或 `uv add deepagents`
2. 创建 agent：`create_deep_agent()`，可指定模型、工具、提示
3. 调用：`agent.invoke()` 或使用 LangGraph 流式特性
4. 定制：添加自定义工具、替换后端、配置权限、启用 human-in-the-loop
5. 调试/监控：使用 LangSmith 进行请求追踪、行为调试、输出评估

---

## 参考资料

- [Deep Agents 官方文档](https://docs.langchain.com/oss/python/deepagents/overview)
- [GitHub 仓库](https://github.com/langchain-ai/deepagents)
- [Deep Agents CLI 文档](https://docs.langchain.com/oss/python/deepagents/cli/overview)
- [LangGraph 官方文档](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangSmith（调试与追踪）](https://docs.langchain.com/langsmith/home)
- [Frameworks, runtimes, and harnesses 概念说明](https://docs.langchain.com/oss/python/concepts/products)
