---
title: langchain标准化prompt
date: 2026-01-11 12:32:42
tags: [llm,langchain]
---

## 1. 问题背景 (Problem Statement)

在传统软件工程中，标准化依赖于精确的语法（Syntax）和强类型系统。而在基于 LangChain 的 LLM 应用开发中，核心逻辑依赖于自然语言（Prompts 和 Tool Descriptions）。

**面临的挑战：**

- **不确定性：** 自然语言具有歧义性，导致系统行为难以像传统代码那样 100% 预测。
- **非标化：** 不同的开发者编写 Prompt 的质量参差不齐，导致系统稳定性波动。

**核心论点：** LangChain 无法消除对 Prompt 的依赖（这是 LLM 的物理特性），但它提供了一套**工程化框架**，将模糊的自然语言逻辑包裹在标准化的软件架构中。

------

## 2. 范式转移：自然语言作为新汇编 (The Paradigm Shift)

在 AI 工程中，我们需要重新映射编程概念：

| **传统编程概念**         | **AI 工程对应物**          | **本质变化**                           |
| ------------------------ | -------------------------- | -------------------------------------- |
| **函数签名 (Signature)** | **工具描述 (Description)** | 从“类型匹配”变为“语义理解”。           |
| **业务逻辑 (Logic)**     | **Prompt Template**        | 从“流程控制 (if/else)”变为“意图引导”。 |
| **编译 (Compile)**       | **推理 (Inference)**       | 从“确定性翻译”变为“概率性预测”。       |

------

## 3. 标准化解决方案 (Standardization Solutions)

为了解决上述挑战，LangChain 引入了三个维度的标准化手段：

### 3.1 接口标准化：Pydantic 强类型约束

不要依赖纯文本来描述工具参数，必须使用类型系统。

- **原理：** 利用 `pydantic` 定义数据模型，强制 LLM 输出符合 Schema 的 JSON。

- **代码规范：**

  - ❌ **非标做法：** 仅在 docstring 中写“请输入城市名”。
  - ✅ **标准做法：** 使用 `args_schema` 强制约束。

  Python

  ```
  class WeatherInput(BaseModel):
      city: str = Field(description="城市的全称，如 Shanghai，不可使用缩写")
      days: int = Field(default=1, le=7, description="预测天数，最大7天")
  
  @tool(args_schema=WeatherInput)
  def get_weather(city: str, days: int): ...
  ```

- **效果：** 将自然语言的随意性限制在严格的 JSON 结构内，后续代码可以安全处理。

### 3.2 资产标准化：Prompt 版本控制

将 Prompt 视为代码资产（Assets），而非硬编码字符串。

- **原理：** 解耦 Prompt 与 Python 代码。
- **工具：** LangChain Hub / PromptTemplates。
- **实施：**
  - 所有的 Prompt 必须提取为独立模板。
  - 引入版本号管理（如 `react-agent-prompt:v2`）。
  - 代码只负责拉取模板和填充变量，不包含具体的 Prompt 文本。

### 3.3 质量标准化：评估驱动开发 (EDD)

既然 Prompt 质量难以凭感觉衡量，就必须建立量化指标。

- **原理：** 用单元测试的思想测试 Prompt。
- **工具：** LangSmith / Evals。
- **流程：**
  1. 建立“黄金数据集”（Golden Dataset），包含 50+ 个典型输入和期望输出。
  2. 每次修改 Tool Description 或 Prompt 后，自动运行测试。
  3. 监控指标：准确率、召回率、幻觉率。

------

## 4. 最佳实践：防御性编程 (Defensive Programming)

在“概率性”内核之外，构建“确定性”的防护层。

1. **文档即代码 (Docs as Code):** 工具描述必须精确、无歧义。例如，不要写“查询数据”，要写“通过用户 ID 查询最近 3 笔订单的详细信息”。
2. **结构化输出 (Structured Output):** 永远优先要求模型输出结构化数据（JSON/XML），而非自然语言，以便程序解析。
3. **错误恢复 (Self-Correction):** 针对 LLM 可能的格式错误或幻觉，编写校验器（Validator）。如果校验失败，自动将错误信息回传给 LLM 让其重试（Retry）。
4. **人机回环 (Human-in-the-loop):** 对于高风险操作，标准化流程中必须包含人工确认步骤。

------

## 5. 未来展望 (Future Outlook)

随着技术演进，手动编写 Prompt 的需求将逐渐降低，标准化程度将进一步提高：

- **自动优化 (DSPy):** 未来我们将定义“任务目标”和“评估指标”，由算法自动搜索和优化最佳 Prompt，彻底消除人工编写 Prompt 的非标化问题。
- **模型内化:** 越来越多的通用工具调用能力正在被训练进模型权重中，对外部 Prompt 的依赖正在减少。

## 6. 总结

LangChain 开发的本质是 Manager（管理者） 角色：

你无法直接控制员工（LLM）大脑里的每一个神经元（概率），但你可以通过制定严格的工作手册（Pydantic）、提供标准的话术（Prompt Templates）和进行定期的绩效考核（Evaluation），来保证产出的标准化和高质量。
