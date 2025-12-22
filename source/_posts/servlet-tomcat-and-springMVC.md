---
title: servlet tomcat and springMVC
date: 2025-12-11 20:37:21
tags: [Java,Springboot]
---

# 1 概述

本篇文档总结了前面关于 **Servlet、Tomcat、Spring MVC**（以及 Jetty、Spring Boot）的所有回答，目的是帮你把它们的概念和关系串成一个完整、可记忆的知识体系。

可以先用一句话概括三者关系：

> **Servlet：Java Web 的底层标准接口**
>  **Tomcat：实现 Servlet 标准的 Web 服务器 / 容器**
>  **Spring MVC：建立在 Servlet 之上的 Web MVC 框架**

------

# 2 Servlet 基础

## 2.1 什么是 Servlet

Servlet 不是一个软件，而是：

1. **一套规范 / 标准（Servlet 规范）**
    定义了：
   - `Servlet` / `HttpServlet` 接口与类
   - 生命周期方法：`init()`、`service()`、`doGet()`、`doPost()`、`destroy()`
   - `HttpServletRequest`、`HttpServletResponse` 的行为
   - Session、Cookie 等基础机制的行为规则
2. **你写的那个 Java 类**
    继承 `HttpServlet`，重写 `doGet()` / `doPost()` 来处理 HTTP 请求。

本质上：

> **Servlet 就是运行在服务器端、专门用来处理请求并返回响应的 Java 类。**

例如，一个最简单的 Servlet：

```java
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req,
                         HttpServletResponse resp) throws IOException {
        resp.setContentType("text/html;charset=UTF-8");
        resp.getWriter().write("<h1>Hello, Servlet!</h1>");
    }
}
```

但注意：**光有这个类，它不会自动“接收 HTTP 请求”**，需要有“Servlet 容器”来调用它，这就引出 Tomcat。

## 2.2 Servlet 的生命周期

典型 Servlet（继承 `HttpServlet`）有三个核心生命周期阶段：

1. **`init()`**
   - Servlet 被创建时调用一次
   - 通常用于初始化资源（读取配置、初始化对象等）
2. **`service()` / `doGet()` / `doPost()`**
   - 每一次 HTTP 请求都会调用
   - `service()` 会根据 HTTP 方法（GET/POST/PUT/DELETE…）去调用对应的 `doXxx()` 方法
   - 业务逻辑大部分写在 `doGet()`、`doPost()` 等方法里
3. **`destroy()`**
   - Web 应用卸载或容器关闭时调用一次
   - 可在此释放资源（关闭连接、线程池等）

## 2.3 Servlet 在 Web 请求中的角色

一个 HTTP 请求通过 Servlet 的典型流程：

1. 浏览器发送请求（例如 `GET /hello`）
2. 容器解析请求，找到对应的 Servlet
3. 创建 `HttpServletRequest` / `HttpServletResponse`
4. 调用 Servlet 的 `service()` / `doGet()` / `doPost()`
5. Servlet 通过 `response` 写出 HTML / JSON / 文件流…
6. 容器把响应通过网络发回客户端

Servlet 只负责**“业务逻辑 + 读写请求/响应”**，网络监听、线程管理等底层工作由容器负责。

------

# 3 Tomcat 与 Servlet 容器

## 3.1 Tomcat 是什么

**Tomcat 是一个 Web 服务器 + Servlet 容器**：

- 监听端口（默认 8080）
- 接收并解析 HTTP 请求
- 加载和管理 Web 应用（war 包 / webapps）
- 管理 Servlet 的生命周期
- 处理线程池、连接、Session 等底层细节

它的一个核心身份是：

> **Servlet 规范的实现者**

也就是说：Tomcat 实现了规范定义的接口与行为，使你写的 Servlet 能被正确加载、调用。

## 3.2 Tomcat 与 Servlet 规范的关系

可以这么理解：

- **Servlet 规范**：
  - 制定“插座标准”：针脚怎么排布、电压多少、接口形状如何
- **Tomcat**：
  - 按这个标准做出来的“插线板”：可以“插入”任意符合这个标准的“插头”

类比到 Java：

- Servlet 规范给出接口和行为规则
- 你写的 Servlet 类实现/继承这些接口
- Tomcat 负责：
  - 按 URL 映射找到你的 Servlet
  - 帮你调用 `init()`、`doGet()`、`doPost()`、`destroy()`

**同一套 Servlet 代码也可以跑在 Jetty、Undertow、GlassFish 等其他 Servlet 容器中**，不必只用 Tomcat。

## 3.3 Tomcat 中可以运行什么程序

### 3.3.1 基于 Servlet 的 Java Web 应用

Tomcat 的“主战场”就是运行各种基于 Servlet 的 Java Web 应用：

- 原生 Servlet + JSP
- Spring MVC / Spring Boot（war 部署）
- 其他框架（Struts、JSF、Shiro 等）

在 Tomcat 看来，这些都是“多个 Servlet + Filter + Listener”的组合。

### 3.3.2 静态资源

Tomcat 也可以：

- 提供静态 HTML、CSS、JS、图片
- 文件下载等

靠的是 Tomcat 内置的默认 Servlet 和静态资源处理器。
 **即便你不写任何 Servlet，Tomcat 仍然可以当一个简易 Web 服务器使用。**

### 3.3.3 其他 Java / 外部程序（技术上可以，工程上不推荐）

你可以在 Web 应用里：

- 启动后台线程、定时任务
- 使用 `Runtime.exec()` 或 `ProcessBuilder` 调用外部程序（如 Python、Shell）

但问题是：

- Tomcat 本来是 Web 容器，不是任务调度平台
- 把一堆后台任务塞进 Tomcat 进程容易导致：
  - 稳定性差，某个任务挂了拖垮整个 Web
  - 运维、排错复杂

因此：**技术上可以，工程上一般建议拆成独立服务。**

### 3.3.4 非 Java Web 应用（Python/Node 等）

Tomcat 不能直接运行 Python Flask / Node.js 应用，因为它们需要各自的运行时。

常见架构是：

- 每个后端（Python/Node/Java）有自己的进程
- 前面再加一个 Nginx 之类的反向代理，把不同路径或域名转发到不同后端

------

# 4 Spring MVC

## 4.1 Spring MVC 的定位

如果只用原生 Servlet：

- 你需要手写参数解析（`req.getParameter`）
- 手动处理 URL 到类/方法的映射
- 手动输出 JSON 或 JSP 等

这非常原始。

**Spring MVC 是一个建立在 Servlet 之上的 Web MVC 框架**，主要目标：

- 封装这些繁琐细节
- 提供：
  - 注解式控制器（`@Controller`、`@RestController`）
  - URL 映射（`@RequestMapping`、`@GetMapping` 等）
  - 自动参数绑定、数据校验
  - 视图解析、返回 JSON 等能力

## 4.2 DispatcherServlet：Spring MVC 的核心 Servlet

Spring MVC 的核心是一个特殊的 Servlet：

> **`DispatcherServlet`**

特点：

- 继承自 `HttpServlet`，本质也是一个 Servlet
- 在容器中被注册为“前端控制器”（Front Controller）
- 所有匹配的请求都先交给它处理

请求进入 `DispatcherServlet` 后，Spring 开始接管：

1. 通过 `HandlerMapping` 找到对应的控制器方法（某个 `@Controller` / `@RestController` 上的某个方法）
2. 用 `HandlerAdapter` 去调用该方法
3. 处理方法参数（路径变量、请求参数、JSON、表单…）
4. 返回值通过视图解析或消息转换器输出为 HTML / JSON 等

## 4.3 注解式编程模型（`@Controller`、`@GetMapping` 等）

在 Spring MVC 中，你写的不再是直接继承 `HttpServlet` 的类，而是：

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/user")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // 业务逻辑...
        return userService.findById(id);
    }
}
```

这里：

- `@RestController` / `@Controller`：标记该类是一个控制器，让 Spring 扫描并管理
- `@RequestMapping("/user")`：定义类级别路径前缀
- `@GetMapping("/{id}")`：映射 GET `/user/{id}` 请求到该方法
- 方法参数和返回值都由 Spring 自动进行绑定和序列化

**关键点：**这些注解不是 Tomcat 的、也不是 Servlet 的，而是 **Spring MVC 自己的东西**，依赖的是：

- Spring 容器（Bean 管理）
- `DispatcherServlet`（统一入口）
- 运行在一个 Servlet 容器里（Tomcat/Jetty 等）

------

# 5 Jetty 与其他 Servlet 容器

## 5.1 Jetty 与 Tomcat 的关系

Jetty、Tomcat、Undertow 都属于：

> **Servlet 容器（实现 Servlet 规范的服务器）**

区别主要在：

- 实现方式、性能特点
- 嵌入式使用体验
- 配置风格

但在 **Servlet 视角** 和 **Spring MVC 视角**，它们都是：

> “能接收 HTTP 请求并调用 Servlet 的容器”。

## 5.2 在 Jetty 上使用 `@Controller` / `@GetMapping` 的前提

要在 Jetty 上使用 Spring MVC 注解，前提是：

1. 引入 Spring MVC 相关依赖（例如 `spring-webmvc`）
2. 在 Jetty 中正确注册 `DispatcherServlet`
3. 启动 Spring 容器并开启组件扫描（XML 或 Java Config 都可）

这样：

- Jetty 负责把请求交给 `DispatcherServlet`
- Spring MVC 解析 `@Controller` / `@GetMapping` 等
- 最终效果与使用 Tomcat 几乎完全一样

如果你只是“裸 Jetty + 原生 Servlet”，没有集成 Spring MVC，那么：

- `@Controller`、`@GetMapping` 这些注解不会起作用，只是摆设。

------

# 6 三者关系与完整请求链路

## 6.1 分层结构（逻辑分层）

可以把浏览器到业务代码的调用链画成这样：

```text
[客户端 / 浏览器]
        │  HTTP 请求
        ▼
[Servlet 容器：Tomcat / Jetty / Undertow]
        │  找到对应 Servlet，并调用其 service()
        ▼
[Servlet 层：原生 HttpServlet 或 Spring 的 DispatcherServlet]
        │  若是 DispatcherServlet，则继续分发
        ▼
[Spring MVC 层：@Controller / @RestController / @GetMapping 方法]
        │  调用业务逻辑、访问数据库等
        ▼
[组装响应，通过 Servlet 容器发回给浏览器]
```

从“控制权”的角度看：

1. 浏览器 → 容器（Tomcat/Jetty）
2. 容器 → 某个 Servlet（可能是 DispatcherServlet）
3. DispatcherServlet → 某个 Controller 的方法
4. 方法返回结果 → DispatcherServlet → 容器 → 浏览器

## 6.2 请求处理的时序简述（以 Spring MVC 为例）

以 `GET /user/1` 为例：

1. 浏览器发起请求

2. Tomcat 监听 8080，接收请求并解析

3. Tomcat 根据 URL 映射，找到 Spring 注册的 `DispatcherServlet`

4. Tomcat 调用：

   ```java
   dispatcherServlet.service(request, response);
   ```

5. `DispatcherServlet`：

   - 通过 HandlerMapping 找到 `UserController#getUser`
   - 做参数绑定（`@PathVariable Long id`）
   - 调用 `getUser(1L)`
   - 拿到返回的 `User` 对象
   - 使用 JSON 转换器写入 `response`

6. `DispatcherServlet` 结束，控制权回到 Tomcat

7. Tomcat 把构造好的 HTTP 响应发送给浏览器

------

# 7 与 Spring Boot 的关系

Spring Boot 主要是**帮你自动把这一整套配好、打包好**：

1. 默认内嵌一个 Servlet 容器（默认 Tomcat，可换 Jetty/Undertow）

2. 自动注册 `DispatcherServlet`

3. 自动扫描 `@RestController`、`@Controller` 等

4. 提供一键启动的 `main` 方法：

   ```java
   @SpringBootApplication
   public class Application {
       public static void main(String[] args) {
           SpringApplication.run(Application.class, args);
       }
   }
   ```

底层仍然是同一条链路：

```text
浏览器
  → 内嵌 Tomcat/Jetty
  → DispatcherServlet (Servlet)
  → Spring MVC Controller 方法
```

**换容器不影响你的 `@Controller` / `@GetMapping` 代码**，只要 Spring MVC 还在。

------

# 8 常见误区与记忆方式

## 8.1 常见误区

1. **把 Servlet 当成“某个框架或服务器”**
    实际上 Servlet 是规范 + 接口 + 你写的类。
2. **认为 `@Controller` / `@GetMapping` 是 Tomcat 提供的**
    它们是 Spring MVC 的注解，只依赖：
   - Spring 容器
   - DispatcherServlet
   - 一个 Servlet 容器（Tomcat/Jetty/Undertow 等都可以）
3. **觉得“Spring MVC 取代了 Servlet”**
    不对，Spring MVC 是建立在 Servlet 之上，核心仍是一个 Servlet（DispatcherServlet）。
4. **以为只能在 Tomcat 上用 Spring MVC**
    任何支持 Servlet 规范的容器（Jetty、Undertow 等）都可以。

## 8.2 记忆口诀

你可以用这一句来记住三者关系：

> **Servlet 是地基标准，Tomcat 是这块地上的物业（容器），Spring MVC 是在地基上盖的框架大楼。**

或者更程序员一点的说法：

> Servlet：接口规范
>  Tomcat：接口的实现 + 运行平台
>  Spring MVC：在接口之上封装出来的高级库（用一个特制的 Servlet：DispatcherServlet 起家）

------

# 9 总结

- **Servlet**：
  - Java Web 的核心标准接口
  - 定义如何用 Java 类处理 HTTP 请求
  - 你的业务类可以继承 `HttpServlet` 来实现一个 Servlet
- **Tomcat**：
  - 实现 Servlet 规范的 Web 服务器 / 容器
  - 负责接收 HTTP 请求、管理 Servlet 生命周期、调用 `doGet()` / `doPost()`
  - 可以运行基于 Servlet 的各种 Java Web 应用和静态资源
- **Spring MVC**：
  - 建立在 Servlet 之上的 Web MVC 框架
  - 用 `DispatcherServlet` 作为核心 Servlet
  - 提供 `@Controller` / `@RestController` / `@GetMapping` 等注解来简化 Web 开发
- **Jetty / Undertow 等**：
  - 也是 Servlet 容器，只要集成 Spring MVC，同样可以使用注解式 Controller，与 Tomcat 无本质差别。

如果你之后想要做 PPT 或笔记，我还可以帮你把这份内容拆成“几页 PPT 结构”和“时序图 / 结构图草稿”，直接拿去用。
