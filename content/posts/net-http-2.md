---
title: "HTTP/2：漫谈 RFC 7540"
author: "Chenghao Zheng"
tags: ["Net"]
categories: ["Reading notes"]
date: 2022-11-08T13:19:47+01:00
draft: false
---

HTTP/2 在引入头部字段压缩和允许单连接并发交换后，使更加高效的网络资源使用和更低的接受延迟成为可能。同时也引入了从服务端到客户端的主动推送。本篇结合 [RFC 7540](https://www.rfc-editor.org/rfc/rfc7540) 谈谈 HTTP/2




