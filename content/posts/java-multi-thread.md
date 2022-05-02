---
title: "Java：初识多线程、原理及实现"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-01-12T14:49:47+01:00
draft: false
---

# 多线程原理

### 1. 为什么需要多线程

#### 1.1 Java 代码的执行是同步阻塞模型

Java 代码在执行的时候，是从 `main` 代码块开始逐行执行，但当当前代码耗时过长时，下一行的代码就必须等待当前任务执行完毕，代码才能继续往下执行。

#### 1.2 CPU的运算实在太快了

取主频为 3GHz CPU 为例，CPU 执行单个指令所需的时间约为 0.3ns；与之相对的，在内存中进行读取1MB大小文件所需时间大概为  250us，SSD随机读取耗时约为 1ms，HDD约为 20ms。（推荐阅读：[我是一个CPU：这个世界慢！死！了！](https://zhuanlan.zhihu.com/p/58431253)）

相比与内存，硬盘，网络传输，CPU 的运算实在是太快了，但是由于阻塞模型，如若当前代码执行的是较为耗时的 IO 操作，线程中的下一个任务就必须等待当前任务结束才能继续执行。使得单线程的代码执行非常没有效率，另外也浪费了 CPU 的运算性能。

![CPU 这个世界慢死了](/images/CPU_too_fast.jpg)

### 2. Java 线程简介

#### 2.1 多线程的方法栈

一般的 Java 程序都是从启动类的 main 函数入口开始执行，随着 main 函数的结束而停止.  这条执行路径就是 Java 程序的主线程。Java 虚拟机允许拥有同时运行多个线程，当新线程运行时，就会在栈中生成一个新的方法栈，新线程独享单独的执行流和局部变量，而静态变量则是被所有线程共享的。

#### 2.2 多线程的性能

使用多线程就只有好处没有坏处吗？并不是，使用多线程是要付出代价的，如果没有耗时的任务，使用多线程，效率反而更低，因为CPU 在线程间的切换也需要耗费时间。引用知乎上看到的一句话：

> 多线程在 CPU 密集型的作业下的确不能提高性能甚至更浪费时间，但是在 IO 密集型的作业下则可以提升性能（或者更准确点说叫平均响应时间）。

由上得出的结论是，正确的使用多线程可以提高程序的运行效率。

### 3. 多线程问题的来源

#### 3.1 多线程下的数据安全问题

多线程同时对一个共享的全局变量进行非原子操作将会引发严重的数据安全问题。

* 原子操作可理解为不可分隔的操作，而非原子操作例如 `i++` 则可以分解为 `temp = i;  temp = temp + 1;  i = temp;` 这样的取值、运算、赋值三个独立的操作。

* 多线程下非原子操作造成数据错误的根源是：在某个线程 A 进行 `取值、运算` 时，在 `赋值` 操作前，被另一个线程 B 在中途抢进来并提前完成了同样的 `取值、运算、赋值` 过程，使得线程 A 的赋值与期望值不符的情况，即多线程下的数据错误。

#### 3.2 多线程造成的死锁

在Java中使用多线程，就会有可能导致死锁问题，[即哲学家就餐问题]([https://zh.wikipedia.org/wiki/%E5%93%B2%E5%AD%A6%E5%AE%B6%E5%B0%B1%E9%A4%90%E9%97%AE%E9%A2%98](https://zh.wikipedia.org/wiki/哲学家就餐问题))。死锁会让程序一直卡住，不再程序往下执行。我们只能通过中止并重启的方式来让程序重新执行。

* 这是我们非常不愿意看到的一种现象，我们要尽可能避免死锁的情况发生！

![死锁](/images/deadLock.png)

造成死锁的原因可以概括成三句话：

* 当前线程拥有其他线程需要的资源
* 当前线程等待其他线程已拥有的资源
* 双方都不放弃自己拥有的资源

示例代码如下：

```java
public class TestDeadLock {
    private static Object lock1 = new Object();
    private static Object lock2 = new Object();
    public static void main(String[] args) {
        new Thread1().start();
        new Thread2().start();
    }
    static class Thread1 extends Thread {
        @Override
        public void run() {
            synchronized(lock1) {
                try {
                    Thread.sleep(500);    // ms
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized(lock2) {
                    System.out.println("Proceed successfully!");
                }
            }
        }
    }
    static class Thread2 extends Thread {
        @Override
        public void run() {
            synchronized(lock2) {
                try {
                    Thread.sleep(100);    // ms
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized(lock1) {
                    System.out.println("Proceed successfully!");
                }
            }
        }
    }
}
```

进程 `Thread1` 拥有锁 `Lock1`，并请求锁 `Lock2`，而进程 `Thread2` 拥有锁 `Lock2` 并请求锁 `Lock1`，双方都不释放自己拥有的锁，此时就满足了产生死锁的四个条件 `互斥、不剥夺、请求和保持、循环等待`， 使两个线程都没办法继续执行下去，形成死锁。

#### 3.3 死锁的排查

通过 Java 自带的 `jps` 命令确定当前执行任务的进程号

* `jstack 26497 > output.txt`  执行 jstack 命令查看当前进程堆栈信息，可以看到当前所有线程的状态以及是否在等待哪个锁。



# 多线程实现

### 1. volatile 关键字与内存可见性

#### 1.1 共有变量 - 线程间互不可见

示例代码：

```java
public class TestVolatile {
    public static void main(String[] args) {
        VolatileDemo volatileDemo = new VolatileDemo();
        new Thread(volatileDemo).start();

        while (true) {
            if (volatileDemo.isFlag()) {
                System.out.println("-------------------------");
                break;
            }
        }
    }

    static class VolatileDemo implements Runnable {
        private boolean flag = false;

        public boolean isFlag() {
            return flag;
        }

        public void setFlag(boolean flag) {
            this.flag = flag;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            setFlag(true);
            System.out.println("flag= " + isFlag());
        }
    }
}
```

JVM会给每个线程分配一个独立的缓存用来提高效率

* 新的线程 `flag`，会先从主存中读取私有实例变量 `flag = false` 到该线程的缓存中，然后进行改值操作，然后把改好的值写回主存中。

* 但 main 线程此时读取了原先的主存中的私有实例变量 `flag = false` 到自己线程的缓存中，此后的 `if` 判断都通过判断自己线程缓存中的 `flag` 变量值来进行，导致 `while` 循环无法终止。
* 内存可见性问题是：当多个线程操作共享数据时，彼此不可见。

#### 1.2 解决方法：

* `synchronized` 同步：效率低，新的线程来取值前先判断当前锁当前是否被占有，被占有的话就造成了阻塞。阻塞后线程被挂起，等待下一次 CPU 再次分配任务。
* `volatile` 关键字：当多个线程进行操作共享数据时，可以保证内存中的数据可见（相较于 synchronized 是一种较为轻量级的同步策略）。
  * 实现：调用了计算机底层的内存栅栏，把缓存中的变量及时刷新到主存中 （可以理解中读写都在主存中完成）效率也会降低一些，但是比同步的效率高。
  * 注意：
    * `volatile` 不具备 "互斥性" ，`synchronized` 的互斥性：一个线程拿到锁之后，其他的线程就拿不到了，就是 互斥性。
    * `volatile` 不能保证变量的 "原子性"。

### 2. 原子变量与 CAS 算法

JDK 5 之后在 `java.util.concurrent.atomic` 包中引入了原子变量工具集，它支持在单个变量上进行无锁、线程安全的编程，同时使在当代处理器上可用的高效机器级原子命令的实现成为可能。 `AtomicInteger`、`AtomicReference`、`AtomicBoolean`、`AtomicLong` 等类的实例各自提供对相应类型的单个变量的访问和更新的方法。

* 原子变量如何保证线程安全
  * 原子变量首先都是 `volatile` 修饰的，保证内存可见性
  * CAS 算法保证数据的原子性 （Compare and Swap）

* `CAS ` 算法是硬件对于并发操作共享数据的支持，由底层 硬件 / JVM 提供
  * CAS包含了三个操作数：内存值 V，预估值 A，更新值 B，在赋值操作前再次从主存中读取变量值为预估值 A，当且仅当 V 的值等于 A 时，CAS 通过原子方式用新值 B 来更新 V 的值，否则将不作任何操作。
  * CAS 效率比锁高？ CAS 判据否时，不会造成线程阻塞，线程会立即再去更新变量值进行赋值更新操作。

### 3. 同步容器类 ConcurrentHashMap

JDK 5 在 JUC 包中提供了多种并发容器来改进同步容器的性能。

* 新增的 ConcurrentHashMap 采用“锁分段”机制代替了 HashTable 的独占锁，进而提高性能。
  * `HashTable` 是线程安全的，它用锁锁住了整个哈希表，导致同时只能有一个线程访问哈希表（并行变成了串行），效率低。
  * 同时 HashTable 的复合操作存在风险，如 “若不存在则添加”，“若存在则删除”，因为 contains 和 put 操作各自拥有一把锁，在这复合操作的间隙可能会被别的线程抢先执行判断添加操作，从而引发多线程下的数据安全问题。
* `ConcurrentHashMap` 采用了 “锁分段” 机制（JDK 5），它有一个并发级别 concurrentLevel 默认为 16 段 Segments，每个段拥有独立的锁守护哈希表的若干个桶 HashBucket ------> 每个桶是键值对连接起来的链表结构，从而实现高效率的并行，并保证了安全性。
  ![ConcurrentHashMap 的分段锁](/images/ConcurrentHashMap.png)
* JDK 8 抛弃了 Segment 分段锁机制，采用 CAS（不会阻塞，更加高效） + Synchronized 来保证并发更新的安全，底层采用数组+链表+红黑树的存储结构。
  * 除此，想用线程安全的 Map ：`Collections.SynchronizedMap` ------> 把集合类中的每个方法都加了 synchronized 关键字，在调用单个方法时是线程安全的，但是在复合操作时（如迭代时修改）仍是不安全的，需要额外的同步。
  
* `CopyOnWriteArrayList`：解决了 `Collections.synchronizedArrayList` 在多线程迭代时的并发修改异常问题。
  * CopyOnWriteArrayList：当你每次写入时，在底层完成新列表的复制，再进行写入。这样新列表和别的互相都不干涉；
  * 添加操作多时，效率低，因为每次添加时都会进行复制，开销非常大。
  
  * 每次复制效率较低，所以适合读数和遍历的操作数远远大于列表的更新数的并发情况。

示例代码：

```java
public class TestCopyOnWriteArrayList {
    public static void main(String[] arge) {
        HelloThread ht = new HelloThread();
        for (int i = 0; i < 10; i++) {
            new Thread(ht).start();
        }
    }
}
class HelloThread implements Runnable {
    //private static List<String> list = Collections.synchronizedList(new ArrayList<>());
    private static CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
    static {
        list.add("AAA");
        list.add("BBB");
        list.add("CCC");
    }

    @Override
    public void run() {
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            System.out.println(it.next());
            list.add("DDD");
        }
    }
}
```

### 4. 创建执行线程的第三种方式：Callable

创建执行线程的方式有四种：继承 Thread，Runnable，Callable，线程池。

* 相比 Runnable，Callable 可以返回值，抛出异常。

* 实现：

  * 创建 Callable 的实现类，重写 call 方法
  * 将实现类的实例作为参数创建 FutureTask
  * 再将 FutureTask 的实例作为参数创建新线程开始执行

  * `futureTask.get()` 接收返回值，该操作会等着上述线程执行完毕后再执行 ------> 相当于闭锁。

示例代码：

```java
public class TestCallable {
    public static void main(String[] args) {
        CallableDemo callableDemo = new CallableDemo();
        FutureTask<Integer> futureTask = new FutureTask<>(callableDemo);

        new Thread(futureTask).start();
        try {
            System.out.println(futureTask.get());    // 相当于闭锁
            System.out.println("--------------------------");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
    
    static class CallableDemo implements Callable<Integer> {
        @Override
        public Integer call() throws Exception {
            int sum = 0;
            for (int i = 0; i < Integer.MAX_VALUE; i++) {
                sum += i;
            }
            return sum;
        }
    }
}
```

### 5. 闭锁 CountDownLatch

CountDownLatch 是一个同步辅助类

* 在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。（当前线程的运算，只有在其他所有线程运算完成后，才能继续执行）
  * 确保某些活动直到其他活动都完成才继续执行

* 应用：统计多个文件 ------> 等前面所有的线程都运算完了再在主线程中汇总结果。
* 实现：
  * 创建一个倒计时为 10 的闭锁 `final CountDownLatch latch = new CountDownLatch(10)`
  * 在其他线程执行完毕后将倒计时减一
  * 等待，直到倒计时为零时向下继续执行代码。

示例代码：

```java
public class TestCountDownLatch {
    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(10);
        LatchDemo ld = new LatchDemo(latch);

        long start = System.currentTimeMillis();
        for (int i = 0; i < 10; i++) {
            new Thread(ld).start();
        }
        try {
            latch.await();     // 等待倒计时到零
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long end = System.currentTimeMillis();
        System.out.println("耗费时间为：" + (end - start));
    }

    static class LatchDemo implements Runnable {
        CountDownLatch latch;

        public LatchDemo(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void run() {
            synchronized(this) {
                try {
                    for (int i = 0; i < 50000; i++) {
                        if (i % 2 == 0) {
                            System.out.println(i);
                        }
                    }
                } finally {
                    latch.countDown();    // 倒计时减一
                }
            }
        }
    }
}
```

### 6. 同步锁

用于解决多线程安全问题的方式：

* 同步代码块（synchronized 隐式锁） 
* 同步方法 
* JDK 5 之后，同步锁（显式锁）需要通过 `lock()` 上锁，`unlock()` 释放锁 （放到 finally 中以免忘记）

特点：

* 使用可重入锁 `ReentrantLock` 实现手动上锁及释放锁的步骤。
* 灵活：比如它可以在一个方法内加锁，到另一个方法再解锁释放。
* 风险：必须要释放锁，不然存在极大的风险。

示例代码：

```java
static class Ticket implements Runnable{
        private static int tick = 100;
        private static Lock lock = new ReentrantLock();

        @Override
        public void run() {
            while (true) {
                lock.lock();
                try {
                    if (tick > 0) {
                        try {
                            Thread.sleep(200);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println(Thread.currentThread().getName() + 
                                           "完成售票，余票为：" + --tick);
                    }
                } finally {
                    lock.unlock();
                }
           }
     }
}
```

### 7. 生产者消费者案例及虚假唤醒

#### 7.1 生产者消费者案例可能产生什么问题？

添加和创建数据的线程就是生产者，删除和销毁数据的线程就是消费者，如果生产者线程过快，就会导致消费者来不及接收，产生数据丢失问题；如果消费者线程过快，就可能产生重复 / 错误的数据等。

* 需要等待唤醒机制

注意：

* `wait()` 所在的条件判据应该总在循环 `while` 中，避免虚假唤醒的发生。（常发生于多个生产者消费者的场景）
* wait、notify 机制需要和 `synchronized` 搭配使用，来保证 while 条件检查与 wait，以及 while 条件更新与notify 是互斥的。

示例代码：

```java
public class ProducerConsumer {
    public static void main(String[] args) throws InterruptedException {
        Container container = new Container();
        Object lock = new Object();
        Producer producer = new Producer(container, lock);
        Consumer consumer = new Consumer(container, lock);

        producer.start();
        consumer.start();

        producer.join();
        producer.join();
    }

    public static class Producer extends Thread {
        private Container container;
        private Object lock;

        public Producer(Container container, Object lock) {
            this.container = container;
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock) {
                    for (int i = 0; i < 10; i++) {
                        while (container.getProductNum() >= 1) {
                            try {
                                lock.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        container.produceANewProduct();
                        System.out.println("Producing one product: current product number is " + container.getProductNum());
                        lock.notifyAll();
                    }
                }
            }
        }
        
    public static class Consumer extends Thread {
        private Container container;
        private Object lock;

        public Consumer(Container container, Object lock) {
            this.container = container;
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 10; i++) {
                        while (container.getProductNum() <= 0) {
                            try {
                                lock.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                        container.consumeAProduct();
                        System.out.println("Consuming one product : current product number is " + container.getProductNum());
                        lock.notifyAll();
                    }
                }
            }
        }
        
    static class Container {
        private int productNum = 0;
        
        public int getProductNum() {
            return productNum;
        }

        public void produceANewProduct() {
            this.productNum += 1;
        }

        public void consumeAProduct() {
            this.productNum -= 1;
        }
    }
}
```

### 8. Condition 线程通信

* Condition 接口描述了可能会与锁有关联的条件变量。这些变量在用法上与使用 Object.wait 访问的隐式监视器类似，但提供了更强大的功能。需要指出的是，**单个 Lock 可能与多个 Condition 对象关联**。为了避免兼容性问题，Condition 方法的名称与对应的 Object 版本中的不同。
* Condition 实例实质上被绑定到一个锁上。要为特定 Lock 实例获得 Condition 实例，请使用其 `new Condition()` 方法。在 Condition 对象中，与 wait、notify、notifyAll 方法对应的分别是 await、signal 和 signalAll。
* Condition 需要和同步锁一起搭配使用，**不可与同步代码块 / 同步方法 一起使用**。

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

* 使用可重入锁 + Condition 线程通信可以实现线程的按序交替。

### 9. ReadWriteLock 读写锁

也是一种乐观锁，可以多个线程同时读，但只能有一个线程在某个时刻进行写的操作。写写 / 读写的线程需要互斥，一次只能有一个；读读 不需要互斥。

读写锁拥有一对相关的锁：读锁和写锁。读锁可以同时被多个线程持有（只要没有 writer），而写锁只能被一个线程拥有 —— 相比独占锁提高了效率。

```java
ReadWriteLock lock = new ReentrantReadWriteLock();
lock.readlock.lock();      // 加在读操作外边。写锁同理
lock.readlock.unlock();
```

### 10. 线程池

相比频繁新建线程，线程用完就销毁的调用方法，线程池提供了一个线程队列，队列中保存着所有等待状态的线程，这样可以避免额外的系统开销，提高响应速度。

![线程池的体系结构](/images/线程池体系结构.jpg)

> 尽管 ThreadPoolExecutor 提供了许多可调节的参数和扩展性，程序设计者被督促使用更为方便的，为大多数通用场景预制配置的 Executors 工厂方法：`Executors.newCachedThreadPool` 有自动回收功能的无限线程池，`Executors.newFixedThreadPool` 固定数量线程池，`Executors.newSingleThreadExecutor` 一个后台线程。

使用 `submit` 向线程池提交任务，使用完后 `shutdown` 关闭线程池（不关闭当前 Java 程序就无法结束），用 `Future` 接收线程池返回的结果。

需要线程延迟或定时（调度）地执行任务时，可以使用 `scheduledThreadPool.schedule(Callable, int, Unit)` 。

### 11. ForkJoinPool 分支合并框架及工作窃取

在必要的情况下，将一个大任务，进行拆分（fork）成若干个小任务（拆到不可再拆时），再将一个个小任务的运算结果进行 join 汇总。

* 工作窃取模式：可以使空闲 / 阻塞的线程从别人线程尾部偷取任务执行，避免了线程阻塞导致的后续任务的停滞，能更好的利用 CPU 的资源，效率更高。

* 使用 forkJoinPool 实例调用 `invoke`（ExcursiveTask）实现。

```java
class ForkJoinSumCalculate extends ExcursiveTask<Long> {
    private long start;
    private long end;
    public ForkJoinSumCalculate(long start, long end;) {
        this.start = strat;long end;
        this.end = end;
    }
    @Override
    protected Long compute() {
        long length = end - start;
        if (length <= THURSHOLD) {
            long sum = 0L;
            for (long i = start; i <= end; i++) {
                sun += i;
            }
            return sum;
        } else {
            long middle = (start + end) / 2;
            ForkJoinSumCalculate left = new ForkJoinSumCalculate(start, middle);
            left.fork();   // 拆分，同时压入线程队列
            ForkJoinSumCalculate right = new ForkJoinSumCalculate(middle+1, end);
            right.fork();
            return left.join() + right.join();
        }
    }
}
```





