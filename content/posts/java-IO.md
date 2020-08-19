---
title: "二：深入分析 Java I/O 的工作机制"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Reading notes"]
date: 2020-08-17T09:19:47+01:00
draft: false
---

I/O 是任何编程语言都无法回避的问题，它是人机交互中机器获取和交换信息的主要渠道，可以说大部分 Web 应用系统的瓶颈都是 I/O 瓶颈。

本篇以 [《深入分析 Java Web 技术内幕》](https://book.douban.com/subject/25953851/) 第二章 深入分析 Java I/O 的工作机制 的内容为参考，此篇讲解 Java I/O 类库、磁盘 I/O、网络 I/O、NIO 等。

### Java I/O 类库的基本架构

#### 基于字节的 I/O 操作接口

无论是磁盘还是网络传输，最小的存储单元都是字节，而不是字符，所以 I/O 操作的都是字节而不是字符，Java 中基于字节的 I/O 操作接口输入和输出分别是 InputStream 和 OutputStream，其类层次关系如下图所示。

![](/images/InputStream.jpg)

需要明确的是，字节流从哪里读或是写入到哪里，操作数据的方式是可以组合使用的，如：

```java
OutputStream outputStream = new BufferedOutputStream(new FileOutputStream(new File("pathName")));
```

#### 基于字符的 I/O 操作接口

虽然 I/O 操作的都是字节，但我们在程序中通常都是操作字符，为了方便 JDK 也提供了一个直接写字符的 I/O 接口 Reader 和 Writer，这样我们就可以直接将字符写入到文件或者网络流去。

例如写字符的操作接口为 `void write(char[] cbuf, int off, int len)`


![](/images/Writer.jpg)

#### 字节与字符的转化接口

字符在写入文件持久化或者网络传输之前，都需要先经过编码转换，下图中 InputStreamReader 就是从字节到字符的转化桥梁，在初始化时需要指定编码字符集，否则会采用操作系统默认的字符集，很可能出现乱码问题。

![](/images/Charset.png)


### 磁盘 I/O 工作机制

#### 几种访问文件的方式
读取和写入文件 I/O 都需要 [系统调用](https://baike.baidu.com/item/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8) 才能实现，那么就肯定存在内存用户空间和内核空间的切换，对用户态和内核态不熟悉的可以先去我之前的博客 [Java：线程池原理、源码分析](https://chenghao.monster/2020/java-threadpool/) 再温习一下。

操作系统将用户内存空间和内核空间隔离开，虽然保障了内核程序运行的安全性，但使磁盘 I/O 多了一步从内核空间往用户空间复制的过程，使 I/O 成为了非常耗时的操作。为此，操作系统也是在内核空间做了缓存机制，如果用户程序访问的是缓存中的数据，就会从内核缓存中直接取出返回，以此减少 I/O 的响应时间。

![](/images/文件访问.jpg)

1. 标准访问：写入时将数据从用户空间复制到 **内核空间的缓存** 中即完成操作，写入磁盘操作由操作系统 sync 同步来完成。
2. 直接 I/O：不使用内核空间的缓存，而使用用户程序的应用缓存，实现对 **热点数据** 的管理，可以减少一次数据从内核缓存区到用户空间的复制，加速数据的访问效率。但如果应用缓存没有命中，程序会直接访问磁盘，加载会非常慢。
3. 同步访问：
4. 异步访问：

### 网络 I/O 工作机制



### NIO 的工作方式




### 适配器模式与装饰器模式

