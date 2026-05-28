---
title: LangChain调用工具详解
date: 2026-01-20 16:34:25
tags: langchain
---

这份技术文档旨在为开发者提供关于在 **LangChain** (LCEL) 和 **LangGraph** 中定义、绑定及执行工具（Tools）的完整指南。

## 1. 概述 (Overview)

在 LLM 应用开发中，“工具使用”（Tool Calling / Function Calling）是指 LLM 输出结构化数据（JSON）以请求执行外部代码的能力。

- **LangChain (LCEL)**：侧重于工具的**定义**与**绑定**，以及线性的调用流程。
- **LangGraph**：侧重于工具的**自动化执行**、**状态管理**及**循环反馈**（Agent 架构）。

------

## 2. 定义工具 (Defining Tools)

无论是在 LangChain 还是 LangGraph 中，定义工具的方式是通用的。核心是创建一个带有**名称**、**描述**和**参数类型 (Schema)** 的可执行单元。

### 最佳实践：使用 `@tool` 装饰器

这是最简洁的方法，LangChain 会自动解析 Python 函数的类型提示（Type Hints）生成 JSON Schema。

Python

```
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """查询指定城市的当前天气信息。"""
    # 模拟 API 调用
    if "东京" in city:
        return "18°C"
    elif "北京" in city:
        return "22°C"
    return "未知"

@tool
def calculate_tax(income: int) -> int:
    """根据收入计算税额。"""
    return int(income * 0.2)

# 工具列表
tools = [get_weather, calculate_tax]
```

------

## 3. 在 LangChain (LCEL) 中使用工具

在传统的 LangChain 链（Chain）中，工具的使用通常是**线性**的。你需要手动处理 LLM 的响应并决定是否执行工具。

### 3.1 核心步骤

1. **Bind (绑定)**: 将工具 Schema 注入 LLM。
2. **Invoke (调用)**: 获取 LLM 响应。
3. **Parse & Execute (解析与执行)**: 检查 `tool_calls` 并运行函数（通常需自定义逻辑或使用辅助链）。

### 3.2 代码实现

Python

```
from langchain_openai import ChatOpenAI

# 1. 初始化模型并绑定工具
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools(tools)

# 2. 调用模型
query = "东京的天气怎么样？"
response = llm_with_tools.invoke(query)

# 3. 检查并手动执行 (LCEL 风格)
print(f"AI 意图: {response.tool_calls}") 
# 输出: [{'name': 'get_weather', 'args': {'city': '东京'}, 'id': '...'}]

if response.tool_calls:
    for tool_call in response.tool_calls:
        # 查找匹配的工具
        selected_tool = {"get_weather": get_weather, "calculate_tax": calculate_tax}[tool_call["name"]]
        # 执行工具
        tool_result = selected_tool.invoke(tool_call["args"])
        print(f"工具结果: {tool_result}")
```

**局限性**：在纯 LCEL 中，构建一个“LLM -> 工具 -> 结果回传 -> LLM 继续回答”的完整闭环（ReAct 模式）代码量较大且难以管理状态。

------

## 4. 在 LangGraph 中使用工具

LangGraph 引入了**图（Graph）**的概念，专门用于解决循环依赖和状态持久化。它提供了预构建组件（`ToolNode`）来自动化工具执行流程。

### 4.1 核心组件

- **`State`**: 存储对话历史（Messages）。
- **`ToolNode`**: 一个可运行节点，自动接收 `AIMessage`，解析 `tool_calls`，并行执行工具，并返回 `ToolMessage`。
- **`tools_condition`**: 一个预置的路由逻辑，判断是去往 `ToolNode` 还是结束对话。

### 4.2 代码实现 (Standard ReAct Pattern)

Python

```
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

# 1. 定义状态 (State)
class State(TypedDict):
    # add_messages reducer 负责将新消息追加到历史记录中
    messages: Annotated[list, add_messages]

# 2. 准备组件
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools(tools) # 依然需要绑定

# 3. 构建图
builder = StateGraph(State)

# 定义节点函数：调用 LLM
def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

# 添加节点
builder.add_node("chatbot", chatbot)
builder.add_node("tools", ToolNode(tools)) # ToolNode 自动处理执行逻辑

# 4. 定义边 (Edges) - 构建控制流
builder.add_edge(START, "chatbot")

# 条件边：chatbot 结束后，检查是否需要调用工具
builder.add_conditional_edges(
    "chatbot",
    tools_condition, 
    # 默认映射: 如果返回 True (有工具调用) -> "tools"
    #          如果返回 False (无工具调用) -> END
)

# 闭环：工具执行完，必须回到 chatbot 让 AI 生成最终回复
builder.add_edge("tools", "chatbot")

# 5. 编译与运行
graph = builder.compile()

# 执行
inputs = {"messages": [("user", "东京天气如何？")]}
for event in graph.stream(inputs):
    for key, value in event.items():
        print(f"Node '{key}':")
        print(value)
        print("---")
```

------

## 5. 对比总结 (Comparison)

| **特性**     | **LangChain (LCEL)**                                     | **LangGraph**                                         |
| ------------ | -------------------------------------------------------- | ----------------------------------------------------- |
| **主要用途** | 简单的、线性的工具调用链。                               | 复杂的、循环的 Agent 系统。                           |
| **执行机制** | 通常需要开发者手动解析 `tool_calls` 并调用 `.invoke()`。 | 通过 `ToolNode` 自动解析并并行执行。                  |
| **流程控制** | 线性 (DAG)。实现循环（如重试）很困难。                   | 循环 (Cyclic)。原生支持 `Loop` 和 `Feedback`。        |
| **状态管理** | 无内置状态管理（Stateless），需外部传递。                | 内置 `State`，自动追踪消息历史和工具输出。            |
| **错误处理** | 需在代码逻辑中手动 try/except。                          | `ToolNode` 支持 `fallback` 机制，可将错误反馈给 LLM。 |

### 什么时候用什么？

- **使用 LangChain (LCEL)**：如果你只需要 LLM 帮你提取一次参数（例如：从文本中提取日期和地点），然后你的 Python 代码拿着这些参数去跑后续流程，不需要再回头问 LLM。
- **使用 LangGraph**：如果你需要构建一个 Agent，它需要自己决定调用什么工具，根据工具的结果（可能报错）进行反思，多次交互直到完成任务。

------

好的，这是基于前文内容的扩展，为您补充 **Human-in-the-loop (人工介入)** 和 **错误恢复 (Error Recovery)** 的详细技术文档。这两部分是将 Demo 级应用转化为生产级 Agent 的关键。

------

# 6. 进阶模式：Human-in-the-loop (人工介入)

在涉及敏感操作（如写入数据库、发送邮件、大额转账）时，完全自动化的工具调用是危险的。LangGraph 提供了**中断（Interrupt）**机制，允许在执行特定节点（如 `tools`）之前暂停，等待人工确认或修改。

### 6.1 核心机制

1. **Checkpointer (检查点保存器)**: 必须配置 Checkpointer（如 `MemorySaver`），用于持久化保存图的状态。
2. **`interrupt_before`**: 在编译图时指定在哪个节点**之前**中断。
3. **Thread ID**: 使用 `thread_id` 来追踪同一个对话会话，确保中断后能找回状态。

### 6.2 代码示例：审批敏感工具

假设我们有一个“转账”工具，在真正执行前需要人工确认。

Python

```
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START
from langgraph.prebuilt import ToolNode, tools_condition

# ... (假设已定义 State, llm, tools 等，参考前文)

# 1. 初始化 Checkpointer (用于保存暂停时的状态)
memory = MemorySaver()

# 2. 构建图
builder = StateGraph(State)
builder.add_node("chatbot", chatbot)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "chatbot")
builder.add_conditional_edges("chatbot", tools_condition)
builder.add_edge("tools", "chatbot")

# 3. 关键步骤：编译时配置中断
graph = builder.compile(
    checkpointer=memory,
    # 在进入 'tools' 节点前暂停，等待批准
    interrupt_before=["tools"] 
)

# 4. 执行流程模拟
thread_config = {"configurable": {"thread_id": "session_1"}}

print("--- 阶段 1: AI 发起请求 ---")
# 用户请求转账
initial_input = {"messages": [("user", "请帮我向 Alice 转账 100 美元")]}

# 图会运行到 'chatbot' -> 决定调用工具 -> 准备进入 'tools' 时暂停
for event in graph.stream(initial_input, config=thread_config):
    print(event)

print("\n--- 状态检查: 系统已暂停 ---")
# 获取当前状态快照
snapshot = graph.get_state(thread_config)
next_step = snapshot.next
print(f"下一步计划执行的节点: {next_step}") # 输出: ('tools',)

# 此时，你可以检查 snapshot.values['messages'] 里的 tool_call 内容
# 例如展示给前端用户： "AI 想要调用 transfer(to='Alice', amount=100)，是否批准？"

print("\n--- 阶段 2: 人工批准并恢复 ---")
user_approval = input("是否批准操作? (y/n): ")

if user_approval.lower() == "y":
    # 传入 None 表示"不修改状态，直接继续运行"
    # 图会从暂停的地方(即进入 tools 节点)继续执行
    for event in graph.stream(None, config=thread_config):
        print(event)
else:
    print("操作已取消。")
    # 实际场景中，这里可以注入一条 ToolMessage 表示"用户拒绝执行"
```

------

# 7. 错误恢复 (Error Recovery)与自修正

LLM 在调用工具时常犯两类错误：

1. **参数错误**（Hallucination）：例如 API 需要 JSON 格式，LLM 给了字符串；或者遗漏了必填参数。
2. **运行时错误**：例如查询的天气服务 API 挂了，或者数据库连接超时。

如果没有错误处理，程序会抛出异常并崩溃。LangGraph 的哲学是：**将错误转化为反馈（Feedback），让 LLM 自己去修。**

### 7.1 策略一：自动捕获并反馈 (`handle_tool_error`)

这是最简单的方法。在定义工具时启用 `handle_tool_error`。当工具抛出异常时，LangChain 会自动生成一个包含错误信息的 `ToolMessage` 发回给 LLM。

**LLM 的反应**：看到错误信息后，LLM 会分析原因，并尝试生成新的参数重新调用工具。

Python

```
from langchain_core.tools import tool

# 自定义错误处理函数 (可选)
def handle_error(error: Exception) -> str:
    return f"工具执行出错: {error}。请检查你的参数格式是否正确，然后重试。"

@tool(handle_tool_error=handle_error) 
def search_database(query: str):
    """查询数据库。Query 必须是合法的 SQL 语句。"""
    if "SELECT" not in query.upper():
        # 抛出异常，触发 handle_error
        raise ValueError("Invalid SQL: Query must start with SELECT")
    return f"Results for {query}"

# 或者直接设为 True，使用默认的错误消息
# @tool(handle_tool_error=True)
```

### 7.2 策略二：在 ToolNode 层面处理 (Fallbacks)

如果你使用的是预构建的 `ToolNode`，可以在初始化时配置回退逻辑。这对于处理不可预知的系统级错误（如网络超时）非常有用。

Python

```
from langgraph.prebuilt import ToolNode

# 定义工具列表
tools = [search_database]

# 初始化 ToolNode，并配置异常捕获
tool_node = ToolNode(tools).with_fallbacks(
    [fallback_llm_node], # 如果工具彻底失败，可以跳转到一个备用的 LLM 节点解释原因
    exception_key="error" # 将异常信息存储在 state 中
)
```

### 7.3 完整的自修正流程示例

当 `handle_tool_error` 生效时，对话历史会变成这样，形成一个**自我修正的闭环**：

1. **User**: "查询数据库：用户列表"
2. **AI (AIMessage)**: `tool_call: search_database(query="users")` *(错误：这不是 SQL)*
3. **Tool (ToolMessage)**: `content="工具执行出错: Invalid SQL: Query must start with SELECT..."` *(捕获了异常)*
4. **AI (AIMessage)**: *(LLM 看到错误，思考修正)* `tool_call: search_database(query="SELECT * FROM users")`
5. **Tool (ToolMessage)**: `content="[Alice, Bob, Charlie]"` *(执行成功)*
6. **AI**: "数据库中的用户有 Alice, Bob 和 Charlie。"

------

# 8. 最佳实践总结

| **场景**                      | **推荐方案**          | **关键技术点**                                       |
| ----------------------------- | --------------------- | ---------------------------------------------------- |
| **只读操作** (查询天气、搜索) | **全自动**            | `handle_tool_error=True` (允许 LLM 出错重试)         |
| **写操作** (转账、删库)       | **Human-in-the-loop** | `interrupt_before=["tools"]`, `MemorySaver`          |
| **复杂参数** (如生成代码)     | **Pydantic 校验**     | 使用 Pydantic 定义工具参数，提供详细的校验错误信息   |
| **调试**                      | **LangSmith**         | 使用 LangSmith 追踪 Trace，查看 LLM 为什么参数传错了 |

通过结合 **Human-in-the-loop** 确保安全，以及 **Error Recovery** 确保鲁棒性，您就可以构建出真正可用的企业级 Agent。
