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

* corePoolSize：核⼼线程数
* maximumPoolSize：池内最大线程数

> 最大线程数 = 核心线程 + 非核心线程。非核心线程如果长时间闲置，就会被销毁。

* keepAliveTime / unit：非核心线程闲置超时时长 / 时间单位
* workQueue：阻塞的任务队列，保存那些即将被执行的任务

> 阻塞队列：在任意时刻，不管并发有多高，永远只有一个线程能够进行队列的入队或者出队操作！无界|有界；队列满，只能进行出队操作，所有入队的操作必须等待，也就是被阻塞；队列空，只能进行入队操作，所有出队的操作必须等待，也就是被阻塞。

* threadFactory：创建线程的工厂
* rejectedExecutionHandler：当任务队列满且无法再创建非核心线程时会执行拒绝处理策略
  * AbortPolicy 默认拒绝处理策略，丢弃任务并抛出RejectedExecutionException异常。  
  * CallerRunsPolicy：由调⽤线程处理该任务  
  * DiscardOldestPolicy：丢弃队列头部（最旧的）的任务，然后重新尝试执⾏程序
  * DiscardPolicy：丢弃新来的任务  

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

#### newCachedThreadPool

```java
	public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

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

两种线程池都⼏乎不会触发拒绝策略，但是原理不同。FixedThreadPool 是因为阻塞队列可以很⼤（最⼤为Integer最⼤值），故⼏乎不会触发拒绝策略；CachedThreadPool 是因为线程池很⼤（最⼤为Integer最⼤值），⼏乎不会导致线程数量⼤于最⼤线程数，故⼏乎不会触发拒绝策略 。

#### newSingleThreadExecutor

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

有且仅有⼀个核⼼线程，使⽤了 LinkedBlockingQueue（容量很⼤），所以，不会创建⾮核⼼线程。所有任务按照先来先执⾏的顺序执⾏。如果这个唯⼀的线程不空闲，那么新来的任务会存储在任务队列⾥等待执⾏。  

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

四种常⻅的线程池基本够我们使⽤了，但是《阿⾥把把开发⼿册》**不建议** 我们直接使⽤ Executors 类中的线程池，⽽是通过 ThreadPoolExecutor 的⽅式，这样的处理⽅式让写的同学需要更加明确线程池的运⾏规则，规避资源耗尽的⻛险。

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

