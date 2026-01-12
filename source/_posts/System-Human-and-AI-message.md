---
title: System Human and AI message
date: 2026-01-09 21:44:10
tags: langchain1.0
---

## 1. 概述

在 LangChain 的 Chat Model 架构中，消息对象（Message Objects）是与大语言模型（LLM）交互的原子单位。它们对底层 API（如 OpenAI Chat Completion API）的角色（Roles）进行了标准化封装，确保了不同模型提供商之间的接口一致性。

## 2. 组件定义

### 2.1 SystemMessage (系统消息)

- **定义**: 用于设定对话环境、模型行为、角色和边界条件的初始指令。
- **对应 LLM 角色**: `system`
- **生命周期**: 通常位于消息列表的索引 0 位置，且贯穿整个对话会话。
- **技术用途**:
  - **角色注入 (Persona Injection)**: 定义模型是谁（如“资深法律顾问”）。
  - **格式约束 (Output Guardrails)**: 强制规定输出格式（如“仅输出 JSON”）。
  - **安全围栏 (Safety Guidelines)**: 设定拒绝回答的话题范围。

### 2.2 HumanMessage (人类消息)

- **定义**: 代表来自外部用户（或客户端应用）的输入内容。
- **对应 LLM 角色**: `user`
- **生命周期**: 每一轮对话交互的触发点。
- **技术用途**:
  - **用户查询 (Query)**: 承载用户的实际问题。
  - **指令输入 (Prompt)**: 提供具体的任务指令。
  - **Few-Shot 示例输入**: 在构建样本时，充当“问题示例”。

### 2.3 AIMessage (AI 消息)

- **定义**: 代表大语言模型生成的响应内容。
- **对应 LLM 角色**: `assistant`
- **生命周期**: 作为 `HumanMessage` 的响应生成；或作为历史上下文被手动构造。
- **技术用途**:
  - **响应载体**: 承载模型返回的文本结果。
  - **工具调用 (Tool Calling)**: 包含 `tool_calls` 参数，指示需执行的外部函数。
  - **对话记忆 (Conversation History)**: 在多轮对话中，开发者需将之前的模型回复封装为 `AIMessage` 再次传回给模型，以保持上下文连贯。
  - **Few-Shot Prompting (少样本提示 / 伪造历史)**：

------

## 3. 交互架构图

在标准的 Request/Response 循环中，消息流如下：

Plaintext

```
[Input List to Model]
├── SystemMessage (1个)       --> 设定基准："你是一个Python专家"
├── HumanMessage  (历史)      --> 上下文："什么是List？"
├── AIMessage     (历史)      --> 上下文："List是可变序列..."
└── HumanMessage  (当前)      --> 触发点："那Tuple呢？"

       ⬇️ (Invoke Model)

[Output from Model]
└── AIMessage     (响应)      --> 结果："Tuple是不可变序列..."
```

------

## 4. 实现代码规范

以下代码展示了如何在 LangChain 1.0 环境下规范使用这三种消息。

### 4.1 基础调用

Python

```
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

# 1. 初始化模型
chat = ChatOpenAI(model="gpt-4o", temperature=0)

# 2. 构建消息列表
messages = [
    # System: 全局设定
    SystemMessage(content="你是一个精通SQL的数据库管理员，只返回SQL代码，不包含解释。"),
    
    # Human & AI: 历史对话注入 (Few-Shot Learning)
    HumanMessage(content="查询所有名为 'Alice' 的用户"),
    AIMessage(content="SELECT * FROM users WHERE name = 'Alice';"),
    
    # Human: 当前任务
    HumanMessage(content="统计过去24小时的订单总量")
]

# 3. 调用模型
response = chat.invoke(messages)

# 4. 处理结果
# response 类型为 AIMessage
print(f"生成的SQL: {response.content}")
```

### 4.2 最佳实践 (Best Practices)

1. **始终引入 `langchain_core`**: 避免从旧的 `langchain.schema` 导入，以确保 1.0 版本的兼容性和轻量化。
2. **角色隔离**: 严禁将 System Instruction 放入 `HumanMessage`，这会降低指令遵循能力（Instruction Following），尤其是在处理越狱攻击（Jailbreaking）防护时。
3. **记忆管理**: 在构建 Chatbot 时，必须将 `AIMessage` 存入 Memory 组件（如 `ChatMessageHistory`），否则模型将无法“记住”上一句话。

------

## 5. 属性对比矩阵

| **特性维度** | **SystemMessage**         | **HumanMessage**           | **AIMessage**          |
| ------------ | ------------------------- | -------------------------- | ---------------------- |
| **语义权重** | **最高** (High Attention) | 高 (Recent) / 低 (History) | 中等 (Contextual)      |
| **可变性**   | 通常静态 (Static)         | 高度动态 (Dynamic)         | 由模型生成 (Generated) |
| **内容源**   | 开发者 / 提示词工程师     | 终端用户                   | 模型推理引擎           |
| **典型用途** | 设定规则 (`Rules`)        | 提出需求 (`Ask`)           | 提供结果 (`Answer`)    |

------

我可以为您做的下一步：

这份文档是否满足您的需求？如果您正在构建多轮对话应用，我可以为您演示如何将这三种消息与 ChatPromptTemplate 和 MessagesPlaceholder 结合，实现自动化的历史记录管理。
