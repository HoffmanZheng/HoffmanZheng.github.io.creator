---
title: "Java：Collection、Map 集合工具类"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-02T16:11:47+01:00
draft: false
---

本篇介绍 Java 中集合类框架的基础知识，包括 List，Set，Map。此外，关于使用频率最高的容器 HashMap 则会在另一篇博文 [Java：HashMap 源码解读](http://chenghao.monster/2020/java-hashmap/) 中详细分析；关于这些集合类的线程安全问题，会在 [Java：多线程下的安全容器](http://chenghao.monster/2020/java-thread-safe-collection/) 中进行讲述。



# Collection 继承体系

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



# 集合类灵魂拷问

### 1. ArrayList、Vector 及 LinkedList

#### 1.1 ArrayList 与 LinkedList 区别

* 是否线程安全：两者都是不同步的，都不保证线程安全。

* 底层数据结构：ArrayList 底层使用的是 Object 数组；LinkedList 底层使用的是 双向链表数据结构（JDK 1.6之前为循环链表，JDK 1.7 取消了循环）。
* 插入和删除是否受元素位置的影响： 
  * ArrayList 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。 比如：执行 `add(E e)` 方法的时候， ArrayList 会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是O(1)。但是如果要在指定位置 i 插入和删除元素的话 `add(int index, E element)` 时间复杂度就为 O(n-i)。因为在进行上述操作的时候集合中第 i 和第 i 个元素之后的 (n-i) 个元素都要执行向后位/向前移一位的复制操作。 
  * LinkedList 采用链表存储，所以对于 add(E e) 方法的插入，删除元素时间复杂度不受元素位置的影响，近似 O(1)，如果是要在指定位置 i 插入和删除元素的话 `add(int index, E element)` 时间复杂度近似为O(n) 因为需要先移动到指定位置再插入。
* 是否支持快速随机访问：LinkedList 不支持高效的随机元素访问，而 ArrayList 支持。快速随机访问就是通过元素的序号快速获取元素对象（对应于get(int index) 方法）。
* 内存空间占用： ArrayList 的空间浪费主要体现在 List 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间（因为要存放直接后继和直接前驱以及数据）。

#### 1.2 RandomAccess 接口

 `RandomAccess` 接口声明了一个约定：实现这个接口的类具有随机访问功能。

`Collections.binarySearch` 方法中，会先判断 List 是否是 RandomAccess 的实例，然后调用不同的二分查找方法。

```java
public static <T>
    int binarySearch(List<? extends Comparable<? super T>> list, T key) {
        if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
            return Collections.indexedBinarySearch(list, key);
        else
            return Collections.iteratorBinarySearch(list, key);
    }
```

ArrayList 实现了 RandomAccess 接口，而 LinkedList 没有实现。

* 即声明了 ArrayList 具有快速随机访问的功能，它的底层是数组，数组天然支持随机访问，时间复杂度为 O(1)。
* 声明了 LinkedList 不具有随机访问的功能，它的底层是链表。链表需要遍历到特定位置才能访问特定位置的元素，时间复杂度为 O(n)。
* 实现了 RandomAccess 接口的 List，优先选择普通 for 循环 遍历，其次 foreach。
* 未实现 RandomAccess 接口的 List，优先选择 iterator 遍历（foreach遍历底层也是通过iterator实现的）大 size 的数据，**千万不要使用普通 for 循环**。

#### 1.3 双向链表和双向循环链表

* 双向链表： 包含两个指针，一个prev指向前一个节点，一个next指向后一个节点。

![](/images/双向链表.png)

* 双向循环链表： 最后一个节点的 next 指向head，而 head 的prev指向最后一个节点，构成一个环。

![](/images/循环双向链表.jpg)

#### 1.4 ArrayList 与 Vector 的区别，为什么要用 ArrayList 取代 Vector 

* `Vector` 类通过将所有的方法声明 synchronized 来保证数据的线程安全。但同步的确是整个 vector 实例，导致一个线程在访问实例时，其他线程都需要等待的尴尬情形，较大的同步开销完全不符合并发设计的初衷。此外，多线程下对 verctor 的复合操作（如遍历+修改）仍会导致 **并发修改异常**。
* ArrayList 不是同步的，所以在不需要保证线程安全时建议使用 ArrayList。

#### 1.5 ArrayList 的扩容机制

* 当调用 ArrayList 的 add 或者 addAll 方法的时候，会先进行判定现有容量是否足够 `ensureCapacity`，如果不够则会进行 `动态扩容` ，即新建一个更大容量（一般为之前容量的 1.5 倍）的 ArrayList，然后把原有的数据拷贝进去，最后赋值给之前的引用对象。
* 如果要在指定位置 i 插入和删除元素的话 `add(int index, E element)`，会调用 `System.arraycopy` 来实现 i 位置及之后元素的移动；而在扩容时则调用了 `Arrays.copyOf` 方法

```java
public static native void arraycopy(Object src, int srcPos,
                           Object dest, int destPos, int length);

public static int[] copyOf(int[] original, int newLength) {
        int[] copy = new int[newLength];
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

### 2. HashMap、HashTable 及 HashSet

#### 2.1 HashMap 和 Hashtable 的区别

* 线程是否安全：HashMap 是非线程安全的，`HashTable` 是线程安全的；HashTable 内部的方法基本都经过 `synchronized` 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）
* 效率：因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它。
* 对 Null key 和 Null value 的支持：HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛出 `NullPointerException`。
* 初始容量大小和每次扩容大小的不同：
  * 创建时如果不指定容量初始值，Hashtable 默认的初始大小为11，之后每次扩充，容量变为原来的 2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。
  * 创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小。
* 底层数据结构：JDK 1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

#### 2.2 HashMap 和 HashSet区别

如果你看过 `HashSet` 源码的话就应该知道：HashSet 底层就是基于 HashMap 实现的。（HashSet 的源码非常非常少，因为除了 `clone() `、`writeObject()`、`readObject()`是 HashSet 自己不得不实现之外，其他方法都是直接调用 HashMap 中的方法。

|              HashMap               |                           HashSet                            |
| :--------------------------------: | :----------------------------------------------------------: |
|          实现了 Map 接口           |                        实现 Set 接口                         |
|             存储键值对             |                          仅存储对象                          |
|    调用 `put()`向map中添加元素     |               调用 `add()`方法向Set中添加元素                |
| HashMap 使用键（Key）计算 HashCode | HashSet 使用成员对象来计算HashCode 值，对于两个对象来说 HashCode 可能相同，所以 equals() 方法用来判断对象的相等性， |

#### 2.3 HashSet 如何检查重复

当你把对象加入 `HashSet` 时，HashSet 会先计算对象的 `HashCode` 值来判断对象加入的位置，同时也会与其他加入的对象的 HashCode 值作比较，如果没有相符的 HashCode，HashSet 会假设对象没有重复出现。但是如果发现有相同 HashCode 值的对象，这时会调用`equals()`方法来检查 HashCode 相等的对象是否真的相同。如果两者相同，HashSet 就不会让加入操作成功。

* hashCode() 约定

> 1. 假如没有 equals 的比较信息被修改，无论何时在 Java 程序执行期间，hashCode 方法在同一个对象上的多次调用都必须始终返回相同的整数。~~这个整数在一个应用程序的一次执行到另一次执行不必保持一致。~~
> 2. 如果两个对象相等（根据 equals 方法），那么在每个对象上调用 hashCode 方法都必须产生相同的整数结果。
> 3. 如果两个对象不相等， hashCode 方法一定要产生不同的整数结果，则是没有被要求的。然而编程者需要知晓，不相等的对象产生不同的哈希值可能会提高哈希表的性能表现。

* equals() 约定

> 反射性、对称性、传递性、一致性
>
> 对于一个非空引用，`x.equals(null)` 应当返回 false

* 综上，如果 equals 方法被覆盖过，则 HashCode 方法也必须被覆盖，单独重写 equals 方法会让业务中使用哈希数据结构的数据失效。

* `hashCode()` 的默认行为是对堆上的对象产生独特值。如果没有重写 hashCode()，则该 class 的两个对象无论如何都不会相等（即使这两个对象指向相同的数据）。

**==与equals的区别**

1. == 是判断两个变量或实例是不是指向同一个内存空间 equals 是判断两个变量或实例所指向的内存空间的值是不是相同
2. == 是指对内存地址进行比较 equals() 是对字符串的内容进行比较
3. == 指引用是否相同 equals() 指的是值是否相同

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

   // 重写 compareTo 方法实现按年龄来排序
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

* 主要根据集合的特点来选用，比如我们需要根据键值获取到元素值时就选用 Map 接口下的集合，需要排序时选择 TreeMap，不需要排序时就选择 HashMap，需要保证线程安全就选用 ConcurrentHashMap。
* 当我们只需要存放元素值时，就选择实现 Collection 接口的集合，需要保证元素唯一时选择实现 Set 接口的集合比如 TreeSet 或 HashSet，不需要就选择实现 List 接口的比如 ArrayList 或 LinkedList，然后再根据实现这些接口的集合的特点来选用。 





