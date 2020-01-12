---
title: "Java：Collection、Map 工具类及常见方法"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-02T16:11:47+01:00
draft: false
---



本篇介绍 Java 中的集合类框架的基础知识，其源码实现、相关面试题会在之后更新，敬请期待。



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



# 二、集合框架常见面试问题（敬请期待）

### 1. ArrayList 与 LinkedList 区别?
* 补充内容:RandomAccess接口
* 补充内容:双向链表和双向循环链表
* ArrayList 与 Vector 区别呢?为什么要用 ArrayList 取代 Vector 呢？
* 说一说 ArrayList 的扩容机制吧
  * 当调用 `ArrayList` 的 `add` 或者 `addAll` 方法的时候，会先进行判定现有容量是否足够，如果不够则会进行 `动态扩容` ，即新建一个更大容量（一般为之前容量的 1.5 倍）的 `ArrayList` ，然后把原有的数据拷贝进去，最后赋值给之前的引用对象。

### 2. HashMap 和 Hashtable 的区别
* HashMap 和 HashSet区别
* HashSet如何检查重复
* HashMap的底层实现
  * JDK1.8之前
  * JDK1.8之后
* HashMap 的长度为什么是2的幂次方
* HashMap 多线程操作导致死循环问题
* ConcurrentHashMap 和 Hashtable 的区别
* ConcurrentHashMap线程安全的具体实现方式/底层具体实现
  * JDK1.7（上面有示意图）
  * JDK1.8 （上面有示意图）

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

