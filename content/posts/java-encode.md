---
title: "深入分析 Java Web 中的中文编码问题"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Reading notes"]
date: 2020-08-29T09:19:47+01:00
draft: false
---

作为非英语国家程序员，编码问题是我们一直都绕不过去的一道坎。

本篇以 [《深入分析 Java Web 技术内幕》](https://book.douban.com/subject/25953851/) 第三章 深入分析 Java Web 中的中文编码问题 的内容为参考，讲解常见的编码格式、Java 中的编解码、Java Web 中的编解码，及常见乱码问题分析等。

计算机中存储信息的最小单元是 1 个字节，即 8 bits，所以能表示的字符范围是 0~255 个，而我们人类的语言太多，表示这些语言的符号太多，因而必须要经过一些翻译工作，才能让计算机理解我们的语言。这个翻译工作就是从 char 到 byte 的编码过程。

### 常见的编码格式

#### ASCⅡ 码

使用一个字节的低 7 位表示，范围为 0~127 共 128 个，包括了英文大小写字母，数字及常用的数学符号。0~31 是控制字符如换行、回车、空字符等，32~126 是打印字符，127 为删除。

#### IOS-8859-1

ASCⅡ 码的 128 个字符显然是不够用的，ISO 组织在 ASCⅡ 码基础上扩展出 ISO-8859-1 至 ISO-8859-15，其中 ISO-8859-1 涵盖了大多数西欧语言字符，应用最广泛。ISO-8859-1 仍然是 **单字节编码**，总共能表示 256 个字符。

#### GB 2312

《信息交换用汉字编码字符集》，双字节编码，范围为 A1~F7，包含 682 个符号和 6763 个汉字。

#### GBK

《汉字内码扩展规范》扩展了 GB2312，加入更多的汉字，编码范围是 8140~FEFE，能表示 21003 个汉字，**兼容 GB 2312**。也就是说用 GB 2312 编码的汉字可以用 GBK 来解码且不会用乱码。

#### GB 18030

《信息技术 中文编码字符集》，它可能是单字节、双字节或四字节编码，与 GB 2312 兼容，虽然是我国的国家标准，但在实际应用系统中使用得并不广泛。

#### UTF-16

双字节定长的 UniCode 统一码，世界上所有的语言都可以通过 UniCode 来编码。UTF-16 表示字符非常方便，每两个字节表示一个字符，**大大简化了字符串操作**，编码效率较高，适合在本地磁盘和内存之间使用（但不适合在网络之间传输，网络传输损坏字节流后很难恢复），所以 Java 采用 UTF-16 作为内存的字符存储格式，即一个 char 占两个字节 16 bits。

#### UTF-8

UTF-16 统一采用两个字节来表示一个字符，虽然简单方便，但很大一部分字符一个字节就足够表示了现在却要用两个字节表示，对 **存储空间和网络带宽** 都不是很友好；而 UTF-8 采用 **变长技术**，每个编码区域有不同的字码长度（1~6 个字节），相比 UTF-8 更适合网络传输，单个字节损坏也不会影响后面的其他字符，在编码效率上介于 GBK 和 UTF-8 之间。

### Java 中的编解码

通过 [深入分析 Java I/O 的工作机制](https://hoffmanzheng.github.io/2020/java-io/) 我们知道 Java 中 InputStream 可以读取输入的字节流，而 `InputStreamReader` 则在 I/O 读取字节流的过程中完成了字节到字符的转换，这个解码过程它又委托 `StreamDecoder` 去做，这个过程中需要用户指定 Charset 编码格式。

![](/images/Charset.png)

如果用户没有指定，将会使用本地环境中的默认字符集，如在中文环境中使用 GBK 编码。如果编解码都在中文环境中，通常也没有问题，但还是强烈建议不要使用操作系统的默认编码，因为这样会使应用程序的编码格式和运行环境绑定起来，在跨环境时很可能出现 **乱码问题**。

除了 I/O 过程涉及编码外，Java 也提供了在内存中字符和字节之间的相互转换接口，如 `string.getBytes(charset)` 和 `charset.encode(string);charset.decode(byteBuffer)`，可以通过 `Charset.forName("UTF-8")` 来获取指定的编码字符集，只要我们设置的编码格式统一，一般都不会出现问题。

以 `string.getBytes(charset)` 为例，Java 实现编码的流程如下：

![](/images/CharsetEncoder.png)

### 在 Java Web 中涉及的编解码

在以字节为单位的网络传输中，请求方需要先将请求内容编码成字节流才能发送，服务器也需要将收到的字节流解码才能正常地处理请求。在一个 HTTP 请求中，需要编码的地方有：URL、Cookie、POST 表单、响应体

![](/images/web中的编码.jpg)

#### URL 的编解码

当在百度中搜索 "技术内幕" 时，我们得到的 URL 是这样的：`https://www.baidu.com/s?ie=UTF-8&wd=%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95`，显然游览器对其中的中文字符进行了 UTF-8 编码，并以十六进制表示了出来。

```java
String string = "技术内幕";
byte[] utf = string.getBytes("UTF-8");
StringBuilder sb = new StringBuilder();
for (byte b : utf) {
     sb.append(String.format("%02X", b));
}
System.out.println(sb.toString());
// prints:  E68A80E69CAFE58685E5B995
```

查阅 [URL 编码规范 RFC 3986](https://datatracker.ietf.org/doc/rfc3986/) 可知，游览器编码 URL 是将 **非 ASCⅡ 字符** 按照某种编码格式编码成百分号加十六进制表示的字节 `percent-encoded octets` ，不同游览器对 PathInfo 和 QueryString 的编码是不一样的，那服务器如 Tomcat 又是如何对接收到的 URL 进行解码的呢。

对 URL 的 URI 部分的解码的过程是在 `org.apache.catalina.connector.CoyoteAdapter` 中的 `convertURI` 方法中完成的，它会使用 `<Connector URIEncoding="UTF-8" />` 中定义的编码格式进行解析，如果没有定义则将以默认编码 ISO-8859-1 解析。

对 QueryString 以及 POST 请求中的表单参数，都是通过 `org.apache.catalina.connector.Request` 中的 `request.getParameter` 获取参数值，对它们的解码是在这个方法第一次被调用时，会调用 parseParameters 这个方法。

对 QueryString 的解码，会先去获取 Connector 中的 **useBodyEncodingForURI** 参数，如果为 true，就会使用 Header 中 ContentType 定义的 Charset 进行解码，否则就使用默认的 ISO-8859-1。

可以看到，对于 URL 的编码和解码并不是我们在应用程序中能完全控制，所以我们应该尽量在 URL 中使用非 ASCⅡ 码字符，并且在服务器端设置 `<Connector URIEncoding="UTF-8" useBodyEncodingForURI="true">` ，不然就很可能会碰到乱码问题。

#### HTTP Header 的编解码

除了上述的 URL 信息，一个 HTTP 请求还可能在 Header 中传递其他参数，如 Cookie、redirectPath 等，对 Header 信息的解码也是在调用 request.getHeader 时进行的，会在 MessageBytes 中的 toString() 方法中完成 byte 到 char 的转化，使用的是 ISO-8859-1 解码格式，我们也不能设置 Header 的其他解码格式，所以 **如果在 Header 中有非 ASCⅡ 码字符，解码中肯定会有乱码**。

#### POST 表单的编解码

POST 表单提交的参数是通过 HTTP BODY 传递到服务端的，它会先在游览器上根据 ContentType 的 Charset 编码格式进行编码，到服务器端也是用 ContentType 中的字符集进行解码的，所以 POST 表单提交的参数 **一般不会出现乱码** 问题。

此外这个字符集也可以通过 request.setCharacterEncoding(charset) 来设置，但是要在第一次调用 request.getParameter 方法之前就设置，如果项目中使用了 Filter，Filter 可能会提前调用 request.getParameter 出现不可预料的乱码问题。

### 常见编解码问题分析

#### 中文变成西欧字符

例如字符串 “技术内幕” 变成了 “¼¼ÊõÄÚÄ»”，中文字符变成了奇怪的西欧字符，且是一个汉字字符变成两个乱码字符，用代码复现其编码解码过程如下：

```java
    String string = "技术内幕";
    char[] chars = string.toCharArray();
    for (char cha : chars) {
        System.out.print(Integer.toHexString(cha) + " ");
    }
    System.out.println(" ");

    byte[] gbk = string.getBytes("GBK");   // GBK 编码
    StringBuilder sb = new StringBuilder();
    for (byte b : gbk) {
        sb.append(String.format("%02X", b));
        sb.append(" ");
    }
    System.out.println(sb.toString());

    String result = new String(gbk, "ISO-8859-1");   // ISO-8859-1 解码
    System.out.println(result);

    // 输出的结果为：
    // 6280 672f 5185 5e55  
    // BC BC CA F5 C4 DA C4 BB 
    // ¼¼ÊõÄÚÄ»
```

编码与解码过程如下：

![](/images/encode.png)

#### 一个汉字变成一个问号

中文字符经过 ISO-8859-1 编解码后，所有字符都变成了问号，这是因为 ISO-8859-1 进行编解码时，对不在码值范围内的字符会统一用 3F 表示，即 ISO-8859-1 会把不认识的字符都变成问号。

![](/images/encode2.png)

中文字符经过 ISO-8859-1 编码会丢失信息，通常我们称之为 "**黑洞**"。由于现在大部分基础的 Java 框架或系统默认的字符集都是 ISO-8859-1，所以很容易出现乱码问题。

#### 一个汉字变成两个问号

有的时候一个中文字符会变成两个问号，这种情况比较复杂，可能会涉及多次编解码的过程，这时要仔细查看每一步的编解码环节，找出错误的地方。比如下图这种情况：

![](/images/encode3.png)

调试后发现，GBK 的编码委托给了 DoubleByte 类的 encodeChar 方法执行，成员变量 c2b 即是 GBK 的码表，观察发现，GBK 的编码格式是 **不支持非 ASCⅡ 码的西欧字符** 的，GBK 在对不认识的字符编码的时候会将其替换成 63 (3F) ，对应 ASCⅡ 码表中的问号。

![](/images/encode4.png)

![](/images/encode5.png)