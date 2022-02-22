---
title: "Couchbase: Blazing Fast. Surprisingly Affordable."
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2022-02-21T13:19:47+01:00
draft: false
---

[Couchbase](https://www.couchbase.com/) 是为企业应用设计的现代分布式文档数据库，具有强大的搜索引擎和内置的操作和分析能力。它拥有 NoSQL 的强大功能，并提供快速、高效的数据双向同步。

本篇根据 [Couchbase 官网文档](https://docs.couchbase.com/home/server.html) 介绍其数据模型、集群可用性特点等。

### 概览



### 数据

Couchbase 中的每个数据都是一个键值对型的 item，它拥有在 bucket 中唯一的键，它的值可以是任何形式的二进制或者 JSON (可以包含基本/复杂的数据类型)。

```js
{
  "a1" : number,
  "a2" : "string",
  "a3" : {   // nested array
    "b1" : [ number, number, number ]
    },
  "a4" : [   // array contains objects
    { "c1" : "string", "c2" : number },
    { "c1" : "string", "c2" : number }
  ]
}
```

#### 数据模型

JSON 提供了快速的序列化与反序列化，并且是大多数 REST API 的返回类型。文档通常被认为等效于关系型数据库中的行记录，但它可以存储嵌套文档或数组，提供了更多的灵活性。

文档可以包含嵌套结构，这让开发者可以在不使用引用或连接表的情况下表达多对多的数据关系，并自然地展示分层数据。一个支持用户根据日期检索的在线航班预订应用，它的关系模型需要多张表 —— 航班、航线、日程，如下图所示：

![](/images/couchbase-relationalDataModel.png)

相对地，文档模型却只需要一个嵌套了所有航班日程的文档：

![](/images/couchbase-jsonDataModel.png)

因此在文档模型中，每个文档都可以 **高度自包含**。这将支持应用需求的快速实现，并且对可扩展性和延迟都具有重要意义：可以在不访问其他文档的情况下原子地复制或更改一个文档；消除了负责的节点间协调需求，并减少了征用。

在文档模型中，schema 是应用结构化文档数据的结果，它完全由应用定义和管理。Couchbase 并 **不强求一致性**，文档结构可以在文档间存有差异，这允许数据以一种高效的方式被呈现。schema 也可以逐渐被应用进化：属性和结构可以被添加到文档，而无需同时更新其它文档（这个灵活性对于大型长期应用是一个特别的优势）。

Couchbase 对文档操作提供了原子性、一致性、隔离性和持久性。单个写操作被保证为完全成功或失败，操作不会导致文档处于部分更新的状态。在 **设计文档结构** 时需要考虑性能和可扩展性：

* 有的时候定义较少嵌套复杂信息的富文档比较合适，这允许信息在单个操作中被读取/写入。因为独立的对象间只有较少的关联，可扩展性也因此被加强。这也使分组属性较为容易的维持在一个相互一致的状态。
* 当访问模式可预测且数据量较少时，定义大量相互引用的文档可能较为合适，可以减少网络带宽的消耗。文件可以通过 key 相互引用。

#### N1QL

在关系型数据库中，数据以一致的结构存储在表中，并通过主键相互关联。相比之下，N1QL 将数据作为格式自由的文档处理，数据被聚集在一个称作 `keyspaces` 的大集合中。

```js
(HRData keyspace)
 {
     'Name': 'Jamie'
     'SSN': 234
     'Wage': 123
     'History':   // 在关系型数据库中，这数组会存在另一个关联表中
      [
       ['Yahoo', 2005, 2006],
       ['Oracle', 2006, 2012],
     ]
 },

 {
     'Name': Steve
     'SSN':  123,
     'Wage': 456,
 }
```

N1QL 中的 `FROM` 被用来在不同的数据源 keyspaces 中检索。同样每个文档也会视自身为一个数据源，并在其嵌套元素中执行查询。嵌套元素可以通过点操作符 `.` 进入，数组元素通过中括号 `[]` 来索引。检索出的属性可以通过 `AS` 操作符来重命名：

```shell
SELECT firstjob FROM HRData.History[0] AS firstjob
{
     'firstjob': ['Yahoo', 2005, 2006]
}

SELECT firstjob[2] FROM HRData.History[0] AS firstjob
{
     'firstjob[2]': 2006
}
```

### Buckets, 内存与存储

Couchbase 将数据保存在 Bucket 中，Couchbase buckets 在内存和硬盘中都有，而 Ephemeral bucket 作为暂时 bucket 只存在于内存中。bucket 可以设置以 **压缩** 形式存储数据来最大化资源效率，文档也可以（像 Redis 那样）设置一个过期时间。

Couchbase 使用内存来保证高性能和可扩展性：内存中会存放经常被请求的数据，最近没被使用的数据会被从内存中驱逐出去。多线程的读取和写入提供了数据的同时读取和写入，来保证交高的吞吐量。

#### Buckets

Bucket 是在逻辑上用来分组键值对的集合，共有三种 bucket 类型：

* Couchbase bucket
  * 持久存储数据，且存在于内存中。
  * 允许数据被自动复制来保证高可用，可以通过跨数据中心复制 XDCR 在集群中动态扩容。
* Ephemeral bucket
  * 当数据持久性不被要求时可作为 Couchbase buchet 的候选，用于在硬盘高度负载时提供高度一致的内存性能；也可用于更快的节点重平衡/重启
  * 当 bucket 的内存限额被超过时，会根据创建时的配置拒绝接收新的数据，或者逐出老数据。因为 Ephemeral bucket 中的数据只存在内存中，被逐出的数据将不能被再次检索
* Memcached bucket
  * 通过缓存常用数据来减少数据服务必须执行的请求数量，并不会持久化到磁盘上。

`vBuckets` （有时也被称作分片 `shards`）是集群中高效分散数据的虚拟 buckets，并且支持在多个节点间复制。Couchbase / Ephemeral bucket 都实现了 vBuckets，每个 bucket 拥有 1024 个 vBuckets 均分集群的内存和存储。写入操作会在 active vBuckets 中执行，大多数读取操作也会在 active vBuckets 中执行，当必要时数据可以从 replica vBuckets 中读取。

对象在写入或检索时会先根据它的 key 用 `CRC32` 哈希算法计算出对象存储的 vBuckets 下标值，再通过 vBuckets map 在集群中找到映射的单个节点，最后在那个服务节点上执行操作，其结构关系如下图所示：

![](/images/couchbase-vbucketToNodeMapping.png)

#### 内存

Couchbase 服务器完全集成了缓冲层，提供高速数据访问。需要写入 Couchbase bucket 中的数据，会先进去缓冲层，然后同时放入磁盘队列和复制队列（示意如下图），因此 replica bucket 可以被更新。由于配置了内存限额，缓冲层中不常用的对象会被写入磁盘，然后从内存中被移出来释放空间，这个异步过程被称为 `ejection`。

![](/images/couchbase-createDocSequence3.png)



#### 存储



### 服务与索引



### 集群与可用性



