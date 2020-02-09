---
title: "Java：多线程下的安全容器"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-16T13:19:47+01:00
draft: false
---

在我之前的博客 [Java：初识多线程、原理及实现](http://chenghao.monster/2020/java-multi-thread/) 提及，多线程下对数据的非原子性操作会造成数据错误，为此 JDK 也提供了一些线程安全的容器。本篇主要介绍 List 类的安全容器：Vector 和 CopyOnWriteArrayList，以及 Map 类的 HashTable 和 ConcurrentHashMap。从他们之前的区别，后者对前者的改进，具体的实现原理这几个角度进行分析。

# List 类的安全容器

### 1. Vector 与 SynchronizedList

#### 1.1 Vector

`Vector` 类通过将所有的方法声明 synchronized 来保证数据的线程安全。但同步的确是整个 vector 实例，导致一个线程在访问实例时，其他线程都需要等待的尴尬情形，较大的同步开销完全不符合并发设计的初衷。此外，多线程下对 verctor 的复合操作（如遍历+修改）仍会导致 **并发修改异常**。

```java
public synchronized boolean add(E e);
public synchronized boolean isEmpty();
public synchronized E get(int index);
public synchronized void sort(Comparator<? super E> c);
```

#### 1.2 SynchronizedList

`SynchronizedList` 类和 vector 类似，只不过是在方法内进行同步锁：

```java
public E get(int index) {
    synchronized (mutex) {return list.get(index);}}
public E set(int index, E element) {
    synchronized (mutex) {return list.set(index, element);}}
public void add(int index, E element) {
    synchronized (mutex) {list.add(index, element);}}
public E remove(int index) {
    synchronized (mutex) {return list.remove(index);}}
```

#### 1.3 Vector 和 SynchronizedList 可能会出现的问题

虽然 Vector 和 SynchronizedList 的单个方法都是线程安全的，但是对其的复合操作仍然需要用户手动地同步。SynchronizedList 在使用迭代器遍历的时候也会有同样的问题，源码已经提醒我们要 **手动加锁** 了。

```java
public ListIterator<E> listIterator() {
            return list.listIterator(); // Must be manually synched by user
        }

        public ListIterator<E> listIterator(int index) {
            return list.listIterator(index); // Must be manually synched by user
        }
```

### 2. CopyOnWriteArrayList

总的来说，JUC 下支持并发的容器与老一代的线程安全类相比，总结起来就是加锁粒度的问题。

#### 2.1 CopyOnWriteArrayList 实现原理

首先引用维基上对 Copy-on-Write 机制的解释：

> Copy-on-Write 机制：如果有多个调用者（callers）同时请求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正 **复制一份专用副本（private copy）给该调用者**，而其他调用者所见到的最初的资源仍然保持不变。优点是如果调用者没有修改该资源，就不会有副本（private copy）被建立，因此多个调用者只是读取操作时可以共享同一份资源。

再看下源码文档的解释：

> CopyOnWriteArrayList 是 ArrayList 线程安全的一种变体，底层的所有修改操作通过复制数组的方式来实现。这通常会过于昂贵，但在遍历操作远大于修改操作时更加高效；并且无需在并发遍历时进行同步。Iterator 通过引用在迭代器被创建时的“快照”形式的列表，保证列表在迭代周期不发生改变来排除干扰，保证不抛出并发修改异常。在迭代器上的修改操作将不被支持，它们会抛出 UnsupportedOperationException。

#### 2.2 CopyOnWriteArrayList 的结构与方法

看源码可知 CopyOnWriteArrayList 底层是数据，加锁通过 `ReentrantLock` 来实现。

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();

    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;

    /**
     * Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
     */
    final Object[] getArray() {
        return array;
    }
```

相比 Vector / SynchronizedList 需要在遍历时手动加锁的情况，CopyOnWriteArrayList 在使用迭代器时不需要显式加锁，可以看下它的 mutation 方法如 add：

可以看到在进行 add 时就上锁，复制一个新数组，增加操作在新数组上完成，将 array 指向到新数组中，最后解锁。这种同步形式对于其他的修改操作 remove、clear、set 等也是类似的。

```java
public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            // 复制新数组，并在长度上加一
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            // 将实例指向新数组
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

至此，CopyOnWriteArrayList 线程安全的策略就很明显了：

* 所有的非修改操作不加锁，修改操作加锁。
* 在修改时，先复制出一个新数组，修改的操作在新数组中完成，最后将新数组交由 array 变量指向。

#### 2.3 为什么 CopyOnWriteArrayList 迭代时不需要显式加锁

通过观察 CopyOnWriteArrayList 的迭代器不难发现，它在迭代器内使用了 final 修饰的引用类型数组 snapshot，这就使所有的修改操作中将实例指向新数组的 `setArray(newElements)` 这个操作在迭代器的生命周期内无法执行，以此保证在迭代过程中不会抛出并发修改异常。

```java
public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }

static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
    }
```

#### 2.4 CopyOnWriteArrayList 的缺点

- **内存占用**：如果 CopyOnWriteArrayList 经常要增删改里面的数据，经常要执行 `add()、set()、remove()` 的话，那是比较耗费内存的。
  - 因为这些修改操作都要 **复制一个数组** 出来。
- **数据一致性**：CopyOnWrite 容器 **只能保证数据的最终一致性，不能保证数据的实时一致性**。
  - 从上面的例子也可以看出来，比如线程 A 在迭代 CopyOnWriteArrayList 容器的数据。线程B在线程A迭代的间隙中将 CopyOnWriteArrayList 部分的数据修改了（已经调用`setArray()`了）。但是线程 A 迭代出来的是原有的数据。

# Map 类的安全容器

### 1. HashTable

`Hashtable` ：将 get / put 所有相关操作都 synchronized 化，这相当于给整个哈希表加了一把**大锁**，多线程访问时候，只要有一个线程访问或操作该对象，那其他线程只能阻塞，相当于将所有的操作**串行化**，在竞争激烈的并发场景中性能就会非常差。

![HashTable 的全表锁](/images/HashTableLock.png)

### 2. JDK 1.7 的 ConcurrentHashMap

`ConcurrentHashMap`：对整个桶数组进行了分割分段 Segment，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 

一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和 HashMap 类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry 数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

![JDK 1.8 之前 ConrrentHashMap 的分段锁](/images/HashMapSegmentsLock.png)

### 3. JDK 1.8 的 ConcurrentHashMap

到了 JDK 1.8 的时候已经摒弃了分段锁，而是直接用 Node 数组 + 链表 + 红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（ JDK 1.6 以后 对 synchronized 锁做了很多优化， 整个看起来就像是优化过且线程安全的 HashMap，虽然在 JDK 1.8 中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本）

![JDK 1.8 后的并发哈希表的锁](/images/HashMapLock.jpeg)

#### 3.1 ConcurrentHashMap 源码文档解读

源码的注释文档主要说了这么几点：

1. 支持高并发的检索和更新，所有的操作都是线程安全，并且在检索时是不加锁的。
2. 未完待续

#### 3.2 ConcurrentHashMap 线程安全实现原理