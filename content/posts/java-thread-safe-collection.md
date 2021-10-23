---
title: "Java：多线程下的安全容器"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-16T13:19:47+01:00
draft: false
---

在我之前的博客 [Java：初识多线程、原理及实现](https://hoffmanzheng.github.io/2020/java-multi-thread/) 提及，多线程下对数据的非原子性操作会造成数据错误，为此 JDK 也提供了一些线程安全的容器。本篇主要介绍 List 类的安全容器：Vector 和 CopyOnWriteArrayList，以及 Map 类的 HashTable 和 ConcurrentHashMap。从他们之前的区别，后者对前者的改进，具体的实现原理这几个角度进行分析。

# List 类的安全容器

### 1. Vector 与 SynchronizedList

#### 1.1 Vector

`Vector` 类通过将所有的方法声明 synchronized 来保证数据的线程安全。但同步的确是整个 vector 实例，导致一个线程在访问实例时，其他线程都需要等待的尴尬情形，较大的同步开销完全不符合并发设计的初衷。此外，多线程下对 Verctor 的复合操作（如遍历+修改）仍会导致 **并发修改异常**。

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

`Hashtable` ：将 get / put 所有相关操作都 synchronized 化，这相当于给整个哈希表加了一把 **大锁**，多线程访问时候，只要有一个线程访问或操作该对象，那其他线程只能阻塞，相当于将所有的操作 **串行化**，在竞争激烈的并发场景中性能就会非常差。

HashTable 与 Vector 一样，由于较大的同步开销和复合操作的并发异常 **已被弃用**，如今以 HashMap 和 ConcurrentHashMap 来代替其使用。

![HashTable 的全表锁](/images/HashTableLock.png)

### 2. JDK 1.7 的 ConcurrentHashMap

`ConcurrentHashMap`：使用继承了 `ReentrantLock` 的 segments 数组代替整个桶数据，segment 实例包含一个 `HashEntry<K, V>[]` table，即 segment 就是一个小的 HashMap，并且自带了一把锁。每个 segment 守护着若干个 HashEntry 键值对链表，当对这些 HashEntry 链表数据进行修改时，必须首先获得对应的 Segment 实例的锁。每个 segment **只锁容器其中一部分数据**，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 

```java
final Segment<K,V>[] segments;
static final int DEFAULT_CONCURRENCY_LEVEL = 16;  // 并发级别
static final class Segment<K,V> extends ReentrantLock implements Serializable ｛
    transient volatile HashEntry<K,V>[] table;
｝
```

![JDK 1.8 之前 ConrrentHashMap 的分段锁](/images/HashMapSegmentsLock.png)

看下源码中的构造器可以得知：

1. ConcurrentHashMap 根据初始容量 initialCapacity 和并发级别 `concurrencyLevel` 来决定需要创建的 segments 数组的数量，以及每个 segment 中键值对的桶数组大小。（这两个数都为 2 的次方数）
2. 新建 segments 数组及一个 segment 实例，并将实例放到数组的第 0 个位置。

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {  // 根据并发级别决定segment数组大小
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;  // 根据c算出每个segment里有几个桶数组
        if (c * ssize < initialCapacity)
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;  
        // create segments and segments[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
```

观察其 put 方法：

1. put 时会先计算 key 的哈希值所在的 segments 数组下标
2. 判断所要 put 的 segment 是否为空，通过 `s = ensureSegment(j)` 构造 segment 实例对象。在该方法内，会先拿到构造器中创建的 segments 数组中第 0 个segment 作为模版 prototype，拿到模版 segment 中的桶数组大小、装载因子信息来创建当前需要的 segment 实例。
3. 调用 segment 中的 put 方法：
   * 使用 `tryLock()` 尝试获取锁（segment实例），获取失败后会通过 `scanAndLockForPut(key, hash, value)` 用循环的方式反复请求这把锁。
   * 算得 segment 中哈希桶的数组下标，拿到桶内第一个键值对 first，接下来的 put 操作和在 HashMap 中的类似，不再赘述。
   * `modCount` 用来保证 fail-fast 机制。

```java
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)  // value值不允许为null
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;  // segments 数组下标
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }

// segment 中的 put 方法
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value); // 加锁
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;  // segment中哈希桶的数组下标
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();  // 解锁
            }
            return oldValue;
        }
```

### 3. JDK 1.8 的 ConcurrentHashMap

到了 JDK 1.8 的时候已经摒弃了分段锁，而是使用 synchronized 桶数组头节点 Node 和 CAS 来实现并发控制。（ JDK 1.6 以后 对 synchronized 锁做了很多优化， 整个看起来就像是优化过且线程安全的 HashMap，虽然在 JDK 1.8 中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本）

![JDK 1.8 后的并发哈希表的锁](/images/HashMapLock.jpeg)

#### 3.1 ConcurrentHashMap 源码文档解读

源码的注释文档主要说了这么几点：

1. 支持高并发的检索和更新，即使所有的操作都是线程安全，但检索时是不加锁的，也不支持锁定整个表。
2. 检索 / get 操作不会阻塞，检索会反映最近一次完成更新时的结果（即在当前时间点的值）。虽然不会抛出并发修改异常，迭代器却是被设计为在单线程下使用的。
3. 一些查询状态的方法如 size、isEmpty、containsValue 在多线程下只反映表的临时状态，结果可用于监控状态，而不能用于条件判定（高并发下统计数据如 size 等其实是没有意义的，因为它在下一时刻就被改变了）。
4. 支持批量操作。
5. ConcurrentHashMap 的 key 和 Value 都不能为 null。

#### 3.2 ConcurrentHashMap 线程安全实现原理

相比 Java 1.7 的 ConcurrentHashMap 使用 "锁分段" 来保证数据的线程安全，Java 1.8 的ConcurrentHashMap 通过 **同步指定哈希桶的第一个节点** 来实现这个功能。

* 若头节点不存在则会调用 `casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null))` 以线程安全的方式创建一个新的头节点。
* `volatile Node<K, V> next;` 用 volatile 修饰节点来保证每次获取的都是最新设置的值。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 获取桶数组下标
                if (casTabAt(tab, i, null,  // 如果桶为空，执行cas方法来new一个新节点，同时保证并发数据安全
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED) // 当前表正在转移数据，用当前线程帮忙转移
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {  // 以第一个节点作为锁
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) { // 遍历链表找key更新value
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) { // 遍历到链表尾部，插入一个新节点
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { // 红黑树的put
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD) // 判断是否树化
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

