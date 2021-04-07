---
title: "Java：线程池原理、源码分析"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-03-27T10:19:47+01:00
draft: false
---

本篇为 [Java：初识多线程、原理及实现](https://nervousorange.github.io/2020/java-multi-thread/) 的续篇，介绍 Java 中的线程池，主要内容有：进程与线程、线程池化的意义、Executors 工具类、线程池工作流程、线程复用原理等。

# 为什么需要线程池

### 进程与线程

#### 起源

**进程** 是在批处理操作系统的基础上，为进一步提高计算机效率，改善操作系统串行的运行方式的产物。进程的出现改变了内存中始终只有一个程序运行的事实，进程是在内存中独享一片内存空间的程序，各个进程之间互不干扰。

CPU 采用 **时间片轮转** 的方式运行进程，使用上下文切换的方式让操作系统的并发成为可能。虽然并发从宏观上看有多个任务在执行，但对于单核 CPU 来说，任意时刻都只有一个任务在占用 CPU 资源。

> 上下文指某一时刻 CPU 寄存器和程序计数器的内容，通过在内存中保存 / 读取来完成其切换。上下⽂切换通常是计算密集型的，意味着此操作会消耗⼤量的 CPU 时间，故线程也不是越多越好。如何减少系统中上下⽂切换次数，是提升多线程性能的⼀个重点课题。

如果说进程让操作系统的并发性成为了可能，那么 **线程** 就让进程的内部并发成为了可能。每个线程执行进程中的一个子任务，使得杀毒软件一遍检测用户电脑一遍清理垃圾成为可能。

#### 进程和线程的区别

1. 进程间的通信比较复杂，而线程间的通信比较简单。线程相比进程更为轻量级，多线程并发相比多进程开销更小。
2. 进程和线程本质的区别是 **是否单独占有内存地址空间及其他的系统资源（比如 I/O）**：进程间存在内存隔离，而线程共享所属进程占有的内存地址空间和隔离，数据共享简单，但是同步复杂。
3. 进程是操作系统进行资源分配的基本单位，而线程是操作系统进行调度的基本单位。一个进程出现问题不会影响其他进程；而一个线程崩溃可能影响整个程序的稳定性。

### Java 线程与操作系统内核线程

#### 用户级线程与内核级线程

推荐阅读：[用户级线程和内核级线程，你分清楚了吗？](https://cloud.tencent.com/developer/article/1541432)

1. 用户级线程 ULT：由应用程序实现和管理（创建、同步、调度等），线程阻塞则整个进程阻塞。对操作系统来说，用户级线程具有不可见性、透明性，ULT 下 CPU 的调度还是以进程为单位。
2. 内核级线程 KLT：需通过 **系统调用** 创建，由系统内核管理，可实现多核 CPU 并行处理。线程阻塞不会影响同进程内其他线程的运行。

![](/images/system-memory.png)

#### 线程池的意义

* JVM 运行在用户态，通过调用系统库来创建内核线程，由 CPU 来完成线程的调度；但是从用户态到内核态的权限提升和状态切换需要相当的 **系统开销**，频繁的创建和销毁线程将不利于程序性能的提升。
* 为此，将线程池化管理，对线程进行统一分配、调优和监控，避免频繁的线程创建，减少状态切换带来的资源消耗，**重用线程** 就是使用线程池的意义。
* 线程池比较适合处理数量庞大，但是处理时间较短的任务。如果某个任务耗时过长，会导致池内任务堆积。

# 线程池原理

### ThreadPoolExecutor 的构造器参数

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) 
```

* corePoolSize：核⼼线程数，在没有任务执行时线程池的大小
* maximumPoolSize：池内最大线程数，在工作队列满了的情况下会创建出超过非核心的线程

> 最大线程数 = 核心线程 + 非核心线程。非核心线程如果长时间闲置，就会被销毁。

* keepAliveTime / unit：非核心线程闲置超时时长 / 时间单位，当非核心线程的空闲时间超过存活时间后会被标记为可回收的，帮助回收空闲线程占有的资源
* workQueue：阻塞的任务队列，保存那些即将被执行的任务

> 阻塞队列：在任意时刻，不管并发有多高，永远只有一个线程能够进行队列的入队或者出队操作！无界|有界；队列满，只能进行出队操作，所有入队的操作必须等待，也就是被阻塞；队列空，只能进行入队操作，所有出队的操作必须等待，也就是被阻塞。

* threadFactory：创建线程的工厂
* rejectedExecutionHandler：当任务队列满且无法再创建非核心线程时会执行拒绝处理策略
  * AbortPolicy 默认拒绝处理策略，丢弃任务并抛出 RejectedExecutionException 异常。  
  * CallerRunsPolicy：由调⽤线程处理该任务，如果将 WebServer 改为有界队列和 “调用者运行” 饱和策略，当线程池中所有线程都被占用，并且工作队列也被填满后，下一个任务会 **在主线程中执行**。由于执行任务需要一定的时间，因此主线程至少在一段时间内不能提交任务任务。在这期间到达的请求会被保存在 TCP 层的队列中而不是在应用程序的队列中，如果持续过载，那么 TCP 层最终会发现它的请求队列被填满，然后开始抛弃请求。
  * DiscardOldestPolicy：丢弃队列头部（最旧的）的任务，然后重新尝试执⾏程序，如果工作队列是一个优先级队列，那么 “抛弃最旧的” 策略将导致抛弃优先级最高的任务
  * DiscardPolicy：悄悄丢弃新来的任务  

### 线程池的状态

#### 线程池的五种状态

* RUNNING：线程池创建后处于 RUNNING 状态，能接受新任务以及处理已添加的任务
* SHUTDOWN：调用 shutdown() 方法后处于该状态，线程池不能接受新任务，可以处理队列中的任务，并清除一些空闲的 worker
* STOP：调用 shutdownNow() 后处于该状态，线程池不能接受新任务，丢弃队列中的任务，并且中断正在处理的任务
* TIDYING：当所有的任务已经终止，ctl 记录的 “任务数量” 为 0，线程池会处于 TIDYING 状态
* TERMINATED：在 TIDYING 状态执行完 terminated() 方法后，线程池彻底终止，转变为TERMINATED 状态

![](/images/线程池状态.png)

#### 源码分析

ThreadPoolExecutor 中 使用 `AtomicInteger ctl` 记录线程池的运行状态与活动线程数量

以一个整数 32 位为例：

* COUNT_BITS = 29；CAPACITY = 0001 1111 1111 1111 1111 1111 1111 1111；（高 3 位为0，后 29 位皆为 1）

* 以 ctl 的 **高三位** 记录当前的线程池状态

  * | 线程池状态 | ctl 高 3 位 （后 29 位皆为 0） |
    | :--------: | :----------------------------: |
    |  RUNNING   |              111               |
    |  SHUTDOWN  |              000               |
    |    STOP    |              001               |
    |  TIDYING   |              010               |
    | TERMINATED |              011               |

* 如此，通过当前 ctl 的值与 CAPACITY  的与运算就可以求得当前线程池状态 / 当前活跃线程数量

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;       // = 29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
```

### 四种常见的线程池

Executors 工具类中提供的⼏个静态⽅法来创建线程池。⼤家到了这⼀步，如果看懂了前⾯讲的 ThreadPoolExecutor 构造⽅法中各种参数的意义，那么⼀看到 Executors 类中提供的线程池的源码就应该知道这个线程池是⼲嘛的。  

不过 [阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20)  中指明 **不允许** 使用 Executors 创建线程池，而是通过 ThreadPoolExecutor 的方式，下面会通过源码进行分析。

#### newCachedThreadPool

```java
	public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

当一个任务提交时，corePoolSize 为 0 不创建核心线程，SynchronousQueue 是一个不存储元素的队列，可以理解为队里永远是满的，因此最终会创建非核心线程来执行任务。

对于非核心线程空闲 60 s 时将被回收。因为 Integer.MAX_VALUE 非常大，可以认为是可以无限创建线程的，在资源有限的情况下容易引起 **OOM 异常**。

当需要执⾏很多 **短时间** 的任务时，CacheThreadPool 的线程复⽤率⽐较⾼， 会显著的提⾼性能。⽽且线程 60s 后会回收，意味着即使没有任务进来，CacheThreadPool 并不会占⽤很多资源 。

#### newFixedThreadPool

```java
	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

可以看到 newFixedThreadPool 创建的都是核心线程，如果任务队列中没有任务可取，线程会一直阻塞在 getTask() 方法，而 newCachedThreadPool 会在等待 60s 后收回非核心线程。所以在没有任务的情况下 FixedThreadPool 会 **占用更多的资源**。

和 SingleThreadExecutor 类似，都使用了无界队列，唯一的区别就是核心线程数不同，并且由于使用的是 LinkedBlockingQueue，在资源有限的时候容易引起 **OOM 异常**。

两种线程池都⼏乎不会触发拒绝策略，但是原理不同。FixedThreadPool 是因为阻塞队列可以很⼤（最⼤为 Integer 最⼤值），故⼏乎不会触发拒绝策略；CachedThreadPool 是因为线程池很⼤（最⼤为Integer最⼤值），⼏乎不会导致线程数量⼤于最⼤线程数，故⼏乎不会触发拒绝策略 。

#### newSingleThreadExecutor

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    } 
```

单线程的线程池，可以通过线程封闭的方式确保不会有任务并发执行，实现线程安全性。因为 LinkedBlockingQueue 是长度为 `Integer.MAX_VALUE` 的队列，可以认为是 **无界队列**，因此往队列中可以插入无限多的任务，在资源有限的时候容易引起 **OOM 异常**，同时因为无界队列，maximumPoolSize 和 keepAliveTime 参数将无效，压根就不会创建非核心线程。

#### newScheduledThreadPool

```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

⼀个定⻓线程池，⽀持定时及周期性任务执⾏。  

四种常⻅的线程池基本够我们使⽤了，但是《阿⾥巴巴开发⼿册》**不建议** 我们直接使⽤ Executors 类中的线程池，⽽是通过 ThreadPoolExecutor 的⽅式，这样的处理⽅式让写的同学需要更加明确线程池的运⾏规则，规避资源耗尽的⻛险。

- FixedThreadPool 和 SingleThreadExecutor => 允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而引起 OOM 异常
- CachedThreadPool => 允许创建的线程数为 Integer.MAX_VALUE，可能会创建大量的线程，从而引起 OOM 异常

#### 如何定义线程池参数

**CPU密集型 =>** 线程池的大小推荐为CPU数量 + 1，CPU数量可以根据 `Runtime.availableProcessors` 方法获取

**IO密集型 =>** CPU 数量 * CPU利用率 * (1 + 线程等待时间 / 线程 CPU 时间)

**混合型 =>** 将任务分为 CPU 密集型和 IO 密集型，然后分别使用不同的线程池去处理，从而使每个线程池可以根据各自的工作负载来调整

**阻塞队列 =>** 推荐使用有界队列，有界队列有助于避免资源耗尽的情况发生

**拒绝策略 =>** 默认采用的是 AbortPolicy 拒绝策略，直接在程序中抛出 RejectedExecutionException 异常【因为是运行时异常，不强制catch】，这种处理方式不够优雅。处理拒绝策略有以下几种比较推荐：

- 在程序中捕获 RejectedExecutionException 异常，在捕获异常中对任务进行处理。针对默认拒绝策略
- 使用 CallerRunsPolicy 拒绝策略，该策略会将任务交给调用 execute 的线程执行【一般为主线程】，此时主线程将在一段时间内不能提交任何任务，从而使工作线程处理正在执行的任务。此时提交的线程将被保存在 TCP 队列中，TCP 队列满将会影响客户端，这是一种 **平缓的性能降低**
- 自定义拒绝策略，只需要实现 RejectedExecutionHandler 接口即可
- 如果任务不是特别重要，使用 DiscardPolicy 和 DiscardOldestPolicy 拒绝策略将任务丢弃也是可以的

### 线程池的使用

此处结合 [Java 并发编程实战](https://book.douban.com/subject/10484692/) 第八章 线程池的使用 介绍在使用任务执行框架时需要注意的各种危险，以及一些使用 Executor 的高级示例。

#### 任务与执行策略之间的隐形耦合

只有当任务都是同类型的并且相互独立时，线程池的性能才能达到最佳。由于线程池中的线程的 **可重用性**，必须在任务完成后在 try-finally 块中清除 `ThreadLocal` 变量，避免影响后续业务逻辑和造成内存泄漏等问题。

如果提交的任务 **依赖** 于其他任务，那么除非线程池无限大，否则将可能造成死锁。试想在单线程的 Executor 中，一个任务将另一个任务提交到同一个 Executor，并等待这个被提交任务的结果，那么通常会引发死锁。第二个任务停留在队列中，等待第一个任务完成，而第一个任务又无法完成，因为它在等待第二个任务的完成。在更大的线程池中，如果所有正在执行任务的线程都由于等待其他仍处于队列中的任务而阻塞，则会发生同样的问题，称为 **线程饥饿死锁**。

如果将 **运行时间较长** 的与运行时间较短的任务混合在一起，除非线程池很大，否则将可能造成 “拥塞”。执行时间较长的任务不仅会造成线程池堵塞，甚至还会增加执行时间较短任务的服务时间。如果线程池中的 **线程数量远小于** 在稳定状态下执行时间较长的任务数量，那么到最后可能所有的线程都会运行这些执行时间较长的任务，从而影响整体的响应性。

有一项技术可以缓解执行时间较长任务造成的影响，即限定等待资源的时间，大多数可阻塞方法都提供了限时版本和无限时版本，如果 **等待超时** 就把任务标记为失败，这样就能将线程释放出来以执行一些能更快完成的任务。

#### 线程池的配置与扩展

在 [Java：并发编程实战](https://nervousorange.github.io/2021/java-concurrency/) 中介绍了无限制地创建线程将导致系统的不稳定性。虽然可以通过使用固定大小的线程池来解决这个问题，然而在高负载下如果新请求的到达速率超过了线程池的处理速率，请求会 **在队列中累计** 起来，应用程序仍可能耗尽资源。

相比使用 newFixedThreadPool 和 newSingleThreadExecutor 默认的无界队列 `LinkedBlockingQueue`，使用有界队列如 `ArrayBlockingQueue` 可以有助于避免资源耗尽的情况发生。需要注意的是，只有当任务相互独立时，为线程池或工作队列设置界限才是合理的。如果任务之间存在依赖性，那么有界的线程池或队列就可能导致线程饥饿死锁问题。队列满后新到的任务将会根据饱和策略进行处理。有界队列的大小必须与线程池大小一起调节，如果线程池较小而队列较大，那么有助于减少内存使用量，降低 CPU 的使用率，同时还可以减少上下文切换，但付出的代码是可能会限制吞吐量。

对于非常大或者无界的线程池（比如 newCachedThreadPool），可以通过使用 `SynchronousQueue` 来避免任务排队，它不是一个真正的队列（没有容量），而是一种在线程之间进行移交的机制。要将一个任务放入 SynchronousQueue 中，就必须有另一个线程正在等待这个任务，否则将会根据当前线程池的大小创建一个新的线程或者按照饱和策略拒绝掉这个任务。

如果希望给线程池中的线程定制一些行为，例如指定一个 UncaughtExceptionHandler、给线程取一个更有意义的名称等，可以通过使用定制的线程工厂来实现：

```java
public class MyThreadFactory implements ThreadFactory {
    public final String poolName;
    
    public MyThreadFactory(String poolName) {
        this.poolName = poolName;
    }
    
    public Threaed newThread(Runnable runnable) {
        return new MyAppThread(runnable, poolName);
    }
}

public class MyAppThread extends Thread {  
    private static final AtomicInteger created = new AtomicInteger();
    private static volatile boolean debugLifecycle = false;
    private static final AtomicInteger alive = new AtomicInteger();  // 存活线程数
    
    public MyAppThread(Runnable runnable, String name) {
        super(runnable, name + "-" + created.incrementAndGet());
        setUncaughtExceptionHandler(    // 定制 UncaughtExceptionHandler
        	new Thread.UncaughtExceptionHandler() {
                public void uncaughtException(Thread t, Throwable e) {
                    log.log(Level.SERVER, "UNCAUGHT in thread " + t.getName(), e);
                }
            });
    }
    
    public void run() {
        boolean debug = debugLifecycle;
        if (debug) { log.log(Level.FINE, "Created " + getName()); }
        try {
            alive.incrementAndGet();
            super.run();
        } finally {
            alive.decrementAndGet();
            if (debug) { log.log(Level.FINE, "Exiting " + getName()); }
        }
    }
    
    public static int getThreadAlive() { return alive.get(); }
}
```

// TODO

#### 递归算法的并行化

### 线程池主要的任务处理流程

ThreadPoolExecutor 的 `execute(Runnable command)` 处理用户新提交的任务：

``` java
// JDK 1.8
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
    	 // 任务为空，抛异常
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            // 当前有核心线程空闲，调用核心线程执行任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
            // 若 addWorker 调用核心线程失败，则更新 ctl，执行下方代码，将任务添加进队列
        }
    
        if (isRunning(c) && workQueue.offer(command)) {
            // 当前核心线程无空闲，则将任务添加到任务队列中
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 再次检查线程池状态，不在 RUNNING 状态则从队列中移除任务，执行拒绝策略
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
            // 若线程池处于 RUNNING 状态，但没有 worker，则创建线程
        }
    
        else if (!addWorker(command, false))
            reject(command);
   	// 如果任务队列满了无法加入，则会创建非核心线程执行任务，若失败则执行拒绝策略
    }
```

然后再看一下在 execute() 中多次被调用的 `addWorker(Runnable firstTask, boolean core)` 方法：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            // 

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 根据 core 的布尔值创建核心 / 非核心线程，若当前线程数以达到预设值，则返回 false
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 满足可以新增线程的条件，CAS 更新 ctl，跳出循环
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
                // CAS 更新失败，若线程池状态没变，重复当前 for 循环，状态变了则从 retry 重来
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);               // 这里会创建一个新的线程
            final Thread t = w.thread;
            if (t != null) {             // 若线程创建失败，则跳至最下方 finally
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s; 
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();                   // 启动这个线程，开始工作
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
            // 创建线程失败，则执行 addWorkerFailed(w)
        }
        return workerStarted;         // 返回是否成功创建线程并开始任务
    }
```

### 如何做到线程复用

再看下 Worker 类，在调用它的构造器的时候会新建一个线程，并且它实现了 Runnable 接口，是一个线程任务，所以当 `addWorker(Runnable firstTask, boolean core)` 方法调用 `t.start()` 时，会调用 Worker 中的 run() 及 `runWorker(Worker w)` 方法：

```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable      // 实现了 Runnable 接口
    {
        final Thread thread;
        Runnable firstTask;
        volatile long completedTasks;
        
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);  // 调用构造器时，会新开一个线程
        }

        public void run() {       // 线程开始工作时调用这个 run 
            runWorker(this);
        }

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
            // 执行自己的任务或者从任务队列中取出一个任务
                w.lock();
                // 将 Worker 自身上锁
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                // 检测线程池状态，若线程池处于中断状态，当前线程将中断
                try {
                    beforeExecute(wt, task);       // 执行 beforeExecute
                    Throwable thrown = null;
                    try {
                        task.run();       // 执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);      // 执行 afterExecute
                    }
                } finally { n
                    task = null;
                    w.completedTasks++;
                    w.unlock();   // 解锁 Worker
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
        

```

⾸先去执⾏创建这个 worker 时就有的任务，当执⾏完这个任务后，worker 的⽣命周期并没有结束，在 while 循环中，worker 会不断地调⽤ `getTask()` ⽅法从阻塞队列中获取任务然后调⽤ `task.run()` 执⾏任务，从⽽达到 **复⽤线程** 的⽬的。只要 getTask() ⽅法不返回 null，此线程就不会退出。

当然，核⼼线程池中创建的线程想要拿到阻塞队列中的任务，先要判断线程池的状态，如果 STOP 或 TERMINATED，返回 null 。  

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // allowCoreThreadTimeOut 变量默认是 false，核⼼线程即使空闲也不会被销毁
            // 如果为 true，核⼼线程在 keepAliveTime 内仍空闲则会被销毁。

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
            // 如果运⾏线程数超过了最⼤线程数，但是缓存队列已经空了，这时递减 worker 数量。
            // 如果有设置允许线程超时或者线程数量超过了核⼼线程数量，
            // 并且线程在规定时间内均未 poll 到任务，且队列为空则递减 worker 数量
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

核⼼线程的会 **⼀直** 卡在 workQueue.take() ⽅法，被阻塞并挂起，不会占⽤ CPU 资源，直到拿到 Runnable 然后返回（当然如果 allowCoreThreadTimeOut 设置为 true，那么核⼼线程就会去调⽤ poll ⽅法，因为 poll 可能会返回 null，所以这时候核⼼线程满⾜超时条件也会被销毁）。  

⾮核⼼线程会 workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ，如果超时还没有拿到，下⼀次循环判断 compareAndDecrementWorkerCount 就会返回 true，导致 getTask() 返回 null，Worker 对象的 run() ⽅法循环体的判断为 null，就会执行 `processWorkerExit(w, completedAbruptly)` 线程被系统回收。

