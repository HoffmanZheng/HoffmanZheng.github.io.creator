---
title: "Database：Redis 设计与实现"
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2021-05-08T13:19:47+01:00
draft: false
---

Redis 是一个开源的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件，支持多种类型的数据结构和范围查询。 Redis 内置了复制（replication）、事务（transactions）和不同级别的磁盘持久化（persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（high availability）。

本篇将结合 [《Redis 设计与实现》](https://book.douban.com/subject/25900156/) 讲解 Redis 的内部机制、单机特性以及多机特性。

### Redis 数据结构

Redis 有五种不同的 **对象类型**，包括字符串、列表、哈希、集合和有序集合，每种对象都用到了至少一种数据结构，这样就可以根据不同的使用场景，为对象设置不同的数据结构（内部编码）实现，从而优化对象在不同场景下的使用效率。

![](/images/redis-data-structure.jpg)

如上图所示，可以看到每种数据结构都有 2 种以上的 **内部编码实现**，例如 String 数据结构就包含了 raw、int 和 embstr 三种内部编码。同时，有些内部编码可以作为多种外部数据结构的内部实现，例如 ziplist 就是 hash、list 和 zset 共有的内部编码，而 set 的内部编码可能是 hashtable 或者 intset。

Redis 使用对象来表示数据库中的键和值，每次当我们在 Redis 数据库中新创建一个键值对时，我们至少会创建两个对象，一个键对象，一个值对象。每个对象都由一个 `redisObject` 结构表示：

~~~c
typedef struct redisObject {
	unsigned type : 4;        // 类型
	unsigned encoding : 4;    // 编码
	void *ptr;                // 指向底层实现数据结构的指针
	// ...
} robj;
~~~

#### 字符串

Redis 中的字符串对象是 **可以修改** 的，称为 SDS `Simple Dynamic String`，简单动态字符串。它有三种不同的编码实现：int、raw、embstr。当一个 key 的 value 是整型时，Redis 就将其编码为 int 类型，这种编码类型节省了内存。Redis 默认会缓存 10000 个整型值（#define OBJSHAREDINTEGERS 10000），这就意味着，如果有 10 个不同的 KEY，其 value 都是 10000 以内的值，事实上全部都是 **共享同一个对象**。

~~~shell
127.0.0.1:6379> set msg 9999
OK
127.0.0.1:6379> object encoding msg
"int"
127.0.0.1:6379> set msg hello
OK
127.0.0.1:6379> object encoding msg
"embstr"
127.0.0.1:6379> set story "long, long, long ago there lived a king."
OK
127.0.0.1:6379> strlen story
(integer) 40
127.0.0.1:6379> object encoding story
"raw"
~~~

`embstr` 和 raw 编码的长度界限是 39，长度超过 39 字节以后，就是 raw 编码类型。embstr 编码将创建字符串对象所需的 **空间分配的次数** 从 raw 编码的两次降低为一次。因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比起 raw 编码的字符串对象能更好地利用缓存带来的优势，并且释放 embstr 编码的字符串对象只需要调用一次内存释放函数，而释放 raw 编码对象的字符串对象需要调用两次内存释放函数。

raw 编码的 SDS 数据结构如下：

~~~~C
struct sdshdr {
 int len;    // buf 数组中已使用的字节数量
 int free;   // buf 数组中未使用的字节数量
 char buf[];
}
~~~~

相比 C 字符串，SDS 具有以下优点：

1. O(1) 复杂度获取字符串长度，C 字符串需要遍历获取，复杂度为 0(N)
2. 杜绝了因忘记重分配内存而导致的缓冲区溢出
3. 通过 **空间预分配**（小于 1MB，分配 2 * len + 1，大于等于 1MB 分配 1MB 的未使用空间）
和 **惰性空间释放**（free 记录未使用空间代替内存回收）减少了修改字符串时所需的 **内存重分配** 次数
4. 使用 len 属性值而不是空字符判断结尾，可以保存任意格式的二进制数据
5. 兼容部分 C 字符串函数

#### 列表

列表对象的编码可以是 ziplist 或者 linkedlist。`linkedlist` 就是我们非常熟悉的双向链表，特征有：双端、无环、带表头和表尾指针、带长度计数器、多态。当列表保存的所有字符串元素长度都小于 64 字节，并且列表长度小于 512 时，列表使用 `ziplist` 编码：

```shell
127.0.0.1:6379> rpush numbers 1 three 5
(integer) 3
127.0.0.1:6379> type numbers
list
127.0.0.1:6379> object encoding numbers
"ziplist"
```

压缩列表是 Redis 为了 **节约内存** 而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，其结构为：

`<zlbytes><zltail><zllen><entry><entry> ... <entry><zlend>`

| 属性    | 长度(字节) | 用途                                                         |
| :------ | :--------- | :----------------------------------------------------------- |
| zlbytes | 4          | 表示这个 ziplist 占用了多少字节，其中包括了 zlbytes 本身占用的 4 个字节：在对压缩列表进行内存重分配，或者计算 zlend 的位置时使用 |
| zltail  | 4          | 表示到 ziplist 中最后一个元素的偏移量，有了这个值，pop 操作的时间复杂度就是 O(1) 了，即不需要遍历整个 ziplist |
| zllen   | 2          | 表示 ziplist 中有多少个 entry，即保存了多少个元素。由于这个字段占用 16 位，所以最大值是2^16-1，当这个值等于 UNIT16_MAX(65535) 时，节点的真实数量需要遍历整个压缩列表才能计算得出 |
| entryX  | 不定       | 压缩列表包含的各个节点，节点的长度由保存的内存决定           |
| zlend   | 1          | 特殊值 0xFF(255)，用于标记压缩列表的末端                     |

每个压缩列表节点都是由 previous_entry_length、encoding、content 三个部分组成，`previous_entry_length` 记录了前一个节点的长度，程序可以通过指针运算，根据当前节点的起始地址来 **计算出前一个节点的起始地址**，从而实现从表尾向表头遍历。如果前一个节点的长度小于 254 字节，那么 previous_entry_length 会使用 1 字节记录它的长度，如果前一节点的长度大于等于 254 字节，那么 previous_entry_length 的长度为 5 字节，且属性的第一个字节会被设置为 0xFE(254)，之后四个字节则用于保存前一节点的长度。`encoding` 属性记录了节点的 content 所存数据的类型及长度，节点值 `content` 可以是一个字节数据或者整数。

如果在一个压缩列表中，有多个连续的、长度介于 250 字节到 253 字节之前的节点 e1 至 eN，这时将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点就会触发 **连锁更新**。如下图所示，因为 e1 的 previous_entry_length 属性仅长 1 字节，它没办法保存新节点 new 的长度，所以程序将对压缩列表执行空间重分配操作，并将 e1 的 previous_entry_length属性从原来的 1 字节长扩展为 5 字节长。但这又导致了 e2 的 previous_entry_length 无法记录 e1 的长度，导致需要再次执行空间重分配操作...

![](/images/redis-list-chain-update.jpg)

除了添加新节点可能会引发连锁更新之外，删除节点也可能会引发连锁更新。在最坏的情况下需要对压缩列表执行 N 次空间重分配操作，而每次空间重分配的最坏复杂度为 O(N)，所以连锁更新的最坏复杂度为 O(N^2)。尽管连锁更新的复杂度较高，但它真正造成性能问题的 **几率是很低的**，压缩列表里要恰好有多个连续的、长度介于 250 字节至 253 字节之间的节点，才有可能引发连锁更新；即使出现连锁更新，但只要被更新的节点数量不多，就不会对性能造成任何影响，所以 ziplistPush 等命令的平均复杂度仅为O(N)。

#### 哈希

哈希对象的编码可以是 ziplist 或者 hashtable。当哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节，且保存的键值对数量小于 512 个，哈希对象就使用 `ziplist` 编码，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的节点推入压缩列表表尾，再将保存了值的节点推入到压缩列表表尾，如下图所示：

![](/images/redis-hashmap-ziplist.jpg)

当上述的两个条件不满足时，哈希对象就会使用 hashtable 编码，由于 Redis 所使用的 C 语言 **没有内置** 字典这种数据结构，因此 Redis 构建了自己的字典实现，其结构如下图所示：

![](/images/redis-hashmap-dicht.jpg)

`dicht` 中的 table 指向一个数组，数组中的每个元素都指向一个 `dictEntry` 结构，它保存着一个键值对。dictEntry 包含指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决 **键冲突** 的问题。`ht` 是一个包含两个项的数组，数组中的每个项都是一个 dicht 哈希表，一般情况下，字典只使用 ht[0] 哈希表，ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。

Redis 中字典的 rehash 动作并不是一次性、集中式地完成的，而是 **分多次、渐进式** 地完成的

#### 哈希

#### 集合

#### 有序集合



### 发布与订阅

### 事务

### 排序

# 单机数据库的实现

### 数据库

### RDB 持久化

### AOF 持久化

### 服务器

# 多机数据库的实现

### 复制

### Sentinel

### 集群
