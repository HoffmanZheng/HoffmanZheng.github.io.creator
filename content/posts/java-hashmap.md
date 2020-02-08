---
title: "Java：HashMap 源码解读"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-28T14:11:47+01:00
draft: false
---

# HashMap 的实现原理

### 1. 哈希表底层实现原理

![](/images/hashmap底层结构.png)

　

哈希表是一个使用数组与链表（红黑树）实现的键值对集合。哈希桶由 Node 数组构成，键值对用实现了 `Entry<K, V>` 接口的内部类 Node 存储，它具有 next 指针，将同一个哈希桶中的键值对结构链表化；当桶中的键值对数量超过树化阈值 8 时，哈希桶会转变成红黑树结构。

```java
transient Node<K,V>[] table;
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
｝
```

### 2. 为什么采用数组+链表的数据结构

数组的随机访问功能为 O(1) 的哈希表（get / put）操作提供了保障；链表用于解决哈希冲突，存放具有哈希冲突的元素；同时链表数据结构在增删元素的时候比较方便，不用像数组一样在扩容的时候 new 一个再复制。

为了改善链表不适合查找的问题，JDK 8 提供了将链表转换成适合查找的红黑树的功能。

### 3. 哈希冲突还有哪些解决办法

1. 开放地址法：
2. 链地址法：
3. 再哈希法：
4. 公共溢出区域法：

# HashMap 扩容

### 1. HashMap 在什么条件下扩容

### 2. 为什么扩容是2的n次幂

相比于 % 取模，哈希表选择 `(n - 1) & hash` 这个更高效率的按位与运算作为哈希桶的索引运算，2^n 的二进制为 10000.. 而这样哈希桶的最大 index = 2^n -1，二进制就成了 01111.. 再与对象的哈希值做按位与运算，就能快速的计算出对应哈希桶 index 值，并且是分布均匀的。

若在创建哈希表时给定初始容量，哈希表也会用 `roundUpToPowerOf2` / `tableSizeFor` 将其扩充为2的幂次方大小。

```java
// Returns a power of two size for the given target capacity. 
static final int tableSizeFor(int cap) {
	int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

### 3. 为什么要先高16位异或低16位再取模运算



# HashMap 的 get / put 过程

### 1. 哈希表的 put 过程

![](/images/哈希表put.png)

需要注意的是，put 方法的 **返回值是 oldValue**。

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    	// 对key的hashCode()做hash
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    // Ⅰ：哈希表为空或为null则初始化，n为哈希桶数量
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
    // Ⅱ：若key对应的哈希桶i的第一个节点为null，则直接新建节点
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // Ⅲ：若当前key与数组第一个元素的相同（哈希值相等，并且key相同：==/equals）跳至下方if更新value
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // Ⅳ：若当前数组Node已经是红黑树，在树中插入新的键值对
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // Ⅴ：在链表尾部找到空的节点，插入新Node
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        // Ⅵ：若加入新节点后，该链表长度大于等于8，变成红黑树结构
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // Ⅶ：若链表中存在同一个key，跳至下方if更新value
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // Ⅷ：key插入成功，更新value
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
    	// Ⅸ：若size超过阈值，则扩容
        afterNodeInsertion(evict);
        return null;
    }
```

### 2. 哈希表的 get 过程

对key的hashCode()做hash运算，计算index;

如果在bucket里的第一个节点里直接命中，则直接返回；

如果有冲突，则通过key.equals(k)去查找对应的Entry;

- 若为树，则在树中通过key.equals(k)查找，O(logn)；
- 若为链表，则在链表中通过key.equals(k)查找，O(n)。

### 3. 还有哪些哈希算法

先说一下hash算法干嘛的，Hash函数是指把一个大范围映射到一个小范围。把大范围映射到一个小范围的目的往往是为了节省空间，使得数据容易保存。

比较出名的有MurmurHash、MD4、MD5等等

### 4. 字符串中 hashCode 的实现

```java
public int hashCode() {
    int h = hash;
    // 若当前字符串已经计算过哈希值，则直接返回
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
            // val[i]为第i+1个字符的ASCⅡ码
        }
        hash = h;
    }
    return h;
}
```



# JDK 1.8 对 HashMap 的改进

### 1. JDK 1.7 的 HashMap 存在的问题

#### 1.1 HashMap CVE 安全隐患

`Tomcat` 在2011年邮件中报道了哈希表可以通过精心构造的恶意 http 请求造成链表性能退化，并引发网站 DoS 拒绝服务攻击的现象。这个问题在之后的 JDK 7 中得到了解决：对传进来的字符串进行特殊的哈希运算 `stringHash32` ，来避免恶意的字符串传值造成的哈希表链表性能退化的情况。

```java
final int hash(Object k) {
	if (0 != h && k instanceof String) {
		return sun.misc.Hashing.stringHash32((String) k);
	}
    ...
}
```

#### 1.2 HashMap 多线程操作导致死循环问题

主要原因在于并发下 HashMap 扩容时，往新的 HashMap 转移数据可能产生循环链表（此处链表插入是往前插入的，导致和原哈希桶中的位置相反，产生环形链表），导致之后在该哈希桶中搜索键值时可能会发生的死锁现象。[详情查看酷壳：Java HashMap 的死循环](https://coolshell.cn/articles/9606.html)

不过，JDK 1.8 后解决了这个问题（使用了保持顺序的扩容 transfer 操作），但是还是不建议在多线程下使用 HashMap，因为多线程下使用 HashMap 还是会存在其他问题比如数据丢失。并发环境下推荐使用 ConcurrentHashMap 。

### 2. JDK 1.8 之前的 HashMap 源码实现

JDK 1.8 之前 `HashMap` 底层是 **数组和链表** 结合在一起使用也就是 链表散列。HashMap 通过 key 的 hashCode 经过 **扰动函数处** 理过后得到 hash 值，然后通过 `(n - 1) & hash` 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

* 扰动函数（减少哈希碰撞）

```java
/* 源码注释：对原有对象的哈希值进行一次补充的哈希运算得到结果值，
来防止低质量的哈希方程。这是非常重要的，
因为哈希表用2的幂用为哈希表的长度，在低位bits相同时就会发生哈希碰撞。*/

final int hash(Object k) {
	h ^= (h >>> 20) ^ (h >>> 12);
	return h ^ (h >>> 7) ^ (h >>> 4);
}
```

* 链地址法解决哈希碰撞

思想：为每个哈希值建立一个单链表，当发生哈希碰撞时，将记录插入到链表中。

![链地址法解决哈希碰撞](/images/HashBucketLinked.png)

### 3. JDK 1.8 之后的 HashMap 底层实现

相比于之前的版本， JDK 1.8 之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。（TreeMap、TreeSet 以及 JDK 1.8 之后的 HashMap 底层都用到了红黑树。）

**为什么选择 8 作为阈值** 

因为理想情况下，随机哈希值遵循参数为 0.5 的泊松分布，如此在同一个哈希桶中出现 8 个以上数据的概率是极低的。

![JDK 1.8 之后的哈希表结构](/images/HashMapTreeSet.jpeg)

### 4. 新的哈希算法

JDK 1.8 的 hash 方法相比于 JDK 1.7 hash 方法原理不变，但是减少了扰动次数，更加简化，并提高了运算性能。

```java
/* 源码注释：将哈希值的较高位对低位进行异或运算，来避免低位相同、只有高位不同的哈希值发生的碰撞。
这是一种在速度、实用性和位扩展之间的权衡。因此我们用最便宜的方式，减少系统损失，以及合并高位哈希值
的影响，否则这些高位哈希值将由于哈希表长度范围的限制永远不会在索引计算中使用*/

static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### 5. 为什么不直接用红黑树，而选择先用链表再转红黑树

### 6. 能否用二叉查找树代替红黑树

### 7. 为什么阈值是 8 呢

### 8. 当链表转为红黑树后，什么时候退化为链表



# HashMap 的并发问题

### 1. HashMap在并发编程环境下有什么问题啊

### 2. 在jdk1.8中还有这些问题么

### 3. 你一般怎么解决这些问题的



# HashMap key 问题

### 1. 键可以为 Null 值么
### 2. 你一般用什么作为 HashMap 的 key
### 3. 我用可变类当 HashMap 的 key 有什么问题
### 4. 如果实现让一个自定义的 class 作为 HashMap 的 key 



# HashMap Javadoc

### 1. 概述

> 哈希表是基于 Map 接口的实现。它提供了所有可选的 Map 操作并且允许 null 作为 key 和 value。HashMap大体上与 `HashTable` 相同，只是 **没有同步** 和允许了 null 作为键值。这个类 **不保证 map 的有序性**，特别是，不保证数据的顺序随着时间不会改变。

> 假设哈希方程把数据均匀的分散在哈希桶中，这个实现可以提供 O(1) 的基础操作（get / put）性能。哈希表中的迭代所需要的时间和哈希表的容量（哈希桶的数量）+键值对的数量成正比。因此，当迭代效率很重要时，不把初始容量设置的过高是非常重要的。
>
> since 1.2

### 2. 初始容量及装载因子

> 影响一个哈希表实例性能的两个参数：初始容量和装载因子。哈希表的容量是表中哈希桶的数量，初始容量    `initial capacity` 则是哈希表被创建时的容量。装载因子 `load factor` 是用来描述哈希桶自动扩容前所允许的数据最大填充程度的一个衡量。当哈希表中键值对的数量超过装载因子和当前容量的乘积后，哈希表会执行 `rehash` 操作， 即内部的数据结构会进行重构，哈希桶的数量会变成现在的两倍。

>  作为一条基本规则，默认的装载因子 0.75 提供了一个较好的时间和空间的权衡。高的装载因子会降低空间开销但提高了查询成本（反应在哈希表的大多数操作上，包括 get / put）。在设置哈希表初始容量时需要考虑 map 中预期存储的键值对的数量以及装载因子，来尽可能的降低 `rehash` 发生的次数。如果初始容量大于键值对最大数 / 装载因子，rehash 将永远不会发生。

> 当很多映射被存入一个哈希表中时，如果 **创建一个容量足够大的实例**，将会使数据存储更加高效，而不是让它在需要时自动执行 rehash 来让哈希表变大。 注意使用许多相同哈希值的 keys 会使哈希表的性能变慢。为了改善影响，当 keys 是 Comparable 时，这个类可能会使用 keys 之间的比较顺序来帮助打破平局。

### 3. 多线程问题

> 注意：这个实现并不是同步的。如果多线程并发地进入一个哈希表，和至少一个线程结构性地修改了 map，它必须是在外部被同步的。（一个结构性的修改指任何添加或者删除一个或多个映射的操作；仅仅改变一个已有键值对的 value 并不是一个结构性的修改。）这通常通过同步某个自然封装 map 的对象来实现的。

> 如果不存在这样的对象，map 应该通过 `Collections.synchronizedMap` 方法来被包裹。这最好在创建哈希表的时候就被完成，来避免意外的不同步地访问 map。

> 此类的所有由 "collection view methods" 返回的迭代器都是 `fail-fast` 机制的：如果 map 在迭代器创建后的任何时候被结构性的修改，除了通过迭代器自己的 `remove` 方法，迭代器将会丢出 **并发修改异常**。因此，面对并发修改，迭代器将快速而干净地失败，而不是在未来的不确定时间内冒任意，不确定行为的风险，

> 注意：迭代器的 fail-fast 行为无法得到保证，因为通常来说，在存在不同步的并发修改的情况下，不可能做出任何严格的保证。fail-fast 的迭代器会尽最大努力抛出并发修改异常。 因此，编写依赖于此异常以确保其正确性的程序是错误的：迭代器的快速失败行为应仅用于检测错误。

### 4. 底层实现

> 这个 map 通常用作 bin 化（bucket 化）的哈希表，但是当 bin 太大时，它们将转换为 TreeNode 的 bin，每个 bin 的结构与 java.util.TreeMap 中的结构类似。大多数 map 方法尝试使用普通的 bin，但是在应用时中继到 TreeNode 中的对应方法（只需通过检查节点的 instanceof）。TreeNodes 的 bin 可以像其他任何 bin 一样被遍历和使用，但在数据元素过多时额外地支持更快的查找。 但是，由于正常使用中的绝大多数 bin 没有过多的元素，检查是否存在 TreeNode 可能会耽误 map 方法的进程。

> TreeNode bin 主要由哈希值来排序，但哈希值相等，如果两个元素是同一个实现了 Comparable 的类的实例，则通过它们的 compareTo 方法来排序。（我们通过反射保守地检查泛型类型以验证这一点 - 参见方法 compareableClassFor）。当键具有不同的哈希值或可排序时，**增加 tree bin 的复杂度在最坏情况是 O(log n) 操作**，因此，意外或恶意的使用会逐渐降低 map 的性能，包括返回结果分布较差（其中许多 keys 共享一个哈希值，它们也都是可比较的）的 hashCode() 方法。（如果这两种方法都不适用，那么与不采取预防措施相比，我们可能会浪费大约两倍的时间和空间。但是，唯一已知的情况是由于不良的用户编程实践，它已经如此之慢，以至于几乎没有什么区别。）

### 5. 红黑树阈值

> 因为 TreeNode 的 **大小大约是常规节点的两倍**，所以我们仅在 bin 包含足够节点时才保证使用它们（请参阅 `TREEIFY_THRESHOLD`）。当它们变得太小（由于移除或调整大小）时，它们会转换回普通 bin。 在使用分布良好的 hashCode 的情况下，tree bins 几乎很少被使用。 理想情况下，在随机 hashCodes 下，bin 中节点的频率遵循 Poisson 分布，默认调整大小阈值 0.75 的在此处的参数平均约为 0.5，尽管由于调整大小粒度而差异很大。 忽略方差，列表大小 k 的预期出现次数是（exp（-0.5）* pow（0.5，k）/ factorial（k））。

随机哈希值条件下，元素在哈希桶中遵循泊松分布，如此，同一个哈希桶中出现 8 个以上元素的概率是相当相当小的，因此选择 8 作为链表结构红黑树化的阈值。

### 6. 其他实现（一堆废话）

> Tree bin 的根通常是它的第一个节点。但是，有时（当前仅在 Iterator.remove 上），根目录可能在其他位置，但可以通过父链接（方法TreeNode.root())）进行恢复。

> 所有应用的内部方法均接受哈希值作为参数（通常由公共方法提供），允许它们相互调用而无需重新计算元素的哈希值。 大多数内部方法还接受“ tab”参数，该参数通常是当前 map，但在调整大小或转换时可以是新的或旧的 map。

> 当 bin 列表被树化，拆分或未树化时，我们将它们保持在相同的相对访问/遍历顺序（即字段 Node.next）中，以更好地保留局部性，并略微简化调用 iterator.remove 时拆分和遍历的处理。在 insert 中使用比较器时，为了保持重新平衡的总顺序（或此处要求的接近度），我们将类和identityHashCodes 作为决胜局进行比较。

> 子类 `LinkedHashMap` 的存在使普通模式与树模式之间的使用和转换变得复杂。 有关定义为在插入，删除和访问时调用的挂钩方法的信息，请参见下文，该方法允许 LinkedHashMap 内部保持独立于这些机制。 （这还要求将 map 实例传递给可能创建新节点的某些实用程序方法。）
>
> 类似并发编程的基于SSA的编码样式有助于避免在所有复杂的指针操作中产生混淆错误。



