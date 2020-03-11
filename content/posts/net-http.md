---
title: "Net：概述与应用层"
author: "Chenghao Zheng"
tags: ["Net"]
categories: ["Study notes"]
date: 2020-02-27T13:19:47+01:00
draft: false
---


本篇为 [《计算机网络自顶向下方法》](https://book.douban.com/subject/30280001/) 前两章的读书笔记，主要介绍：分层模型、HTTP 协议、非持续连接和持续连接、报文格式、cookie 与 session、Web 缓存（控制的关键字）、 FTP、DNS 流程与作用等。



# 网络协议分层模型

ISO 提出的计算机网络 OSI（Open System Interconnection Reference Model） 七层模型：

![](/images/OSI分层模型.png)

TCP / IP 分层模型：

![](/images/TCP分层模型.png)

分层体系的优点：HTTP 协议不用担心数据丢失，也 **不关注** TCP 从网络的数据丢失和乱序故障中恢复的细节。那是 TCP 以及协议栈较低层协议的工作。

# 应用层

现代网络应用有两种主流体系结构：客户 - 服务器体系结构 或 对等（P2P）体系结构。网络应用通过 socket 软件接口向网络发送报文 message 和从网络接收报文来实现在 **不同端系统上的进程间的通信**。

应用层协议定义了：

* 换的报文类型，例如请求报文和响应报文 。
* 各种报文类型的语法，如报文中的各个字段及这些字段是如何描述的 。
* 字段的语义，即这些字段中包含的信息的含义 。
* 一个进程何时以及如何发送报文，对报文进行响应的规则。  

### HTTP 协议

Web 的应用层协议是超文本传输协议 (HyperText Transfer Protocol , HTTP)   ，客户端和服务器端通过交换 HTTP 报文进行会话，HTTP 定义了这些报文的结构以及客户和服务器进行报文交换的方式。

HTTP 定义了 Web 客户向 Web 服务器请求 Web 页面的方式，以及服务器向客户传送 Web 页面的方式。HTTP 使用 TCP 作为它的支撑运输协议，TCP 为 HTTP 提供可靠数据传输服务。

#### 非持续连接和持续连接

非持续性连接（non-persistent connection）是每个请求 / 响应对是经一个单独的 TCP 连接发送，而持续连接（persistent connection）是所有的请求及其响应经相同的 TCP 连接发送。HTTP 在其默认方式下使用持续连接，HTTP 客户和服务器也能配置成使用非持续连接 。  

在非持续连接状态下， HTTP 服务器进程会在每次向客户发送响应后通知 **TCP 断开连接**，导致向客户返回一个 Web 页面需要产生多个 TCP 连接。非持续连接的 **缺点**：

* 每个 TCP 连接都需要两个往返时间（Round-Trip Time，RTT），一个 RTT 用于创建 TCP，另一个 RTT 用于请求和接受一个对象，如此每个对象都要经受两倍 RTT 的交付时延。

* 对于每个连接，在客户和服务器中都要分配 TCP 的缓冲区和保持 TCP 变量，这给 Web 服务器带来了严重的负担，因为一 台 Web 服务器可能同时服务于数以百计不同的客户的请求 。   

![](/images/非持续连接.png)

而在采用持续连接的情况下，服务器在发送响应后 **保持该 TCP 连接** 打开，使一个完整的 Web 页面请求甚至对同个客户可以用单个持续的 TCP 连接进行传输。如果一个连接经过一段时间间隔仍未被使用，HTTP 服务器就关闭该连接。

#### HTTP 报文格式

以访问 [www.baidu.com](https://www.baidu.com) 为例，HTTP 请求报文格式（每行由一个回车和换行符结束）：

![](/images/请求头.png)

当需要提交表单时，GET 方法会将请求参数放在 URL 中（见 [Web：浅析 URL](http://chenghao.monster/2020/web-url/)），POST 方法会将请求参数放在请求体之中。

服务器返回的响应头为（返回的 HTML 文件会放在响应的实体体 entity body 中）：

![](/images/响应头.png)

#### Cookie 与 Session

因为 HTTP 服务器并不保存关于客户的任何信息，所以我们说 HTTP 是一个 **无状态协议**（stateless protocol），这简化了服务器的设计，并且允许工程师去开发可以同时处理数以千计的 TCP 连接的高性能 Web 服务器。通过 Cookie 允许了站点对用户进行跟踪，识别用户并将内容与用户身份联系起来。

#### Web 缓存

### FTP：文件传输协议

### DNS：因特网的目录服务

### HTTPS 





