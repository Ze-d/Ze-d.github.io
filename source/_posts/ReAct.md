---
title: ReAct
date: 2026-01-10 22:11:32
tags: [LLM,langchain]
---

这是一份关于 **ReAct (Reasoning + Acting)** 技术的深度技术白皮书。它涵盖了从核心原理、底层实现到进阶架构的完整知识体系。

## 1. 核心定义与背景 (Introduction)

**ReAct** 是 **Reasoning（推理）** 与 **Acting（行动）** 的合成词。它是一种用于构建大语言模型（LLM）代理（Agent）的范式。

- **起源:** Google Research (Yao et al., 2022) 提出的论文 *"ReAct: Synergizing Reasoning and Acting in Language Models"*。
- **解决的问题:**
  - 单纯的 **推理 (Chain-of-Thought)** 容易产生幻觉，无法获取外部世界的新知。
  - 单纯的 **行动 (Action)** 缺乏规划，容易在复杂任务中迷失方向。
- **核心理念:** 让 LLM 在执行任务时，交替进行“思考（生成推理轨迹）”和“行动（调用外部工具）”，并将行动的结果（观察）反哺给模型，形成闭环。

------

## 2. 技术架构与工作流 (Architecture & Workflow)

ReAct 不是模型本身的一个功能，而是一种**应用层的控制流（Control Flow）**。

### 2.1 核心循环 (The Loop)

一个标准的 ReAct Loop 包含三个原子步骤：

1. **Thought (思考):** LLM 分析当前的用户需求和历史状态，决定下一步做什么。
   - *Output:* 文本描述或结构化数据。
2. **Action (行动):** LLM 生成具体的工具调用指令（函数名+参数）。
   - *Trigger:* 系统检测到工具调用意图，拦截流程。
3. **Observation (观察):** 外部系统（Runtime）执行工具，将运行结果（API 返回值、代码输出）捕获回来。
   - *Input:* 将结果追加到 Prompt 中，再次喂给 LLM。

### 2.2 状态机表示 (State Machine)

从计算机科学角度看，ReAct 是一个状态机：

$$S_{t+1} = LLM(S_t, O_t)$$

其中：

- $S_t$: 当前的对话历史（包含之前的 Thoughts 和 Actions）。
- $O_t$: 上一步 Action 的执行结果（Observation）。
- $LLM$: 状态转移函数（大模型）。

------

## 3. 实现技术的演进 (Evolution)

ReAct 的实现方式经历了两个主要阶段：

### 阶段一：Prompt Engineering (基于文本解析)

- **机制:** 在 System Prompt 中强行规定格式。

  > "如果你要使用工具，请按以下格式输出：Action: [工具名], Action Input: [参数]"

- **触发:** 框架（如 LangChain）使用 **正则表达式 (Regex)** 扫描 LLM 的输出。

- **缺点:** 脆弱，模型容易输出无效格式，参数解析困难（例如 JSON 引号嵌套问题）。

### 阶段二：Native Tool Calling (基于微调与语法约束)

- **机制:** 现代 LLM (GPT-4, Claude 3, Qwen) 经过专门的 Function Calling 微调。
- **协议:**
  - 开发者通过 `bind_tools` 传入 JSON Schema。
  - 模型输出专门的 `tool_calls` 结构化字段（非纯文本）。
- **触发:** 框架直接检查 `response.tool_calls` 对象。
- **优点:** 鲁棒性极高，支持强类型的参数验证，支持并行调用多个工具。

------

## 4. 关键技术组件 (Key Components)

构建一个生产级的 ReAct Agent 需要以下组件协同工作：

### A. 路由器 (The Brain)

- 负责决策。它判断是直接回答用户（Final Answer），还是发起工具调用。
- **技术点:** 上下文窗口管理（Context Window Management），防止历史记录撑爆 Token 限制。

### B. 工具执行器 (The Hands)

- 负责将 LLM 的指令（JSON）转换为实际的代码执行。
- **技术点:** 异常处理（Error Handling）。如果工具报错，需要将错误信息作为 Observation 反馈给 LLM，而不是让程序崩溃。

### C. 记忆存储 (The Memory)

- **Short-term Memory:** 即 Scratchpad（草稿本），存储当前的 Thought-Action-Observation 链条。
- **Long-term Memory:** 向量数据库（Vector Store），用于检索跨会话的历史经验。

------

## 5. 进阶模式与优化 (Advanced Patterns)

为了解决 ReAct “死循环”、“迷路”或“低效”的问题，衍生出了以下增强技术：

| **模式**                    | **描述**                                                     | **解决痛点**                                    |
| --------------------------- | ------------------------------------------------------------ | ----------------------------------------------- |
| **Plan-and-Solve**          | 先生成完整的 To-Do List，再逐个执行。                        | 解决 ReAct 容易陷入局部细节、缺乏大局观的问题。 |
| **Reflexion**               | 在 Action 报错或结果不理想时，强制 LLM 进行自我反思（Self-Critique）。 | 提高 Agent 的鲁棒性，具备“知错能改”的能力。     |
| **Human-in-the-loop**       | 在关键 Action 之前挂起（Suspend），等待人工批准。            | 解决高风险操作（如退款、删库）的安全合规问题。  |
| **Multi-Agent (LangGraph)** | 将复杂任务拆解，由多个垂直领域的 ReAct Loop 协作完成。       | 解决单体 Agent 能力边界有限、上下文过长的问题。 |

------

## 6. 常见挑战与解决方案 (Challenges)

### 6.1 延迟 (Latency)

- **问题:** ReAct 需要多次串行调用 LLM，响应速度慢。
- **对策:**
  - 使用速度更快的模型（如 GPT-4o-mini, Groq LPU）。
  - **并行工具调用:** 如果模型同时请求查天气和查股票，并行执行。

### 6.2 循环陷阱 (Infinite Loops)

- **问题:** 模型反复尝试同一个失败的操作。
- **对策:**
  - 设置 `max_iterations`（最大循环次数）。
  - 在 System Prompt 中加入指令：“如果连续失败 3 次，请向用户寻求帮助”。

### 6.3 成本 (Cost)

- **问题:** 历史记录不断累积，Token 消耗呈指数级增长。
- **对策:**
  - **Context Trimming:** 仅保留最近 N 轮对话。
  - **Summarization:** 每一轮循环结束后，将之前的长思考过程总结为一句话。

------

## 7. 代码实现参考 (LangGraph 伪代码)

在最新的 LangChain 生态（LangGraph）中，ReAct 显式地由“图”定义：

Python

```
from langgraph.graph import StateGraph, END

# 1. 定义状态
class AgentState(TypedDict):
    messages: list

# 2. 定义节点
def call_model(state):
    # 推理逻辑
    response = model.invoke(state['messages'])
    return {"messages": [response]}

def execute_tools(state):
    # 行动逻辑
    last_message = state['messages'][-1]
    tool_result = run_tool(last_message.tool_calls)
    return {"messages": [tool_result]}

# 3. 定义边 (路由逻辑)
def should_continue(state):
    last_message = state['messages'][-1]
    if last_message.tool_calls:
        return "tools"  # 继续循环
    return END          # 结束，输出给用户

# 4. 构建图
workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", execute_tools)

workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
workflow.add_edge("tools", "agent") # 工具执行完，必须回到 Agent 再思考

app = workflow.compile()
```

------

## 8. 总结

ReAct 是大模型从“聊天机器人”进化为“智能助手”的分水岭技术。它通过 **“推理-行动-观察”** 的迭代循环，赋予了静态模型操作动态世界的能力。

对于开发者而言，理解 ReAct 不仅仅是会用 LangChain 的 API，更重要的是理解其背后的**状态管理、上下文控制以及错误恢复机制**。
