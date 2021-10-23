---
title: "深入分析 Java I/O 的工作机制"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Reading notes"]
date: 2020-08-17T09:19:47+01:00
draft: false
---

I/O 是任何编程语言都无法回避的问题，它是人机交互中机器获取和交换信息的主要渠道，可以说大部分 Web 应用系统的瓶颈都是 I/O 瓶颈。

本篇以 [《深入分析 Java Web 技术内幕》](https://book.douban.com/subject/25953851/) 第二章 深入分析 Java I/O 的工作机制 的内容为参考，讲解 Java I/O 类库、磁盘 I/O、网络 I/O、NIO 工作机制等。

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

字符在写入文件持久化或者网络传输之前，都需要先经过编码转换，下图中 InputStreamReader 就是从字节到字符的转化桥梁，在初始化时需要指定编码字符集，否则会采用操作系统默认的字符集，很可能出现乱码问题。具体的编码解码过程可以查看：[深入分析 Java Web 中的中文编码问题](https://hoffmanzheng.github.io/2020/java-encode/)

![](/images/Charset.png)


### 磁盘 I/O 工作机制

读取和写入文件 I/O 都需要 [系统调用](https://baike.baidu.com/item/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8) 才能实现，那么就肯定存在内存用户空间和内核空间的切换。

> 对用户态和内核态不熟悉的可以先去我之前的博客 [Java：线程池原理、源码分析](https://hoffmanzheng.github.io/2020/java-threadpool/) 再温习一下。

操作系统将用户内存空间和内核空间隔离开，虽然保障了内核程序运行的安全性，但使磁盘 I/O 多了一步从内核空间往用户空间复制的过程，使 I/O 成为了非常耗时的操作。为此，操作系统也是在内核空间做了缓存机制，如果用户程序访问的是缓存中的数据，就会从内核缓存中直接取出返回，以此减少 I/O 的响应时间。

![](/images/磁盘IO.png)

#### 几种访问文件的方式

1. 标准访问：写入时将数据从用户空间复制到 **内核空间的缓存** 中即完成操作，写入磁盘操作由操作系统 sync 同步来完成。
2. 直接 I/O：不使用内核空间的缓存，而使用用户程序的应用缓存，实现对 **热点数据** 的管理，可以减少一次数据从内核缓存区到用户空间的复制，加速数据的访问效率。但如果应用缓存没有命中，程序会直接访问磁盘，加载会非常慢。
3. 同步访问：只有当数据被成功写到磁盘时才返回给应用程序成功的标志，性能较差，适合于对数据安全性较高的场景。
4. 异步访问：线程在发出 I/O 请求后不会阻塞等待，而是接着处理别的事，如此可以提高应用程序的效率，但不会改变访问文件的效率。
5. 内存映射：将磁盘中的文件与操作系统内存的某一块区域关联起来，减少数据从内核空间缓存到用户空间缓存的复制操作，提高效率。

#### Java 访问磁盘文件

上文介绍了 Java 操作字节或字符的接口，但这些字节流最后写到何处，是如何持久化到物理磁盘的呢？

```java
public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);
        path = name;
        open(name);
    }
```

数据在磁盘中的最小描述是文件，文件也是操作系统和磁盘驱动器交互的最小单元，但 Java 中的 `File` 并不代表一个真实的文件对象，而是代表这个路径的 **虚拟对象**， 只有当创建 FileInputStream 实例时才会去真正检查文件存不存在（不存在的话会抛出 FileNotFoundException），创建一个 `FileDescriptor` 对象，这个对象是真正代表一个存在的文件对象的描述，通过它可以直接控制这个磁盘文件。此外，可以调用 FileDescriptor.sync() 将操作系统缓存中的数据强制刷新到物理磁盘中。

#### 磁盘 I/O 优化

我们可以在 Linux 下通过 `iostat` 来查看机器的 I/O 性能是否已经成为程序的性能瓶颈，在有 4 个 CPU 的情况下，I/O wait 参数不应该超过 25%。

此外 IOPS（每秒读写次数）也是衡量磁盘 I/O 性能的一个参数，通常与磁盘转速和数据块大小有关，可以与应用程序所需要的最低的 IOPS 对比看是否能够满足性能要求。

通常提升磁盘 I/O 性能的方法有：

* 增加缓存，减少磁盘访问次数。
* 给存放的数据设计索引，加快磁盘访问。
* 采用异步非阻塞的访问方式
* 采用 RAID 技术，实现并行读写，提升 IOPS

| 磁盘阵列 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| RAID 0   | 数据被平均写到多个磁盘阵列中，实现并行读写，性能最高，100% 空间利用率，但不提供数据冗余保护，一旦数据损坏将无法恢复 |
| RAID 1   | 将数据复制到镜像磁盘阵列中，进行备份提高数据的安全性，空间利用率 50% |
| RAID 5   | 前两种的折中，往一个磁盘中写入数据的奇偶校验信息，磁盘损坏时可以用其他数据块和对应的校验数据来恢复损坏的数据 |

### 网络 I/O 工作机制

数据从一台主机发送到网络中的另一台主机需要先确认有相互沟通的意向和能力，有关如何建立和关闭一个 TCP 连接可以参考我的 [Net：计算机网络读书笔记](https://hoffmanzheng.github.io/2020/net-http-tcp/) 中的可靠数据传输章节。

#### Socket 通信链路

主机之间的程序通信必须通过 Socket（套接字）来建立连接，套接字上联应用进程，下联网络协议栈，提供了应用层进程利用网络协议交换数据的机制。

大部分情况下我们使用的都是基于 `TCP/IP` 的流套接字，建立 TCP 连接后需要底层 IP 来寻址网络中的主机，但是一台主机上可能同时运行着多个应用程序，如何才能与指定的应用程序通讯就要通过 `端口号` 来执行，这样就可以通过一个 Socket 实例来唯一代表一个主机上的应用程序的通信链路了。

![](/images/socket.jpg)

服务端会创建一个 ServerSocket 实例，只要指定的端口号没有被占用一般都会创建成功，操作系统会为该实例创建一个底层数据结构，包含指定监听的端口号和监听地址的通配符，通常是 `*` 即监听所有地址，进入阻塞等待状态，等待客户端的请求。

当有请求到来时，会为这个连接创建一个新的 Socket 数据结构，包含请求源地址和端口，并加入到 ServerSocket 实例的未完成连接的数据结构列表中，服务端与之对应的 Socket 实例会在完成与客户端的三次握手后创建返回，并将该数据结构移到已完成列表中，所以与 ServerSocket 所关联的列表中的每个数据结构都代表一个与客户端建立的 TCP 连接。

#### 数据传输与影响因素

当连接成功建立后，服务端和客户端都会拥有一个 Socket 实例，实例通过 InputStream 和 OutputStream 来传输字节流。它们都有一个一定大小的缓存区，写入端将数据写到 OutputStream 的 SendQ 队列中，当队列满时，数据将被转移到另一端 InputStream 的 RecvQ 队列中，如果这时 RecvQ 已经满了，OutputStream 的 `write` 方法将会 **阻塞**，直到 RecvQ 队列中有足够的空间容纳 SendQ 发送的数据。

这个缓存区的大小及写入端和读取端的速度将会很大程度上影响这个连接的数据传输效率，因此 TCP 拥塞控制需要一个最优的缓存区大小，这个窗口大小（BDP，Bandwidth Delay Product）等于带宽乘以RTT（Round-Trip Time，响应时间）。此外，如果两边同时传送数据可能会产生 **死锁**，这个问题在 NIO 中得到了解决。

#### 网络 I/O 优化

1. 高并发下注意系统可用的端口数，可用端口有限的情况下会有大量请求等待建立连接，可以增大端口范围，或设置 `/proc/sys/net/ipv4/tcp_fin_timeout` 为更小的值来快速释放请求。
2. 减少网络交互的次数，在两端设置缓存，减少对数据库的访问；**合并访问请求**，比如可以将多个 JS 文件合并在一个 HTTP 链接中，每个文件用逗号隔开，然后发送给后端 Web 服务器，根据 URL 再拆分为各个文件，最后一起打包一并返回给前端游览器。、
3. 减少网络传输数据量的大小，通常的做法是将数据压缩后再传输。在设计代理程序时，尽量避免要读取整个通信数据才能取得需要的信息。
4. 尽量减少编码，从字符到字节需要编码，但这个编码过程是比较耗时，减少转化过程可以提高 I/O 效率。
5. 根据应用场景选择合适的交互方式，网络长连接同时传输数据不是很多的情况下可以使用 **同步非阻塞** 的方式，用额外的 CPU 消耗来提升 I/O 性能；分布式数据库写备份记录通常采用异步阻塞的方式，集群之间的消息同步则可以使用异步非阻塞的方式。

### NIO 的工作方式

#### BIO 与 NIO

采用 BIO（Blocking IO）线程在数据写入 OutputStream 或者从 InputStream 读取时都有可能会阻塞，一旦阻塞线程将会 **失去 CPU 的使用权**，这在大规模访问和高性能要求下是不能被接受的。

虽然我们可以采用一个客户端对应一个处理线程的方式，让出现阻塞的线程不影响其他线程的工作，加以使用线程池来减少线程创建和回收的成本，但仍有一些使用场景无法适用，比如服务器需要同时保持大量的 HTTP 长连接，或是想给某些客户端更高的优先级。为此我们需要一种新的 I/O 操作方式。

传统多线程 + BIO 存在的问题：

1. 线程的创建和销毁成本很高，在 Linux 这样的操作系统中，线程本质上就是一个进程。创建和销毁都是重量级的系统函数。
2. 线程本身占用较大内存，像Java的线程栈，一般至少分配 512K～1M 的空间，如果系统中的线程数过千，恐怕整个 JVM 的内存都会被吃掉一半。
3. 线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统 load 偏高、CPU sy 使用率特别高（超过20%以上)，导致系统几乎陷入不可用的状态。
4. 容易造成锯齿状的系统负载。因为系统负载是用活动线程数或CPU核心数，一旦线程数量高但外部网络环境不是很稳定，就很容易造成大量请求的结果同时返回，激活大量阻塞线程从而使系统负载压力过大。
5. 当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。随着移动端应用的兴起和各种网络游戏的盛行，百万级长连接日趋普遍，此时，必然需要一种更高效的 I/O 处理模型。

![](/images/io-model.jpg)

Java NIO 实际上就是 **多路复用 I/O**，会有一个线程不断去轮询多个 socket 的状态（selector.select()），如果没有事件就一直阻塞在哪里，通过一个线程就可以管理多个 socket，只有当 socket 真正有读写事件发生时才会占用资源来进行实际的读写操作。

NIO 同步非阻塞特性：

1. Selector 用于监听多个通道的事件，比如连接打开，数据到达。因此单个线程可以监听多个数据通道。NIO 将数据读到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动数据，增加了处理过程中灵活性。当没有数据可以读取时，线程不是保持阻塞，而是可以继续做其他的事情直至数据变的可以读取。
2. NIO 在 socket 读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的 IO 操作是同步阻塞的（消耗CPU但性能非常高）。
3. BIO 模型的 socket.read() 和 socket.write() 函数 在等待数据的时候无法返回，除了多开线程没有好的可以利用 CPU 的方法；NIO 的读写函数可以 **立刻返回**，如果一个连接不能读写 socket.read() 或者 socket.write() 会立刻返回 0，这就给了我们不另开线程利用 CPU 的最好机会，我们可以把这个事件记录下来（在 Selector 上注册标记位），然后切换到其他就绪的连接（channel）继续进行读写。
4. select 是阻塞的，不用担心在函数中 CPU 空转。另外多路复用 IO 为何比非阻塞 IO 模型的 **效率高** 是因为在非阻塞 IO 中，不断询问 socket 状态是通过用户线程去进行的，而在多路复用 IO 中，轮询每个 socket 状态是在 **内核** 中进行的，这个效率比用户线程要高的多。

JDK 1.4 引入的 `java.nio` 包采用了新的非阻塞的 I/O 操作方式，实际上 `java.io` 也已经被 NIO 重新实现过，即使我们不显式地使用 NIO 编程，也能从中受益。

| BIO                       | NIO                           |
| ------------------------- | ----------------------------- |
| 面向流（Stream Oriented） | 面向缓冲去（Buffer Oriented） |
| 阻塞 IO                   | 非阻塞 IO                     |
| -                         | 选择器（Selector）            |

BIO 是面向流，一次一个字节地处理数据；而 NIO 则面向块（缓冲区），以块的形式处理数据。NIO 主要由三个核心部分组成：Buffer 缓冲区、Channel 管道、Selector 选择器。

#### Buffer 的工作方式

Buffer 缓冲区作为 NIO 中直接与数据交互的部分，底层是一个数组结构，它通过几个成员变量来记录当前缓存的数据状态：

| 索引     | 说明                                 |
| -------- | ------------------------------------ |
| capacity | 缓冲区数组总长度                     |
| position | 下一个要操作的数据元素的位置         |
| limit    | 缓冲区数组不可操作的下一个元素的位置 |
| mark     | 备忘位置，与 reset() 联合使用   |

ByteBuffer 常用 API 如下：

```java
ByteBuffer byteBuffer = ByteBuffer.allocate(10);
System.out.println("初始化一个长度为 10 的 Buffer: " + byteBuffer.toString());

byteBuffer.put(new byte[]{2, 3, 3});
System.out.println("往缓冲区写入三个字节: " + byteBuffer.toString());

byteBuffer.flip();
System.out.println("切换到读模式: " + byteBuffer.toString());

byte[] bytes = new byte[3];
System.out.println("读取前结果集数据: " + Arrays.toString(bytes));
byteBuffer.get(bytes, 0, 3);
// 读取的时候，length 最大可为 limit，不然会抛出 java.nio.BufferUnderflowException
System.out.println(byteBuffer.toString());
System.out.println("读取后结果集数据: " + Arrays.toString(bytes));
```

输出结果如下：

```shell
初始化一个长度为 10 的 Buffer: java.nio.HeapByteBuffer[pos=0 lim=10 cap=10]
往缓冲区写入三个字节: java.nio.HeapByteBuffer[pos=3 lim=10 cap=10]
切换到读模式: java.nio.HeapByteBuffer[pos=0 lim=3 cap=10]
读取前结果集数据: [0, 0, 0]
java.nio.HeapByteBuffer[pos=3 lim=3 cap=10]
读取后结果集数据: [2, 3, 3]
```

在写模式下 limit 等于 capacity，表示最后能够写多少个字节的数据，使用 `flip()` 切换到读模式后， limit 代表最多能够读到多少数据，所以 flip 之后 limit 会被设置成写模式下的 position 值。如果读取超出 limit 的数据则会抛出 `java.nio.BufferUnderflowException`。

```java
public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

当想进行下一次写入时，调用 `clear()` 再次切换成写模式，然后重置 position、limit、mark 值，这样不会擦除缓冲数组中的数据，而是将数据 “遗忘”，准备好再次写入。

如果 Buffer 中仍有未读的数据，且后续还需要这些数据，但是此时想要先先写些数据，那么使用 `compact()` 方法，compact 将所有 **未读的数据拷贝到 Buffer 起始处**，然后将 position设置到最后一个未读元素后面，limit 设置为 capacity，之后就可以写数据了，不会覆盖未读的数据。

```java
public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

public ByteBuffer compact() {
        System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
        position(remaining());
        limit(capacity());
        discardMark();
        return this;
    }
```

mark() 与 reset() 组合使用，mark 用来标记当前下次操作的 position 位置，reset 会将 mark 值重新赋给 position。

```java
public final Buffer mark() {
        mark = position;
        return this;
    }

// Resets this buffer's position to the previously-marked position.
public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
```

从操作系统缓冲区复制数据到用户缓冲区比较耗性能，Buffer 提供了以 **内存映射** 的访问方式，即 `ByteBuffer.allocateDirect(int capacity)`，这个方法返回的 DirectByteBuffer 就是与底层空间关联的缓冲区，通过 Native 代码操作非 JVM 堆的内存空间。每次创建和回收的内存开销都比较大，适合数据量比较大、生命周期比较长的情况。

#### NIO 的工作机制

NIO 作为非阻塞的 I/O 方式其实是在网络层次中理解的，对于 FileChannel 来说一样是阻塞的。NIO 采用了 **多路复用的 I/O 模型**，一个进程可以同时等待多个文件描述符，而这些文件描述符其中的任意一个进入读就绪状态，select 函数就可以返回。

![](/images/nio.png)

Selector 就是一个 I/O 调度系统，它负责监控每个 Channel 的当前运行状态（需提前注册），当 Channel 中有数据时，就把数据分配到对应的 Buffer 中，我们可以控制 Buffer 的容量及扩容机制，典型的 NIO 代码如下：

```java
public void select() throws IOException {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    Selector selector = Selector.open();
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
    serverSocketChannel.configureBlocking(false);  // 非阻塞模式
    serverSocketChannel.socket().bind(new InetSocketAddress(8080));
    // 把通信信道注册到选择器上
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    while (true) {
        // 检查注册在这个选择器上的所有通信信道是否有事件发生
        Set selectedKeys = selector.selectedKeys();  // 获取所有的 key
        Iterator iterator = selectedKeys.iterator();
        while (iterator.hasNext()) {
            SelectionKey selectionKey = (SelectionKey) iterator.next();
            // 监听客户端的连接请求
            if ((selectionKey.readyOps() & SelectionKey.OP_ACCEPT) 
                == SelectionKey.OP_ACCEPT) {
                // 获取通信通道对象
                ServerSocketChannel ssc = (ServerSocketChannel) selectionKey.channel();  
                SocketChannel sc = ssc.accept();  // 接收请求
                sc.configureBlocking(false);
                // 设置该 channel 为读就绪状态
                sc.register(selector, SelectionKey.OP_READ);
                iterator.remove();
                // 处理请求：从读就绪状态的 channel 中读取数据写入 buffer
            } else if ((selectionKey.readyOps() & SelectionKey.OP_READ) 
                       == SelectionKey.OP_READ) {
                SocketChannel sc = (SocketChannel) selectionKey.channel();
                while (true) {
                    buffer.clear();
                    int n = sc.read(buffer); 
                    if (n <= 0) {
                        break;
                    }
                    buffer.flip();
                }
                iterator.remove();
            }
        }
    }
}
```

虽然上述代码把监听请求和处理请求的事件放在同一个线程中，但通常 Web 服务器像 Tomcat 和 Jetty 都是它们放在 **两个线程** 中，一个线程专门以阻塞的方式监听客户端的连接请求，另一个线程真正采用 NIO 的方式负责处理请求，因此每个连接的数据交互都不是阻塞方式。

