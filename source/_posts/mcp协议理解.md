---
title: mcp协议理解
date: 2026-05-21 16:26:15
tags: agent
---

# 技术理解卡：MCP

## 1. 一句话定义

**MCP，Model Context Protocol，是一种让 AI Agent / LLM 应用以标准化方式连接外部工具、数据源和上下文资源的协议。**

它常用于：

- AI IDE 接入代码仓库、文件系统、GitHub、数据库；
- 企业 Agent 接入内部知识库、业务系统、工单系统；
- 多工具 Agent 平台统一管理本地工具、远程工具和外部 API。

简单说：

> MCP 是 Agent 连接外部世界的标准协议层。

官方定位是：MCP 标准化了应用向 LLM 提供上下文的方式，并让 AI 应用可以连接不同数据源和工具。([Model Context Protocol](https://modelcontextprotocol.io/routing?utm_source=chatgpt.com))

------

## 2. 没有它之前怎么做？

没有 MCP 之前，通常是每个 Agent 项目自己写工具集成：

```text
Agent Runtime
  ├── 手写 GitHub API 调用
  ├── 手写数据库查询工具
  ├── 手写文件系统工具
  ├── 手写搜索工具
  ├── 手写 Jira / 飞书 / Notion API 适配
  └── 手写工具 schema 注入逻辑
```

也就是说：

```text
每个 AI 应用 × 每个外部系统 = 一套单独适配代码
```

例如：

- Cursor 要接 GitHub，写一套；
- Claude Desktop 要接 GitHub，再写一套；
- 你自己的 Agent 要接 GitHub，又写一套；
- 每个系统的鉴权、参数校验、返回格式、错误处理都不统一。

------

## 3. 旧方案的核心痛点

- 痛点 1：**集成成本高**

  每个 Agent 应用都要重复对接外部系统，代码复用性差。

- 痛点 2：**工具没有统一发现机制**

  以前工具通常是写死在代码里的。Agent 很难动态知道：当前有哪些工具、工具参数是什么、哪些工具可用、哪些工具不可用。

- 痛点 3：**安全和权限边界混乱**

  如果每个工具都自己处理权限、鉴权、日志、审计，就容易出现工具越权、敏感数据泄露、高风险操作误执行等问题。

------

## 4. 它的核心抽象

- 抽象 1：**Host**

  Host 是 AI 应用本体，比如 Claude Desktop、Cursor、AI IDE、你自己写的 Agent Runtime。
  它负责管理模型调用、用户会话、上下文、权限策略和多个 MCP Client。

- 抽象 2：**Client**

  Client 是 Host 内部的协议客户端。一个 MCP Client 通常连接一个 MCP Server，并维护一条隔离的会话连接。官方架构里，一个 Host 可以创建多个 Client，每个 Client 与一个 Server 建立 1:1 连接。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/architecture?utm_source=chatgpt.com))

- 抽象 3：**Server**

  Server 是能力提供方，负责暴露外部工具、资源和 Prompt 模板。比如 filesystem-server、github-server、database-server、browser-server。MCP Server 可以提供 tools、resources、prompts 等能力。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/architecture?utm_source=chatgpt.com))

核心结构：

```text
User
  ↓
Host：AI 应用 / Agent Runtime
  ↓
MCP Client
  ↓
MCP Server
  ↓
外部系统：文件、数据库、GitHub、搜索、业务 API
```

------

## 5. 它解决了什么问题？

- 解决问题 1：**统一外部能力接入**

  MCP 把文件、数据库、GitHub、搜索、内部 API 等能力统一抽象成标准协议，Host 可以用统一方式连接不同 Server。

- 解决问题 2：**支持动态能力发现**

  MCP Server 可以声明自己支持哪些 tools、resources、prompts。Host 不需要提前把所有工具硬编码死，而是可以连接后动态发现。

- 解决问题 3：**分层隔离和权限治理**

  MCP 把 AI 应用、协议连接、外部能力提供方分开。Host 负责用户授权、上下文聚合和安全策略；Server 只暴露专门能力，不应该看到完整对话，也不应该看到其他 Server 的内容。官方架构原则也强调 Server 不应读取完整对话，跨 Server 交互应由 Host 控制。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/architecture?utm_source=chatgpt.com))

------

## 6. 它引入了什么问题？

- 新问题 1：**安全风险变大**

  MCP 让 Agent 更容易接触真实系统，例如文件、数据库、代码仓库、邮件、工单系统。能力越强，误调用、越权调用、Prompt Injection、敏感信息泄露的风险越高。

- 新问题 2：**工具数量膨胀**

  一个 MCP Server 可能暴露几十个工具，多个 Server 叠加后工具数量会非常多。如果全部塞进模型上下文，会造成 token 浪费、工具选择混乱和调用准确率下降。

- 新问题 3：**工程治理复杂**

  生产环境不能只“能调用 MCP Server”就结束，还要补鉴权、限流、超时、重试、熔断、审计、日志、指标、权限审批、工具结果清洗等能力。

------

## 7. 为什么这样设计？

### 为什么要有 Host？

因为 Host 是用户真正交互的 AI 应用，它最适合做全局控制：

```text
用户权限
模型调用
上下文管理
工具选择
安全策略
用户确认
结果回填
```

如果让 MCP Server 直接管理完整上下文和用户权限，会导致安全边界不清晰。

所以 MCP 的设计是：

```text
完整上下文留在 Host；
Server 只拿到完成任务所需的最小上下文。
```

------

### 为什么要有 Client？

Client 是 Host 和 Server 之间的协议适配层。

它负责：

```text
连接管理
initialize 初始化
协议版本协商
capability 能力协商
消息路由
通知处理
会话隔离
```

官方生命周期中，MCP 连接包含 Initialization、Operation、Shutdown 三个阶段；初始化阶段会进行协议版本兼容和能力协商。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle?utm_source=chatgpt.com))

------

### 为什么要有 Server？

Server 的目标是把外部系统封装成 Agent 可用的能力。

例如：

```text
GitHub MCP Server：封装 GitHub API
Filesystem MCP Server：封装文件读写
Database MCP Server：封装数据库查询
Browser MCP Server：封装网页浏览能力
```

这样外部系统只需要实现一次 MCP Server，就可以被多个 AI Host 使用。

------

### 为什么不是直接 REST API？

REST API 面向普通软件系统，主要是开发者手写调用。

MCP 面向 Agent，更关注：

```text
工具发现
上下文资源
Prompt 模板
能力协商
双向消息
模型采样
用户补充信息
权限隔离
```

MCP 底层消息基于 JSON-RPC，并支持 stdio 和 Streamable HTTP 等传输方式。官方传输规范中，stdio 用于本地进程通信，Streamable HTTP 适合远程 Server，并可结合 SSE 做流式消息。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports?utm_source=chatgpt.com))

------

## 8. 替代方案对比

| 方案                            | 优点                                        | 缺点                                            | 适用场景                             |
| ------------------------------- | ------------------------------------------- | ----------------------------------------------- | ------------------------------------ |
| 手写工具函数                    | 简单、直接、可控                            | 复用差、标准化差、每个项目重复写                | 小 Demo、工具数量少、短期项目        |
| Function Calling / Tool Calling | 模型能结构化表达工具调用意图                | 不负责工具发现、远程连接、协议通信、能力协商    | 单个 Agent 内部调用本地函数          |
| OpenAPI / REST API              | 生态成熟、适合传统服务调用                  | 对 Agent 的上下文、Prompt、资源、能力协商支持弱 | 传统后端服务、业务 API               |
| 插件系统                        | 可以扩展能力                                | 各家插件标准不统一，迁移成本高                  | 特定平台生态                         |
| MCP                             | 标准化、可发现、可组合、适合 Agent 工具生态 | 引入协议复杂度和安全治理成本                    | AI IDE、Agent 平台、企业内部工具接入 |

一句话区分：

```text
Function Calling 解决“模型怎么表达要调用哪个工具”；
MCP 解决“工具从哪里来、怎么发现、怎么连接、怎么隔离、怎么治理”。
```

------

## 9. 适用边界

### 适合什么时候用？

适合：

```text
1. Agent 需要接入多个外部工具
2. 工具来源不固定，需要动态发现
3. 多个 AI 应用希望复用同一批工具
4. 企业内部有很多系统要接入 Agent
5. 需要统一做权限、审计、安全治理
6. 需要本地工具和远程工具混合接入
```

典型场景：

```text
AI IDE
企业知识库 Agent
代码审查 Agent
数据分析 Agent
自动化运维 Agent
Deep Research Agent
多工具个人助手
```

### 不适合什么时候用？

不适合：

```text
1. 只是写一个简单 Demo
2. 只有 1-2 个固定本地函数
3. 工具调用完全写死，不需要动态发现
4. 外部系统没有复用价值
5. 安全治理能力不足，但要接高风险系统
6. 对延迟极其敏感，不希望引入额外协议层
```

例如，只是做一个天气查询 Demo，直接写 function calling 就够了，不一定需要 MCP。

------

## 10. 最小实验

### Happy Path

目标：验证 MCP 的基本工具调用链路。

实验设计：

```text
1. 写一个最小 MCP Server
2. 暴露一个 add(a, b) 工具
3. Host / Client 连接 Server
4. 执行 initialize
5. 调用 tools/list 获取工具列表
6. 调用 tools/call 执行 add
7. 返回结果 3
```

最小链路：

```text
Client -> Server: initialize
Client -> Server: tools/list
Client -> Server: tools/call add {"a":1,"b":2}
Server -> Client: result 3
```

验证点：

```text
连接成功
能力发现成功
参数校验成功
工具执行成功
结果返回成功
```

------

### Failure Path

目标：验证 MCP Server 的错误处理。

实验设计：

```text
1. 调用不存在的工具
2. 传错误参数类型
3. 模拟工具内部异常
4. 模拟 Server 超时
```

例如：

```json
{
  "name": "add",
  "arguments": {
    "a": "hello",
    "b": 2
  }
}
```

期望：

```text
参数校验失败
返回结构化错误
Host 不崩溃
Agent 可以决定重试、换工具或向用户说明失败原因
```

------

### Security Path

目标：验证高风险工具和越权访问控制。

实验设计：

```text
1. 暴露 read_file 工具
2. 只允许访问当前 workspace
3. 尝试读取 ../secret.txt
4. 暴露 delete_file 工具
5. delete_file 被召回后不直接执行，而是进入 pending
6. 用户确认后才执行
```

验证点：

```text
路径逃逸被拒绝
高风险工具不自动执行
工具参数展示给用户
执行记录进入审计日志
```

------

## 11. 工程落地要补什么？

- 日志：

  记录 MCP Server 连接、initialize、tools/list、tools/call、错误码、调用参数摘要、调用结果摘要。

- 指标：

  记录工具调用次数、成功率、失败率、P95/P99 延迟、超时率、重试次数、不同 Server 健康状态。

- 鉴权：

  本地 stdio 要限制 workspace 和环境变量；远程 Streamable HTTP 要做 OAuth、Bearer Token、SSO、租户隔离、权限 scope。MCP 的 Streamable HTTP 传输是远程 Server 常用方式，支持 HTTP POST/GET 和可选 SSE，因此生产环境必须认真处理认证和安全边界。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports?utm_source=chatgpt.com))

- 限流：

  按用户、租户、Server、工具维度限流，避免 Agent 循环调用或外部 API 被打爆。

- 超时：

  每个工具设置超时时间，例如：

  ```text
  search_web: 10s
  query_db: 5s
  run_test: 60s
  browser_action: 15s
  ```

- 重试：

  只对幂等、可重试的工具做重试。写操作、发邮件、支付、删除文件等非幂等操作不能盲目重试。

- 回滚：

  对有副作用的工具设计补偿逻辑。例如创建工单后失败，可以关闭工单；写文件前保存旧版本；数据库变更必须走事务或审批。

- 审计：

  记录谁在什么时间通过哪个 Agent 调用了哪个 MCP Server 的哪个工具，参数是什么，结果是什么，是否经过用户确认。

除此之外，生产级 MCP 还应该补：

```text
ToolCatalog
工具风险分级
工具健康检查
熔断机制
Prompt Injection 检测
工具结果清洗
namespace 隔离
token budget 控制
工具 schema 版本管理
```

------

## 12. 面试表达

1 分钟版本：

> MCP 是 Model Context Protocol，本质上是 Agent 和外部工具、数据源、上下文资源之间的标准化连接协议。没有 MCP 之前，每个 Agent 项目都要自己手写 GitHub、数据库、文件系统、搜索等工具适配，集成成本高，而且工具发现、权限、安全和审计都不统一。
>
> MCP 采用 Host、Client、Server 的架构。Host 是 AI 应用本体，负责模型调用、上下文管理和安全策略；Client 是协议连接层，负责和某个 MCP Server 建立会话、做初始化和能力协商；Server 是能力提供方，暴露 tools、resources、prompts。底层通信基于 JSON-RPC，常见传输方式包括本地 stdio 和远程 Streamable HTTP。
>
> 它的价值是让外部能力从“写死在 Agent 代码里”变成“可发现、可复用、可隔离、可治理的标准能力服务”。但 MCP 也会带来安全和工程治理问题，所以生产落地时不能把所有工具直接暴露给模型，而应该通过 ToolCatalog 做工具召回、权限过滤、健康检查、token budget 控制；对写文件、删数据、发消息这类高风险工具，要进入 pending 状态，用户确认后再执行，并配合日志、指标、限流、超时、审计和工具结果清洗。
