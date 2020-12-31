---
title: "Database：高性能 MySQL - 优化查询"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2020-12-22T13:29:47+01:00
draft: false
---

一个 MySQL 的查询任务，可以看作是一系列子任务，每个子任务都会消耗一定的时间，包括：网络、CPU 计算、生成统计信息和执行计划、锁等待等。如果要优化查询，实际上要优化其子任务，消除其中的一些子任务，减少子任务的执行次数，让子任务运行得更快。因此，了解 **查询的生命周期**、清楚查询的时间消耗情况对于优化查询有很大的意义。

在 [Database：高性能 MySQL - 索引](https://nervousorange.github.io/2020/database-index/) 篇笔者已经讲述了索引对良好的性能有着重要的影响，但如果查询写的很糟糕，即使库表结果再合理、索引再合适，也无法实现高性能。本篇将结合 [《高性能 MySQL》](https://book.douban.com/subject/23008813/) 第六章 查询性能优化 的内容讲解 MySQL 如何真正地执行查询，查询高效和低效的原因何在，以及该如何合理的设计查询。

本篇依然基于和书本相同的 MySQL 5.5 版本，本文所提及的一些 **优化器的限制** 在 MySQL 后续版本或是在 MariaDB 中得到了优化。


# 查询执行的基础

### MySQL 客户端/服务器通信协议

一般来说不需要去理解 MySQL 通信协议的内部实现细节，只需要大致理解通信协议是如何工作的。MySQL 客户端和服务器之间的通信协议是 "**半双工**" 的，这意味着在任何一个时刻，要么是由服务器向客户端发送数据，要么是由客户端向服务器发送数据，这两个动作不能同时发生，无法也无须将一个消息切成小块独立来发送。

这种协议让 MySQL **通信简单快速**，但也给了 MySQL 一些限制，比如：

* 无法进行 **流量控制**，服务器必须接收客户端传来的完整查询消息才能响应，当查询语句过长超过 `max_allowed_packet` 时，服务器会拒绝接收并抛出错误。

* 客户端必须完整地接收整个返回结果，而不能简单地只取前面几条结果，然后让服务器停止发送数据。这也是在必要的时候一定要在查询中加上 `limit` 限制的原因。

* MySQL 通常需要等所有数据都发送给客户端后才能 **释放这条查询所占用的资源**，如果查询返回了一个很大的结果集，库函数会花很多时间和内存来存储所有的结果集，给服务器造成一定的压力。

### 查询执行流程

MySQL 执行一个查询的过程，大体上可分为服务器层和存储引擎层两部分：

#### Server 层

| 阶段     | 职责                                                 |
| -------- | ---------------------------------------------------- |
| 连接器   | TCP 握手后服务器端验证登录用户身份                   |
| 查询缓存 | 先检查查询缓存，如果命中则立刻返回存储在缓存中的结果 |
| 分析器   | 根据语法规则判断输入的 SQL 语句是否满足语法规范      |
| 优化器   | 选择并生成最优的执行计划                             |
| 执行器   | 调用存储引擎的 API 执行查询                          |

#### 存储引擎层

负责数据的存储和提取，MySQL 插件式的存储引擎架构支持 InnoDB、MyISAM、Memory 等多个存储引擎。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5 开始成为了默认的存储引擎。

![](/images/mysql_execution.jpg)

### 查询缓存

TODO

### 查询优化处理

在查询缓存没有命中后，Server 会将 SQL 语句转换成一个 **执行计划**，MySQL 再依照这个执行计划和存储引擎进行交互。这包括多个子阶段：

| 阶段       | 职责                                                         |
| ---------- | ------------------------------------------------------------ |
| 语法解析器 | 通过关键字和 MySQL 语法规则对 SQL 语句进行验证解析，生成一颗 "解析树"。 |
| 预处理器   | 进一步检查解析树是否合法，检查数据表、数据列是否存在，解析名字和别名是否有歧义 |
| 查询优化器 | 将合法的语法树转化成最好的执行计划                           |

MySQL 的查询优化器 **基于其成本模型** 选择最优的执行计划，并引入了估算某些操作（比如执行一次 where 条件比较）代价的因子，可以通过 `show status like 'Last_query_cost'` 可以得知 MySQL 当前查询计算的成本。优化器在评估成本的时候并不考虑任何层面的缓存，它假设读取任务数据都需要一次磁盘 I/0，优化器对于每种类型的查询所做的优化都不尽相同，具体的会在 **优化特定类型的查询** 中进行讲述。

# 慢查询基础：优化数据访问

### 是否请求了不需要的数据

查询性能低下的最基本的原因是访问的数据太多，对于低效的查询，可以分析下查询是否检索了大量 **超过需要的数据**，确认 MySQL 是否分析了大量超过需要的数据行。

* 有些查询会请求超过实际需要的数据，然后 **将多余的数据丢弃**，这会给 MySQL 服务器带来额外的负担，并增加网络开销，且消耗了应用服务器的 CPU 和内存资源。一个常见的误解是认为 MySQL 会只返回需要的数据，实际 MySQL 却是先返回全部结果集再进行计算，一些开发者会使用 select 语句查询大量的结果，然后获取前面的 N 行后关闭结果集，应用程序会在接收全部的结果集数据后抛弃其中大部分数据，最简单有效的解决方法是在这样的查询后面加上 `limit`。

* 一些 DBA 是严格禁止 `select *` 写法的，取出所有列，会让优化器无法完成覆盖扫描这类优化，还会为服务器带来额外的 I/O、内存和 CPU 的消耗。但查询返回超过需要的数据也 **不总是坏事**，一些开发者认为这种浪费数据库资源的方式可以简化开发，提高代码的复用性。如果应用程序对获取的所有的列进行缓存，这相比多个只获取部分列的查询可能更有好处。

* 对于不断重复执行、每次都返回相同数据的查询，比较好的方案是：在初次查询的时候将这个数据 **缓存** 起来，需要的时候从缓存中取出，这样性能显然会更好。

### MySQL 是否在扫描额外的记录

在确定查询只返回需要的数据以后，接下来应该看看查询为了返回结果是否使用了预期之外的访问类型、扫描了过多的数据，即扫描的行数对返回的行数的比率较大。也就是说，查询在存储引擎层执行时 **并不能** 高效地找到需要的数据，导致需要扫描大量的数据返回给服务器层，**在服务器层过滤数据** 后才能取得需要的数据。

在 explain 语句中的 `type` 列反应了查询的访问类型，从全表扫描 all, 到索引扫描 index、范围扫描 range、索引查询 ref、常数引用 const 等，它们的速度从慢到快，扫描的行数也是从大到小。如果查询没有办法使用合适的访问类型，最好的解决办法通常是增加一个合适的索引，这也是笔者在 [Database：高性能 MySQL - 索引](https://nervousorange.github.io/2020/database-index/) 中所讲述的问题。

好的索引可以让查询使用合适的访问类型，**在存储引擎层就使用索引过滤不匹配的记录**，尽可能地只扫描需要的数据行，减少 I/O 次数。但有些查询需要扫描大量的数据却只返回少数的行，比如使用聚合函数 `count() + group by` 分组的查询，对这些查询通常也可以尝试以下技巧去尝试优化：

* 使用索引覆盖扫描，把所有需要的列都放到索引中，这样存储引擎无须回表获取对应行就可以返回结果了。

* 改变库表结构，例如使用单独的汇总表。

### 一个复杂查询还是多个简单查询

在传统的数据库查询实现中，开发者总认为网络通信、查询解析和优化是一件 **代价很高** 的事情，强调数据库层要完成尽可能多的工作，然后写出一个较为复杂的查询。但这样的想法对于 MySQL 并不适用，MySQL 从设计上让连接和断开连接都很 **轻量级**，在返回一个小的查询结果方面很高效。现代的网络速度比以前要快很多，无论是带宽还是延迟，即使是一个千兆网卡也能轻松满足每秒超过 2000 次的查询，所以运行多个小查询现在已经不是大问题了。

~~~mysql
select * from tag 
join tag_post on tag_post.tag_id = tag.id 
join post on tag_post.post_id = post.id 
where tag.tag = 'mysql';

/** 可以分解成下面的查询来代替 */
select * from tag where tag = 'mysql';
select * from tag_post where tag_id = 1234;
select * from post where post.id in (123, 456, 567, 9098, 8904);
~~~~

* 有时候一个大的更新语句可能会一次 **锁住很多数据**、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询，对此可以将一个大的更新语句 **切分** 成多个较小的语句，降低对服务器的影响，减少删除时锁的持有时间。
* 很多高性能的应用都会对关联查询进行分解，将结果再应用中进行关联。**分解关联查询** 有以下好处：
  1. 使用 IN() 代替关联查询，按照索引顺序查询可能比随机的关联更高效
  2. 语句被分解后，单个简单查询可以减少锁的竞争
  3. 可以减少冗余记录的查询，在数据库中做嵌套循环关联可能会重复地访问一部分数据，而在应用层做关联对同样的记录只需要查询一次
  4. 让缓存的效率更高，可以方便地缓存单表查询的结果对象
  5. 在应用层做关联，更容易对数据库进行拆分，做到高性能和高扩展

# 优化特定类型的查询

### 优化 count() 查询

count() 聚合函数很可能是 MySQL 中最容易被误解的前 10 个话题之一。`count(expression)` 统计表达式有值（不为 Null）的结果数，当 expression 不可能为空时，实际上就是在统计行数，比如 count(1)、count(*)，通配符 * 并不会像我们猜想的那样扩展成所有的列，实际上它会 **忽略所有的列**，直接统计行数。

MyISAM 中不带任何 where 条件的 count(*) 非常快。（InnoDB 会选择最小的索引扫描来降低成本）
* 因为无须实际地去计算表的行数，MySQL 可以利用存储引擎的特性直接获得这个值（MyISAM 维护了一个变量来放数据表的行数），当 MySQL 检测到一个表达式可以转化为常数的时候，就会一直 **把该表达式作为常数** 进行优化处理。
* 但当统计带 where 子句的结果集行数，可以是统计某个列值的数量时，MyISAM 的 count() 和其他的存储引擎没有任何不同。
* 有时候可以利用 MyISAM count(*) 全表非常快的特性做一些优化：

~~~mysql
select count(*) from world.City where id > 5

/** 将条件反转，用总数去减，就可以将扫描的行数减少到 5 行以内，
count(*) 子查询会被直接当做一个常数来处理 */
select(select count(*) from world.City) - count(*) from world.City where id <= 5
~~~

例：如果要查询返回各种不同颜色的商品数量

~~~mysql
select sum(IF(color = 'blue', 1, 0)) as blue, 
sum(IF(color = 'red', 1, 0)) as red from items;

/** 或者使用 count() 而不是 sum() 来实现同样的目的，
设置满足条件为真，不满足条件为 Null 即可 */
select count(color = 'blue' OR Null) as blue, 
count(color = 'red' OR Null) as red from items;
~~~

一般来说 count（） 查询都需要扫描大量的行（意味着访问大量数据）才能获得精确的结果，因此是比较难优化的，可以考虑使用 **索引覆盖扫描，或者增加汇总表**（外部缓存）。不过很快就会陷入到一个熟悉的困境，"快速，精确和实现简单"，三者永远满足其二，必须舍弃掉其中一个。


### 优化关联查询

MySQL 优化器最重要的一部分就是关联查询优化，其中涵盖的要点太多，其中需要特别提到的是：

* 确保 on 或者 using 子句中的列上有索引

* 确保任何的 group by 和 order by 中的表达式只涉及到一个表中的列，这样 MySQL 才有可能使用索引来优化这个过程

#### 优化器对关联查询做的优化

* 重新定义关联表的顺序

数据表的关联并不总是按照在查询中指定的顺序进行，决定关联的顺序是优化器很重要的一部分功能，关联查询优化器通过 **评估不同顺序的成本** 来选择一个代价最小的关联顺序。（在遇到右外连接时，MySQL 会将其改写成等价的左外连接。）

比如下面的查询：

~~~mysql
select film.film_id, film.title, film.release_year, 
actor.acotor_id, actor.first_name, actor.last_name
from sakila.film inner join sakila.film_actor using(film_id) 
inner join sakila.actor using(actor_id)
~~~

可以用 film 表作为 **驱动表** 先查找 film_actor 表，然后以此结果为驱动再查找 actor 表（oracle 术语表述），然而 explain 后看到的执行计划却是先用 actor 表驱动的，对此可以看下 explain `straight_join` 的执行计划，发现以 film 做驱动表后第一个关联表 **需要扫描更多的行数**（优化器预估需要读取的数据页数），第二个和第三个关联表都是根据索引查询，速度都很快，所以在实际执行的时候 MySQL 选择了顺序倒转的执行方式，让查询进行更少的嵌套查询和回溯操作。

关联优化器会尝试在所有的关联顺序中选择一个成本最小的来生成执行计划树，遍历每一个表然后逐个做嵌套循环计算每一个可能的执行计划的成本，最后返回一个最优的执行计划。糟糕的是，如果有 n 个表关联，就需要检查 **n 的阶乘** 种关联顺序，那所有可能的执行计划的 "**搜索空间**" 随着关联表个数增长的速度会非常快。当搜索空间非常大的时候，优化器不可能逐一评估每一种关联顺序的成本，当关联的表超过 `optimizer_search_depth` 时，优化器会选择使用 "**贪婪**" 搜索的方式查找 "最优" 的关联顺序，不会计算每一种关联顺序的成本，所以偶尔也会选择一个 **不是最优** 的执行计划。

* 将外连接转化成内连接

并不是所有的 outer join 语句都必须以外连接的方式执行，诸多因素可能让外连接等价于一个内连接。

* 使用等价变换规则

MySQL 可以使用一些等价变换来简化并规范表达式，合并和减少一些比较，移除一些恒成立和一些恒不成立的判断。

* 等值传播

如果两个列的值通过等式关联，那么 MySQL 能够把其中一个列的 where 条件传递到另一个列上。

~~~mysql
select film.film_id from sakila.film 
inner join sakila.film_actor using(film_id) where film.film_id > 500
/** where 子句中的 film_id 不仅适用于 film 表，而且对于 film_actor 表同样适用 */
~~~

* 列表 IN() 的比较

在很多数据库系统中，IN() 完全 **等同于多个 OR 条件** 的子句，这在 MySQL 中是 **不成立** 的，MySQL 将 IN() 列表中的数据先进行排序，然后通过 **二分查找** 的方式来确定列表中的值是否满足条件，这是一个 `O(log n)` 复杂度的操作，而等价转换成 OR 查询的复杂度为 O(n)，对于 IN() 列表中有大量取值的时候，MySQL 的处理速度将会更快。

#### MySQL 如何执行关联查询

理解 MySQL 如何执行关联查询至关重要，`UNION` 查询会先将一系列单个查询的结果放到一个 **临时表** 中（MySQL 的临时表是 **没有任何索引** 的，在编写复杂的子查询和关联查询的时候需要注意这一点），然后再重新读出临时表来完成 union 查询；MySQL 对 `JOIN` 会执行 **嵌套循环关联** 操作，即 MySQL 先在一个表中循环取出单条数据，然后再嵌套循环到下一个表中寻找匹配的行，依次下去直到找到所有表中匹配的行为止。然后根据各个表匹配的行，返回查询中需要的各个列。MySQL 会尝试在最后一个关联表中找到所有匹配的行，如果最后一个关联表无法找到更多的行，MySQL 返回到上一层关联表（**回溯**），看是否能够找到更多的匹配记录，依此类推迭代执行。

* 内连接 INNER JOIN

~~~mysql
select tb1.col1, tb2.col2 from tb1 
innner join tb2 using(col3) where tb1.col1 in (5, 6);

/** 伪代码如下 */
outer_iter = iterator over tb1 where col1 in (5, 6)
outer_row = outer_iter.next
while outer_row
  inner_iter = iterator over tb2 where col3 = outer_row.col3
  inner_row = inner_iter.next
  where inner_row
    output [outer_row.col1, inner_row.col2]
    inner_row = inner_iter.next
  end
  outer_row = outer_iter.next
end
~~~

内连接的泳道图如下所示：

【图6-2】

* 外连接 OUTER JOIN

~~~mysql
select tb1.col1, tb2.col2 from tb1 
left outer join tb2 using(col3) where tb1.col1 in (5, 6)

/** 伪代码如下 */
outer_iter = iterator over tb1 where col1 in (5, 6)
outer_row = outer_iter.next
while outer_row
  inner_iter = iterator over tb2 where col3 = outer_row.col3
  inner_row = inner_iter.next
  if inner_row
    where inner_row
      output [outer_row.col1, inner_row.col2]
      inner_row = inner_iter.next
    end
  else
    output [outer_row.col1, Null]   /** 与内连接区别的地方 */
  end
  outer_row = outer_iter.next
end
~~~

### 优化子查询

MySQL 的子查询 **实现得非常糟糕**，最糟糕的一类查询是 where 条件中包含 IN() 的子查询语句。例如下面的查询：

~~~mysql
select * from sakila.film where film_id in
(select film_id from sakila.film_actor where actor_id = 1)

| id | select_type       |  table     |  type  |  possible_keys          |
| -- | ----------------- | ---------- |------- |------------------------ |
| 1 | PRIMARY            | film       | ALL    | Null                    |
| 2 | DEPENDENT SUBQUERY | film_actor | eq_ref | PRIMARY, idx_fk_film_id |
~~~

一般会认为 MySQL 会先执行子查询返回所有包含 actor_id 为 1 的 film_id 列表，然后用 IN() 列列表查询。很不幸 MySQL **不是** 这样做的，MySQL 会将相关的外层表压倒子查询中，它认为这样可以更高效地查找到数据行，查询会被改写成下面的样子：

~~~mysql
select * from sakila.film where exists
(select * from sakila.film_actor where actor_id = 1 
 and film_actor.film_id = film.film_id)
~~~

这时子查询需要根据 film_id 来关联外部表 film，因为需要 film_id 字段，MySQL 认为无法先执行这个子查询，所以它选择先对 film 表进行 **全表扫描**，然后根据返回的 film_id **逐个执行子查询**。如果外层的表是一个非常大的表，那么这个查询的性能会非常糟糕。当然我们很容易用下面的办法来重写这个查询：

~~~mysql
select film.* from sakila.film 
inner join sakila.film_actor using(film_id) where actor_id = 1
~~~

并不是所有的关联子查询的性能都很差，

仍需注意的是，MySQL 在 from 子句中遇到子查询时，先执行子查询并将其结果放到一个临时表中（MySQL 的临时表是没有任何索引的，在编写复杂的子查询和关联查询的时候需要注意这一点），然后将这个临时表当做一个普通表对待（派生表 derived）。

### 优化排序

### 优化 group by 和 distinct

### 优化 limit 分页

### 优化 UNION 查询

#### UNION 的限制

