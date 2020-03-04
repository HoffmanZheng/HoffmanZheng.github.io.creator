---
title: "Database：MySQL 数据库"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Study notes"]
date: 2020-03-04T13:19:47+01:00
draft: false
---

本篇介绍 MySQL 数据库，主要内容有：MySQL 常见的两种存储引擎（InnoDB 与 MyISAM）、数据库索引和数据库事务隔离级别与锁。

# 存储引擎

### MyISAM 与 InnoDB

#### 区别：

1. InnoDB 支持事务，MyISAM **不支持事务**。这是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

2. InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败；（主键是能确定一条记录的唯一标识，外键用于与另一张表的关联）

3. InnoDB 是聚集索引，MyISAM 是 **非聚集索引**（堆表）。
   * 聚集索引的文件存放在主键索引的叶子节点上，因此 InnoDB 必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询（回表），先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。
   * 聚集索引的顺序就是数据的物理存储顺序，所以 insert 和 update 操作可能会导致数据重排，而导致性能下降。
   * 而 MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。 
4. InnoDB 不保存表的具体行数，执行 `select count(*) from table` 时需要全表扫描。而MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快；（MyISAM 查询效率更高，且支持全文索引）

5. InnoDB 最小的锁粒度是 **行锁**，MyISAM 最小的锁粒度是表锁。一个更新语句会锁住整张表，导致其他查询和更新都会被阻塞，因此并发访问受限。这也是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

![](/images/InnoDB.png)

#### 如何选择：

1. 是否要支持事务，如果要请选择 InnoDB，如果不需要可以考虑 MyISAM；

2. 如果表中绝大多数都只是读查询，可以考虑 MyISAM，如果既有读写也挺频繁，请使用InnoDB。

3. 系统奔溃后，MyISAM 恢复起来更困难，能否接受，不能接受就选 InnoDB；

4. MySQL 5.5 版本开始 InnoDB 已经成为 MySQL 的默认引擎（之前是 MyISAM ），说明其优势是有目共睹的。如果你不知道用什么存储引擎，那就用 InnoDB，至少不会差。
5. MyISAM 更适合 **读密集** 的表，而 InnoDB 更适合写密集的的表。 在数据库做主从分离的情况下，经常选择 MyISAM 作为主库的存储引擎。 一般来说，如果需要事务支持，并且有较高的并发读取频率（MyISAM的表锁的粒度太大，所以当该表写并发量较高时，要等待的查询就会很多了），InnoDB 是不错的选择。如果你的数据量很大（MyISAM 支持压缩特性可以减少磁盘的空间占用），而且不需要支持事务时，MyISAM 是最好的选择。  

# 索引原理

### 行格式与数据页

### 最左前缀匹配原则

### 单表记录数过大时的优化措施

# 事务与锁

事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表。  

### 事务特性与并发事务带来的问题

### 事务隔离级别与读写锁



