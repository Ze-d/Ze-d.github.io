---
title: SSE通信方式
date: 2026-04-10 20:33:07
tags: [cn,agent]
---

SSE 面试知识点总结

## 1. 什么是 SSE

SSE，全称 **Server-Sent Events**，是一种基于 **HTTP** 的服务端向客户端**单向推送**消息的技术。

它的核心特点是：

- 客户端发起一次 HTTP 请求
- 服务端不立即关闭连接
- 后续通过这个连接持续向客户端推送事件数据
- 客户端实时接收并处理这些数据

前端通常使用浏览器原生提供的 `EventSource` 来接收 SSE 消息。

---

## 2. 一句话定义

面试里可以这样回答：

> SSE 是一种基于 HTTP 的服务端单向流式推送技术，适合将服务端持续产生的数据实时发送给客户端，比如大模型流式输出、任务进度通知、日志推送等。

---

## 3. SSE 的本质

SSE 本质上不是一个全新的复杂协议，而是：

- 仍然使用 HTTP
- 响应头通常设置为 `Content-Type: text/event-stream`
- 服务端将响应保持打开
- 按约定格式不断向响应流中写入事件

所以你可以把它理解成：

**“长连接版的 HTTP 响应流 + 事件格式约定”**

---

## 4. SSE 的通信模型

### 4.1 单向通信

SSE 是 **服务端 -> 客户端** 的单向通信。

也就是说：

- 客户端只能在最开始通过 HTTP 请求发起连接
- 后续不能像 WebSocket 那样在同一连接上自由发送消息
- 服务端持续推送内容给客户端

### 4.2 典型场景

适合 SSE 的场景：

- LLM / ChatGPT 类应用的流式输出
- Agent 执行过程展示
- 检索进度推送
- 日志流
- 任务执行状态通知
- 股票价格、监控数据等单向推送场景

---

## 5. SSE 的消息格式

SSE 消息是文本格式，每条消息由若干行组成，事件之间用空行分隔。

例如：

```text
event: token
id: 1
data: hello

event: token
id: 2
data: world
```

### 5.1 常见字段

#### `event`

表示事件名。

例如：

```text
event: token
```

前端可以根据不同事件名做不同处理。

#### `data`

表示事件数据，是最核心的字段。

例如：

```text
data: {"text":"hello"}
```

#### `id`

表示当前事件的唯一 ID。

例如：

```text
id: 123
```

用于断线重连时恢复状态。

#### `retry`

表示建议客户端断线后重连的等待时间，单位毫秒。

例如：

```text
retry: 3000
```

### 5.2 事件结束标志

SSE 中一条完整事件必须以**空行**结束。

------

## 6. SSE 和普通 HTTP 的区别

### 普通 HTTP

普通 HTTP 是：

- 请求一次
- 响应一次
- 连接结束

### SSE

SSE 是：

- 请求一次
- 响应持续不断地产生
- 连接保持打开
- 服务端不断往里面写事件

所以 SSE 其实就是：

**将 HTTP 从“一次性返回”变成“持续流式返回”**

------

## 7. SSE 和 WebSocket 的区别

这是面试里的高频问题。

## 7.1 核心区别总结

| 对比项         | SSE                        | WebSocket                          |
| -------------- | -------------------------- | ---------------------------------- |
| 通信方向       | 单向，服务端推客户端       | 双向，客户端和服务端都可主动发消息 |
| 底层协议       | HTTP                       | 握手后升级为 WebSocket 协议        |
| 复杂度         | 相对简单                   | 更复杂                             |
| 浏览器支持方式 | `EventSource`              | `WebSocket`                        |
| 自动重连       | 浏览器天然支持             | 通常需要自己实现                   |
| 适合场景       | 流式输出、进度、日志、通知 | 聊天、协同编辑、实时对战等强交互   |

------

## 8. 面试时如何回答 SSE 和 WebSocket 的选型

可以这样说：

> 如果场景只是服务端持续向前端推送数据，比如大模型 token 流式输出、Agent 执行进度、日志推送，我会优先选 SSE，因为它基于 HTTP，实现简单，前端接收也方便。
> 如果场景需要高频双向交互，比如聊天室、协同编辑、实时控制，我会选择 WebSocket，因为它支持客户端和服务端双向通信。

------

## 9. SSE 的优点

### 9.1 实现简单

因为 SSE 基于 HTTP，不需要像 WebSocket 那样处理升级协议和复杂双向通信。

### 9.2 对流式输出天然友好

特别适合大模型输出 token 流。

### 9.3 浏览器原生支持

前端可以直接使用 `EventSource`。

### 9.4 自动重连机制

浏览器端默认支持断线重连。

### 9.5 更容易接入现有 HTTP 基础设施

很多网关、鉴权、日志、监控链路都是围绕 HTTP 构建的，SSE 更容易融入现有系统。

------

## 10. SSE 的缺点

### 10.1 只能单向通信

客户端无法像 WebSocket 那样在同一连接中自由发送业务消息。

### 10.2 不适合强交互场景

比如实时聊天、多人协作、游戏对战等。

### 10.3 代理和网关可能缓冲响应

如果代理层开启缓冲，可能导致服务端已经推了数据，但客户端迟迟收不到。

### 10.4 需要处理心跳保活

连接长时间没数据时，中间层可能断开连接。

------

## 11. SSE 的核心 HTTP 头

面试中常被问到。

常见响应头：

```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

### 解释

- `Content-Type: text/event-stream`
  告诉客户端这是 SSE 事件流
- `Cache-Control: no-cache`
  避免缓存影响实时性
- `Connection: keep-alive`
  保持连接不断开

------

## 12. 前端如何接收 SSE

前端最常见写法：

```javascript
const es = new EventSource("/sse");

es.onmessage = (event) => {
  console.log(event.data);
};

es.addEventListener("token", (event) => {
  console.log("token:", event.data);
});

es.onerror = (err) => {
  console.error("error:", err);
};
```

### 面试回答点

- 前端一般通过 `EventSource` 建立 SSE 连接
- 可以监听默认的 `message` 事件
- 也可以监听自定义事件名
- 出错时可以在 `onerror` 里处理
- 浏览器会自动尝试重连

------

## 13. Python 中如何实现 SSE

在 Python 面试中，通常结合 **FastAPI** 来回答。

### 13.1 基本思路

- 写一个生成器或异步生成器
- 每次 `yield` 一段符合 SSE 格式的字符串
- 使用 `StreamingResponse` 返回
- 设置 `media_type="text/event-stream"`

### 13.2 示例

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def event_generator():
    for i in range(5):
        yield f"event: message\ndata: hello {i}\n\n"
        await asyncio.sleep(1)

@app.get("/sse")
async def sse():
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

### 13.3 面试中要能讲清楚

- `StreamingResponse` 用于流式响应
- SSE 本质上是不断 `yield` 数据
- 每个事件要符合 SSE 格式
- `\n\n` 表示一条事件结束
- FastAPI 下常配合协程和异步生成器使用

------

## 14. Java 中如何实现 SSE

Java 面试里通常结合 Spring 来讲。

------

### 14.1 Spring MVC：`SseEmitter`

这是最常见的回答方式。

```java
@GetMapping("/sse")
public SseEmitter stream() {
    SseEmitter emitter = new SseEmitter();

    new Thread(() -> {
        try {
            for (int i = 0; i < 5; i++) {
                emitter.send(SseEmitter.event().name("message").data("hello " + i));
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    }).start();

    return emitter;
}
```

### 面试回答点

- `SseEmitter` 是 Spring MVC 对 SSE 的支持
- Controller 返回 `SseEmitter`
- 后台线程持续往里面 `send`
- 最后调用 `complete()` 结束连接

------

### 14.2 Spring WebFlux：`Flux<ServerSentEvent<T>>`

如果项目使用响应式编程，可以这样实现：

```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(i -> ServerSentEvent.<String>builder()
            .event("message")
            .data("hello " + i)
            .build());
}
```

### 面试回答点

- WebFlux 中通常返回 `Flux`
- 每个元素就是一个 SSE 事件
- 更适合响应式、非阻塞场景
- 比 `SseEmitter` 更适合高并发流式系统

------

## 15. Python 和 Java 实现 SSE 的区别

### Python / FastAPI

思路更偏向：

- 生成器 / 异步生成器
- 手动拼 SSE 格式
- 用 `StreamingResponse` 输出

### Java / Spring MVC

思路更偏向：

- 用 `SseEmitter` 作为抽象
- 后台线程不断 `send`

### Java / WebFlux

思路更偏向：

- 返回响应式事件流 `Flux`
- 框架自动处理流式输出

### 面试总结说法

> Python 更像是自己手动生成事件流；Java Spring MVC 更像是往 `SseEmitter` 中持续发送事件；WebFlux 则是直接返回一个响应式事件流。

------

## 16. SSE 在 LLM / Agent 场景中的应用

这是现在面试中特别高频的点。

### 16.1 为什么适合 LLM 流式输出

LLM 的生成过程天然是增量式的：

- 用户发送一个问题
- 服务端开始调用模型
- 模型持续返回 token
- 服务端边收到边推给前端
- 前端边接收边渲染

这和 SSE 的单向持续推送模型非常契合。

------

### 16.2 Agent 场景中可以推哪些事件

不要只推文本，可以设计为结构化事件：

```text
event: status
data: {"phase":"retrieving"}

event: tool_call
data: {"tool":"search","query":"SSE"}

event: tool_result
data: {"count":3}

event: token
data: {"text":"SSE 是一种..."}

event: done
data: {"finish_reason":"stop"}
```

### 这样设计的好处

- 前端可以区分不同类型的信息
- 更容易展示工具调用过程
- 更适合做 tracing 和调试
- 更利于面试时体现工程思维

------

## 17. SSE 的工程问题

面试里如果你能说到这些，会很加分。

### 17.1 客户端断开检测

客户端刷新页面或关闭页面后，服务端要尽快停止推送，避免资源浪费。

Python/FastAPI 中一般会检测请求是否断开。
Java 中也要监听连接状态或捕获发送异常。

------

### 17.2 心跳机制

为了防止长时间无数据导致连接被中间层断开，可以周期性发送注释行：

```text
: ping
```

这不属于业务事件，但可以起到保活作用。

------

### 17.3 代理缓冲问题

某些 Nginx 或网关默认会缓冲响应，导致前端接收不到实时数据。

所以部署 SSE 时要注意：

- 关闭代理缓冲
- 确保响应是实时刷出的
- 检查超时配置

------

### 17.4 显式结束事件

不要只依赖连接断开来表示结束，最好显式发送一个 `done` 事件。

例如：

```text
event: done
data: {"ok": true}
```

这样前端更容易做状态管理。

------

### 17.5 错误事件统一设计

建议将错误也作为事件输出：

```text
event: error
data: {"message":"tool timeout"}
```

这样前端能更优雅地处理异常，而不是只感知到连接中断。

------

## 18. SSE 的断线重连

SSE 的一个优势是浏览器支持自动重连。

### 相关机制

- 服务端可以发送 `id`
- 客户端重连时可携带上次收到的事件 ID
- 服务端可以基于这个 ID 判断是否需要恢复

### 面试里要注意的点

浏览器自动重连只是基础能力。
如果想真正实现“断点续传”，服务端必须自己保存事件状态或可恢复的上下文。

------

## 19. 高频面试题整理

## 题目 1：SSE 是什么？

**答：**
SSE 是一种基于 HTTP 的服务端单向推送技术，服务端可以通过长连接持续向客户端推送事件流，常用于流式输出、日志、进度通知等场景。

------

## 题目 2：SSE 和 WebSocket 的区别？

**答：**
SSE 是单向的，只能服务端推客户端，基于 HTTP；WebSocket 是双向的，支持客户端和服务端互相主动发消息，适合更强的实时交互场景。

------

## 题目 3：为什么 LLM 流式输出常用 SSE？

**答：**
因为 LLM 输出通常是服务端持续向前端推送 token，属于典型的单向流式场景，SSE 实现简单、前端接收方便、天然适合这种模式。

------

## 题目 4：SSE 的消息格式是怎样的？

**答：**
SSE 消息由文本行构成，常见字段有 `event`、`data`、`id`、`retry`，事件之间通过空行分隔。

------

## 题目 5：FastAPI 如何实现 SSE？

**答：**
通常使用异步生成器不断 `yield` 符合 SSE 格式的字符串，再通过 `StreamingResponse` 返回，并设置 `media_type="text/event-stream"`。

------

## 题目 6：Spring 如何实现 SSE？

**答：**
Spring MVC 通常使用 `SseEmitter`；Spring WebFlux 通常使用 `Flux<ServerSentEvent<T>>` 来实现流式事件推送。

------

## 题目 7：SSE 的缺点是什么？

**答：**
只能单向通信，不适合高频双向交互；在代理和网关场景下可能存在缓冲和超时问题；需要额外考虑心跳和连接管理。

------

## 题目 8：SSE 为什么比 WebSocket 更适合某些 AI 场景？

**答：**
因为很多 AI 场景只需要服务端持续向客户端输出数据，不需要双向通信。SSE 基于 HTTP，接入简单，更容易和现有后端系统整合。

------

## 20. 面试时推荐的总结模板

可以直接背这一段：

> SSE 是基于 HTTP 的服务端单向事件推送机制，客户端通过 `EventSource` 建立连接，服务端通过 `text/event-stream` 持续返回事件数据。
> 它特别适合大模型 token 流式输出、Agent 执行过程展示、日志和进度通知。
> 和 WebSocket 相比，SSE 实现更简单、天然适合单向流式场景，但不支持双向通信。
> 在 Python 中通常通过 FastAPI 的 `StreamingResponse` 配合异步生成器实现，在 Java 中可以通过 Spring MVC 的 `SseEmitter` 或 WebFlux 的 `Flux<ServerSentEvent>` 来实现。
> 工程上还需要关注断线重连、心跳保活、代理缓冲和显式结束事件等问题。

------

## 21. 复习重点清单

面试前重点记住以下内容：

### 必会概念

- SSE 是什么
- 单向通信
- 基于 HTTP
- `text/event-stream`

### 必会对比

- SSE vs WebSocket
- SSE vs 普通 HTTP 流式响应

### 必会实现

- FastAPI `StreamingResponse`
- Spring `SseEmitter`
- WebFlux `Flux<ServerSentEvent<T>>`

### 必会场景

- LLM 流式输出
- Agent 执行步骤展示
- 日志和进度通知

### 必会工程点

- 心跳
- 断线重连
- 代理缓冲
- 显式 done 事件
- 错误事件设计

------

## 22. 最后一句话总结

> SSE 本质上就是“基于 HTTP 的服务端单向事件流推送”，它不是为了替代 WebSocket，而是为“服务端持续输出、客户端实时接收”这类场景提供一种更简单、更贴近 HTTP 的解决方案。

```
如果你需要，我下一条可以继续帮你把这份内容再整理成一份 **更适合背诵的面试八股版**，或者整理成 **Q&A 题库版 md 文档**。
```
