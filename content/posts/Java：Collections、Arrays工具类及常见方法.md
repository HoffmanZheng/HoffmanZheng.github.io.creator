---
title: "Java：Collection、Map 工具类及常见方法"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-02T16:11:47+01:00
draft: false
---



本篇介绍 Java 中的集合类框架的基础知识，并对 HashMap 的源码进行剖析，具体介绍 JDK 1.8 对 HashMap 所做的改动。关于这些集合类的线程安全问题，会在我的另一篇博客 `Java：多线程下的安全容器` 中进行详细的分析。



# 一、Collection 继承体系

![Collection 继承体系](/images/Collections.png)

### 1. 集合 Collection 介绍

* 为什么需要集合？
  * 集合使 Java 可以处理多个同类型的对象（List / Set），亦或是多个键值对类型的数据（Map）。
* 集合的常用功能：
  * 添加：add(Object obj)、addAll(Collection c)；
  * 删除：clear()、remove(Object)、removeAll(Collection)；
  * 判断包含：isEmpty()、contains(Object)、containsAll(Collection)；
    * contains 方法进行判定时，会调用 `equals` 方法，所以在用集合存储引用对象（非原生数据类型）时需要重写 `equals` 和 `HashCode` 方法。
  * 遍历获取：Iterator<E> iterator()；
  * 长度：size()；
  * 交集：retainAll(Collection c)； 若有元素被移除，此操作会改变原有实例集合。
* 迭代器（Iterator）：
  * 以内部类的方式遍历集合中的元素，有以下方法：
    * hasNext()； next()； remove()；
  * 构造思路：
    * 写一个 `iterator()` 方法返回一个自己的迭代器， 创建一个自己的迭代器继承 `Iterator` 接口，重写接口的三个方法。
    * 使用时：用 `iterator()` 创建迭代器，再用迭代器去调用其中的三个方法。 

### 2. List：对付顺序的好帮手

* List 是插入有序的，元素可重复的。
* List 有个自己的迭代器 `ListIterator` ，比普通的迭代器多出几个功能：向前遍历、添加元素、设置元素等。
* List 常用的实现类有 
  * `ArrayList` ：底层数据结构是数组，线程不安全。
  * `LinkedList` ：底层数据结构是链表，线程不安全。
  * `Vector` ：底层数据结构是数组，线程安全。

### 3. Set：注重独一无二的性质

* Set 是插入无序的，元素不可重复的。
* Set 常用实现类有 
  * `HashSet` ：底层数据结构是哈希表（是一个元素为链表的数组），顺序完全随机。
  * `TreeSet` ：底层数据结构是哈希表(是一个元素为链表的数组)，保证元素的插入排序不变。
  * `LinkedTreeSet` ：底层数据结构由哈希表和链表组成，可以用来排序。

### 4. Map：用Key来搜索的专家（映射）

* 如果数据是键值对类型的，就需要选择 `HashMap` 进行存储。如果要保持插入顺序，可以选择 `LinkedHashMap` ，如果需要排序则选择 `TreeMap` 。
* Key 不能重复（是唯一的），每个 Key 最多只能映射到一个值，但多个 Key 可以引用相同的对象。
* Map 的常用方法有：

```java
containsKey(); containsValue();
get(Key); put(K, V);
keySet();  //返回所有的 key
```

* HashMap 的 key 的 set 就是一个 HashSet，HashSet 有的功能 HashMap 都有，所有 HashSet 里面包了一个 HashMap。
* HashMap 的线程不安全性：HashMap的实现 没有被同步 ---> HashMap 的死循环（搜这个）。
  *  多线程环境下 扩容的时候 有可能会变成一个死循环的链表
  *  多线程环境下  使用 ConcurrentHashMap
* HashMap 在 JDK 7 下的改变：链表 ---> 红黑树（HashSet）
  * 不同的对象有相同的 HashCode ---> 桶分布不均匀，性能下降。变成了链表
  * JDK 7 之后，在处理 哈希桶的碰撞（多个元素 `HashCode` 相同，存储在同一个哈希桶中）时，会将链表 变成了红黑树的结构。

### 5. Guava

* 是 `Google` 写的集合框架，在原生 JDK Collection 的基础上增添了更多有用的实现，如：`MultiSet` 可以存储相同的元素，并告诉你他们被存了多少次；`BidirectionMap` 可以实现从 `value` 映射回 `Key` 值。 
* 思想：不要重复发明轮子，尽量使用经过实战检验的类库。



# 二、剖析集合框架

### 1. ArrayList 与 LinkedList 区别?
* 补充内容:RandomAccess接口
* 补充内容:双向链表和双向循环链表
* ArrayList 与 Vector 区别呢?为什么要用 ArrayList 取代 Vector 呢？
  * `Vector` 类的所有方法都是同步的。可以由两个线程安全地访问一个 Vector 对象、但是一个线程访问Vector的话代码要在同步操作上耗费大量的时间。
  * ArrayList不是同步的，所以在不需要保证线程安全时建议使用 ArrayList。
* 说一说 ArrayList 的扩容机制吧
  * 当调用 `ArrayList` 的 `add` 或者 `addAll` 方法的时候，会先进行判定现有容量是否足够，如果不够则会进行 `动态扩容` ，即新建一个更大容量（一般为之前容量的 1.5 倍）的 ArrayList，然后把原有的数据拷贝进去，最后赋值给之前的引用对象。

### 2. HashMap、HashTable 及 HashSet

#### 2.1 HashMap 和 Hashtable 的区别

* 线程是否安全：HashMap 是非线程安全的，`HashTable` 是线程安全的；HashTable 内部的方法基本都经过`synchronized` 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）
* 效率：因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它。
* 对 Null key 和 Null value 的支持：HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛出 `NullPointerException`。
* 初始容量大小和每次扩容大小的不同：
  * 创建时如果不指定容量初始值，Hashtable 默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。
  * 创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小。
* 底层数据结构：JDK 1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

#### 2.2 HashMap 和 HashSet区别

如果你看过 `HashSet` 源码的话就应该知道：HashSet 底层就是基于 HashMap 实现的。（HashSet 的源码非常非常少，因为除了 `clone() `、`writeObject()`、`readObject()`是 HashSet 自己不得不实现之外，其他方法都是直接调用 HashMap 中的方法。

|              HashMap               |                           HashSet                            |
| :--------------------------------: | :----------------------------------------------------------: |
|           实现了Map接口            |                         实现Set接口                          |
|             存储键值对             |                          仅存储对象                          |
|   调用 `put（）`向map中添加元素    |              调用 `add（）`方法向Set中添加元素               |
| HashMap 使用键（Key）计算 HashCode | HashSet 使用成员对象来计算HashCode 值，对于两个对象来说 HashCode 可能相同，所以 equals() 方法用来判断对象的相等性， |

#### 2.3 HashSet 如何检查重复

当你把对象加入`HashSet`时，HashSet会先计算对象的 `HashCode` 值来判断对象加入的位置，同时也会与其他加入的对象的 HashCode 值作比较，如果没有相符的 HashCode，HashSet会假设对象没有重复出现。但是如果发现有相同 HashCode 值的对象，这时会调用`equals（）`方法来检查 HashCode 相等的对象是否真的相同。如果两者相同，HashSet 就不会让加入操作成功。

**HashCode（）与equals（）的相关规定：**

1. 如果两个对象相等，则 HashCode 一定也是相同的。
2. 两个对象相等，对两个 equals 方法返回 true。
3. 两个对象有相同的 HashCode 值，它们也不一定是相等的。
4. 综上，equals 方法被覆盖过，则 HashCode 方法也必须被覆盖，单独重写 equals 方法会让业务中使用哈希数据结构的数据失效。
5. `hashCode()` 的默认行为是对堆上的对象产生独特值。如果没有重写 hashCode()，则该 class 的两个对象无论如何都不会相等（即使这两个对象指向相同的数据）。

**==与equals的区别**

1. == 是判断两个变量或实例是不是指向同一个内存空间 equals 是判断两个变量或实例所指向的内存空间的值是不是相同
2. == 是指对内存地址进行比较 equals() 是对字符串的内容进行比较
3. == 指引用是否相同 equals() 指的是值是否相同

#### 2.4. JDK 1.8 之前 HashMap 的底层实现 

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

#### 2.5 JDK1.8之后 HashMap 的底层实现 

相比于之前的版本， JDK 1.8 之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。（TreeMap、TreeSet 以及 JDK 1.8 之后的 HashMap 底层都用到了红黑树。）

**为什么选择 8 作为阈值** 

因为理想情况下，随机哈希值遵循参数为 0.5 的泊松分布，如此在同一个哈希桶中出现 8 个以上数据的概率是极低的。

![JDK 1.8 之后的哈希表结构](/images/HashMapTreeSet.jpeg)

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

#### 2.6 HashMap 灵魂拷问

**HashMap 的长度为什么是2的幂次方**

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

**负载系数为什么是 0.75**

默认的负载系数 0.75 在时间和空间成本之间提供了一个很好的权衡。 较高的值会减少空间开销，但会增加查找成本（比如在 get / put 操作时）。设置其初始容量时，应考虑预期的哈希桶的数量及其负载因子，以最大程度地减少 `rehash` （扩容中重新计算哈希值） 操作的数量。 如果初始容量大于 最大数据量 / 负载因子，则将不会发生任何 rehash 操作。

**HashMap CVE 安全隐患**

`Tomcat` 在2011年邮件中报道了哈希表可以通过精心构造的恶意 http 请求造成链表性能退化，并引发网站 DoS 拒绝服务攻击的现象。这个问题在 JDK 7 中得到了解决：对传进来的字符串进行特殊的哈希运算 `stringHash32` ，来避免恶意的字符串传值造成的哈希表链表性能退化的情况。

```java
final int hash(Object k) {
	if (0 != h && k instanceof String) {
		return sun.misc.Hashing.stringHash32((String) k);
	}
    ...
}
```

**HashMap resize 性能问题**

哈希表在扩容 `resize` 时是效率非常低的，如果业务需要频繁往哈希表中插入数据，那就在创建哈希表的时候就指定一个容量。避免未来扩容带来的性能问题，以空间换时间。

**HashMap 多线程操作导致死循环问题**

主要原因在于并发下 HashMap 扩容时，往新的 HashMap 转移数据可能产生循环链表（此处链表插入是往前插入的，导致和原哈希桶中的位置相反，产生环形链表），导致之后在该哈希桶中搜索键值时可能会发生的死锁现象。[详情查看：HashMap 的死循环](https://coolshell.cn/articles/9606.html)

不过，JDK 1.8 后解决了这个问题（使用了保持顺序的扩容 transfer 操作），但是还是不建议在多线程下使用 HashMap，因为多线程下使用 HashMap 还是会存在其他问题比如数据丢失。并发环境下推荐使用 ConcurrentHashMap 。



### 3. comparable 和 Comparator的区别

* comparable接口实际上是出自 `java.lang` 包 它有一个 `compareTo(Object obj)` 方法用来排序
  * 想要对引用类型对象进行排序的话，会让他们的类实现 `comparable` 接口。
* comparator接口实际上是出自 `java.util` 包它有一个 `compare(Object obj1, Object obj2)` 方法用来排序
  * comparator 接口一般用于原生数据类型，其本身就可以比较，但是想要重写 `compare` 方法的情况。

#### 3.1 Comparator定制排序

```java
Collections.sort(arrayList, new Comparator<Integer>() {
	@Override
	public int compare(Integer o1, Integer o2) {
		return o2.compareTo(o1);    // 从小到大排序，变成了从大到小。
        }
});               // 用匿名内部类 重写比较器，定制原生数据类型的比较方法。
```

#### 3.2 重写 compareTo 方法实现按年龄来排序

```java
public  class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

   // TODO重写compareTo方法实现按年龄来排序
    @Override
    public int compareTo(Person o) {
        // TODO Auto-generated method stub
        if (this.age > o.getAge()) {
            return 1;
        } else if (this.age < o.getAge()) {
            return -1;
        }
        return age;
    }
}                     // 对于非原生数据类型，需要其实现 comparable 接口，来进行排序。
```

### 4. 如何选用集合?

* 主要根据集合的特点来选用，比如我们需要根据键值获取到元素值时就选用 Map 接口下的集合，需要排序时选择 TreeMap，不需要排序时就选择HashMap,需要保证线程安全就选用 ConcurrentHashMap。
* 当我们只需要存放元素值时，就选择实现Collection接口的集合，需要保证元素唯一时选择实现 Set 接口的集合比如 TreeSet 或 HashSet，不需要就选择实现 List 接口的比如 ArrayList 或 LinkedList，然后再根据实现这些接口的集合的特点来选用。

