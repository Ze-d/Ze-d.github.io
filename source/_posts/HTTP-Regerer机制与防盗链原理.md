---
title: HTTP Regerer机制与防盗链原理
date: 2026-01-06 16:09:24
tags: http
---

## 1. 核心概念定义

### 1.1 什么是盗链 (Hotlinking)？

盗链是指在自己的页面中，通过 `<img>`、`<video>` 或 `<link>` 等标签，直接引用**其他网站**服务器上的资源（图片、视频、文件）。

- **后果：** 最终用户的浏览器会向“被盗链站”发起请求。这会导致“被盗链站”消耗带宽和服务器资源，却没有任何流量收益（用户还在原网站上）。
- **防御：** 为了保护资源和节省成本，服务商（尤其是 CDN 和图床）会实施防盗链策略。

### 1.2 什么是 Referer？

`Referer`（HTTP 来源地址）是一个 HTTP 请求头（Request Header）。

- **作用：** 告诉服务器，当前这个请求是从哪个页面链接过来的。
- **趣闻：** 在 RFC 1945 标准中将 "Referrer" 误拼为 "Referer"，但为了保持向后兼容，这个拼写错误一直沿用至今。

------

## 2. 技术原理与工作流程

当浏览器加载网页中的资源（如图片）时，会自动执行以下握手流程：

### 2.1 请求携带

浏览器解析 HTML，发现 <img src="https://cdn.example.com/img.jpg">，随即发起 HTTP GET 请求。

此时，浏览器会根据当前页面的 URL，自动在请求头中添加 Referer 字段。

**示例请求头 (Request Headers):**

HTTP

```
GET /img.jpg HTTP/1.1
Host: cdn.example.com
Referer: http://100.100.100.100:8080/dashboard  <-- 你的主机B地址
User-Agent: Mozilla/5.0 ...
```

### 2.2 服务器判定 (The Check)

资源服务器（CDN/Nginx）接收到请求后，会检查 `Referer` 字段的值。判定逻辑通常如下：

1. **是否为空？** (直接在浏览器输入地址时为空)。
2. **是否在白名单内？** (例如 `*.google.com`, `localhost`, 自己的域名)。
3. **是否在黑名单内？** (已知的盗链网站)。

### 2.3 响应结果

- **通过 (Pass):** 返回 `200 OK` 和图片数据。
- **拦截 (Block):** 返回 `403 Forbidden`，或者返回一张替代的“禁止盗链”占位图。

------

## 3. 场景复盘：为何主机 A 可行，主机 B 失败？

结合您遇到的情况，技术差异分析如下表：

| **比较项**           | **主机 A (本地环境)**                              | **主机 B (Tailscale/远程环境)** |
| -------------------- | -------------------------------------------------- | ------------------------------- |
| **访问 URL**         | `http://localhost:8080`                            | `http://100.x.y.z:8080`         |
| **发送的 Referer**   | `http://localhost:8080`                            | `http://100.x.y.z:8080`         |
| **CDN 的白名单策略** | 通常**包含** `localhost` 或 `127.0.0.1` (方便调试) | **不包含** 未知的公网/内网 IP   |
| **判定结果**         | ✅ **通过** (被视为开发者调试)                      | ❌ **拦截** (被视为盗链站点)     |
| **HTTP 状态码**      | 200 OK                                             | 403 Forbidden                   |

**结论：** 问题不在于 Docker 容器或代码，而在于**发起请求的来源域名（Referer）**改变了，触发了目标服务器的防御规则。

------

## 4. 解决方案与 Referrer-Policy

要解决主机 B 无法加载图片的问题，核心思路是**控制浏览器发送 Referer 的行为**。

### 4.1 客户端策略 (Referrer-Policy)

W3C 定义了 `Referrer-Policy` 协议，允许网页开发者控制浏览器何时、发送什么样的 Referer。

最常用的解决方案：no-referrer

在 HTML 的 <head> 中添加：

HTML

```
<meta name="referrer" content="no-referrer" />
```

- **效果：** 浏览器在请求图片时，彻底**不发送** Referer 头。
- **原理：** 大多数防盗链策略为了兼容“用户直接下载图片”或“HTTPS 跳转 HTTP”的场景，会**允许 Referer 为空**的请求通过。

### 4.2 常用 Policy 模式对比

| **策略值 (Content)**              | **行为描述**                                    | **适用性**            |
| --------------------------------- | ----------------------------------------------- | --------------------- |
| `no-referrer`                     | 也就是“隐身模式”，完全不发送 Referer。          | **解决 403 的利器**   |
| `strict-origin-when-cross-origin` | (浏览器默认) 同源发送完整路径，跨域只发送域名。 | 安全性平衡            |
| `unsafe-url`                      | 无论去哪都发送完整 URL。                        | 不推荐 (隐私泄露风险) |

------

## 5. 服务端配置示例 (附录)

如果您是**资源提供方**（即您想防止别人盗用您的图片），可以在 Nginx 中按如下方式配置：

Nginx

```
location ~* \.(gif|jpg|png|jpeg)$ {
    # 定义合法的 Referer 列表
    # none: 允许 Referer 为空 (直接访问)
    # blocked: 允许 Referer 无法被防火墙识别的情况
    # *.my-site.com: 允许自己的域名
    # localhost: 允许本地调试 (这就是为什么主机 A 能访问)
    valid_referers none blocked *.my-site.com localhost 127.0.0.1;

    if ($invalid_referer) {
        return 403;
        # 或者重定向到一张“禁止盗链”的图片
        # rewrite ^/ http://my-site.com/forbidden.png image;
    }
}
```

------

## 6. 总结

1. **403 根源：** 目标图片服务器通过检查 HTTP 请求中的 `Referer` 头部来识别请求来源。
2. **环境差异：** `localhost` (主机 A) 通常在白名单中，而 `100.x.y.z` (主机 B) 是陌生 IP，因此被拦截。
3. **解决办法：** 修改网页前端代码，添加 `<meta name="referrer" content="no-referrer" />`，让浏览器在请求图片时不暴露来源地址，从而绕过检查。
