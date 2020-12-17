---
title: "Database：高性能 MySQL - 索引"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2020-12-10T13:19:47+01:00
draft: false
---

在之前的博客 [Database：MySQL 数据库](https://nervousorange.github.io/2020/database-mysql/) 我已经介绍了自己对 MySQL 的一些基本认识，包括它的存储引擎、数据存储结构、索引及事务等，本篇及之后的一篇将着眼于 MySQL 的性能深入探究。

索引是存储引擎用于快速找到记录的一种数据结构，它对良好的性能非常关键，它能让 MySQL 以最高效、扫描行数最少的方式找到需要的记录。本篇将结合 [《高性能 MySQL》](https://book.douban.com/subject/23008813/) 第五章 创建高性能的索引 的内容深入分析，如何使用索引来提高 MySQL 数据库的查询性能。

本篇使用和书本相同的 MySQL 5.5 版本，使用 [官方提供的测试数据库](https://github.com/datacharmer/test_db)，通过 [MySQL Workbench](https://dev.mysql.com/downloads/workbench/) 导入 dump 数据。

# 索引的类型

### B-Tree 索引

在 [Database：MySQL 数据库](https://nervousorange.github.io/2020/database-mysql/) 我已经介绍过 B-Tree 索引了，这里谈下一个面试常问的问题：为什么 MySQL 的索引要使用 B+ 树而不是其他的树形结构，比如 B 树？

* B 树是一颗多叉树，相比于二叉树在相等数据量的情况下，树的高度更低，可以减少查询时的 I/O 次数。
* B+ 树相比 B 树，只有叶子节点存放数据，非叶子节点只存储索引，这样就能在一张 16kb 的非叶子节点数据页中 **存储更多的指针**，降低树的高度，减少查询 I/O 次数，提高查询性能。
* 计算机层面，一次 I/O 就是磁盘的 **寻道、寻点、拷贝到内存** 三步操作。寻道是磁臂移动到指定磁道所需要的时间，一般在 5ms 以下；寻点是从磁道中找到数据存在的那个点，平均为半圈时间，在一个 7200 转/min 的磁盘，大概是 60,000 / 7200 / 2 = 4.17 ms；拷贝到内存很快，相比前面两步可以忽略不计，所以一次 I/O 的时间平均是在 9ms 左右。

#### 优缺点

* 索引列是顺序组织存储的，适合范围查询，还可用于 `order by` 和 `group by`
* **最左前缀匹配原则** 给 B-Tree 索引的使用带来一些限制，比如一定要按索引的最左列开始查找、无法使用左侧通配符、范围查询会导致右侧索引列失效，不能跳过索引中的列等。

### 哈希索引

哈希索引基于哈希表实现，将对索引列计算得到的 **哈希码** 作为键值存储于索引中，如果多个列的哈希值相同，将以 **链表** 的方式存储于同一个哈希条目中。因为哈希索引只存储哈希值，所以索引结构十分紧凑，这使哈希索引查找的速度非常快，然而哈希索引也有许多限制：

1. 哈希索引并不是按照索引值顺序存储的，无法用于排序
2. 由于对比的是由全部索引列的全部内容计算得到的哈希值，所以哈希索引 **只支持等值比较查询**，不支持部分索引列，同时也不支持范围查询
3. 当出现 **哈希冲突** 时，存储引擎必须遍历表中所有的行指针，逐行比较直到找到所有符合条件的行。

虽然有这些限制，但哈希索引带来的性能上的显著提升，使它仍适用于某些特定的场合，比如：

* 数据仓库应用中的 “星型” schema，通过哈希索引查找很多的关联表
* 自适应哈希索引
* 对长字段比如 URL 创建的伪哈希索引

# 高性能的索引策略

### 独立的列

如果查询中的列不是独立的，MySQL 就不会使用索引。“独立的列” 指索引列不能是表达式的一部分，也不能是函数的参数。例如下面的查询，因为 `emp_no + 1` 表达式不是独立的列，使查询无法使用主键 `emp_no` 索引，只能使用效率低下的全表扫描。

~~~mysq
explain(select emp_no, birth_date, first_name, last_name, gender, hire_date
FROM employees.employees where emp_no + 1 = 10003)

id|select_type|table    |type|possible_keys|key|key_len|ref|rows  |Extra      |
--|-----------|---------|----|-------------|---|-------|---|------|-----------|
 1|SIMPLE     |employees|ALL |             |   |       |   |300030|Using where|

~~~

业务中我们常常对日期列做一些函数运算再比较，但这样就无法使对应的索引列生效，如下面的查询。我们应该养成简化 `where` 条件的习惯，始终 **将索引列单独放在比较符号的一侧**。

~~~mysql
explain(select emp_no, salary, from_date, to_date from salaries
where emp_no = 10001 and TO_DAYS(CURRENT_DATE) - TO_DAYS(from_date) <= 365 * 20)

id|select_type|table   |type|possible_keys|key    |key_len|ref  |rows|Extra      |
--|-----------|--------|----|-------------|-------|-------|-----|----|-----------|
 1|SIMPLE     |salaries|ref |PRIMARY      |PRIMARY|4      |const|  17|Using where|

/** 可以看到使用独立的列的查询，使用到了 PRIMARY 的两个索引列，扫描了更少的行 */
explain(select emp_no, salary, from_date, to_date from salaries
where emp_no = 10001 and from_date > '2000-01-01')

id|select_type|table   |type |possible_keys|key    |key_len|ref|rows|Extra      |
--|-----------|--------|-----|-------------|-------|-------|---|----|-----------|
 1|SIMPLE     |salaries|range|PRIMARY      |PRIMARY|7      |   |   3|Using where|
~~~

### 前缀索引和索引选择性

很长的字符串列，会让索引变得大且慢，除了对其使用伪哈希索引，通常还可以索引开始的部分字符，这样可以大大节约索引空间，提高索引效率，但这样也会降低索引的选择性。

`索引的选择性 = 不重复的索引值 / 数据表的记录总数 (#T)`，选择性高的索引可以在查找时过滤掉更多的行，查询效率较高，唯一索引的选择性最高 为 1。

可以创建一个合适长度的 **前缀索引** `alter table <tableName> add key (<columnName>(<length>))` ，保证前缀索引的选择性接近于完整列的选择性，可以使索引更小、更快，但它也有缺点，比如无法使用前缀索引做 `order by` 和 `group by` ，也无法用前缀索引做覆盖扫描。

### 多列索引

很多人对多列索引的理解都不够，一个常见的误区就是为多个列创建独立的索引。多个单列索引大部分情况下 **并不能** 提高 MySQL 的查询性能，MySQL 5.0 引入了 “**索引合并**”，一定程度上可以使用表上的多个单列索引来定位指定的行。

```mysql
SELECT emp_no, dept_no FROM employees.dept_emp 
where dept_no = 'd005' or emp_no  = 10002

/** 在 MySQL 5.0 之前的版本中，将查询改成 UNION 方式能更好的使用索引 */
select emp_no, dept_no from dept_emp where dept_no = 'd005' union ALL 
select emp_no, dept_no from dept_emp where  emp_no  = 10002 and dept_no != 'd005'
```

索引合并虽然能够用到多个单列索引进行嵌套查询操作，但需要 **耗费大量 CPU 和内存资源** 在算法的缓存、排序、合并操作上。更为重要的是，优化器只关心随机页面读取，不会把这些计算到 “查询成本” 中，使得查询的成本被低估，导致该执行计划还不如直接走全表扫描。

通常来说，与其让优化器选择使用索引合并，还不如像 MySQL 5.0 之前的版本那样，将查询改为 UNION 的方式往往更好（可以通过参数 `optimizer_switch` 来关闭索引合并功能）。

### 选择合适的索引列顺序

B-Tree 索引首先按照最左列排序，其次是第二列，等等。由于最左前缀匹配原则的存在，查询使用索引需要从索引最左列开始，如此多列索引的顺序就显得至关重要。

经验法则：将 **选择性最高的列** 放索引最前列，这在某些场景是有帮助的（比如不考虑排序、分组和范围条件时），但考虑问题需要更加全面，往往需要根据 **运行频率的高低** 来调整索引列的顺序。

### 聚簇索引

聚簇索引是一种数据存储方式，或者说是索引组织表 index-organized table，

| 索引       | 非叶子节点 | 叶子节点      |
| ---------- | ---------- | ------------- |
| 聚簇-主键  | 索引列     | 数据          |
| 聚簇-二级  | 索引列     | 索引列+主键值 |
| 非聚簇索引 | 索引列     | 数据所在行号  |

![](/images/cluster-index.jpg)

### 覆盖索引

### 使用索引扫描来做排序

### 压缩索引（MyISAM）

### 松散索引

### 冗余和重复索引

### 未使用的索引

### 索引和锁

# 索引案例学习

### 支持多种过滤条件

### 避免多个范围条件

### 优化排序

