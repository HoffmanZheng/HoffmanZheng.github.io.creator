---
title: "Java：多线程下的安全容器"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-16T13:19:47+01:00
draft: false
---



在我之前的博客 `Java：初识多线程、原理及实现` 提及，多线程下对数据的非原子性操作会造成数据错误，为此 JDK 也提供了一些线程安全的容器。本篇主要介绍 List 类的安全容器：Vector 和 CopyOnWriteArrayList，以及 Map 类的 HashTable 和 ConcurrentHashMap。从他们之前的区别，后者对前者的改进，具体的实现原理这几个角度进行分析。



# List 类的安全容器

### 

# Map 类的安全容器

**ConcurrentHashMap 和 Hashtable 的区别**

`Hashtable` ：将 get / put 所有相关操作都 synchronized 化，这相当于给整个哈希表加了一把**大锁**，多线程访问时候，只要有一个线程访问或操作该对象，那其他线程只能阻塞，相当于将所有的操作**串行化**，在竞争激烈的并发场景中性能就会非常差。

![HashTable 的全表锁](/images/HashTableLock.png)

`ConcurrentHashMap`：对整个桶数组进行了分割分段 Segment，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 

一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和 HashMap 类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry 数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

![JDK 1.8 之前 ConrrentHashMap 的分段锁](/images/HashMapSegmentsLock.png)

到了 JDK 1.8 的时候已经摒弃了分段锁，而是直接用 Node 数组 + 链表 + 红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（ JDK 1.6 以后 对 synchronized 锁做了很多优化， 整个看起来就像是优化过且线程安全的 HashMap，虽然在 JDK 1.8 中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本）

![JDK 1.8 后的并发哈希表的锁](/images/HashMapLock.jpeg)