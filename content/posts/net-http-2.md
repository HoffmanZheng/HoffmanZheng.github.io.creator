---
title: "HTTP/2：漫谈 RFC 7540"
author: "Chenghao Zheng"
tags: ["Net"]
categories: ["Reading notes"]
date: 2022-11-09T13:19:47+01:00
draft: false
---

HTTP/2 在引入头部字段压缩和允许单连接并发交换后，使更加高效的网络资源使用和更低的接收延迟成为可能。同时也引入了从服务端到客户端的主动推送。本篇结合 [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540) 谈谈 HTTP version 2。

### 前言

超文本传输协议是一个大获成功的协议，尽管如此，HTTP/1.1 使用的基础传输却有着会对当下的 **应用性能** 造成不利影响的特性。具体地讲，HTTP/1.0 只允许在单个 TCP 连接中处理一个请求，在 HTTP/1.1 新增请求管道后，也只实现了部分的并发请求，并仍遭受着头部阻塞。因此，HTTP/1.0 和 HTTP/1.1 的客户端为了实现并发并降低延迟，需要使用多个与服务端的连接来发送多个请求。此外，HTTP 头部字段经常是重复且冗余的，导致了不必要的网络流量，且拥塞窗口会被快速填充。这将在单个 TCP 连接处理多个请求时导致过度的延迟。

HTTP/2 通过在基础连接中定义一个更优的 HTTP 语义映射来处理这些问题。具体地，它允许在同个连接中多个请求和响应消息的交织，并对 HTTP 头部字段采用了高效的编码。它还支持请求的优先级，让一些更重要的请求能够更快地完成，以此来提升性能。生成的协议对网络更加友好，因为相比 HTTP/1.x 只需使用更少的连接。这也意味着与其他流和长连接之间更少的竞争，反过来能更好地利用可用的网络容量。

### HTTP 2 协议概览

HTTP/2 给 HTTP 语义提供了一个改良的传输，它支持 HTTP/1.1 的所有核心特征，但旨在某些方面变得更加高效。HTTP/2 的基础协议单元是一个框架 `frame`，每个框架类型服务一个不同的目的。例如，`HEADERS` 和 `DATA` 框架组成了 HTTP 请求和响应的基础，其他框架类型如 `SETTINGS`、`WINDOW_UPDATE` 和 `PUSH_PROMISE` 被用来支持 HTTP/2 的其他特性。

请求的多路复用通过让每个 HTTP 请求/响应与它自己的流交换来实现。流是 **各自独立** 的，因此一个被阻塞的请求或响应不会阻止其他流的进程。流的控制和优先级使使用多路的流成为可能。流的控制保证了被接收者使用的数据被传输了。优先级保证了有限的资源能被首先引导至更重要的流上。

HTTP/2 增加了一个新的交互模式，借此一个服务可以向客户端推送响应。服务推送允许一个服务在权衡网络使用和潜在的延迟收益后，向客户端任意的发送（预计它将会需要的）数据。服务端通过合成一个发送 `PUSH_PROMISE` 框架的请求来完成这个。随后服务端就能在一个独立的流中给这个合成的请求发送一个响应。因为在连接中使用的 HTTP 头部字段可能包含了大量多余的数据，包含它们的框架可以被压缩。这对通常情况下的请求大小有着特殊的优势，且允许许多请求被压缩在一个包里。

### HTTP 2 连接

一个 HTTP/2 连接是一个运行在 TCP 连接之上的应用层协议。客户端是 TCP 连接的初始化器。HTTP/2 使用与 HTTP/1.1 相同的 http 和 https URI 框架。HTTP/2 共享着相同的默认端口，http URIs 80，https URIs 443。因此，对于目标资源比如 “http://example.org/foo” 或者 “https://example.com/bar” 的 URLs 的请求实现被要求首先判断（客户端希望立即与之建立连接的）上游服务是否支持 HTTP/2。

客户端可以在没有前置知识的情况下通过在 HTTP/1.1 请求中包含一个带有 h2c 令牌的升级头字段和一个 `HTTP2-Settings` 头字段来支持 HTTP/2。（基于 TLS 的 HTTP/2 使用 `h2` 作为协议标识符）

```http
GET / HTTP/1.1
Host: Server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```

一个不支持 HTTP/2 的服务端能够在缺少升级头字段的情况下响应请求。而支持 HTTP/2 的服务会给出一个 101 交换协议的响应。在一个结束 101 响应的空行后，服务可以开始发送 HTTP/2 框架。

```javascript
// 不支持 HTTP/2 的服务
HTTP/1.1 200 OK
Content-Length: 243
Content-Type: text/html
...

// 支持 HTTP/2 的服务
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c

[ HTTP/2 connection ... ]
```

这个提前发送来升级的请求会被认定为流 1，在 HTTP/2 连接开始后，流 1 会被用来作响应。因为升级只会应用在接下来的连接，客户端在发送 HTTP2-Settings 头字段时必须也发送 HTTP2-Settings 作为连接头字段的连接选项来防止请求被转发。

### HTTP 框架

### 流与多路复用

### 框架定义

### 额外的 HTTP 要求与考虑

### 安全性考虑
