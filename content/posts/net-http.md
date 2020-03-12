---
title: "Net：概述与应用层"
author: "Chenghao Zheng"
tags: ["Net"]
categories: ["Study notes"]
date: 2020-02-27T13:19:47+01:00
draft: false
---


本篇为 [《计算机网络自顶向下方法》](https://book.douban.com/subject/30280001/) 前两章的读书笔记，主要介绍：网络分层模型、HTTP 协议、非持续连接和持续连接、报文格式、cookie 与 session、 FTP 协议、HTTPS 协议 等。



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

首次访问某网站时，返回的响应头中会包含 `set-cookie` 首部行，用户端系统的游览器会对接收到的 cookie 进行保存管理；当之后再访问该网站时，游览器会从 cookie 文件中获取对应的识别码，并放到 HTTP 请求头首部行中发送出去。

在这种方式下，该网站服务器可以跟踪用户在其站点的活动，虽然不知道具体的用户信息，但用户在什么时间访问了哪些页面都可以被记录，网站通过 cookie 来记住用户名、提供购物车服务、推荐产品、保留用户地址支付方式信息等。

![](/images/cookie.png)

与 cookie 相对的，`session` 是在服务器端记录信息确定用户身份的机制。客户端浏览器再次访问时只需要从该 session 中查找该客户的状态就可以了。每个用户访问服务器都会建立一个session，那服务器是怎么标识用户的唯一身份呢？事实上，用户与服务器建立连接的同时，服务器会自动为其分配一个 sessionId。客户端通过 cookie 将 sessionId 带到服务器。当一个用户提交了表单时，浏览器会将用户的 sessionId 自动附加在 HTTP 头信息中（这是浏览器的自动功能，用户不会察觉到），当服务器处理完这个表单后，将结果返回给 sessionId。

推荐阅读：[cookie、session和application都是些什么神？](https://mp.weixin.qq.com/s?__biz=Mzg2NzA4MTkxNQ==&mid=2247487067&idx=3&sn=4c5e4d7dbd78487b5d94737514232162&chksm=ce40458ff937cc997b713085dbb7298fc7fee1ba7b7825f06e40dd5a4b5020e7a96174d65157&scene=0&xtrack=1&key=2495f607594ce962957990176ff0ddb668a77980f3c3dec4db1de46351a0c8d3ada572bdb632e2fa802062bd8e53ec924c538ca1346257e0b89e57b1e713d519891bf408009d3e9adfc970424927a87d&ascene=14&uin=MjE3NTkwMDQyMA%3D%3D&devicetype=Windows+10&version=62080079&lang=zh_CN&exportkey=AQGUunafp%2FrGut2qObQGz7w%3D&pass_ticket=7l13YI2o1Gse5In5S%2BVZ4HZ%2BGP20HBV2if7cHqRZPNOKVsi2ybcmgKVvdMgryGy3)

### FTP：文件传输协议

HTTP 和 FTP 都是运行在 TCP 上，然而 FTP 使用了两个并行的 TCP 连接来传输文件，一个是控制连接（control connection），一个是数据连接（data connection）。

![](/images/FTP.png)

控制连接用于发送用户的标识和口令，发送改变远程目录的命令，它会贯穿整个用户会话期间（持续连接），而数据连接会在每次用户需要传输文件时被打开，传送后被关闭（即数据连接是非持续的）。  此外 HTTP 是无状态的，而 FTP 服务器会在整个会话旗舰保留用户的状态。

### HTTPS 

**HTTP** 协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了 Web 浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此，HTTP 协议不适合传输一些敏感信息，比如：信用卡号、密码等支付信息。

**HTTPS**（Hypertext Transfer Protocol Secure：超文本传输安全协议）是一种透过计算机网络进行安全通信的传输协议。HTTPS 经由 HTTP 进行通信，但利用 SSL / TLS 来加密数据包。HTTPS 开发的主要目的，是提供对网站服务器的身份认证，保护交换数据的隐私与完整性。

#### HTTP 与 HTTPS 区别

- HTTP 明文传输，数据都是未加密的，安全性较差，HTTPS（SSL+HTTP） 数据传输过程是加密的，安全性较好。
- 使用 HTTPS 协议需要到 CA（Certificate Authority，数字证书认证机构） 申请证书，一般免费证书较少，因而 **需要一定费用**。证书颁发机构如：Symantec、Comodo、GoDaddy 和 GlobalSign 等。
- HTTP 页面 **响应速度比 HTTPS 快**，主要是因为 HTTP 使用 TCP 三次握手建立连接，客户端和服务器需要交换 3 个包，而 HTTPS除了 TCP 的三个包，还要加上 SSL 握手需要的 9 个包，所以一共是 12 个包。
- HTTP 和 HTTPS 使用的是完全不同的连接方式，用的端口也不一样，前者是 80，后者是 443。
- HTTPS 其实就是建构在 SSL / TLS 之上的 HTTP 协议，所以，要比较 HTTPS 比 HTTP 要更**耗费服务器资源**。

![](/images/非对称加密.jpg)

#### HTTPS 工作原理

客户端在使用HTTPS方式与Web服务器通信时有以下几个步骤，如图所示。

1. 客户使用 HTTPS 的 URL 访问 Web 服务器，要求与 Web 服务器建立 SSL 连接。
2. Web 服务器收到客户端请求后，会将网站的证书信息（证书中包含 **公钥**）传送一份给客户端。
3. 客户端的浏览器与 Web 服务器开始协商 SSL 连接的安全等级，也就是信息加密的等级。
4. 客户端的浏览器根据双方同意的安全等级，建立会话密钥，然后利用网站的**公钥将会话密钥加密**，并传送给网站。
5. Web服务器利用自己的 **私钥解密出会话密钥**。
6. Web服务器利用会话密钥加密与客户端之间的通信。

