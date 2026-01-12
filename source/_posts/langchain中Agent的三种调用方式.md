---
title: langchain中Agent的三种调用方式
date: 2026-01-11 15:11:37
tags: [langchain,llm]
---

这份技术文档专门针对 **LangChain Agent (智能体)** 场景，深度解析三种核心调用方式。

相比于简单的线性链（Chain），Agent 包含复杂的循环、工具调用和自我修正逻辑，因此其调用模式（Execution Patterns）具有独特的行为特征。

## 1. 概述 (Overview)

在 LangChain 1.0 架构中，Agent 本质上是一个实现了 `Runnable` 协议的复杂状态机。尽管其内部包含 ReAct 循环、工具路由和错误重试机制，但对外依然暴露统一的标准接口：

- **`invoke` (同步):** 任务交付型。最常用，封装了复杂的中间过程。
- **`stream` (流式):** 体验增强型。用于实时获取最终回答的 Token，或监控中间步骤。
- **`batch` (批量):** 效率优化型。用于并发处理多个独立的 Agent 任务。

------

## 2. 同步调用：`agent.invoke()`

这是开发 Agent 时**最默认、最推荐**的调用方式。

### 2.1 核心逻辑

Agent 的工作往往是“结果导向”的。调用 `invoke` 意味着你启动了一个**黑盒进程**。

1. **输入:** 用户目标（如“帮我总结这周的销售数据并发邮件”）。
2. **黑盒运行:** Agent 内部进行 N 次 `While` 循环（思考 $\rightarrow$ 调 Excel 工具 $\rightarrow$ 调邮件工具 $\rightarrow$ 报错重试 $\rightarrow$ 成功）。
3. **输出:** 在所有步骤都成功完成后，一次性返回最终结果（State 或 Dict）。

### 2.2 为什么它是 Agent 的首选？

- **原子性保证:** Agent 的中间步骤（如工具调用）通常是阻塞式的原子操作，无法流式传输。`invoke` 等待这些操作全部完成。
- **内置容错:** 如果 Agent 在第 2 步工具报错，它可以自动自我修正重跑第 3 步。`invoke` 会掩盖这些尴尬的错误过程，只给用户看最终成功的结论。

### 2.3 代码示例

Python

```
# 定义 Agent
agent_executor = AgentExecutor(agent=agent, tools=tools)

# 同步调用
# 程序会在此处“卡住”，直到 Agent 跑完所有逻辑
result = agent_executor.invoke({"input": "查询上海明天的天气"})

# result 是一个包含完整上下文的字典
print(f"最终答案: {result['output']}")
print(f"中间步骤: {result['intermediate_steps']}") # 包含工具调用的详细历史
```

------

## 3. 流式调用：`agent.stream()`

对 Agent 使用流式调用比较复杂，因为 Agent 产生的数据流是**混合的**（Hybrid Stream）。

### 3.1 核心逻辑

当对 Agent 调用 `.stream()` 时，输出流中可能包含两种不同性质的数据块：

1. **Final Answer Chunks:** 最后生成给用户的自然语言（类似 ChatGPT 的打字机效果）。
2. **Intermediate Steps (可选):** 如果配置了相关参数，可能会流出工具调用的动作（JSON 片段）或工具的返回结果。

### 3.2 两种流式策略

在 Agent 开发中，我们需要区分“流式输出答案”和“流式展示思考过程”。

#### 策略 A: 仅流式输出最终答案 (Standard Stream)

这是最简单的用法。前端用户在等待期间看不到“正在搜索...”，但当 Agent 开始说话时，字是逐个蹦出来的。

Python

```
# 这种方式只会流式输出 Agent 决定“说话”时的内容
# 在 Agent “做事”（调工具）期间，流是静默的
for chunk in agent_executor.stream({"input": "给我讲个笑话"}):
    # chunk 通常包含最终生成的文本片段
    if "output" in chunk:
        print(chunk["output"], end="", flush=True)
```

#### 策略 B: 流式展示全过程 (Stream Events - **v0.2 推荐**)

如果你想实现类似 Kimi/Perplexity 那种 **“正在思考... 正在搜索 Google...”** 的效果，标准的 `.stream()` 是不够的，需要使用 **`.stream_events()`** API。

Python

```
# 使用 astream_events 获取细粒度事件
async for event in agent.astream_events({"input": "查天气"}, version="v1"):
    kind = event["event"]
    
    if kind == "on_chat_model_stream":
        # LLM 正在生成 Token (可能是思考，可能是回答)
        print(f"Token: {event['data']['chunk'].content}")
        
    elif kind == "on_tool_start":
        # Agent 决定调用工具了
        print(f"\n[正在调用工具]: {event['name']}...")
        
    elif kind == "on_tool_end":
        # 工具运行完毕
        print(f"\n[工具结果]: {event['data'].get('output')}")
```

------

## 4. 批量调用：`agent.batch()`

用于高并发场景。LangChain 会利用线程池并行启动多个 Agent 实例（即并行的 ReAct Loop）。

### 4.1 核心逻辑

- **并行 Loop:** 系统同时启动 N 个 `While` 循环。
- **独立状态:** 每个 Agent 维护自己独立的 Scratchpad（历史记录），互不干扰。
- **木桶效应:** 整体返回时间取决于“最费劲”的那个任务。

### 4.2 适用场景

- **数据富集:** 给 Excel 表里的 100 个公司名称，让 Agent 并行去 Google 搜索它们的 CEO 和 官网。
- **自动化评测:** 并行运行 50 个测试用例，评估 Agent 的准确率。

### 4.3 代码示例

Python

```
inputs = [
    {"input": "苹果公司的CEO是谁"},
    {"input": "微软公司的CEO是谁"},
    {"input": "英伟达公司的CEO是谁"}
]

# max_concurrency 控制并发数，防止触发 API Rate Limit
results = agent_executor.batch(inputs, config={"max_concurrency": 5})

for res in results:
    print(res['output'])
```

------

## 5. 总结对比矩阵 (Decision Matrix)

| **特性**       | **Invoke (同步)**            | **Stream (流式)**                                          | **Batch (批量)**           |
| -------------- | ---------------------------- | ---------------------------------------------------------- | -------------------------- |
| **Agent 视角** | 启动一个闭环任务，直到完成。 | 启动任务，并实时广播内部产生的数据。                       | 同时启动多个闭环任务。     |
| **返回数据**   | 完整的最终状态 (`Dict`)      | 数据片段 (`Chunk`) 或 事件 (`Event`)                       | 状态列表 (`List[Dict]`)    |
| **前端体验**   | 用户需等待 Loading 圈转完。  | 1. 打字机效果 (Final Answer) 2. 步骤进度条 (Stream Events) | 通常用于后台，无前端交互。 |
| **开发复杂度** | ⭐ (简单，推荐默认使用)       | ⭐⭐⭐ (复杂，需解析混合流)                                   | ⭐⭐ (中等，需关注并发限制)  |
| **最佳场景**   | **工具执行、复杂任务交付**   | **聊天机器人、长文本生成**                                 | **数据清洗、批量爬虫**     |

## 6. 专家建议 (Best Practices)

1. **默认使用 `invoke`:** 除非你有明确的理由（如降低首字延迟），否则 Agent 开发应默认使用 `invoke`。它能保证逻辑完整性，且屏蔽了大量脏数据。
2. **UI 展示用 `stream_events`:** 如果产品经理要求展示 Agent 的“思考过程”（UI 上的动态步骤卡片），不要费力去解析 `.stream()` 的原始字符串，直接使用 `.stream_events()` API，它是专门为此设计的。
3. **注意 `batch` 的消耗:** Agent 的 `batch` 消耗是惊人的。如果 1 个任务循环 5 次，`batch(10)` 就意味着短时间内触发 50 次 LLM 调用和 50 次工具调用，极易触发 API 风控。务必设置 `max_concurrency`。
