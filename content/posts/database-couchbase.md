---
title: "Couchbase: Blazing Fast. Surprisingly Affordable."
author: "Chenghao Zheng"
tags: ["Database"]
categories: ["Reading notes"]
date: 2022-02-21T13:19:47+01:00
draft: false
---

[Couchbase](https://www.couchbase.com/) 是为企业应用设计的现代分布式文档数据库，具有强大的搜索引擎和内置的操作和分析能力。它拥有 NoSQL 的强大功能，并提供快速、高效的数据双向同步。

本篇基于 [Couchbase 官网文档](https://docs.couchbase.com/home/server.html) 介绍其数据模型、内存存储、服务与搜索、集群可用性等。

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

在关系型数据库中，数据以一致的结构存储在表中，并通过主键相互关联。相比之下，Couchbase 使用 JSON 这种更灵活的数据模式，以及针对 JSON 增强的 SQL N1QL。N1QL 将数据作为格式自由的文档处理，并将数据聚集在一个称作 `keyspaces` 的大集合中。

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

每个 bucket 的可用内存由 `mem_low_wat` 和 `mem_high_wat` 来管理，当 bucket 中的数据持续地被加载到内存中，且数量达到 mem_low_wat 时，不会触发任何动作。如有更多数据被加载，数量达到 mem_high_wat 时，对象可被 eject 出去，直到内存数量下降至 mem_low_wat。如果 ejection 过程无法释放足够的空间，数据服务将停止吸收数据，并向客户端发送异常信息。

Ejection 基于 NRU (Not Recently Used) 算法，最近没有被访问的数据对象都可能从内存中 eject 出去。随着时间推移，Couchbase 服务器内存就变的碎片化了，这将导致内存无法被高效利用，为此 Couchbase 提供了 `Acitve Memory Defragmenter` ，它会定期扫描缓存，来识别较少使用的页，然后将这些页上的对象们重新打包，来释放空间。

![](/images/couchbase-tunableMemory.png)

#### 存储

Couchbase 会将数据对象压缩后存储在磁盘，当有客户端请求的时候，对象会重新从磁盘恢复到内存来响应访问。需要注意的是，Ephemeral bucket 中的对象只会维持在内存中，而不会被存储在磁盘上。

磁盘的读取和写入 I/O 是后台任务，并不会阻塞客户端的交互。多线程的读取和写入将提供磁盘上同时高性能的读写操作，线程间冲突将通过将线程分配给 1024 个 vBuckets 的特定子集来避免。

磁盘上的对象也会由于删除操作或过期时间而被删除，Couchbase 会在删除对象会保持对象的墓碑 `tombstone`（对象的元信息），以此在节点和集群间提供最终一致性。

### 服务与索引

服务会在管理员初始化节点时配置和部署，每个节点可以选择服务的一个子集，并给它们分配各自的内存限额。每种服务支持一种特殊形式的数据访问，在多个集群节点部署服务将支持较高的工作负荷，并保证最高的性能和资源可用性。

查询和分析服务将依赖索引服务提供的索引，搜索服务提供了自己内置的索引。设计巧妙的索引将对搜索操作提供显著的性能提升。

#### 服务

Couchbase 服务器提供了数据、查询、索引、搜索、分析、事件六大服务，通过多维度扩展，这些服务可以被相互独立地部署、维护，来满足业务场景的高效响应和紧急的工作负荷需求。

服务可以在每个节点上单独设置并分配内存配额（除了查询服务），每个节点可以运行每种服务的至多一个实例。如果一个集群只用作开发或测试，可以快速地为每个服务设置相同的内存配额，如下图所示：

![](/images/couchbase-cbClusterWithServicesDevelopment.png)

服务可以被分配更大的内存配额，并且可能作为节点上的唯一服务来处理更大的工作量。如果一个特别的服务需要在生产环境处理大量数据，要被密集索引，并通过查询和搜索来解决，下图的配置会更有效：

![](/images/couchbase-cbClusterWithServicesProduction.png)

#### 索引

与关系型数据库一样，Couchbase 也拥有基于每个对象唯一 key 的主键索引和基于文档内属性的二级索引（也被称作全局二级索引 GSI）。GSI 可以被独立（不依赖于 vBuckets 的划分）划分在有索引服务的集群节点，也就是说，数据对象和它的索引可以有不同的划分下标，对应不同的集群节点。GSI 可以被复制到其他的集群节点来实现高可用性和高性能（负载均衡）。

应用和数据库驱动会将 N1QL 查询提交到集群中其中一个可用的查询节点，查询节点会分析语句，规划出最优的执行计划。根据执行计划将查询递交给索引节点或数据节点来执行，其流程如下图所示：

![](/images/couchbase-query_execution.png)

有关索引对查询性能的优化可以参考另一篇博文：[Database：高性能 MySQL - 索引](https://hoffmanzheng.github.io/2020/database-index/)

#### 全文搜索

Couchbase 使用基于 [Bleve](http://blevesearch.com/) 的全文搜索，首先要在执行搜索的 bucket 上创建全文索引，默认的 `default` 的 Type Mappings 指可以对 bucket 内的所有文档进行搜索，如下图所示：

![](/images/couchbase-defaultFullTextIndex.png)

当然也可以只索引 bucket 中某个类型的文档，这样有助于减少空间和内存使用。例如指定文档 ID 下划线前的字符串为类型，hotel 为类型映射，然后将 default type mappings 取消即可，如下图所示：

![](/images/couchbase-typeMapping.png)

此外还可以仅索引文档中的某个字段，如下图所示：

![](/images/couchbase-searchField.png)

在网页管理员控制台测试搜索功能后，就可以用 REST API 来实现搜索：

```js
curl -u Administrator:password -X POST -H "Content-Type: application/json" \
http://localhost:8094/api/index/travel-FTS/query \
-d '{
  "explain": true,
  "fields": [
    "*"
  ],
  "highlight": {},
  "query": {
    "query": "hotel"
  }
}'
```

### 集群与可用性

Couchbase 集群包含一个或多个 Couchbase 服务器的实例，每个实例都运行在一个独立的节点上，数据和服务在集群中被共享。当一个 Couchbase 服务器被配置到一个节点上时，它可以被指定为一个新的集群或者作为一个已有集群的参与者。Couchbase 集群管理员运行在集群的每个节点上，并在节点间通信，保证所有节点的健康。

数据会在不影响应用的情况下自动分散在集群中。每个 bucket 会被数据服务存储为 1024 个 vBuckets（虚拟 bucket），并平均分散在所有可用的数据服务节点。vBuckets 可以被跨集群间复制，并将其副本存于其他的节点。Couchbase 会在保证没有数据丢失的前提下自动处理节点的新增、删除和失效。在监测到配置变化时，vBuckets 和它的副本会在可用的节点上进行重新分配。跨数据中心复制 XDCR 可以将数据选择性地复制到一个远端节点，以此实现高可用性。

#### 新增/移除节点

添加节点时，会在持续处理现有的工作量的同时完成数据的重新分布 `redistribution`：

1. 用现有的集群配置更新新的节点
2. 初始化 rebalance，重新计算 vBucket 映射表
3. 节点开始从现有节点接收每个 vBuckets 的复制流，生成新的 vBuckets 副本
4. 随着 vBuckets 的数据被复制，索引也被更新，会发生从老的 vBucket 到新的 vBucket 的原子切换
5. 新节点上的新 vBuckets 会被激活，新的 vBucket 映射表被交流给其他的节点。重复上述过程直到再平衡完成。

移除一个数据服务节点的过程与新增的过程相似：在要维护的节点上创建 vBuckets，并将要移除的节点 vBucket 上的数据复制到那里。当节点上不再有 vBuckets 时，节点就会从集群中移除。当新增或移除一个没有数据服务的节点时，则无需移动数据。

#### 节点失效

在 Couchbase 服务集群中的节点会通过心跳机制提供它们的健康状态。所有实例都会定期提供心跳，其中包含了节点状态的基础统计信息。如果在一个默认时期内不再收到某个节点的心跳，集群会自动转移该故障节点。共有两种故障转移 `Failover` 的类型：

* Graceful: 这是一个在线操作，要求零停机时间。当要移除的节点包含数据服务时，在剩下的集群节点上的副本 vBuckets 会被提升为激活状态，而要移除的节点上的活跃 vBuckets 会是 dead 状态。在整个过程中，数据服务节点会被主动、有序地移除，集群为每个 bucket 保持所有 1024 个活跃的 vBuckets。
* Hard: 当节点失效时自动采取的故障转移过程，被动地从集群中移去节点，因为节点已经变得不可用或不稳定。当失去的节点曾运行数据服务，这个过程会从剩下的集群节点中将副本 vBuckets 提升激活，直到每个 bucket 有 1024 个活跃的 vBuckets。

Hard Failover 应该只在某个节点变成不可用时被使用，当失效节点曾有数据服务时，集群会进入一个降级状态。部分副本 vBuckets 会被升级成活跃 vBuckets，集群则因减少的副本 vBuckets 缺少再一次对抗节点停电的资源。如果失效的节点运行着集群中唯一一个索引服务，索引会在强制故障转移期间变得不可用。ref: [Hard Failover Example](https://docs.couchbase.com/server/6.0/learn/clusters-and-availability/hard-failover.html#hard-failover-example)

#### 集群内复制

bucket 可以拥有至多三个副本，bucket 数据允许在副本中维护和更新。当节点故障时，就可以激活故障节点维护的数据副本。为了在节点故障时保证最大的数据可用性，集群会计算并在可用节点上实施最优的 vBuckets 分配，来减少在单个节点故障时的数据损失，如下图所示：

![](/images/couchbase-vBucketReplication.png)

#### 跨数据中心复制

Cross Data Center Replication (XDCR) 能够在集群间复制数据，来避免数据中心故障，并提供分布全球的高性能数据访问。

XDCR 支持跨集群单向 `Unidirectionally` 复制，在源 bucket 中存储的数据会被复制到指定的目标 bucket。尽管被复制的数据可以用来提供服务，但这事实上是设计用作容灾恢复的备份。

![](/images/couchbase-unidirectional-xdcr.png)

`Bidirectionally` 双向复制：源 bucket 中的数据可以被复制到目标 bucket，相反地，目标 bucket 中的数据也可以被复制回源 bucket。这让两个 buckets 都可以提供数据服务，在不同地区提供更快的数据访问。需要注意的是，XDCR 只提供了基础的单向复制机制，双向复制是通过实施两个相反的单向复制来实现的。

![](/images/couchbase-bidirectional-xdcr.png)

当双向复制的数据被不同地区的应用所修改时，可能会引起 **XDCR 冲突**，即文档数据在同一时刻或多或少被不同地修改了。XDCR 提供了基于序列号或时间戳的冲突解决方法：

* 序列号是每个文档都会维护，并随着每一次文档更新而递增的。冲突时会比较源文档与目标文档的序列号，文档修订数较多的胜出。比如一个网站的点击计数，在冲突时，拥有更多计数的文档更能反应真实的计数值，在这种情况下就可以选择基于序列号的冲突解决。
* 文档最后一次写入的时间戳也可以被用来在冲突时比较，拥有最近更新时间戳的文档胜出。如果新的值持续地被写入到同一个 key，且我们更关注 "当前值" 时，就应该使用基于时间戳的冲突解决，它保证了文档拥有最新的版本。

