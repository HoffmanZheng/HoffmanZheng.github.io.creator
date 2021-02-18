---
title: "Database：InnoDB 存储引擎"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2021-01-07T13:19:47+01:00
draft: false
---

在之前的博客 [Database：MySQL 数据库](https://nervousorange.github.io/2020/database-mysql/) 中笔者已经介绍了自己对于关系型数据库 MySQL 的些许认识，但终觉不够深刻，本篇将结合 [《MySQL技术内幕：InnoDB存储引擎》](https://book.douban.com/subject/24708143/) 讲解作为 MySQL 企业级数据库应用的第一存储引擎 InnoDB 的 **核心实现和工作机制**，主要内容有：缓冲池、后台线程工作机制、日志文件、锁、事务以及数据的备份与恢复等。

### MySQL 存储引擎

MySQL 数据库区别于其他数据库的最重要的一个特点就是其 **插件式的表存储引擎**（Pluggable Storage Engines），其提供了一系列标准的管理和服务支持，如 SQL 分析器和优化器等，这些标准与存储引擎本身无关，而存储引擎是底层物理结构的实现，每个存储引擎开发者可以按照自己的意愿来进行开发。 MySQL 数据库的体系结构如下图：

![](/images/mysql-architecture.png)

需要注意的是，存储引擎是 **基于表的**，而不是数据库。每个存储引擎都有各自的特点，开发者应该根据具体的应用选择适合的存储引擎，以下是一些存储引擎的简单介绍：

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

MySQL 数据库使用 Memory 存储引擎作为临时表来存放 **查询的中间结果集**。如果中间结果集大于 Memory 存储引擎表的容量设置，又或者中间结果中含有 TEXT 或 BLOB 列类型字段，则 MySQL 数据库会把其转换到 MyISAM 存储引擎表而 **放到磁盘中**。之前提到过 MyISAM 不缓存数据文件，因此这时产生的临时表的性能对于查询会有损失。

### InnoDB 存储引擎

早期的 InnoDB 的版本随着 MySQL 数据库的更新而更新，从 5.1 版本开始，MySQL 允许存储引擎开发商以动态方式加载引擎，官方称 **InnoDB Plugin**，其各个版本的功能升级如下表：

| 版本     | 功能                                                 |
| -------- | ---------------------------------------------------- |
| 老版本 InnoDB   | 支持 ACID、行锁设计、MVCC                        |
| InnoDB 1.0.x（MySQL 5.1） | 增加了 compress 和 dynamic 页格式      |
| InnoDB 1.1.x（MySQL 5.5） | 增加了 Linux AIO、多回滚段             |
| InnoDB 1.2.x（MySQL 5.6） | 增加了全文索引支持、在线索引添加         |

下图简单展示了 InnoDB 存储引擎的体系架构，可以看到 InnoDB 主要由一个大的内存池和多个后台线程组成

![](/images/InnoDB-architecture.jpg)

#### 缓冲池

InnoDB 是 **基于磁盘** 的数据库系统（Disk-base Database），由于 CPU 速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池，通过 **内存的速度** 来弥补磁盘速度较慢对性能的影响。查询时先判断该页是否在缓冲池中，如果在则称该页在缓冲池被命中，直接读取该页，否则读取磁盘上的页；修改时先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。为了提高数据库的整体性能，页从缓冲区刷新回磁盘的操作 **并不是在每次页发生更新时触发**，而是通过一种称为 `Checkpoint` 的机制刷新回磁盘。

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

~~~shell
show engine innodb status;

----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 137363456; in additional pool allocated 0
Dictionary memory allocated 10088749
Buffer pool size   8191     // 缓冲池中页的数量，除了 LRU 还可能有自适应哈希索引、插入缓冲、锁信息等
Free buffers       0        
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
  3. 重做日志不可用时，刷新脏页（重做日志缓冲 redo log buffer `innodb_log_buffer_size` 一般不需要设置得很大（默认 8 MB），因为每一秒钟都会将重做日志缓冲刷新到日志文件；磁盘中的重做日志文件设计成循环使用的，不再需要的重做日志可以被重复使用）

~~~shell
show engine innodb status;

---
LOG
---
Log sequence number 49109625455      // 重做日志当前的 LSN
Log flushed up to   49109625455      // 刷新到重做日志文件的 LSN
Last checkpoint at  49109625197      // 已经刷新回磁盘最新页的 LSN
0 pending log writes, 0 pending chkp writes
6626161 log i/o's done, 0.08 log i/o's/second
----------------------
~~~

InnoDB 通过 LSN Log Sequence Number 来标记版本，每个页有 LSN，重做日志中也有 LSN，Checkpoint 也有 LSN。InnoDB 有两种 Checkpoint 机制，在数据库关闭时将所有脏页刷新回磁盘的 `Sharp Checkpoint` （运行时使用影响数据库的可用性）和引擎内部使用的 `Fuzzy Checkpoint`，在以下几种情况下可能会发生 Fuzzy Checkpoint：
  1. Master Thread Checkpoint：每秒/每十秒以异步方式刷新一定比例的脏页，不阻塞用户查询线程
  2. FLUSH_LRU_LIST Checkpoint：检查 LRU 列表是否有 100 个可用空闲页（阻塞用户查询线程），否就刷新 LRU 尾端的脏页，从 MySQL 5.6 开始，这个检查被单独放在了 Page Cleaner 线程中，可以通过 `innodb_lru_scan_depth` 来控制 LRU 列表中可用页的数量
  3. Async/Sync Flush Checkpoint：重做日志不可用时，强制刷新一些脏页回磁盘，保证重做日志的循环使用的可用性。从 MySQL 5.6 开始这个检查也被放入了 Page Cleaner 线程中来避免对用户查询线程的阻塞。
  4. Dirty Page too much Checkpoint：为保证缓冲池中有足够可用的页，当脏页数量比例超过 `innodb_max_dirty_pages-pct` （MySQL 5.5 及之后版本，默认为 75）时进行刷新操作。

#### 后台线程

InnoDB 是多线程的模型，后台有多个不同的后台线程，负责处理不同的任务：

* Master Thread

核心的后台线程，完成 InnoDB 主要工作，负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、UNDO 页的回收等。

**InnoDB 1.0.x（MySQL 5.1）** 版本之前的 Master Thread

Master Thread 内部由多个循环组合，大多数的操作在主循环中，其中有两大部分的操作 —— 每秒钟的操作和每十秒的操作，其行为伪代码如下：

~~~java
void master_thread() {
  goto loop;
}
loop:
for (int i = 0; i < 10; i++) {               // ---------- 每秒钟的操作 -------------
  thread_sleep(1)  // sleep 1 second
  do log buffer flush to disk                // 刷新日志缓冲到磁盘，即使这个事务还没提交
  if (last_one_second_ios < 5)              
    do merge at most 5 insert buffer         // I/O 压力很小时（小于每秒 5 次），合并至多 5 个插入缓冲
  if (buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct)
    do buffer pool flush 100 dirty page      // 当前缓冲池脏页比例过大时，刷新 100 个脏页到磁盘
  if (no user activity) 
    goto backgroud loop                      // 如果当前没有用户活动，则切换到 backgroud loop
}
                                             // ---------- 每十秒钟的操作 -------------
if (last_ten_second_ios < 200) 
  do buffer pool flush 100 dirty page        // I/O 压力较小时（过去 10 秒小于 200 次），刷新 100 个脏页到磁盘
do merge at most 5 insert buffer             // 合并至多 5 个插入缓冲
do log buffer flush to disk                  // 将日志缓冲刷新到磁盘
do full purge                                // 删除无用的 Undo 页
if (buf_get_modified_ratio_pct > 70%)
  do buffer pool flush 100 dirty page        // 如果缓冲池脏页比例过大，刷新 100 个脏页，否则刷新 10 个脏页
else
  buffer pool flush 10 dirty page
goto loop
backgroud loop:
  do something                               // 数据库空闲或者关闭时会被切换到 backgroud loop
goto loop:
~~~

**InnoDB 1.2.x（MySQL 5.6）** 版本之前的 Master Thread

通过看上面 Master Thread 在 InnoDB 1.0.x 版本之前的伪代码，可以发现由于 `hard coding` 导致存储引擎在 1 秒内至多处理 100 个页的写入和 20 个插入缓冲的合并，这在磁盘技术飞速发展的今天（特别是固态磁盘 SSD 出现后）成为了 I/O 性能的瓶颈。

因此 InnoDB Plugin（从 InnoDB）提供一些参数来提高 I/O 性能：

| 参数                        | 功能                                                 |
| --------------------------- | ---------------------------------------------------- |
| innodb_io_capacity          | 新增，默认 200，表示磁盘 I/O 吞吐量，是刷新脏页数量，* 5% 为合并插入缓冲数量        |
| innodb_max_dirty_pages_pct  | 修改默认值从 90 至 75，降低每秒任务中判断缓冲池中脏页是否过多的比例                 |
| innodb_adaptive_flushing    | 新增，通过判断产生重做日志的速度决定最合适的刷新脏页的数量                         |
| innodb_purge_batch_size     | 新增，用于控制每次 full purge 时回收 Undo 页的数量                              |

可以通过 SHOW ENGINE INNODB STATUS 查看当前 Master Thread 的状态信息，如下所示：

~~~shell
show engine innodb status;

=====================================
210111 18:33:26 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 12 seconds     // 过去 12 秒内引擎的状态统计
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 6572378 1_second, 6571948 sleeps, 589956 10_second, 678071 background, 677167 flush
                       // 主线程 每秒操作次数，每秒挂起操作次数，每十秒操作次数，backgroud loop 次数，flush loop 次数  --- 说明服务器压力较小
                       // 如果服务器压力较大，可以看到每秒挂起的次数小于每秒操作的次数；
                       // 因为 InnoDB 做了一些优化，当压力大时不总是等待 1 秒，所以可以通过两者的差值反应当前数据库的负载压力
srv_master_thread log flush and writes: 6663025
----------
~~~

**InnoDB 1.2.x（MySQL 5.6）** 版本的 Master Thread

InnoDB 1.2.x 版本再次对 Master Thread 做了优化，伪代码如下：

~~~shell
if InnoDB is idle
  srv_master_do_idle_tasks();     // 之前版本每 10 秒的操作
else 
  srv_master_do_active_tasks()；  // 之前版本每秒钟的操作
~~~

此外，这个版本将刷新脏页的操作从 Master Thread 线程分离到一个单独的 Page Cleaner Thread 线程，从而减轻了 Master Thread 的工作，同时 **避免** 了 FLUSH_LRU_LIST Checkpoint 和 Async/Sync Flush Checkpoint **对用户查询线程的阻塞**，进一步提高了系统的并发性。

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

### 文件

#### 错误日志与慢查询日志

* 错误日志

错误日志文件对 MySQL 的启动、运行、关闭过程进行的记录。DBA 在遇到问题时应该首先查看该文件以便定位问题，该文件不仅 **记录了所有的错误信息**，也记录一些警告信息或者正确的信息，用户可以通过参数 `log_error` 来定位该文件：

~~~shell
show variables like 'log_error';

Variable_name|Value                    |
-------------|-------------------------|
log_error    |/var/log/mysql/mysqld.log|     // 文件名默认为服务器的主机名
~~~

有时用户可以直接在错误日志文件中得到 **优化** 的帮助，因为有些警告（warning）很好地说明了问题所在：

~~~shell
InnoDB：If you are using big BLOB or TEXT rows, you must set the
InnoDB：combined size of log files at least 10 times bigger than the largest such row.
~~~

* 慢查询日志

通过错误日志可以得到一些关于数据库优化的信息，而慢查询日志（slow log）可帮助 DBA 定位可能存在问题的 SQL 语句，从而进行 **SQL 语句层面的优化**。MySQL 数据库默认并不启动慢查询日志，相关的参数如下表：

| 参数                           | 功能                                                      |
| ----------------------------- | --------------------------------------------------------- |
| log_slow_queries              | 默认 OFF，设置为 ON 开启慢查询日志                            |
| long_query_time               | 默认为 10(秒)，慢查询日志会记录运行时间超过该值的所有 SQL 语句   |
| log_queries_not_using_indexes | 默认 OFF，开启后记录所有 **没有使用索引** 的查询语句            |
| log_throttle_queries_not_using_indexes   | MySQL 5.6.5 新增，默认 0(没有限制)，表示每分钟允许记录到 slow log 的未使用索引的 SQL 语句次数   |

DBA 可以通过慢查询日志来找出有问题的 SQL 语句，对其进行优化，然而慢查询日志随着服务器运行时间的增加越来越多。DBA 可以通过 MySQL 提供的命令 `mysqldumpslow` 来分析慢查询日志文件。MySQL 5.1 开始可以将慢查询日志记录放到一张表中，这使得用户的查询更加方便和直观。

~~~shell
show create table mysql.slow_log;       // 慢查询表在 mysql 架构下名为 slow_log

CREATE TABLE `slow_log` (
  `start_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `user_host` mediumtext NOT NULL,
  `query_time` time NOT NULL,
  `lock_time` time NOT NULL,
  `rows_sent` int(11) NOT NULL,
  `rows_examined` int(11) NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int(11) NOT NULL,
  `insert_id` int(11) NOT NULL,
  `server_id` int(10) unsigned NOT NULL,
  `sql_text` mediumtext NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'       // 默认为 CSV 引擎，可改为 MyISAM，添加索引以提高查询效率

show variables like 'log_output';       // 指定了慢查询输出的格式，设为 TABLE 改用 slow_log 表

Variable_name|Value|     
-------------|-----|
log_output   |FILE |
~~~

#### 二进制日志

二进制日志（binary log）记录了对 MySQL 数据库执行更改的所有操作（包括没有导致数据库发生变化的操作 `0 rows affected`），还记录了执行数据库更改操作的时间等其他额外信息，其主要有以下三种作用：

* 恢复（recovery）：某些数据的恢复需要二进制日志，例如在一个数据库全备文件恢复后，可以进行 `point-in-time` 的恢复

* 复制（replication）：通过复制和执行二进制日志实现主从（slave）数据库的实时同步

* 审计（audit）：通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击

通过在配置文件中配置参数 log-bin [=name] 启动二进制日志，开启后确实会影响性能，但性能的损失十分有限，考虑到可以使用复制和 point-in-time 的恢复，这些性能损失绝对是可以且应该被接收的。

~~~shell
shbu6dbl1:/etc # mysql --help | grep my.cnf              // 查看当 MySQL 数据库实例启动时，会在哪些位置查找配置文件
order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf

datadir = /data/mysql                                    // 二进制日志所在目录
log-bin=master-bin                                       // 开启二进制日志
log-bin-index=master-bin.index

-rw-rw---- 1 mysql mysql       1393 Jun 23  2020 master-bin.000001       // 后缀名为二进制日志的序列号
-rw-rw---- 1 mysql mysql    1030508 Jun 23  2020 master-bin.000002
-rw-rw---- 1 mysql mysql        126 Jun 23  2020 master-bin.000003
-rw-rw---- 1 mysql mysql      16485 Jun 23  2020 master-bin.000004
-rw-rw---- 1 mysql mysql        126 Jun 23  2020 master-bin.000005
-rw-rw---- 1 mysql mysql  249467313 Jul 28 14:28 master-bin.000006
-rw-rw---- 1 mysql mysql      10034 Jul 28 17:02 master-bin.000007
-rw-rw---- 1 mysql mysql        126 Jul 28 17:03 master-bin.000008
-rw-rw---- 1 mysql mysql 1098762106 Sep  9 09:47 master-bin.000009
-rw-rw---- 1 mysql mysql  742541453 Jan 15 16:26 master-bin.000010
-rw-rw---- 1 mysql mysql        200 Sep  9 09:47 master-bin.index        // 二进制的索引文件
~~~

以下配置文件的参数影响着二进制日志记录的信息和行为：

* `max_binlog_size`：指定单个二进制日志文件的最大值（从 MySQL 5.0 开始默认为 1G），如果超过该值，则产生新的二进制文件，后缀名 + 1，并记录到 .index 文件

* `binlog_cache_size`：指定了未提交的二进制日志的缓冲大小（默认 32KB），等事务提交时再直接从缓冲中写入二进制日志文件，该参数是基于会话的，可以通过 `show global status like 'binlog_cache%'` 判断当前的 binlog_cache_size 设置是否合适

~~~shell
show variables like 'binlog_cache%'
Variable_name    |Value|
-----------------|-----|
binlog_cache_size|32768|              // 默认 32KB

show global status like 'binlog_cache%'
Variable_name        |Value  |
---------------------|-------|
Binlog_cache_disk_use|57     |        // 使用临时文件写二进制文件的次数
Binlog_cache_use     |2097944|        // 使用缓冲写二进制日志的次数
~~~

* `sync_binlog`：表示每写缓冲多少次就同步到磁盘，为了获得最大的高可用性，建议设置为 1，即采用同步写磁盘的方式

* `binlog-do-db`：表示需要写二进制日志的库

* `binlog-ignore-db`：表示忽略写入哪些库的日志，默认为空，即需要同步所有库的日志到二进制日志

* `log-slave-update`：如果当前数据库是 slave，则 **不需要** 将从 master 取得并执行的二进制日志写入自己的二进制文件中去。如果是 master=>slave->slave 架构的复制，则必须设置该参数

* `binlog_format`：十分重要的参数，影响了记录二进制日志的格式，它有三种模式：

  * STATEMENT：MySQL 5.1 之前没有这个参数，所有的二进制文件都是基于 SQL 语句级别，binlog 只会 **记录可能引起数据变更的 SQL 语句**；优势：该模式下，因为没有记录实际的数据，所以日志量和 IO 都消耗很低，性能是最优的；劣势：但有些操作并不是确定的，比如 rand、uuid() 函数会随机产生唯一标识，当依赖 binlog 回放时，该操作生成的数据与原数据必然是 **不一致** 的，此时可能造成无法预料的后果
  
  * ROW：在该模式下，binlog 记录的不再是简单的 SQL 语句了，而是记录每次操作的源数据与修改后的目标数据；优势：可以绝对精准的还原，从而保证了 **数据的安全与可靠**，并且复制和数据恢复过程可以是并发进行的（可以将 InnoDB 的事务隔离级别设置为 `READ COMMITTED`，以获得更好的 **并发性**）；劣势：缺点在于 binlog **体积** 会非常大，同时，对于修改记录多、字段长度大的操作来说，记录时 **性能消耗** 会很严重。阅读的时候也需要特殊指令来进行读取数据
  
  * MIXED：是对上述 STATEMENT 跟 ROW 两种模式的混合使用；对于绝大部分操作，使用 STATEMENT 来进行 binlog 的记录，只有以下操作使用 ROW 来实现：表的存储引擎为 NDB，使用了 uuid() 等不确定函数，使用了 insert delay 语句，使用了临时表，使用了用户定义函数

#### InnoDB 存储引擎文件

除了 MySQL 数据库本身的文件外，每个表存储引擎还有其自己独有的文件：

* 表空间文件

InnoDB 将存储的数据按表空间（tablespace）进行存放，用户可以通过参数 `innodb_data_file_path` 对其进行设置：

~~~shell
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
// 将 ibdata1 和 ibdata2 两个文件组成表空间，用完后可以自动增长
~~~

所有基于 InnoDB 存储引擎的表的数据都会记录到该 **共享表空间** 中。如设置了参数 `innodb_file_per_table` 则会基于每个表产生一个独立表空间 `表名.ibd`，需要注意的是，这些单独的表空间文件仅存储该表的数据、索引和插入缓冲等信息，其余信息还是存放在默认的表空间中

![](/images/innodb-ibdata-file.jpg)

* 重做日志文件

在 InnoDB 存储引擎的数据目录下有两个名为 ib_logfile0 和 ib_logfile1 的文件，他们记录了对于 InnoDB 存储引擎的事务日志。在实例因主机掉电失败时，会使用重做日志恢复到掉电前的时刻，以 **保证数据的完整性**。

每个 InnoDB 存储引擎至少有一个重做日志文件组（group），每个文件组下至少有两个重做日志文件。用户可以设置多个镜像日志组（mirrored log groups）以得到更高的可靠性。**重做日志文件的大小** 对存储引擎的性能有非常大的影响，如果设置得很大，则在恢复时可能需要很长的时间；如果设置得过小，可能会导致一个事务的日志需要多次切换重做日志文件，或者导致频繁地发生 `Async Checkpoint`，导致性能的抖动。

写入重做日志的操作不是直接写，而是先写入一个重做日志缓冲 redo log buffer，然后按照一定的条件顺序写入日志文件，如下图：

![](/images/innodb-redo-log.jpg)

主线程每秒会将重做日志缓冲写入磁盘的重做日志文件中，不论事务是否已经提交。另一个触发写磁盘的过程是由参数 `innodb_flush_log_at_trx_commit` 控制，表示在提交操作时，处理重做日志的方式：

| innodb_flush_log_at_trx_commit 的值 | 处理重做日志的方式                         |
| ----------------------------------- | ------------------------------------------ |
| 0                                   | 提交事务时不写，等待主线程每秒的刷新       |
| 1                                   | 执行事务提交时将重做日志 **同步** 写到磁盘 |
| 2                                   | 执行事务提交时将重做日志 **异步** 写到磁盘 |

为了保证事务 **ACID 的持久性**，必须将 `innodb_flush_log_at_trx_commit` 设置为 1，也就是每当有事务提交时就必须确保事务都已经写入重做日志文件。那么当数据库因为意外发生宕机时，可以通过重做日志文件恢复，并保证可以恢复已经提交的事务。

与二进制日志文件的区别：二进制日志是在事务提交完成后就记录所有与 MySQL 数据库（包括其他存储引擎的）有关的日志记录，记录的都是关于一个事务的具体操作内容，是逻辑日志；而重做日志是在事务进行的过程中也在不断记录每个页更改的物理情况（并不是随事务提交的顺序进行写入的）。

### 锁

锁是数据库系统区别于文件系统的一个关键特性，它对数据库的并发访问提供了支持，并确保了每个用户能 **以一致的方式读取和修改数据**，实现了数据库事务的隔离性。

不同的数据库和存储引擎在锁的实现上不尽相同，比如 MyISAM 的 **表锁** 设计在并发插入的时候性能就不尽人意，SQL Server 2005 版本之前的 **页锁** 一定程度上提高了并发性能，但对热点数据页的并发问题仍无能为力，到 2005 版本 SQL Server 开始支持乐观并发和悲观并发，在乐观并发下开始支持行级锁。在 SQL Server 中锁是一种稀有的资源，锁越多开销就越大，因此它会有 **锁升级**，如果行级锁升级到表锁，并发性能就又回到了从前。

InnoDB 存储引擎的锁实现和 Oracle 数据库非常类似，提供一致性的非锁定读、行级锁支持。行级锁 **没有额外的开销**，不需要锁升级，可以同时得到并发性和一致性。

除了共享锁 S Lock 和排他锁 X Lock 外，InnoDB 还支持 **多粒度锁定和意向锁**，如果需要对页上的记录上 X 锁，需要先对数据库、表、页上意向锁 IX，若其中任何一个部分等待，那么该操作需要等待粗粒度锁的完成。意向锁与行级锁的兼容性如下表所示：

![](/images/innodb-lock-granual.jpg)

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | √      | √      | √      | 不兼容 |
| IX   | √      | √      | 不兼容 | 不兼容 |
| S    | √      | 不兼容 | √      | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

InnoDB 支持意向锁设计比较简练，其意向锁即为 **表级别** 的锁，因此意向锁其实不会阻塞除全表扫以外的任何请求。

#### 一致性非锁定读

InnoDB 存储引擎在读取正在执行 DELETE 或 UPDATE 操作的行时，不会等待行上 X 锁的释放，而是去读取行的一个 **快照** 数据。该实现通过事务中回滚数据用的 undo 段来完成，因此快照数据本身没有额外的开销。读取快照数据也不需要上锁，因为没有事务需要对历史的数据进行修改操作。

![](/images/consistent-nonlocking-read.jpg)

非锁定读的机制让读取不会占用和等待表上的锁，极大地 **提高了数据库的并发性**。每行记录可能有不止一个历史版本，由此带来的并发控制称为多版本并发控制 MVCC。InnoDB 在 RC、RR 隔离级别下使用非锁定的一致性读，但采用了不同的读取方式。在 RC 下总是读取被锁定行的最新一份快照数据，而在 RR 下总是读取事务开始时的行数据版本。

需要特别注意的是，RC 的非锁定读 **违反** 了事务 ACID 中的 I 特性，如下表：

| 时间 | 会话 A                                       | 会话 B                                 |
| ---- | -------------------------------------------- | -------------------------------------- |
| 1    | BEGIN                                        |                                        |
| 2    | SELECT * FROM parent WHERE id = 1;           |                                        |
| 3    |                                              | BEGIN                                  |
| 4    |                                              | UPDATE parent SET id = 3 WHERE id = 1; |
| 5    | SELECT * FROM parent WHERE id = 1;（能读到） |                                        |
| 6    |                                              | COMMIT;                                |
| 7    | SELECT * FROM parent WHERE id = 1;（读不到） |                                        |
| 8    | COMMIT                                       |                                        |

#### 一致性锁定读

虽然在默认的 RR 模式下，InnoDB 的 SELECT 操作使用一致性非锁定读，但在某些情况下用户需要显式地对数据库读取操作进行加锁以保证数据逻辑的一致性，InnoDB 支持两种锁定读操作：

| 锁定读操作                    | 锁类型 |
| ----------------------------- | ------ |
| SELECT ... FOR UPDATE         | X 锁   |
| SELECT ... LOCK IN SHARE MODE | S 锁   |

锁定读的锁会在事务提交后释放，因此在使用上述两句 SELECT 锁定语句时，务必加上 BEGIN，`START TRANSACTION` 或者 `SET AUTOCOMMIT=0`

#### 自增长和外键中的锁

自增长是很多 DBA 或开发人员首选的主键方式，InnoDB 对每个含有自增长的表都有一个自增长计数器，在进行插入操作时会依据这个自增长的计数器加 1 赋予自增长列，其实现使用了 `AUTO-INC Locking`，这种锁不是在一个事务完成后释放，而是在 **完成对自增长值插入的 SQL 语句后立即释放**，提高了插入的性能。

虽然 AUTO-INC Locking 一定程度上提高了并发插入的效率，但对于大数量的插入，仍需等待前一个插入的完成。从 MySQL 5.1.22 版本开始，InnoDB 提供了一种 **轻量级互斥量** 的自增长实现方式，提供了一个参数 `innodb_autoinc_lock_mode` 来控制自增长的模式，大大提高了自增长值插入的性能，其有效值如下表所示：

| innodb_autoinc_lock_mode |                              说明                                  |
| ------------------------ | ------------------------------------------------------------------ |
|             0            | MySQL 5.1.22 版本之前自增长的实现方式，通过表锁的 AUTO-INC-Locking 方式，因为有了新的自增长实现方式，0 这个选项不应该是新版用户的首选项   |
|             1（默认）     | 对于能在插入前就确定插入行数的 insert 语句，使用互斥量 `mutex` 对内存中的计数器进行累加的操作；对于不能在插入前确定插入行数的语句使用传统表锁的 AUTO-INC Locking 方式；   |
|             2            | 对所有的插入语句都通过互斥量来实现自增长，这是性能最高的方式，然而在并发插入时会导致 **自增长的值可能不是连续的**，且使用 `statement-base replication` 会出现主从数据不一致，必须使用 `row-base replication` |

MyISAM 自增长的实现和 InnoDB 不同，其采用表锁设计，自增长不用考虑并发插入的问题。如果在 master 上使用 InnoDB 存储引擎，slave 上用 MyISAM 存储引擎的的 replication 架构下，需要考虑这种情况。

外键用户引用完整性的约束检查，对于外键的插入或更新，首先需要查询父表中的记录，这时 **不会** 采用一致性非锁定读的方式，而是使用 `SELECT ... LOCK IN SHARE MODE` 的方式，主动给父表加一个 S 锁。如果这时父表上已经加了 X 锁，字表上的操作会被阻塞。如下表所示，如果访问父表时使用的是一致性非锁定读，这时 Session B 会读到父表有 id = 3 的记录，可以进行插入操作，但是如果会话 A 对事务提交了，则父表中就不存在 id = 3 的记录，这时数据在父、字表就会存在不一致的情况。

| 时间 | 会话 A                           | 会话 B                                                       |
| ---- | -------------------------------- | ------------------------------------------------------------ |
| 1    | BEGIN                            |                                                              |
| 2    | DELETE FROM parent WHERE id = 3; |                                                              |
| 3    |                                  | BEGIN                                                        |
| 4    |                                  | INSERT INTO child SELECT 2,3 <br />  // 第二列是外键，执行该句时被阻塞 |
#### 锁的算法

InnoDB 一共有三种行锁：

| 行锁                    | 锁定范围                                |
| ------------- | ----------------------------------------------- |
| Record Lock   | 单个行记录上的锁                                  |
| Gap Lock      | 间隙锁，锁定一个范围，但不包含记录本身               |
| Next-Key Lock | Gap + Record Lock，锁定一个范围，并且锁定记录本身   |

Record Lock 总是会去 **锁住索引记录**，如果没有索引，则使用隐式的主键进行锁定。Next-Key Lock 设计的目的是为了解决幻读，然而当查询的索引含有唯一属性时（查询条件需包含所有的唯一索引列，不然仍是 range 类型查询，依然使用 Next-Key Lock），InnoDB 会对 Next-Key Lock 进行优化，将其 **降级为 Record Lock**，即仅锁住索引本身，而不是范围，提高应用的并发性。

```mysql
CREATE TABLE z (a INT, b INT, PRIMARY KEY(a), KEY(b));

INSERT INTO z SELECT 1,1;
INSERT INTO z SELECT 3,1;
INSERT INTO z SELECT 5,3;
INSERT INTO z SELECT 7,6;
INSERT INTO z SELECT 10,8;
```

若会话 A 执行 `SELECT * FROM z WHERE b=3 FOR UPDATE`，其通过索引列 b 进行查询，因此使用 Next-Key Locking 技术加锁，对两个索引分别进行锁定。对于聚集索引仅对列 a=5 的索引加上 Record Lock，对辅助索引加上 Next-Key Lock（1, 3），InnoDB 还会对辅助索引的下一个键值加上 Gap Lock，即还有一个范围为（3, 6）的锁。

```shell
SELECT * FROM z WHERE a = 5 LOCK IN SHARE MODE;    // a=5 聚集索引锁占用，阻塞
INSERT INTO z SELECT 4,2;   // b=2 在辅助索引的锁范围内，阻塞
INSERT INTO z SELECT 6,5;   // b=5 在辅助索引的锁范围内，阻塞

// 下面的 SQL 不会被阻塞，可以立即执行
INSERT INTO z SELECT 8,6;
INSERT INTO z SELECT 2,0;
INSERT INTO z SELECT 6,7;
```

可以看到，Gap Lock 的作用是为了阻止多个事务将记录插入到同一个范围内，而这会导致幻读 `Phantom Problem` 问题的产生。例如在上面的例子中，会话 A 已经锁定了 b = 3 的记录，若此时没有 Gap Lock 锁定（3, 6），那么用户可以插入索引列 b = 3 的记录，这会导致会话 A 中的用户再次执行同样查询时会返回不同的记录，导致幻读问题的产生。

#### 锁问题

* 脏读

脏页指在缓冲池中已经被修改的页，但是还没有刷新到磁盘中；脏数据指事务对缓冲池中行记录的修改，并且还没有提交；脏页并不影响数据读取的一致性（当脏页刷新时两者最终会达到一致性），而如果读到了脏数据，即读到了另一个事务中 **未提交的数据**，显然违反了数据库的隔离性。脏读会在事务隔离级别为 READ UNCOMMITTED 下产生，但目前绝大部分的数据库都将事务的隔离级别至少设置成 RC，所以脏读线程在生产环境中并不常发生。

* 幻读 Phantom Problem

幻读指在同一事务下，连续执行两次同样的 SQL 语句，第二次可能会 **返回之前不存在的行**。如下表所示，会话 A 在 RC 下两次读取得到的结果不同，违反了事务的隔离性。


| 时间 | 会话 A                           | 会话 B                                                       |
| ---- | -------------------------------- | ------------------------------------------------------------ |
| 1    | SET SESSION tx_isolation='READ-COMMITTED'     |                                                              |
| 2    | BEGIN;                                         |                                                              |
| 3    | SELECT * FROM t WHERE a > 2 FOR UPDATE // 4    |                                                              |
| 4    |                                  | BEGIN                                                        |
| 5    |                                  | INSERT INTO t SELECT 4;     |
| 6    |                                  | COMMIT;     |
| 7    | SELECT * FROM t WHERE a > 2 FOR UPDATE // 4 5                |             |

不同于 Oracle 数据库需要在 SERIALIZABLE 的事务隔离级别下才能解决幻读问题，InnoDB 在 RR 下就通过 Next-Key Lock 的算法避免了幻读。上述例子中，`SELECT * FROM t WHERE a > 2 FOR UPDATE` 锁住的不是 4 这单个值，而是（2，+∞）这个 **范围**。因此任何对于这个范围的插入都是不被允许的，从而避免了幻读。

如果将事务隔离级别设置为 RC，将会关闭使用 Gap Lock，这样会破坏事务的隔离性，并且造成 replication **主从数据的不一致** 。此外，RC 的性能也不会优于默认的 RR。

* 不可重复读

不可重复读指一个事务两次读取到的数据不一致，因为另一个事务对该行记录进行了修改操作。与脏读的区别是：脏读读到的是未提交的数据，而不可重复读读到的却是 **已经提交的数据**，其违反了数据库事务一致性的要求。

原书上将不可重复读和幻读混为一谈，但幻读指两次读取的行数发生了变化，而不可重复读指行记录的数据内容在两次读取间发生了变化。InnoDB 在 RC 下仍会出现不可重复读的问题，因为 **RC 是读取最新一次提交的快照数据，而在 RR 下是读取事务开始时的快照数据**，因此就避免了不可重复读的问题。一般来说不可重复读是可以接受的，因为读到的是已经提交的数据，很多数据库厂商（如 Oracle，SQL Server）将其数据库事务的默认隔离级别设置为 RC，在这种隔离级别下允许不可重复读的现象。

* 丢失更新

丢失更新指一个事务的更新操作会被另一个事务的更新操作所覆盖，从而导致数据的不一致。在当前数据库的任何隔离级别下都不会出现数据库理论意义上的丢失更新（更新操作给记录上了 X 锁），但在生产应用中会出现 **逻辑意义上的丢失更新** 问题：
  1. 事务 T1 查询一行数据，放入本地内存，并显示给一个终端用户 User1
  2. 事务 T2 页查询该行数据，并将取得的数据显示给终端用户 User2
  3. User1 修改这行记录，更新数据库并提交
  4. User2 修改这行记录，更新数据库并提交

显然这个过程中 User1 的修改更新操作 "丢失" 了，要避免丢失更新发生，需要让事务在这种情况下的操作变成串行化，而不是并行操作，可以在读取记录时上一个排他 X 锁，这样步骤 2）的操作就必须等待步骤 1）和步骤 3）完成：

| 时间 | 会话 A                           | 会话 B                                                       |
| ---- | -------------------------------- | ------------------------------------------------------------ |
| 1    | BEGIN；                          |                                                              |
| 2    | SELECT cash into @cash FROM account WHERE user = pUser FOR UPDATE  |                                  |
| 3    |                               |   SELECT cash into @cash FROM account WHERE user = pUser FOR UPDATE // 等待     |
|      |         ......                         |      ......                                                    |
| m  | UPDATE account SET cash=@cash-9000 WHERE user = pUser   |                                   |
| m+1  |  COMMIT;                                |                                                   |
| m+2  |                                      |      UPDATE account SET cash=@cash-1 WHERE user = pUser       |
| m+3  |                                      |       COMMIT;        |

#### 阻塞与死锁

因为不同锁之前的兼容性问题，有时候事务中的锁需要等待另一个事务中的锁释放它所占用的资源，这就是阻塞，阻塞不是坏事，它可以确保事务可以并发且正常地运行。

InnoDB 中使用参数 `innodb_lock_wait_timeout` 来控制等待的时间（默认为 50 秒），`innodb_rollback_on_timeout` 来设定是否在等待超时时对进行中的事务进行回滚操作（默认为 OFF，代表不回滚）。需要注意的是，默认情况下 InnoDB 存储引擎 **不会回滚** 超时引发的错误异常，在大部分情况下都不会对异常进行回滚。

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。解决死锁问题最简单的一种方式就是超时，即当两个事务互相等待时，当一个等待时间超过设置的某一阈值 `innodb_lock_wait_timeout` 时，其中一个事物进行回滚，另一个等待的事务就能继续进行。

超时机制虽然简单，但根据 FIFO 的顺序选择回滚对象，若超时的事务所占权重比较大，就显得不合适了。除了超时机制，当前数据库还普遍采用 `wait-for graph`（等待图）的方式来进行死锁检测，它是一种较为主动的死锁监测机制，在每个事务请求锁并发生等待时都会判断是否存在回路，若存在则有死锁，通常来说 InnoDB 存储引擎选择 **回滚 undo 量最小的事务**。

此外还存在另一种死锁，即当前事务持有了待插入记录的下一个记录的 X 锁，但是在等待队列中存在一个 S 锁的请求，则可能会发生死锁，如下表所示：

```shell
CREATE TABLE t (
a INT PRIMARY KEY
) ENGINE=InnoDB;        // 测试表

INSERT INTO t VALUES(1),(2),(4),(5);   // 初始数据
```

| 时间 | 会话 A                                                       | 会话 B                                                   |
| ---- | ------------------------------------------------------------ | -------------------------------------------------------- |
| 1    | BEGIN；                                                      |                                                          |
| 2    |                                                              | BEGIN；                                                  |
| 3    | SELECT * FROM t WHERE a = 4 FOR UPDATE；                     |                                                          |
| 4    |                                                              | SELECT * FROM t WHERE a<=4 LOCK IN SHARE MODE；  // 等待 |
| 5    | INSERT INTO t VALUES(3)；  // ERROR 1213(40001)：Deadlock found when trying to get lock；try restarting transaction |                                                          |
| 6    |                                                              | // 事务获得锁，正常运行                                  |

这个问题是由于会话 B 中请求记录 4 的 S 锁而发生等待，但之前请求的锁对于主键值记录 1、2 都已经成功，若会话 A 能在时间点 5 能插入记录，那么会话 B 在获取记录 4 持有的 S 锁后，还需要向后获得记录 3 的记录，这样就显得有点 **不合理**。因此 InnoDB 存储引擎这里主动选择了死锁，而回滚的是 undo log 记录大的事务，和之前提到的方式又有所不同。

### 事务

事务和锁一样，也是数据库区别于文件系统的重要特性之一。事务会把数据库从一种一致状态转换为另一种一致状态，在提交时确保要么所有修改都已经保存了，要么所有的修改都不保存。

事务可以分为以下几种类型：

| 事务类型           | 特征                                                         |
| ------------------ | ------------------------------------------------------------ |
| 扁平事务           | 最简单也是使用最为频繁的事务，但不能提交或者回滚事务的某一部分 |
| 带保存点的扁平事务 | 可通过 **保存点** 实现部分回滚，减少放弃整个事务的巨大开销   |
| 链事务             | 部分提交后，释放不需要的数据对象（锁），且只能回滚到最近一个保存点 |
| 嵌套事务           | 高层事务仅负责逻辑控制，可以并行地执行子事务，单独回滚子事务 |
| 分布式事务         | 对网络中多个节点的数据库实现的事务                           |

#### Redo

重做日志 redo log 用来保证事务的原子性和持久性，有关重做日志缓冲和重做日志已经在前面 `文件-InnoDB 存储引擎文件` 中介绍过了。在每次重做日志缓冲写入重做日志文件时，因为没有使用 `O_DIRECT` 选项，因此重做日志缓冲会先写入文件系统缓存，然后调用一次 `fsync` 确保重做日志写入磁盘。由于 fsync 的效率取决于磁盘的性能，因此磁盘的性能决定了事务提交的性能，也就是数据库的性能。

InnoDB 允许用户手工设置非持久性的情况发生，以此提高数据库的性能，即修改 `innodb_flush_log_at_trx_commit` 的值为 0 或 2。该值为 0 表示事务提交时不进行写入重做日志操作，而是使用 master thread 每秒进行的重做日志文件的 fsync 操作；该值为 2 表示事务提交时将重做日志写入重做日志文件，但 **仅写入文件系统的缓存中**，不进行 fsync 操作，该设置下如果服务器宕机，重启数据库后会 **丢失未从文件系统缓存刷新到重做入职文件那部分事务**。

fsync 的执行次数会直接影响数据库的执行性能，插入 50 万行记录在不同的参数设置 `innodb_flush_log_at_trx_commit` 下的所用时间如下表所示：

| innodb_flush_log_at_trx_commit        | 执行 50 万行记录所用的时间               |
| ------------------ | ------------------------------------------------------------ |
|   0   |  13.90  秒 |
|   1   |  1 分 53.11 秒        |
|   2   |  23.37 秒   |

虽然用户可以设置参数 `innodb_flush_log_at_trx_commit` 为 0 或 2来提高事务提交的性能，但这样会 **丧失事务的 ACID 特性**。为了提高事务的提交性能，应该在将 50 万行记录插入表后进行以此的 COMMIT 操作，而不是在每插入一条记录后进行以此 COMMIT 操作，这样做还可以使事务在回滚时回滚到事务最开始的确定状态。

LSN 代表的是日志序列号，占 8 字节，LSN 表示的含义有：重做日志写入的总量、checkpoint 的位置、页的版本。LSN 记录了重做日志的总量，单位是字节，在每个页中的 LSN 表示该页最后刷新时 LSN 的大小，用来 **判断页是否需要进行恢复操作**。InnoDB 在启动时不管上次数据库运行时是否正常关闭，都会尝试进行恢复操作，由于 checkpoint 表示已经刷新到磁盘上的 LSN，因此在恢复过程中仅需恢复 checkpoint 开始的日志部分。因为重做日志记录的是物理日志，其恢复速度比逻辑日志如二进制日志要快很多，InnoDB 自身也对恢复进行了一定程度的优化，如顺序读取及并行应用重做日志，这样可以进一步地提高数据库恢复的速度。

#### Undo

undo log 用来保证事务的一致性，实现事务回滚及 MVCC 的功能。不同于 redo log，undo 位于共享表空间中，是逻辑日志，在回滚时只是将数据库 **逻辑地** 恢复到原来的样子，但是数据结构和页本身在回滚之后可能大不相同。unno 还实现了 InnoDB 中的 MVCC，使用户可以读取之前的版本信息，实现非锁定读取。

事务提交后并不能马上删除 undo log（insert undo log 除外，insert 操作只对事务本身可见，可以在事务提交后直接删除，不需要 purge 操作）及其所在的页，因为可能还有其他的事务需要通过 undo log 来得到行记录之前的版本。故事务提交时将 undo log 放入一个链表中，是否可以最终删除 undo log 及其所在页由 purge 线程来判断。由于 undo 可以被重用，并不会给每一个事务分配一个单独的 undo 页，因此 purge 操作需要涉及磁盘的离散读取操作，是一个比较缓慢的过程。

  1. delete 操作并 **不直接删除记录**，而只是将记录标记位已删除，记录最终的删除是在 purge 操作中完成的。
  2. update 主键的操作其实分两步完成，首先将原主键记录标记为已删除，之后插入一条新的记录

delete 和 update 操作可能并不直接删除原有的数据，真正删除这行记录的操作其实被 "延时" 到 purge 操作中完成。这样设计是因为 InnoDB 支持 MVCC，所以记录不能在事务提交时立即进行处理，这时其他事务可能正在引用这行，故需要保存记录之前的版本。是否可以删除这条记录通过 purge 来进行判断，若该行记录已不被任何事务引用，就可以进行真正的 delete 操作。

purge 时会从根据事务提交顺序的 history 列表中找到第一个需要被清理的 undo log，然后再从 undo page 中找 undo log，避免大量的随机读取操作，提高 purge 的效率，可以通过 `innodb_purge_batch_size` 来设置每次 purge 操作需要清理的 undo page 数量，每次回收的 undo page 越多，可供重用的 undo page 就越多，减少了磁盘存储空间与分配的开销。当 InnoDB 的压力非常大时，并不能高效地进行 purge 操作，那么 history list 会变得越来越长，当长度大于 `innodb_max_purge_lag` 时，会 **"延缓" DML 的操作**。需要注意的是，delay 的对象是行，参数 `innodb_max_purge_lag_delay` 用来控制 delay 的最大毫秒数，避免由于 purge 操作缓慢导致其他 SQL 线程出现无限制的等待。

#### Group Commit

若每次非只读事务提交都进行一次 fsync 操作来保证重做日志都已经写入磁盘，**fsync 的次数会直接影响数据库的执行性能**。当前数据库都提供了 group commit 功能，即一次 fsync 可以刷新多个事务日志写入文件，这将大大减少磁盘的压力，提高数据库的整体性能。对于写入或更新较为频繁的操作，group commit 的效率尤为明显。

在 InnoDB 1.2 版本之前，在开启二进制日志后，InnoDB 的 group commit 功能会失效，从而导致性能的下降。生产环境多使用 replication，因此这个问题尤为显著。导致这个问题的原因是开启二进制日志后，为了 **保证存储引擎层的事务和二进制日志的一致性**，二者之间使用了两阶段事务，其步骤如下：

1. 当事务提交时 InnoDB 存储引擎进行 prepare 操作

2. MySQL 数据库上层写入二进制日志

3. InnoDB 存储引擎层将日志写入重做日志文件
   
    ​	a）修改内存中事务对应的信息，并且将日志写入重做日志缓冲
    
    ​	b）调用 fsync 确保日志都从重做日志缓冲写入磁盘

![](/images/innodb-group-commit.jpg)

一旦步骤 2）中的操作完成，就确保了事务的提交，即使在执行步骤 3）时数据库发生了宕机。为了保证 MySQL 数据库上层二进制日志的写入顺序和 InnoDB 层的事务提交顺序一致（不一致会影响备份及恢复），MySQL 数据库内部使用了 `preapare_commit_mutex` 这个锁，但在启用这个锁之后，步骤 3）中的步骤 a）步 **不可以** 在其他事务执行步骤 b）时进行，从而导致了 group commit 失效。

MySQL 5.6 采用了 `Binary Log Group Commit` BLGC 的实现方式，在事务提交时将其放入一个队列中，队列的第一个事务称为 leader，其他事务称为 follower，leader 控制着 follower 的行为，BLGC 的步骤分为以下三个阶段：

1. Flush 阶段：将每个事务的二进制日志写入内存中
2. Sync 阶段：将内存中的二进制日志刷新到磁盘，若队列中有多个事务，那么仅一次 fsync 操作就完成了二进制日志的写入，这就是 BLGC 
3. Commit 阶段：leader 根据顺序调用存储引擎层事务的提交，InnoDB 本就支持 group commit，因此修复了原先由于锁 prepare_commit_mutex 导致 group commit 失效的问题

![](/images/innodb-BLGC.jpg)

当有一组事务在进行 Commit 阶段时，**其他新事务可以进行 Flush 阶段，从而使 group commit 不断生效**。当然 group commit 的效果由队列中事务的数量决定。

#### 事务的隔离级别

ISO  和 ANIS SQL 制定了四种事务隔离级别的标准，与标准 SQL 不同的是，InnoDB 在 RR 事务隔离级别下使用 Next-Key Lock 锁避免的幻读的产生，能完全保证事务的隔离性要求，即达到 SQL 标准的 SERIALIZABLE 隔离级别。

隔离级别越低，事务请求的锁越少或保持锁的时间就越短。这也是为什么大多数数据库系统默认的事务隔离级别为 RC。大部分用户可能会质疑 SERIALIZABLE 隔离级别带来的性能问题，但是根据 [《Transaction Processing》](https://book.douban.com/subject/2586390/) 的描述，两者的 **开销几乎是一样的**。同样的，即使使用 RC 的隔离级别，用户也不会得到性能的大幅提升。SERIALIZABLE 级别下的 InnoDB 会对每个 SELECT 语句自动加上 `LOCK IN SHARE MODE` ，对一致性的非锁定读不再予以支持，SERIALIZABLE 主要用于 InnoDB 存储引擎的分布式事务。

在 MySQL 5.1 中 RC 隔离级别默认只能工作在二进制日志为 ROW 的格式下，如果二进制日志工作在默认的 STATEMENT 下，则会出现如下的错误：

~~~sql
ERROR 1598 (HY000): Binary logging not possible. 
Message: Transaction level 'READ-COMMITTED' in InnoDB is not safe for binlog mode 'STATEMENT'
~~~

RC + STATEMENT 可能会在产生数据不一致问题：

~~~shell
CREATE TABLE t (
a INT PRIMARY KEY
) ENGINE=InnoDB;        // 测试表
INSERT INTO t VALUES(1),(2),(4),(5);   // 初始数据

// 会话 A 执行如下事务，且不要提交
BEGIN;
DELETE FROM t WHERE a <= 5;

// 会话 B 执行如下事务，且提交
BEGIN;
INSERT INTO t SELECT 3;
COMMIT;

// 接着在会话 A 提交，并查看表中数据
COMMIT;
SELECT * FROM t;    // 3

// 但是在 slave 上却看不到任何记录
~~~

因为 RC 下事务并 **没有使用 gap lock 进行锁定**，因此会话 B 中可以再小于等于 5 的范围内插入一条记录，STATEMENT 格式记录的是 master 上产生的 SQL 语句，因此在 master 服务器上执行的顺序为先删后插，但是在 binlog 中记录的却是先插后删，逻辑顺序上产生了不一致。

### 备份与恢复


