---
title: "Java：并发编程实战"
author: "Chenghao Zheng"
tags: ["java"]
categories: ["Reading notes"]
date: 2021-04-02T13:19:47+01:00
draft: false
---

在 [摩尔定律](http://www.mooreslaw.org) 失效的今天，通过提高时钟频率来提升处理器的性能已变得越来越困难，如何高效地使用并发，充分发挥多核处理器的强大计算能力变得益发重要。操作系统的出现使得计算机可以每次运行多个程序，实现了 **进程级别的并发**，操作系统为每个进程分配各自的资源（比如内存），不同的进程之间可以通过一些通信机制来交换数据，包括套接字、文件、共享内存等。

线程的出现允许在同一个进程中同时存在 **多个程序控制流**，它们共享进程中的资源，并且可以被同时调度到多个 CPU 上运行。多线程程序通过对多核处理器的充分利用来提升系统吞吐率，然而这却会引入更多的问题：指令重排、并发修改、死锁、饥饿、活锁、上下文切换、线程调度开销等。

本篇将结合 [Java 并发编程实战](https://book.douban.com/subject/10484692/) 讲解在 Java 项目中如何利用线程来提高并发程序的吞吐量或响应性，如何确保并发程序的执行与预期一致，避免安全性和活跃性问题。

# 并发任务的控制

在设计并发程序时，第一步就是要找出清晰的 **任务边界**，抽象出相互独立、可以并行执行的工作单元。大多数服务器应用程序都提供了一种自然的任务边界选择方式：以独立的客户请求为边界，主线程不断接收外部连接、**分发请求**并创建一个新线程来处理请求，但这种方法存在一些缺陷：

* 线程生命周期的开销非常高：大量轻量级的请求会在线程的创建和销毁上消耗较多的计算资源
* 资源消耗：如果可运行的线程数大于可用处理器的数量，闲置线程将会占用内存，且给垃圾回收带来压力；在 CPU 忙碌的状态下再创建更多的线程反而会降低性能
* 稳定性：没有对可创建的线程数做一个限制，容易耗尽资源造成 `OutOfMemory` 异常

### Executor 框架

串行执行的问题在于其糟糕的响应性和吞吐量，而 “为每个任务分配一个线程” 却产生了复杂的 **资源管理** 问题。为此 Java 提供了 Executor 作为 Thread 的替代，它提供了一种标准的任务执行框架，可以将任务的提交过程与实际执行过程 **解耦** 开来，从而无须太大的困难就可以为某种类型的任务指定和修改 **执行策略**。

笔者在 [Java：线程池原理、源码分析](https://nervousorange.github.io/2020/java-threadpool/) 中已经介绍过线程池相关的内容，线程池通过对线程的统一分配、调优和监控等，实现了从 “为每个任务分配一个线程” 策略到基于线程池的策略的飞跃，服务器不会再创建数千个线程来争夺有限的 CPU 和内存资源，也不会再在高负载情况下失败了。

#### 处理非正常的线程中止

在延迟任务及周期任务上，`java.util.Timer` 提供了基于绝对时间的调度机制， 但 Timer 在执行所有定时任务时只会创建一个线程，如果某个任务的执行时间过长，可能会影响后续的任务执行。此外如果 TimerTask 抛出一个 **未检查异常**，定时线程将会被 **终止**，Timer 也不会恢复线程的执行，而是会错误地认为整个 Timer 都被取消了。因此已经被调度但尚未执行的 TimerTask 将不会执行，新的任务也不能被调度，导致 `线程泄漏`。（推荐使用 DelauQueue 或者 ScheduledThreadPoolExecutor）

```java
public class OutOfTime {
 public static void main(String[] args) throws Exception {
        Timer timer = new Timer();
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(1);                        // 程序到此就结束了
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(5);
    }
    
    static class ThrowTask extends TimerTask {
        public void run() {
            throw new RuntimeException();
        }
    }
}
```

线程因未捕获异常而终止时，应用程序可能看起来仍然在工作，所以这个错误可能会被忽略。任何代码都可能抛出一个 RuntimeException，不要盲目地认为它一定会正常返回，因此线程应该在 `try-catch` 代码块中调用任务，捕获那些未检查的异常，也可以使用 `try-finally` 代码块来 **确保框架能够知道** 线程非正常退出的情况，并作出正确的响应。如以下展示了如何在线程池内部处理非受检异常：

```java
public void run() {
    Throwable thrown = null;
    try {
        while (!isInterrupted()) {
            runTask(getTaskFromWorkQueue());
        } 
    } catch (Throwable e) {
        thrown = e;
    } finally {
        threadExited(this, thrown);      // 通知框架该线程已经终结
        // 框架可能会用新的线程代替这个工作线程，确保不会影响到后续任务的执行
    }
}
```

除了上述主动 **处理未检查异常** 的方法，Thread 中提供了 `UncaughtExceptionHandler`，它能检测出某个线程由于未捕获异常而终结的情况，然后做出相应的处理（默认为将栈追踪信息输出到 System.err）：例如将错误信息及相应的栈追踪信息写入应用程序中、尝试重新启动线程、执行其他修复或诊断等。

如果想为线程池中的所有线程设置一个 UncaughtExceptionHandler，需要为 ThreadPoolExecutor 的构造函数提供一个 ThreadFactory（`thread.setUncaughtExceptionHandler(new CustomUEH());`），将任务封装在能够捕获异常的 Runable 或 Callable 中，或者改写 ThreadPoolExecutor 的 `afterExecute` 方法。需要注意的是，只有通过 execute 提交的任务，才能将它抛出的异常交给未捕获异常处理，而通过 submit 提交的任务，都会将异常封装在 Future.get 的 ExecutionException 中以重新抛出。

#### 一组任务的结果收集

如果希望得到一组任务的计算结果，可以保留与每个任务关联的 Future，然后反复调用 get 方法，timeout 指定为 0，通过轮询判断任务是否完成。这种方法虽然可行，但有些繁琐，幸运的是还有一种更好的方法：`CompletionService`。它将 Executor 和 BlockingQueue 的功能融合在一起，在提交任务后通过队列的 take 和 poll 等方法来获取已完成的结果。

### 任务的取消与线程的关闭

任务和线程的启动很容易，在大多数时候，我们都会让它们运行直到结束，或者让它们自行停止。然而有时候我们希望 **提前结束任务或线程**，或许是因为用户取消了操作，或者应用程序需要被快速关闭。Java 没有提供任何机制来安全地终止线程，只提供了中断（Interruption）作为 **协作** 机制，去 **通知** 另一个线程去终止当前的工作，因为任务本身的代码比发出取消请求的代码更清楚如何执行清除工作（避免将共享的数据置于不一致的状态）。

> [Why are Thread.stop, Thread.suspend, Thread.resume and Runtime.runFinalizersOnExit Deprecated?](http://java.sun.com/j2se/1.5.0/docs/guide/misc/threadPrimitiveDeprecation.html) 中讲述了 Thread.stop 和 suspend 等方法存在的一些缺陷，以及废弃的原因：
>
> * Thread.stop 是不安全的，终止一个线程会抛出 `ThreadDeath` 并 unlock 所有它拥有的锁，导致这些锁保护的共享对象处在一个不一致的状态。如果其他线程在这些共享对象上操作可能会导致任意的行为，ThreadDeath 会悄悄地杀死线程，因此并没有警告说应用程序处在一个危险的状态。
> * 如果想终止一个等待中的线程，可以使用 `Thread.interrupt` ，它使用了基于状态的信号机制。如果一个方法 catch 了 InterruptedException 但是没有声明抛出这个受检异常，那它应该通过 `Thread.currentThread().interrupt();` 重新中断线程。
> * Thread.suspend 本质上容易发生死锁，如果目标线程在挂起时持有了一个锁，那么其他线程在目标线程恢复前将无法获取这个锁保护的对象。如果恢复目标线程的线程在调用 `Thread.resume` 之前尝试获取这个锁，则会导致死锁。
> * 可以使用 `Object.wait` 挂起线程，`Object.notify` 恢复线程。注意 wait 和 notify 方法都在同步代码块中进行调用，保证序列化运行以减少竞争。

#### 取消任务

如果通过在任务中设置 “已请求取消” **标志位** + 轮询的方式实现任务的取消，那么在请求取消的时刻和下一次检查标志位之间就会存在延迟，在与阻塞方法联合使用时可能会导致更严重的问题：任务可能 **永远不会检查** 取消标志。

```java
class BrokenPrimeProducer extends Thread {
 private final BlockingQueue<BigInteger> queue;
    private volatile boolean cancelled = false;        // 标志位
    
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!cancelled) {
                queue.put(p = p.nextProbablePrime());   // 队列满，一直阻塞，无法通过 cancel 取消
            }
        } catch (InterruptedException consumed) {}
    }
    
    public void cancel() {cancelled = true;}
}
```

为此 Java 使用中断作为一种协作的方式去通知另一个线程 **在合适的情况下** 停止当前的工作。每个线程都有一个 boolean 类型的中断状态，在调用 interrupt() 方法后被设置，它并不会真正地去中断一个正在运行的线程，而只是发出中断请求，然后由线程在下一个合适的时刻中断自己。

```java
public class Thread implements Runable {
    public void interrupt() {...}                // 中断该线程
    public boolean isInterrupted() {...}         // 判断当前线程是否被中断
    public static boolean interrupted() {...}    // 测试这个线程是否被中断，清除中断状态，并返回它之前的值  -> 除非想屏蔽这个中断，否则必须对它进行处理，抛出 InterruptedException 或者调用 interrupt
    
    /**
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public static native void sleep(long millis) throws InterruptedException;
}
```

一些阻塞库中的方法，比如 Thread.sleep 和 Object.wait 等，都会检查线程何时中断，清除中断状态后抛出 `InterruptedException` ，因此上述通过标志位取消任务的机制可以优化为使用中断来实现：

```java
class BrokenPrimeProducer extends Thread {
 private final BlockingQueue<BigInteger> queue;
    
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()) {  // 检测中断
                // 这里显式的检测是不必要的，但执行后会对中断提供更高的响应性
                queue.put(p = p.nextProbablePrime());          // 响应中断
            }
        } catch (InterruptedException consumed) {}
    }
    
    public void cancel() {interrupt();}
}
```

在响应中断方面，非线程所有者应该小心地 **保存中断状态**（传递 InterruptedException / 使用 interrupt 恢复中断状态），这样拥有线程的代码才能对中断做出响应。这也是大多数可阻塞的库函数都只是抛出 InterruptedException 作为中断响应的原因，它们的取消策略就是尽快退出执行流程，并把中断信息传递给调用者，从而使调用栈中的上层代码可以采取进一步的操作。

如果代码不会调用可中断的阻塞方法，仍然可以通过在任务代码中选择合适的频率轮询当前线程的中断状态来响应中断。以 ThreadPoolExecutor 为例，它的工作者线程在检测到中断时，如果线程池正在关闭，它会在结束之前执行一些线程池清理工作，否则它可能创建一个新线程将线程池恢复到合理的规模。

此外还可以使用 `future.cancel(boolean mayInterruptIfRunning)` 来实现取消，参数 mayInterruptIfRunning 表示能否中断运行中的任务。使用 AbstractExecutorService.newTaskFor + 实现 RunnableFuture 可以实现非标准的取消，定制的取消代码可以实现日志记录或者收集取消操作的统计信息，以及一些不响应中断的操作。

```java
public static void timeRun (Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
    Future<?> task = taskExec.submit(r);
    try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        // 接下来任务会被取消
    } catch (ExecutionException e) {
        throw launderThrowable(e.getCause()); // 重新抛出任务中的异常
    } finally {
        task.cancel(true);
        // 超时后取消任务，如果任务成功返回，这个操作不会有任何影响
    }
}
```

虽然 Java 没有提供抢占式的中断机制，然而通过推迟中断请求的处理，开发人员能制定更灵活的中断策略，从而使应用程序在响应性和健壮性之间实现合理的平衡。

#### 停止基于线程的服务

应用服务通常会创建拥有多个线程的服务，且这些服务的生命周期通常比创建它们的方法的生命周期 **更长**。如果应用程序准备退出，那么这些服务所拥有的线程也需要结束。由于无法通过抢占式的方法来停止线程，因此它们需要自行结束（需提供生命周期方法）。

* 比如日志服务，应用线程调用 log 方法将日志放入某个队列，由专门的日志线程负责写入，是一种多生产者单消费者的设计。如果直接关闭日志服务，就会 **丢失** 那些正在等待被写入到日志的信息，并且由于队列满了其他正在调用 log 的线程将会被阻塞。为此可以设置一个标志位，在日志放入队列前进行判断，但这样就存在 “先判断再运行” 的竞态条件问题。由于队列的 put 本身可以阻塞，可以以原子方式检查关闭请求，并递增一个计数器来保持提交消息的权利：

```java
public class LogService {
    private final BlockingQueue<String> queue;
    private final LoggerThread loggerThread;
    private final PrintWriter writer;
    private boolean isShutdown;                 // 标志位  
    private int reservations;                   // 计数器
    
    public void start() { loggerThread.start(); }
    
    public void stop() {
        synchronized(this) { isShutdown = true; }
        loggerThread.interrupt();
    }
    
    public void log (String msg) throws InterruptedException {
        synchronized (this) {
            if (isShutdown) {                   // 在应用线程调用 log 时先检查标志位
                throw new IllegalStateException(...);
            }
            ++reservations;
        }
        queue.put(msg);                         
        // 对共享的标志位和计数器上锁已经足够了，队列本身会阻塞
    }
    
    private class LoggerThread extends Thread {
        public void run() {
            try {
                while (true) {
                    try {
                        synchronized (LogService.this) {
                            if (isShutdown && reservation == 0) {
                                break;
                            }
                        }
                        String msg = queue.take();
                        synchronized (LogService.this) { --reservations; }
                        writer.println(msg);
                    } catch (InterruptedException e) { /** retry */}
                }
            } finally {
                writer.close();
            }
        }
    }
} 
```

* 另一种关闭生产者-消费者服务的方式就是使用 “毒丸 Poison Pill” 对象。当从队列中获取到这个毒丸对象时就停止服务。在先进先出 FIFO 的队列中，毒丸对象能确保在关闭消费者之前能够完成队列中之前提交的所有工作，而生产者在提交毒丸对象后也不会再提交任务工作。 遗憾的是，毒丸对象只适用于生产者和消费者的数量都已知的情况。
* 在调用 shutdownNow 关闭线程池时，除了想得到所有已提交但尚未开始的任务，还想获取所有已经开始但尚未结束的任务，可以通过 **封装** ExecutorService 并使得 execute 记录哪些任务是在线程池关闭后被取消的：

```java
public class TrackingExecutor extends AbstractExecutorService {
    private final ExecutorService exec;
    private final Set<Runnable> tasksCancelledAtShutdown = Collections.synchronizedSet(new HashSet<Runnable>);
    
    public void execute (final Runnable runnable) {
        exec.execute(new Runnable()) {
            public void run() {
                try {
                    runnable.run();
                } finally {
                    if (isShutdown() && Thread.currentThread().isInterrupted()) {
                        tasksCancelledAtShutdown.add(runnable);
                    } 
                }
            }
        }
    }
    
    public List<Runnable> getCancelledTasks() {
        if (!exec.isTerminated()) {
            throw new IllegalStateException(...);
        }
        return new ArrayList<Runnable>(tasksCancelledAtShutdown);
    }
    
    // 将 ExecutorService 的其他方法委托给 exec
}
```

# 活跃性、安全性与并发测试

### 线程安全性

### 避免活跃性危险

### 并发性能与测试



# Java 对并发的支持

### 显式锁与条件队列

### AQS

### CAS

### Java 内存模型