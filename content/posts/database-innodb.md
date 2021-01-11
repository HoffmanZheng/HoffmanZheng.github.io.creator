---
title: "Database：InnoDB 存储引擎"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2021-01-07T13:19:47+01:00
draft: false
---

在之前的博客 [Database：MySQL 数据库](https://nervousorange.github.io/2020/database-mysql/) 中笔者已经介绍了自己对于关系型数据库 MySQL 的些许认识，但终觉不够深刻，本篇将结合 [《MySQL技术内幕：InnoDB存储引擎》](https://book.douban.com/subject/24708143/) 讲解作为 MySQL 企业级数据库应用的第一存储引擎 InnoDB 的 **核心实现和工作机制**，主要内容有：缓冲池、线程工作机制、日志文件、锁、事务以及数据的备份与恢复等。

### MySQL 存储引擎

MySQL 数据库区别于其他数据库的最重要的一个特点就是其 **插件式的表存储引擎**（Pluggable Storage Engines），其提供了一系列标准的管理和服务支持，如 SQL 分析器和优化器等，这些标准与存储引擎本身无关，而存储引擎是底层物理结构的实现，每个存储引擎开发者可以按照自己的意愿来进行开发。 MySQL 数据库的体系结构如下图：

![](/images/mysql-architecture.png)

需要注意的是，存储引擎是 **基于表的**，而不是数据库。每个存储引擎都有各自的特点，开发站应该根据具体的应用选择适合的存储引擎，以下是一些存储引擎的简单介绍：

#### InnoDB

支持事务，行锁，外键，使用 MVCC 获得高并发性，实现了 SQL 的 4 种隔离级别，默认为 REPEATABLE，使用一种被称为 `next-key locking` 的策略来避免幻读（phantom）现象的产生，聚集索引，按主键顺序，从 MySQL 5.5.8 开始成为 MySQL 默认的存储引擎，众多互联网公司的成功应用已经证明了 InnoDB 存储引擎的高可用性、高性能以及高可扩展性。

#### MyISAM

不支持事务，表锁，支持全文索引，在 5.5.8 之前是默认的存储引擎，只缓存索引文件，不缓冲数据文件。

#### NDB

**集群** 存储引擎，数据全部放在内存中，主键查找速度极快，通过添加 NDB 数据存储节点（Data Node）可以线性地提高数据库性能；有一个问题值得注意，那就是 NDB 存储引擎的连接操作 JOIN 是在 MySQL 数据库层完成的，而不是在存储引擎层完成的。这意味着，复杂的连接操作需要巨大的网络开销，因此关联查询速度很慢。

#### Memory

数据存放于内存，数据库重启或者发生崩溃，表中的数据都将消失，使用了 **哈希索引**，只支持表锁，并发性能较差；MySQL 使用 Memory 存储引擎作为 **临时表** 来存放查询的中间结果集。

#### Archive

只支持 INSERT SELECT 操作，支持索引，压缩比例 1：10，非常 **适合存储归档数据**，如日志信息。使用行锁来实现高并发的插入操作，但并不是事务安全的引擎。

#### Federated

不存放数据，它只是指向一台远程 MySQL 数据库服务器上的表，类似于 SQL Server 的链接服务器和 Oracle 的透明网关，但只支持 MySQL 的数据库表。

#### Maria

由 MySQL 创始人之一的 Micheal Widenius 开发，可以看做是 MyISAM 的后续版本。特点是支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事务和非事务安全的选项，以及更好的 BLOB 字符类型的处理性能。

#### MySQL 的临时表

MySQL 数据库使用 Memory 存储引擎作为临时表来存放查询的中间结果集。如果中间结果集大于 Memory 存储引擎表的容量设置，又或者中间结果中含有 TEXT 或 BLOB 列类型字段，则 MySQL 数据库会把其转换到 MyISAM 存储引擎表而 **放到磁盘中**。之前提到过 MyISAM 不缓存数据文件，因此这时产生的临时表的性能对于查询会有损失。

### InnoDB 存储引擎

早期的 InnoDB 的版本随着 MySQL 数据库的更新而更新，从 5.1 版本开始，MySQL 允许存储引擎开发商以动态方式加载引擎，官方称 InnoDB Plugin，其各个版本的功能升级如下表：

| 版本     | 功能                                                 |
| -------- | ---------------------------------------------------- |
| 老版本 InnoDB   | 支持 ACID、行锁设计、MVCC                        |
| InnoDB 1.0.x（MySQL 5.1） | 增加了 compress 和 dynamic 页格式      |
| InnoDB 1.1.x（MySQL 5.5） | 增加了 Linux AIO、多回滚段             |
| InnoDB 1.2.x（MySQL 5.6） | 增加了全文索引支持、在线索引添加         |

下图简单展示了 InnoDB 存储引擎的体系架构，可以看到 InnoDB 主要由一个大的内存池和多个后台线程组成

![](/images/InnoDB-architecture.jpg)

#### 缓冲池

InnoDB 是 **基于磁盘** 的数据库系统（Disk-base Database），由于 CPU 速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池，通过 **内存的速度** 来弥补磁盘速度较慢对性能的影响。查询时先判断该页是否在缓冲池中，如果在则称该页在缓冲池被命中，直接读取该页，否则读取磁盘上的页；修改时先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。为了提高数据库的整体性能，页从缓冲区刷新回磁盘的操作 **并不是在每次页发生更新时触发**，而是通过一种称为 checkpoint 的机制刷新回磁盘。

可见缓冲池的大小直接影响着数据库的整体性能：

~~~mysql
show variables like 'innodb_buffer_pool_size';

Variable_name          |Value    |
-----------------------|---------|
innodb_buffer_pool_size|134217728|
~~~

具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo 页、插入缓冲、自适应哈希索引、InnoDB 存储的锁信息、数据字典信息等。

![](/images/InnoDB-buffer-pool.jpg)

* LRU

数据库中的缓冲池是通过 LRU（Lastest Recent Used，最近最少使用）算法来管理的，即最频繁使用的页在 LRU 列表的前端，而最少使用的页在 LRU 列表的尾端。当缓冲池不能存放新读取到的页时，将 **首先释放 LRU 列表中尾端的页**。

InnoDB 对传统的 LRU 算法做了一些优化，新增了 `midpoint insertion strategy` 算法，新读取到的页并不是直接放入到 LRU 列表的首部，而是放入到 LRU 列表的 midpoint 位置，midpoint 位置由参数 `innodb_old_blocks_pct` 控制，默认位置在 LRU 列表长度的 5/8 （37%）处。midpoint 之后的列表称为 old 列表，之前的列表称为 new 列表，可以简单理解为 new 列表中的页都是最为活跃的热点数据。

~~~mysql
show variables like 'innodb_old_blocks_pct'

Variable_name        |Value|
---------------------|-----|
innodb_old_blocks_pct|37   |
~~~

若是直接将读取到的页放入到 LRU 的首部，那么某些 SQL 操作（索引或者数据的扫描操作）可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。如果页被放入 LRU 列表的首部，可能将所需要的 **热点数据页** 从 LRU 列表中移除，为此 InnoDB 引入了 `innodb_old_blocks_time`，表示页读取到 mid 位置后需要等待多久才会被加入到 LRU 列表的热端。当页从 LRU 列表的 old 部分加入到 new 部分时，称此操作为 `page made young`，因设置了 innodb_old_blocks_time 导致页没有从 old 移动到 new 部分的操作称为 page_not_made_young

~~~mysql
show variables like 'innodb_old_blocks_pct'

----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 137363456; in additional pool allocated 0
Dictionary memory allocated 10088749
Buffer pool size   8191     // 缓冲池中页的数量，除了 LRU 还可能有自适应哈希索引、插入缓冲、锁信息等
Free buffers       0        // 
Database pages     7545     // LRU 列表中页的数量
Old database pages 2765
Modified db pages  0        // 脏页的数量
Pending reads 0
Pending writes: LRU 0, flush list 1, single page 0
Pages made young 1115739907, not young 0     // LRU 列表中页移动到前端的次数
0.00 youngs/s, 0.00 non-youngs/s
Pages read 592119280, created 299958, written 7230109
0.00 reads/s, 0.00 creates/s, 1.75 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000    // 缓冲池的命中率，若小于 95% 需要观察 LRU 列表是否被污染
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7545, unzip_LRU len: 0              // LRU 列表中页的数量，unzip LRU 中页的数量
I/O sum[4]:cur[7], unzip sum[0]:cur[0]
--------------
~~~

在 LRU 列表中的页被修改后，称该页为 **脏页**（dirty page），即缓冲池中的页和磁盘上的页的数据产生了不一致，脏页既存在于 LRU 列表中，来管理页的可用性，也存在于 Flush 列表中，通过 checkpoint 机制刷新回磁盘。

* Checkpoint

缓冲池协调了 CPU 速度与磁盘速度的鸿沟，而脏页的产生使得数据库需要将新版本的页从缓冲池刷新到磁盘。这里就出现了几个矛盾点：
  1. 若每次一个页发生变化就刷新到磁盘，开销会非常大。
  2. 刷新过程中发生了宕机，会导致数据丢失。
当前的事务数据库系统普遍采用了 `Write Ahead Log` 策略，即当事务提交时，写先 **重做日志** redo log，再修改页。当发生宕机导致数据丢失时，通过重做日志来完成数据的恢复，这也是事务 ACID 中 D（Durability 持久性）的要求。

即使缓冲池足够大（可以缓存数据库中所有的数据）并且重做日志可以无限增大，宕机后数据库重新应用重做日志的 **数据恢复时间** 也会非常久，因此 checkpoint 技术解决了以下三个问题：
  1. 缩短数据库的恢复时间（宕机后只需对 checkpoint 后的重做日志进行恢复，这样就大大缩短了恢复时间）
  2. 缓冲池不够用时，刷新脏页到磁盘
  3. 重做日志不可用时，刷新脏页


#### 后台线程

* Master Thread

核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、UNDO 页的回收等。

* I/O Thread

异步 I/O 线程来处理写 I/O 请求，以极大地提高数据库的性能。可以通过 `show engine innodb status` 来观察 InnoDB 中的 IO Thread，默认为 1 个 insert buffer thread，1 个 log thread，4 个 read thread 和 4 个 write thread，可以使用 `innodb_read_io_threads` 和 `innodb_write_io_threads` 参数进行设置。

~~~mysql
show engine innodb status;

--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
~~~

* Purge Thread

在事务提交后，**回收已经使用并分配的 undo 页**。在 InnoDB 1.1 版本之前，purge 操作仅在 Master Thread 中完成，1.1 版本开始 purge 操作可以独立到单独的线程中进行，1.2 版本开始支持多个 Purge Thread 来加快 undo 页的回收，以此来减轻 Master Thread 的工作，从而提高 CPU 的使用率以及提升存储引擎的性能。

~~~mysql
show variables like 'innodb_purge_threads';

Variable_name       |Value|
--------------------|-----|
innodb_purge_threads|4    |
~~~

* Page Cleaner Thread

InnoDB 1.2.x 引入的，将 **脏页刷新** 操作放入单独的线程来完成，减轻原 Master Thread 的工作及对于用户查询线程的阻塞， 进一步提高 InnoDB 存储引擎的性能。

### 日志文件

### 锁

### 事务

### 备份与恢复


