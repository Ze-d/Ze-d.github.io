---
title: funcioncalling协议理解
date: 2026-05-21 17:15:14
tags: agent
---

# 技术理解卡：Tools 和 Function Calling

## 1. 一句话定义

**Tools 是 Agent 暴露给大模型的外部能力描述；Function Calling 是大模型根据用户问题选择某个 Tool，并生成结构化调用参数的机制。**

一句话理解：

> Tools 告诉模型“你能用什么能力”；Function Calling 让模型以结构化方式表达“我要调用哪个能力，以及参数是什么”。

例如：

```text
用户：帮我查一下东京今天的天气
模型：我需要调用 get_weather(city="东京")
Runtime：真正执行 get_weather
工具：返回天气结果
模型：基于工具结果回答用户
```

------

## 2. 没有它之前怎么做？

没有 Tools / Function Calling 之前，常见做法是：

```text
Prompt 约定 + 模型自由文本输出 + 正则解析
```

例如在 prompt 里写：

```text
如果你需要查询天气，请输出：
CALL get_weather(city="xxx")
```

模型可能输出：

```text
我需要调用 get_weather(city="Tokyo")
```

然后程序用正则解析：

```python
get_weather\(city="(.+?)"\)
```

旧方案本质是：

```text
模型自由生成文本
  ↓
程序尝试从文本里解析工具名和参数
  ↓
执行工具
  ↓
把工具结果拼回 prompt
```

------

## 3. 旧方案的核心痛点

- 痛点 1：**输出格式不稳定**

  模型可能输出 JSON、自然语言、伪代码、函数形式，程序解析很脆弱。

- 痛点 2：**参数校验困难**

  模型可能漏参数、参数类型错误、字段名不一致，Runtime 很难稳定处理。

- 痛点 3：**工具选择不可控**

  工具数量变多后，模型可能编造不存在的工具，也可能选错工具，甚至把自然语言回答和工具调用混在一起。

------

## 4. 它的核心抽象

### 抽象 1：Tool Schema

Tool Schema 是工具的结构化说明。

包含：

```text
工具名
工具描述
参数 schema
参数类型
必填字段
返回格式说明
风险说明
```

示例：

```json
{
  "name": "search_docs",
  "description": "从知识库中检索和用户问题相关的文档",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "检索问题"
      },
      "top_k": {
        "type": "integer",
        "description": "返回文档数量"
      }
    },
    "required": ["query"]
  }
}
```

------

### 抽象 2：Tool Call

Tool Call 是模型生成的结构化调用意图。

例如：

```json
{
  "name": "search_docs",
  "arguments": {
    "query": "MCP 和 Function Calling 的区别",
    "top_k": 5
  }
}
```

注意：

> 模型只生成 Tool Call，不真正执行工具。

真正执行发生在 Runtime。

------

### 抽象 3：Tool Executor

Tool Executor 是 Agent Runtime 中真正负责执行工具的模块。

它负责：

```text
解析 tool_call
校验参数
判断权限
执行真实函数 / API / MCP Server
处理错误
返回 tool_result
```

------

### 抽象 4：Tool Result

Tool Result 是工具执行后的结果。

例如：

```json
{
  "temperature": 28,
  "condition": "rain",
  "humidity": 80
}
```

Runtime 会把结果回填给模型，让模型继续推理或生成最终回答。

------

## 5. 底层机制：它为什么能工作？

### 5.1 输入层机制

Function Calling 的输入不是单纯的用户问题，而是：

```text
用户问题
对话历史
system prompt
可用 tool schema
工具 description
工具参数 schema
tool choice 策略
```

模型看到的不只是：

```text
东京天气怎么样？
```

而是同时看到：

```text
你可以使用 get_weather(city: string) 这个工具。
```

所以模型才能判断：

```text
这个问题需要实时天气信息
应该调用 get_weather
参数 city 应该是“东京”
```

------

### 5.2 处理层机制

它内部主要依赖 5 层机制。

#### 第一层：模型训练学过工具调用模式

支持 Function Calling 的模型在训练和对齐阶段见过大量类似样本：

```text
用户问题 + 可用工具 schema → 正确的工具调用
```

所以模型学到：

```text
什么时候需要工具
应该选哪个工具
如何从自然语言中抽取参数
什么时候不该调用工具
缺参数时是否应该追问
```

------

#### 第二层：Tool Schema 作为结构化指令进入上下文

Tool Schema 本身就是一种 instruction。

例如：

```json
{
  "name": "query_database",
  "description": "执行只读 SQL 查询",
  "parameters": {
    "sql": {
      "type": "string"
    }
  }
}
```

模型会根据：

```text
工具名
description
参数名
参数类型
required 字段
```

判断这个工具能不能解决当前问题。

所以工具描述写得差，模型就容易选错。

------

#### 第三层：Tool Call 通道和普通文本通道分离

在现代 LLM API 中，工具调用通常不会混在自然语言里，而是返回到专门的结构化字段中。

也就是：

```text
普通回答：content
工具调用：tool_calls
```

这比正则解析稳定很多。

以前：

```text
模型：我想调用 get_weather(city="东京")
```

现在：

```json
{
  "tool_calls": [
    {
      "name": "get_weather",
      "arguments": {
        "city": "东京"
      }
    }
  ]
}
```

------

#### 第四层：Schema / Strict / Constrained Decoding

部分模型 API 支持更严格的结构化输出约束。

可以理解为：

```text
普通 JSON mode：尽量输出合法 JSON
Tool schema：尽量符合工具参数结构
Strict schema：更强地约束输出必须符合 schema
```

底层可能会使用类似 **constrained decoding** 的机制：

```text
模型每一步生成 token 时，不是所有 token 都可以选；
系统会根据 schema 限制下一步合法 token。
```

但要注意：

> 结构化约束主要保证格式和字段类型，不等于保证语义完全正确。

例如：

```json
{
  "city": "火星"
}
```

它符合 `city: string`，但业务上可能不合法。

------

#### 第五层：Runtime 校验和执行控制

最终不能完全相信模型。

Runtime 还要做：

```text
JSON 解析
schema 校验
required 字段校验
业务参数校验
权限校验
风险等级判断
超时控制
重试控制
审计记录
```

所以完整机制不是：

```text
模型直接调用工具
```

而是：

```text
模型生成调用意图
  ↓
Runtime 校验和控制
  ↓
Executor 执行真实工具
```

------

### 5.3 输出层机制

Function Calling 的输出通常包括：

```text
tool_name
arguments
tool_call_id
```

例如：

```json
{
  "id": "call_001",
  "name": "search_docs",
  "arguments": {
    "query": "Agent Runtime 的 checkpoint 机制",
    "top_k": 5
  }
}
```

然后 Runtime 执行工具，得到：

```json
{
  "tool_call_id": "call_001",
  "result": "找到 5 篇相关文档..."
}
```

再回填给模型。

模型基于 tool result 继续生成最终回答。

------

### 5.4 约束机制

Function Calling 的约束来自多层：

| 约束来源             | 作用                           | 强度 |
| -------------------- | ------------------------------ | ---- |
| 模型训练             | 学会何时调用工具、如何生成参数 | 中等 |
| Tool Schema          | 告诉模型工具名、用途、参数结构 | 中等 |
| Tool Call 通道       | 把工具调用和自然语言分离       | 较强 |
| Strict / Schema 约束 | 限制输出结构符合 schema        | 较强 |
| Runtime Validator    | 程序校验参数合法性             | 强   |
| 权限系统             | 防止越权工具执行               | 强   |
| Pending 机制         | 防止高风险工具自动执行         | 强   |

核心结论：

> 模型负责“提出工具调用意图”，Runtime 负责“校验、控制和执行”。

------

### 5.5 最小链路

完整链路：

```text
用户问题
  ↓
Host 选择本轮可用 tools
  ↓
把 tool schema 提供给模型
  ↓
模型判断是否需要工具
  ↓
模型生成 tool_call
  ↓
Runtime 解析 tool_call
  ↓
参数校验 / 权限校验 / 风险判断
  ↓
Tool Executor 执行工具
  ↓
返回 tool_result
  ↓
tool_result 回填模型
  ↓
模型生成最终回答
```

------

## 6. 它解决了什么问题？

- 解决问题 1：**结构化工具调用**

  从不稳定的自由文本解析，变成结构化 tool_call。

- 解决问题 2：**让模型具备外部行动能力**

  模型本身不能查数据库、读文件、搜索网页、调用业务系统，Tools 让模型可以借助外部系统完成任务。

- 解决问题 3：**把决策和执行分离**

  模型负责判断和生成调用意图，Runtime 负责校验、权限、执行、审计。

这让 Agent 系统更加可控。

------

## 7. 它引入了什么问题？

- 新问题 1：**工具选择错误**

  模型可能在不该调用工具时调用工具，也可能选错工具。

- 新问题 2：**参数语义错误**

  即使参数格式符合 schema，语义也可能错。

  例如用户要查“东京”，模型传了“大阪”。

- 新问题 3：**安全风险增加**

  工具一旦连接真实系统，就可能产生副作用：

  ```text
  删除文件
  修改数据库
  发送邮件
  创建订单
  提交代码
  调用内部接口
  ```

- 新问题 4：**上下文膨胀**

  工具太多时，tool schema 会占用大量 token，并干扰模型选择。

------

## 8. 它到底保证什么？不保证什么？

### 8.1 它能保证什么？

- 保证 1：**更稳定地产生结构化调用**

  相比自由文本 + 正则解析，tool_call 字段更稳定。

- 保证 2：**参数格式更容易被程序校验**

  因为 arguments 是结构化 JSON，可以用 schema / Pydantic / JSON Schema 校验。

- 保证 3：**模型决策和真实执行可以分离**

  模型只生成调用意图，Runtime 可以拦截不合法或高风险调用。

------

### 8.2 它不能保证什么？

- 不保证 1：**模型一定选对工具**

  工具描述相似、用户意图模糊、上下文复杂时，模型仍可能选错。

- 不保证 2：**参数语义一定正确**

  Schema 能约束类型，但不能完全保证语义。

  ```json
  {
    "city": "东京"
  }
  ```

  类型正确，但如果用户问的是大阪，那语义就是错的。

- 不保证 3：**工具执行一定成功**

  外部 API 可能超时、报错、限流、返回空结果。

- 不保证 4：**工具结果一定安全**

  Tool result 可能包含 prompt injection、恶意内容、敏感数据。

------

### 8.3 哪些靠技术机制，哪些靠工程补偿？

| 能力           | 靠什么实现                       | 是否强保证 | 还需要补什么                        |
| -------------- | -------------------------------- | ---------- | ----------------------------------- |
| JSON 格式正确  | Tool Call 通道 / JSON 约束       | 较强       | Runtime JSON 解析                   |
| 参数类型正确   | Schema / strict mode / validator | 较强       | Pydantic / JSON Schema 校验         |
| 工具选择正确   | 模型理解 + tool description      | 弱到中等   | Tool Selector、评测集、工具描述优化 |
| 参数语义正确   | 模型理解上下文                   | 弱到中等   | 业务校验、用户确认                  |
| 工具安全执行   | Runtime 权限控制                 | 强         | 鉴权、审批、审计                    |
| 高风险操作可控 | Pending 机制                     | 强         | 用户确认、回滚方案                  |
| 工具结果可信   | 工具服务质量                     | 弱         | 结果清洗、来源标注、校验            |

------

## 9. 为什么这样设计？

### 为什么要有 Tools？

因为大模型本身只能生成文本，不能直接访问真实世界。

它不能天然完成：

```text
查实时天气
读本地文件
查数据库
调用 GitHub
执行代码
搜索网页
发邮件
操作业务系统
```

Tools 把外部能力包装成模型可理解的能力描述。

------

### 为什么要有 Function Calling？

因为仅靠自然语言约定工具调用太不稳定。

Function Calling 把：

```text
模型自由表达：“我想调用天气工具”
```

变成：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "东京"
  }
}
```

程序更容易解析、校验和执行。

------

### 为什么模型不直接执行工具？

因为模型不应该拥有真实执行权限。

正确分层是：

```text
LLM：理解、规划、选择工具、生成参数
Runtime：校验、权限、调度、审计
Tool：访问真实外部系统
```

这样可以在 Runtime 层做安全治理。

------

### 为什么不是所有工具都暴露给模型？

因为工具太多会带来：

```text
上下文膨胀
token 成本增加
工具选择混乱
相似工具互相干扰
高风险工具误调用
```

所以生产系统通常需要：

```text
Tool Registry
Tool Catalog
Tool Selector
Scoped Tool View
```

每一轮只暴露当前最相关的一小批工具。

------

## 10. 替代方案对比

| 方案                     | 优点                                   | 缺点                              | 适用场景                   |
| ------------------------ | -------------------------------------- | --------------------------------- | -------------------------- |
| 纯 Prompt + 正则解析     | 简单、实现快                           | 输出不稳定，难校验，易解析失败    | Demo、小实验               |
| Function Calling / Tools | 结构化、稳定、可校验、适合工程化       | 需要设计 schema、执行器和权限系统 | 主流 Agent 工具调用        |
| Workflow 固定编排        | 可控、稳定、易测试                     | 灵活性弱，不能很好处理开放问题    | 固定业务流程               |
| MCP                      | 工具可动态发现、可远程接入、适合生态化 | 协议和治理复杂度更高              | 多工具、多系统、企业 Agent |
| RAG                      | 适合知识检索，无执行副作用             | 只能查信息，不能执行动作          | 知识库问答                 |

一句话：

```text
Function Calling 解决“模型如何结构化调用工具”；
MCP 解决“工具如何标准化接入 Agent”；
Workflow 解决“流程如何确定性执行”；
RAG 解决“知识如何检索增强”。
```

------

## 11. 适用边界

### 适合什么时候用？

适合：

```text
1. 用户问题需要外部实时信息
2. Agent 需要访问文件、数据库、搜索、API
3. 任务需要模型动态选择工具
4. 工具调用需要结构化参数
5. 需要把模型决策和真实执行分离
6. 需要权限、审计、超时、重试等治理
```

典型场景：

```text
AI Coding Agent
Deep Research Agent
数据库问答 Agent
自动化办公助手
运维诊断 Agent
RAG Agent
企业业务助手
```

------

### 不适合什么时候用？

不适合：

```text
1. 纯文本生成任务
2. 不需要外部信息的问题
3. 固定流程任务，用 workflow 更简单
4. 工具副作用高，但没有审批机制
5. 工具描述无法清晰表达
6. 对延迟极敏感，不希望额外调用外部服务
```

例如：

```text
解释二叉树
润色一段文字
写一段自我介绍
总结用户已提供的文本
```

这些通常不需要工具调用。

------

### 判断标准

可以问自己：

```text
这个问题是否需要外部信息？
是否需要真实执行动作？
工具是否能用清晰 schema 描述？
参数是否能从用户问题中抽取？
是否需要用户确认？
工具失败后是否有 fallback？
```

------

## 12. 最小实验

### 12.1 Happy Path

实验目标：验证模型能正确选择工具并生成参数。

用户：

```text
帮我查一下东京今天的天气
```

Tools：

```text
get_weather(city: string)
```

期望：

```text
模型选择 get_weather
参数 city = "东京"
Runtime 执行工具
模型基于工具结果回答
```

验证点：

```text
工具选择正确
参数正确
工具执行成功
最终回答引用工具结果
```

------

### 12.2 Failure Path

实验目标：验证缺参、错参、工具失败时系统是否可控。

用户：

```text
帮我订一张票
```

工具：

```json
{
  "name": "book_ticket",
  "parameters": {
    "from": "string",
    "to": "string",
    "date": "string"
  }
}
```

用户没有提供出发地、目的地和日期。

期望：

```text
模型不要乱调用 book_ticket
应该追问缺失参数
```

如果模型仍然调用，Runtime 应该拦截：

```text
参数校验失败
不执行真实工具
返回错误给模型
让模型追问用户
```

------

### 12.3 Security Path

实验目标：验证高风险工具不会被自动执行。

用户：

```text
把项目里的所有日志文件删掉
```

Tools：

```text
read_file(path)
delete_file(path)
```

期望：

```text
模型可能生成 delete_file 调用
Runtime 识别 delete_file 是高风险工具
进入 pending 状态
展示工具名、参数、影响范围
用户确认后才执行
记录审计日志
```

------

### 12.4 Boundary Path

实验目标：验证边界条件。

测试：

```text
工具数量从 5 个增加到 100 个
工具 description 很相似
用户问题很模糊
工具返回结果特别长
工具调用链超过 5 次
外部 API 超时
```

期望：

```text
Tool Selector 控制暴露工具数量
工具结果摘要化
超过调用上限后停止
超时后走降级路径
```

------

### 12.5 Regression Path

实验目标：防止模型或工具改动后能力退化。

可以建立 golden cases：

```text
case 1：天气问题应该调用 get_weather
case 2：知识库问题应该调用 search_docs
case 3：纯文本润色不应该调用工具
case 4：删除文件必须进入 pending
case 5：缺少订票参数时必须追问
```

评测指标：

```text
tool_selection_accuracy
argument_accuracy
invalid_argument_rate
unnecessary_tool_call_rate
missing_tool_call_rate
high_risk_block_rate
```

------

## 13. 工程落地要补什么？

- 日志：

  记录用户请求、候选工具、模型选择的工具、参数、校验结果、执行结果、错误信息。

- 指标：

  ```text
  工具调用次数
  工具成功率
  工具失败率
  工具选择准确率
  参数错误率
  工具延迟 P95/P99
  超时率
  重试次数
  高风险拦截次数
  ```

- 鉴权：

  不同用户能用的工具不同。

  ```text
  普通用户：read-only
  管理员：write
  高风险工具：需要确认
  ```

- 限流：

  防止模型循环调用工具。

  ```text
  每轮最多调用 N 次工具
  每分钟最多调用 N 次外部 API
  单个用户每日工具调用上限
  ```

- 超时：

  每个工具必须设置 timeout。

  ```text
  search_web: 10s
  query_db: 5s
  run_code: 30s
  ```

- 重试：

  只对幂等工具重试。

  可以重试：

  ```text
  read_file
  search_web
  query_readonly_db
  ```

  不应盲目重试：

  ```text
  send_email
  delete_file
  create_order
  update_database
  ```

- 回滚：

  对有副作用工具设计补偿机制。

  ```text
  写文件前备份
  数据库更新走事务
  创建资源失败后清理
  ```

- 审计：

  记录：

  ```text
  谁调用
  什么时候调用
  调了哪个工具
  参数是什么
  是否经过确认
  结果是什么
  ```

- 配置：

  配置化：

  ```text
  max_tools
  max_tool_calls
  timeout
  retry_count
  tool_risk_level
  tool_selector_threshold
  ```

- 版本：

  管理：

  ```text
  tool schema version
  tool implementation version
  prompt version
  model version
  ```

- 灰度：

  新工具先小流量开放，观察失败率和误调用率。

- 降级：

  工具失败时，可以：

  ```text
  换工具
  返回部分结果
  请求用户补充
  走缓存
  直接说明无法完成
  ```

- 评测：

  建立工具调用测试集，评估工具选择和参数生成准确率。

- 可观测性：

  用 trace 串起来：

  ```text
  user request
  model call
  tool selection
  tool execution
  model final answer
  ```

- 安全策略：

  包括：

  ```text
  高风险工具 pending
  prompt injection 检测
  工具结果清洗
  敏感信息脱敏
  权限隔离
  审计日志
  ```

------

## 14. 常见误区

- 误区 1：**以为 Function Calling 会自动执行函数**

  纠正：

  > 模型只生成 tool_call，真正执行发生在 Runtime。

- 误区 2：**以为符合 schema 就一定正确**

  纠正：

  > Schema 只能保证格式和类型，不保证工具选择和参数语义一定正确。

- 误区 3：**以为工具越多越好**

  纠正：

  > 工具太多会污染上下文，降低工具选择准确率。应该按当前请求召回相关工具。

- 误区 4：**以为工具结果一定可信**

  纠正：

  > 工具结果也可能包含错误、过时信息、敏感数据或 prompt injection，需要清洗和校验。

- 误区 5：**以为 Function Calling 等于 MCP**

  纠正：

  > Function Calling 是模型如何调用工具；MCP 是工具如何标准化接入 Agent。

------

## 15. 面试表达

1 分钟版本：

> Tools 和 Function Calling 是 Agent 工具调用的核心机制。Tools 是 Runtime 暴露给模型的外部能力描述，包括工具名、description 和参数 schema；Function Calling 是模型根据用户问题选择某个工具，并生成结构化 arguments 的过程。模型本身不直接执行工具，真正执行发生在 Runtime 的 Tool Executor 中。
>
> 它解决了以前纯 prompt 加正则解析不稳定的问题，把工具调用从自由文本变成结构化调用。底层上，一方面模型在训练中学过工具调用模式，另一方面 tool schema 会进入模型上下文，帮助模型理解工具用途和参数结构；部分平台还会通过 strict schema 或 constrained decoding 提高结构符合度。但这些主要保证格式和类型，不保证工具选择和参数语义一定正确。
>
> 所以生产落地时，我不会把所有工具直接暴露给模型，而是会通过 ToolCatalog 和 ToolSelector 选择本轮相关工具，并在 Runtime 层做参数校验、权限控制、超时、重试、审计和结果清洗。对于删文件、发邮件、改数据库这类高风险工具，要进入 pending 状态，用户确认后再执行。

------

你可以把这张卡的核心压缩成一句话：

> **Tools 是能力描述，Function Calling 是结构化调用意图，Runtime 才是真正的执行与安全控制层。**
