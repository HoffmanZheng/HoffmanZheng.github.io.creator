---
title: "Database：高性能 MySQL - 索引"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2020-12-10T13:19:47+01:00
draft: false
---

在之前的博客 [Database：MySQL 数据库](https://hoffmanzheng.github.io/2020/database-mysql/) 我已经介绍了自己对 MySQL 的一些基本认识，包括它的存储引擎、数据存储结构、索引及事务等，本篇及之后的一篇将着眼于 MySQL 的性能深入探究。

索引是存储引擎用于快速找到记录的一种数据结构，它对良好的性能非常关键，它能让 MySQL 以最高效、扫描行数最少的方式找到需要的记录。本篇将结合 [《高性能 MySQL》](https://book.douban.com/subject/23008813/) 第五章 创建高性能的索引 的内容深入分析，如何使用索引来提高 MySQL 数据库的查询性能。

本篇使用和书本相同的 MySQL 5.5 版本，使用 [官方提供的测试数据库](https://github.com/datacharmer/test_db) 作为示例，通过 [MySQL Workbench](https://dev.mysql.com/downloads/workbench/) 导入 dump 数据。

# 索引的类型

### B-Tree 索引

在 [Database：MySQL 数据库](https://hoffmanzheng.github.io/2020/database-mysql/) 我已经介绍过 B-Tree 索引了，这里谈下一个面试常问的问题：为什么 MySQL 的索引要使用 B+ 树而不是其他的树形结构，比如 B 树？

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

聚簇索引是一种数据存储方式，或者说是索引组织表 index-organized table，其与非聚簇索引（如 MyISAM）在数据存储上的差异如下所示：

| 索引       | 非叶子节点    | 叶子节点        |
| ---------- | ------------- | --------------- |
| 聚簇-主键  | 索引列 + 指针 | 行数据          |
| 聚簇-二级  | 索引列 + 指针 | 索引列 + 主键值 |
| 非聚簇索引 | 索引列 + 指针 | 数据所在行号    |

![](/images/cluster-index.jpg)

#### 优缺点

1. 将索引和数据保存在同一个 B-Tree 中，数据访问更快
2. 可将相关的数据根据主键保存在一起，可以减少查询的时候需要访问的数据页数量
3. 更新聚簇索引列的代价很高，行移动可能会产生 “**页分裂**”，导致表占用更多的磁盘空间
4. 二级索引的叶子节点保存了引用行的主键列，可能比非聚簇的要大（这样的策略减少了当出现行移动或者数据页分裂时二级索引是维护工作）
5. 使用二级索引访问时需要回表操作，进行两次索引查找（通过自适应哈希索引能够减少这样的重复工作）
6. 并发插入时，顺序的主键会导致 **间隙锁竞争**

#### 注意点

* 最好避免随机的（不连续且值的分布范围非常大）聚簇索引，避免使用 UUID 作为聚簇索引，它会使索引的插入变成完全随机，页分裂和碎片会导致插入花费更长的时间
* 随机插入 ---> 先读取写入的目标页到内存中 --->  页分裂导致需要移动大量的数据 ---> 频繁的页分裂使页变得稀疏并被不规则地填充，最终数据会有碎片
* 随机值被载入到聚簇索引后，也需要做一次 `OPTIMIZE TABLE` 来重建表并优化页的填充，所以尽可能地使用单调增加的聚簇键值来插入新行。

### 覆盖索引

如果索引的叶子节点中已经包含要查询的数据，MySQL 也可以 **使用索引直接获取列的数据**，这样就不需要再读取数据行，如果一个索引包含所有需要查询的字段的值，我们就称之为 “覆盖索引”。

#### 优缺点

1. 如果只需读取索引，就能极大地 **减少数据访问量**，减轻缓存的负载；因为索引比数据更小，更容易全部放入内存中（尤其 MyISAM 还能压缩索引）
2. 因为索引是按照列值顺序存储的，所以范围查询的 I/O 会比随机读取少得多
3. 对是聚簇索引的 InnoDB 引擎，二次索引如果能够覆盖查询，就可以避免对主键索引的二次查询
4. 覆盖索引必须要存储索引列的值，MySQL 只能使用 B-Tree 索引做覆盖索引，哈希索引、空间索引、全文索引等都不适用。

当查询使用覆盖索引的时候，在 explain 的 Extra 列可以看到 `Using index` 的信息，这与 type 列的 index 很容易搞混淆，type 列表示的是查询访问数据的方式，type 列的 index 表示的是以遍历索引树的方式进行全表扫描。

```mysql
explain(select emp_no from dept_emp where dept_no  = 'd005' limit 10)

id|select_type|table   |type|possible_keys|key    |key_len|ref  |rows  |Extra                   |
--|-----------|--------|----|-------------|-------|-------|-----|------|------------------------|
 1|SIMPLE     |dept_emp|ref |dept_no      |dept_no|4      |const|145708|Using where; Using index|
```

#### 延迟关联

很多时候没有任何索引能够覆盖查询所需的列，可以使用延迟关联来延迟对列的访问，使查询的第一阶段使用覆盖索引。在本文最后的案例学习中，就使用了延迟关联来避免在 limit 偏移量分页时对不需要数据的回表操作。

### 使用索引扫描来做排序

MySQL 有两种方式可以生成有序的结果：通过排序操作 `Using filesort`，或者按索引顺序扫描，从一条索引记录移动到紧接着的下一条记录，使用索引排序后在 explain 的执行计划中就没有出现 Using filesort 字样了。

```mysql
Table    |Non_unique|Key_name |Seq_in_index|Column_name|Collation|Cardinality|Sub_part|Packed|Null|Index_type|Comment|Index_comment|
---------|----------|---------|------------|-----------|---------|-----------|--------|------|----|----------|-------|-------------|
employees|         0|PRIMARY  |           1|emp_no     |A        |     300584|        |      |    |BTREE     |       |             |
employees|         1|last_name|           1|last_name  |A        |       2918|        |      |    |BTREE     |       |             |
employees|         1|last_name|           2|first_name |A        |     300584|        |      |    |BTREE     |       |             |
/** employees 表中存在二级索引（last_name，first_name） */

explain(select * from employees where last_name = 'Simmel' order by first_name)
id|select_type|table    |type|possible_keys|key      |key_len|ref  |rows|Extra      |
--|-----------|---------|----|-------------|---------|-------|-----|----|-----------|
 1|SIMPLE     |employees|ref |last_name    |last_name|18     |const| 167|Using where|
```

使用索引排序需要满足以下几个条件：

1. 关联多张表排序时，order by 子句引用的字段需要全部为 **执行计划中第一个表的**
2. order by 子句和查询的限制是一样的，也需要满足索引的最左前缀的要求，order by 子句的顺序需要和索引的列顺序完全一致
3. 所有排序列的排序方向需要都一致，都是 desc 或者都是 asc

```mysql
explain(select t1.emp_no, t2.last_name, t2.first_name, t2.hire_date 
from dept_emp t1 inner join employees t2 on t1.emp_no =t2.emp_no 
where dept_no = 'd005' order by t2.hire_date asc)

/** 因为 order by 子句引用了第二张表的列，所以只能在服务器层进行文件排序 */
id|select_type|table|type  |possible_keys                |key                  |key_len|ref                |rows  |Extra                                                    |
--|-----------|-----|------|-----------------------------|---------------------|-------|-------------------|------|---------------------------------------------------------|
 1|SIMPLE     |t1   |ref   |PRIMARY,dept_no_and_from_date|dept_no_and_from_date|4      |const              |166144|Using where; Using index; Using temporary; Using filesort|
 1|SIMPLE     |t2   |eq_ref|PRIMARY                      |PRIMARY              |4      |employees.t1.emp_no|     1|                                                         |

alter table dept_emp add key dept_no_and_from_date(dept_no, from_date)
explain(select t1.emp_no, t2.last_name, t2.first_name, t2.hire_date 
from dept_emp t1 inner join employees t2 on t1.emp_no =t2.emp_no 
where dept_no = 'd005' order by t1.from_date asc)

/** 只有使用执行计划中第一张表的字段排序，才能使用索引排序 */
id|select_type|table|type  |possible_keys                |key                  |key_len|ref                |rows  |Extra                   |
--|-----------|-----|------|-----------------------------|---------------------|-------|-------------------|------|------------------------|
 1|SIMPLE     |t1   |ref   |PRIMARY,dept_no_and_from_date|dept_no_and_from_date|4      |const              |166144|Using where; Using index|
 1|SIMPLE     |t2   |eq_ref|PRIMARY                      |PRIMARY              |4      |employees.t1.emp_no|     1|                        |
```

### 松散索引

MySQL 是不支持松散索引（相当于 oracle 中的跳跃索引扫描 `skip index scan`）扫描的，无法按照不连续的方式扫描一个索引。

假设存在索引（a, b）然后执行查询 `select ... from tb1 where b between 2 and 3;`，因最左前缀匹配原则的存在 MySQL 是无法使用该索引的，只能通过全表扫描来找到匹配的行。但还有一个更快的方法执行上面的查询：先扫描 a 列第一个值对应的 b 列的范围，然后再跳到 a 列第二个值去扫描对应的 b 列的范围，即使用 **松散索引** 来找出匹配的行数据。这样就无须在服务器端再根据 where 条件来过滤数据，因为松散索引扫描已经跳过了所有不需要的记录。

![](/images/skip-index-scan.jpg)

虽然对上述场景，新增一个合适的索引也可以达到优化的效果，但对于某些场景，比如第一个索引列是范围条件，第二个索引列是等值条件的查询，光靠增加索引是无法解决问题的。

MySQL 5.0 之后的版本，在某些特殊的场景下是可以使用松散索引扫描的，例如在一个分组查询中找到分组的最大值和最小值，这时会在 explain 的 Extra 字段中显示 `Using index for group-by`，在 MySQL 能够很好地支持松散索引扫描之前，可以通过给前面的索引列加上可能的常数值来绕过最左匹配的限制，这在之后的案例学习中会有具体的介绍。

```mysql
alter table employees add key(last_name, birth_date);
explain(select last_name , max(birth_date) from employees e group by last_name)

id|select_type|table|type |possible_keys|key        |key_len|ref|rows|Extra                   |
--|-----------|-----|-----|-------------|-----------|-------|---|----|------------------------|
 1|SIMPLE     |e    |range|             |last_name_2|18     |   | 201|Using index for group-by|
```

### 冗余和重复索引

MySQL 允许在相同列上创建多个索引，如果存在冗余或者重复索引，MySQL 需要 **单独维护** 这些冗余或者重复的索引，并且在优化器优化查询的时候逐个地进行考虑，这会影响性能。

#### 重复索引

在相同的列上按照相同的顺序创建的相同类型的索引。有时会在不经意间创建了重复索引，例如下面的代码：

```mysql
create table duplicateIndex (
id int not null primary key,
a int not null,
b int not null,
unique(id),
index(id)
) engine = InnoDB;
```
MySQL 的唯一限制和主键限制都是通过索引实现的，因此唯一索引、主键索引、二级索引其实是三个相同类型的索引，是重复索引。应该尽量避免创建重复索引，发现以后也应该立即移除。

#### 冗余索引

某个索引是另一个索引的前缀索引，比如（A）是（A，B）的冗余索引（这种冗余只是对 B-Tree 索引来说的）。冗余索引通常发生在为表添加新索引的时候，对于已经存在的索引前缀列，大多数情况下应该首先考虑扩展已有的索引而不是创建新的索引。

当然扩展已有的索引会导致该索引变大，从而影响其他使用该索引的查询，出于性能方面的考虑使用冗余索引。但表中的索引越多，由于对二级索引的维护开销，DML 操作的速度就会越慢。

可以使用 Percona Toolkit 中的 `pt-duplicate-key-checker` 工具来分析表结构，找出冗余和重复的索引，在删除索引的时候也需要非常小心，由于二级索引的叶子节点包含了主键值，所以在列（A）上的索引就相当于在（A，id）上的索引，像 `select ... from tb1 where A = 5 order by id` 在索引被扩展成（A，B）后就无法使用索引做排序，而只能使用文件排序了。建议使用 Percona 工具箱中的 `pt-upgrade` 来仔细检查计划中的索引变更。

### 未使用的索引

服务器上可能存在一些永远不用的索引，这样的索引完全是累赘，建议考虑删除（除了唯一索引，为了避免重复）。

可以通过在 Percona Server 中打开 `userstates` 服务器变量，让服务器正常运行一段时间，再通过查询 `INFORMATION_SCHEMA.INDEX_STATISTICS` 就可以看到每个索引的使用频率，帮助定位在服务器上未被使用的索引。

另外还可以使用 Percona Toolkit 中的 `pt-index-usage`，该工具可以读取查询日志，并对日志中每条查询执行 explain 操作，不仅可以找出哪些索引是未使用的，还可以了解查询的执行计划，帮助 **定位那些偶尔服务质量差的查询**。该工具可以将结果写入到 MySQL 的表中，方便查询结果。

### 索引和锁

索引可以让查询锁定更少的行，从而减少锁定行时带来的额外开销，**减少锁争用提高并发性**，但这只有当 InnoDB 在存储引擎层过滤掉所有不需要的行时才有效。

如果索引无法过滤掉无效的行，那么在 InnoDB 检索到数据并返回给服务器层以后，MySQL 服务器才能应用 `where` 子句，这时已经 **无法避免** 锁定行了（在 explain 的 Extra 列出现了 `Using where`，表示 MySQL 服务器将存储引擎返回行以后再应用 where 过滤条件）。

在早期的 MySQL 版本中，InnoDB 只有在事务提交后才能释放锁；在 MySQL 5.1 及之后的版本中，InnoDB 可以在服务器端过滤掉行后就释放锁。

```mysql
set autocommit = 0;
begin;
select emp_no from employees where emp_no < 10005 and emp_no <> 10001 for update;

/** 保持第一个连接打开，然后开启第二个查询，这个查询会被挂起，直到第一个事务释放第一行的锁 */
set autocommit = 0;
begin;
select emp_no from employees where emp_no = 10001 for update;
```

# 索引案例学习

假设要设计一个在线约会网站，支持多种不同特性（国家、地区、城市、性别、肤色等）的 **各种组合条件** 来搜索用户，并根据用户的最后在线时间、其他会员对用户的评分进行排序，该如何设计索引来满足上面的复杂需求呢？

### 支持多种过滤条件

虽然选择选择性好的索引能够更加有效地过滤掉不需要的行，但在这个应用场景下，往往选择性高的索引被使用的频率较低。

因此选择使用频率较高的（sex，country）作为索引的前缀列，就算某个查询不限制性别，也可以使用 `and sex in ('m','f')` 用 **多个等于条件来代替范围条件**，来让优化器选择该索引，虽然这样写不会过滤任何行，但是必须加上这个列的条件，MySQL 才能匹配索引的最左前缀。但如果列有太多不同的值，就会让 IN() 列表太长，这样做就不行了。

* 由于用户的不同特性可以组成许多不同的 where 条件组合，这样就会需要大量的索引，应该尽可能 **重用索引** 而不是建立大量的组合索引，可以使用 IN() 技巧来绕过最左前缀匹配的限制。如果未来 MySQL 能够实现松散索引，就不需要为范围查询考虑的 IN() 列表了。

* 对于一些生僻的搜索条件如（has_pictures、eye_color、hair_color、education 等），虽然这些列的选择性很高，但使用并不频繁可以选择 **忽略** 它们，让 MySQL 多扫描一些额外的行即可

* 对于多半是范围查询的 age 列，一般会选择放在索引的最后面，以便优化器能使用尽可能多的索引列

* 使用 IN() 绕过最左前缀匹配的限制不可滥用，因为每额外增加一个 IN() 条件，**优化器需要做的组合将以指数形式增加**，可能会极大地降低查询性能。新版本的 MySQL 在组合数超过一定数量后就不再进行执行计划评估了，这可能会导致 MySQL 不能很好地利用索引。

### 避免多个范围条件

```mysql
alter table employees add index (hire_date, birth_date)

/** 范围条件查询 */
explain(SELECT emp_no, birth_date, first_name, last_name, gender, hire_date
FROM employees.employees where hire_date > '2000-01-20' and birth_date = '1964-06-12')

id|select_type|table    |type |possible_keys|key      |key_len|ref|rows|Extra      |
--|-----------|---------|-----|-------------|---------|-------|---|----|-----------|
 1|SIMPLE     |employees|range|hire_date    |hire_date|3      |   |   3|Using where|

/** 多个等值条件查询，相比范围条件查询，IN() 列表可以继续使用后面的索引列 */
explain(SELECT emp_no, birth_date, first_name, last_name, gender, hire_date
FROM employees.employees where hire_date in ('2000-01-22', '2000-01-28', '2000-01-23') and birth_date = '1964-06-12')

id|select_type|table    |type |possible_keys|key      |key_len|ref|rows|Extra      |
--|-----------|---------|-----|-------------|---------|-------|---|----|-----------|
 1|SIMPLE     |employees|range|hire_date    |hire_date|6      |   |   3|Using where|
```

从 explain 给出的执行计划看，MySQL 是无法区分范围查询和多个等值条件查询的，但这两种的访问效率是不同的，对于范围条件查询，MySQL 无法再使用范围列后面的其他索引列了，但是对于多个等值条件查询则没有这个限制。

如果想要检索出最近一周上线过并且年龄在 18~25 岁的用户，查询语句大概是下面这个样子：

```mysql
select ... 
where ... 
and last_online > DATE_SUB(now(), INTERVAL 7 DAY) 
and age between 18 and 25
```

这样一个查询中就有了两个范围条件查询，没有一个直接的方法可以同时使用 last_online 和 age 两个索引列，只能将其中的一个范围查询转换为一个简单的等值比较，比如准备一个由定时任务维护的 active 列，将过去七天未曾登录的用户的值设置为 0。虽然 active 列并不是完全精确的，这类查询对精度的要求也没有那么高，如果需要精确数据，可以将 last_online 放到 where 子句，但不加入到索引中。


### 优化排序

在使用 limit 偏移量实现翻页时，如果翻页翻到比较靠后时查询可能会比较慢，因为 MySQL 需要花费大量的时间来扫描需要丢弃的数据，对此可行的策略有：反范式化、预先计算、缓存或是限制用户能够翻页的数量。

```mysql
explain(SELECT emp_no, birth_date, first_name, last_name, gender, hire_date
FROM employees.employees order by hire_date asc limit 1000, 10)

id|select_type|table    |type |possible_keys|key      |key_len|ref|rows|Extra|
--|-----------|---------|-----|-------------|---------|-------|---|----|-----|
 1|SIMPLE     |employees|index|             |hire_date|6      |   |1010|     |
```

此外还有一个比较好的策略是使用 **延迟关联**，通过覆盖索引查询返回需要的主键，再根据主键关联原表获得需要的行，这样可以减少对不需要的数据的回表次数，具体写法如下：

```mysql
explain(select t1.emp_no, birth_date, first_name, last_name, gender, hire_date from employees t1
inner join(select emp_no from employees order by hire_date limit 1000, 10) t2 on t1.emp_no = t2.emp_no)

id|select_type|table     |type  |possible_keys|key      |key_len|ref      |rows|Extra      |
--|-----------|----------|------|-------------|---------|-------|---------|----|-----------|
 1|PRIMARY    |<derived2>|ALL   |             |         |       |         |  10|           |
 1|PRIMARY    |t1        |eq_ref|PRIMARY      |PRIMARY  |4      |t2.emp_no|   1|           |
 2|DERIVED    |employees |index |             |hire_date|3      |         |1010|Using index|
```

