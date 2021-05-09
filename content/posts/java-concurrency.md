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

本篇将结合 [《Java 并发编程实战》](https://book.douban.com/subject/10484692/) 讲解在 Java 项目中如何利用线程来提高并发程序的吞吐量或响应性，如何确保并发程序的执行与预期一致，避免安全性和活跃性问题。

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

# 活跃性问题与并发的性能

我们使用加锁机制来确保线程安全，但如果过度使用加锁，则可能导致 **锁顺序死锁**。同样我们使用线程池和信号量来限制对资源的使用，但它们可能会导致资源死锁。在数据库系统的设计中提供了 **主动监测死锁及从死锁中恢复**（选择一个牺牲者放弃事务，释放它持有的资源，详见 [Database：InnoDB 存储引擎](https://nervousorange.github.io/2021/database-innodb/)）的机制，但 Java 应用程序是 **无法** 从死锁中恢复过来的，因此在设计时就一定要排除可能导致死锁出现的条件。

### 避免活跃性危险

#### 死锁

如果两个线程试图以 **不同的顺序** 来获得相同的锁，则可能会发生锁顺序死锁，如下图所示：

![](/images/lock-ordering-deadlock.jpg)

想要避免锁顺序死锁，需要对程序中的加锁行为进行全局分析，查看是否存在 **嵌套的锁获取** 操作。有时候看似能够控制锁顺序的代码依然没有避免死锁的发生，如下列的代码，如果一个线程从 X 向 Y 转账，另一个线程从 Y 向 X 转账，那么就有可能会发生死锁。

~~~java
public void transferMoney (Account fromAccount, Account toAccount, DollarAmount amount) 
    throws InsufficientFundsException {
    synchronized (fromAccount) {
        synchronized (toAccount) {    // 嵌套的锁获取操作
            if (fromAccount.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            } else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
}
~~~

由于我们无法控制参数的顺序，因此必须在整个程序中定义锁的顺序，可以使用 `System.identityHashCode` 方法重新定义锁的顺序，来保证所有的线程 **以固定的顺序** 来获取锁：

````java
private static final Object tieLock = new Object();

public void transferMoney (Account fromAccount, Account toAccount, DollarAmount amount) 
    throws InsufficientFundsException {
    class Helper {
        public void transfer() throws InsufficientFundsException {
            if (fromAccount.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            } else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
    
    int fromHash = System.identityHashCode(fromAccount);
    int toHash = System.identityHashCode(toAccount);
    
    if (fromHash < toHash) {
        synchronized (fromAccount) {
            synchronized (toAccount) {
                new Helper().transfer();
            }
        }
    } else if (fromHash > toHash) {
        synchronized (toAccount) {
            synchronized (fromAccount) {
                new Helper().transfer();
            }
        }
    } else {
        synchronized (tieLock) {          
            // 极少情况拥有相同的散列值，加入另一个锁以保证串行
            synchronized (fromAccount) {
                synchronized (toAccount) {
                    new Helper().transfer();
                }
            }
        }
    }
}
````

在持有锁的时候调用某个外部方法也可能出现活跃性问题，如果在这个外部方法中可能会获取其他锁（可能导致锁顺序死锁），或者阻塞时间过长，导致其他线程无法及时获得当前被持有的锁。如果在调用某个方法时不需要持有锁，这种调用表被称为 **开放调用**。这种通过开放调用来避免死锁的方法，类似于采用封装机制来提供线程安全的方法，在分析的时候更为简单，更容易确保采用一致的顺序来获得锁。

#### 死锁的避免与诊断


如果一个程序每次至多只能获得一个锁，那么就不会产生锁顺序死锁。如果必须获取多个锁，在设计时必须考虑锁的顺序：尽量减少潜在的加锁交互数量，将获取锁时需要遵循的协议写入正式文档并始终遵循这些协议。

还有一项技术可以检测死锁和从死锁中恢复过来，即使用显示锁中的 **定时 tryLock** 或者可轮询的锁。如下列代码所示，使用 tryLock 来获取两个锁，如果不能同时获得，那么就回退并重新尝试。在休眠时间中加入随机部分，从而降低活锁的可能性。如果在指定时间内不能获得所有需要的锁，就返回一个失败状态，从而使该操作平缓地失败。

~~~~java
public boolean transferMoney (Account fromAccount, Account toAccount, 
                              DollarAmount amount, long timeout, TimeUnit unit) 
    throws InsufficientFundsException, InterruptedException{
    long stopTime = System.nanoTime() + unit.toNanos(timeout);
    while (true) {
        if (fromAccount.lock.tryLock()) {
            try {
                if (toAccount.lock.tryLock()) {
                    try {
                        if (fromAccount.getBalance().compareTo(amount) < 0 ) {
                            throw new InsufficientFundsException();
                        } else {
                            fromAccount.debit(amount);
                            toAccount.credit(amount);
                            return true;
                        }
                    } finally {
                        toAccount.lock.unlock();
                    }
                }
            } finally {   // 获取 toAccount 锁失败，释放掉 fromAccount 锁
                fromAccount.lock.unlock();
            }
        }
        if (System.nanoTime() < stopTime) { return false; }   
        // 在指定时间内不能获得所有需要的锁，返回失败的状态
        NANOSECONDS.sleep(new Random().nextInt() % 1000);   
        // 休眠一段时间，降低活锁发生的可能性
    }
}
~~~~

#### 其他活跃性危险

1. 当线程由于无法访问它所需要的戏院而不能继续执行时，就发生了 **饥饿**，比如对线程的优先级使用不当，或者在持有锁时执行一些无法结束的结构（无限制地等待某个资源）。在 Thread API 中定义的线程优先级只是作为线程调度的参考，JVM 会根据需要将它们映射到操作系统的调度优先级，由操作系统的线程调度器尽力地提供公平的、活跃性良好的调度。尽量 **不要改变线程的优先级**，只要改变了线程的优先级，程序的行为就将与 **平台相关**，并且有导致发生饥饿问题的风险。

2. **活锁** 是另一种形式的活跃性问题，虽然不会阻塞线程，但是线程将不断重复执行相同的操作，且总会失败。活锁通常是由过度的错误恢复代码造成的，当多个相互协作的线程都对彼此进行响应从而修改各自的状态，并使得任何一个线程都无法继续执行时，就发生了活锁。这就像两个过于礼貌的人在半路上面对面地相遇：他们彼此都让出对方的路，然后又在另一条路上相遇了，因此他们就这样反复地避让下去。要解决这种活锁问题，就需要 **在重试机制中引入随机性**，比如等待一段随机的时间再重试。

### 性能与可伸缩性

提升性能意味着用更少的资源做更多的事情，当操作性能由于某种特定的资源受到限制时，我们常将该操作称为资源密集型的操作，比如：[CPU 密集型](https://zh.wikipedia.org/zh/CPU%E5%AF%86%E9%9B%86%E5%9E%8B)、I/O 密集型等。尽管使用多个线程的目标是提升整体性能，但却引入了一些额外的开销，包括：线程之间的协调（加锁、触发信号及内存同步等），增加的上下文切换， 线程的创建和销毁，以及线程的调度等。如果 **过度地使用线程**，那么这些开销甚至会超过由于提高吞吐量、响应性或者计算能力所带来的性能提升。

使用线程常常是为了充分利用多个处理器的计算能力，因此在并发程序性能的讨论中，通常更多地将侧重点放在 **吞吐量和[可伸缩性](https://baike.baidu.com/item/%E5%8F%AF%E4%BC%B8%E7%BC%A9%E6%80%A7)** 上，而不是服务时间。Amdahl 定律告诉我们，程序的可伸缩性取决于在所有代码中必须被 **串行执行** 的代码（比如独占锁）比例。因此通常可以通过以下方式来提升可伸缩性：减少锁的持有时间，降低锁的粒度，以及采用非独占的锁或非阻塞锁来代替独占锁。

#### Amdahl 定律

大多数并发程序都是由一系列的并行工作和串行工作组成的。Amdahl 定律描述的是：在增加计算资源的情况下，程序在理论上能够实现最高加速比，取决于程序中 **可并行组件与串行组件所占的比重**。如果 F 为必须被串行执行的部分，在包含 N 个处理器的机器中，最高的加速比为：

![](/images/speedup-Amdahl.png)

当 N 趋近无穷大时，最大的加速比趋近于 1/F。Amdahl 定律还量化了串行化的效率开销，在拥有 10 个处理器的系统中，如果程序有 10% 的部分需要串行执行，那么最高的加速比为 5.3（53% 的利用率），在拥有 100 个处理器的系统中加速比可以达到 9.2（9% 的利用率）。下图给出了处理器利用率（加速比 / 处理器的数量）在不同串行比例以及处理器数量情况下的变化曲线，可以明显地看到，即使串行部分所占的百分比很小，也会极大地限制当增加计算资源时能够提升的吞吐率。

![](/images/Amdahl-graph.jpg)

如果要预测应用程序在某个多处理器系统中将实现多大的加速比，需要 **找出任务中的串行部分**，除了任务本身执行的部分外，还有一些常见的串行操作容易被忽略：比如从队列中取出任务的部分，或是对结果进行处理的部分。下图给出从一个共享 Queue 取出元素时，增加线程对吞吐量的变化：

![](/images/Amdahl-concurrentLinkedQueue.jpg)

可以看到只需改变队列的实现方式，就能对可伸缩性产生明显的影响。吞吐量的差异来源于两个队列中不同比例的串行部分，同步的 LinkedList 采用单个锁来保护整个队列的状态，并且在 offer 和 remove 等方法的调用期间都将持有这个锁；ConcurrentLinkedQueue 使用了一种更复杂的非阻塞队列算法，**减少了各个操作的串行部分**，获得了更好的吞吐量。当线程数达到一定值之后，吞吐量基本保持不变，这是由于线程增加后锁上的竞争增加，每个操作消耗的时间大部分都用于上下文切换和调度延迟，再加入更多的线程也不会提高太多的吞吐量。

#### 线程引入的开销

多个线程的调度和协调过程中都需要一定的性能开销：对于为了提升性能而引入的线程来说，并行带来的性能提升必须超过并发导致的开销。

* 上下文切换

如果可运行的线程数大于 CPU 的数量，那么操作系统最终会将某个正在运行的线程调度出来，从而使其他线程能够使用 CPU，这将导致一次上下文切换（在大多数处理器中，上下文切换的开销相当于 5000~10000 个时钟周期，大约几微妙）。当线程在等待某个锁而 **阻塞** 时，JVM 通常会将这个线程挂起，在程序中发生的阻塞越多，就会发生越多的上下文切换，在 JVM 和操作系统的代码中消耗越多的 CPU 时钟周期，应用程序的可用 CPU 时钟周期就越少，从而增加调度开销，并因此降低吞吐量。

* 内存同步

synchronized 和 volatile 提供的内存可见性保证可能会使用特殊指令：内存栅栏。内存栅栏可以刷新缓存，使缓存无效，刷新硬件的写缓存以及停止执行管道，但同时 **抑制** 一些编译器优化操作（比如指令重排序），对性能带来间接的影响。synchronized 机制对无竞争的同步进行了优化（例如去掉一些不会发生竞争的锁，通过逸出分析、锁粒度粗化减少锁获取操作次数等）（volatile 通常是非竞争的），通常只消耗 20~250 个时钟周期，对应用程序整体性能的影响微乎其微。

* 阻塞

竞争的同步可能需要操作系统介入，从而增加开销。在锁上竞争失败的线程会发生阻塞，如果等待时间较短适合采用自旋等待的方式，而如果等待时间较长则会通过操作系统挂起线程，在这过程中将包含两次额外的 **上下文切换**，以及所有必要的 **操作系统操作** 和缓存操作：被阻塞的线程在其执行时间片还未用完之前就被交换出去，而在随后当要获取的锁或其他资源可用时，又再次被切换回来。

#### 减少锁的竞争

串行操作会降低可伸缩性，并且上下文切换也会降低性能。在锁上发生竞争时将同时导致这两种问题，因此减少锁的竞争能够提高性能和可伸缩性。有两个因素将影响在锁上发生竞争的可能性：**锁的请求频率，以及每次持有该锁的时间**。如果二者的乘积很小，那么大多数获取锁的操作都不会发生竞争，不会对可伸缩性造成严重影响。然而如果在锁上的请求量很高，那么需要获取该锁的线程将被阻塞并等待，在极端情况下，即使仍有大量工作等待完成，处理器也会被 **闲置**。

* 缩小锁的范围

降低发生锁竞争可能性的一种有效方式就是尽可能缩短锁的持有时间。例如可以将一些与锁无关的代码 **移出同步代码块**，尤其是那些开销较大的操作，不需要访问 **共享状态** 的操作，以及可能被阻塞的操作。根据 Amdahl 定律，这样消除了限制可伸缩性的一个因素，因为串行代码的总量减少了。

* 减小锁的粒度

另一种减小锁的持有时间的方式是降低线程请求锁的频率，这可以通过 **锁分解和锁分段** 等技术来实现。在这些技术中将采用多个相互独立的锁来保护独立的状态变量，改变这些变量在之前由单个锁来保护的情况，从而减小锁操作的粒度，实现更高的可伸缩性。`ConcurrentHashMap` 就采用了锁分段技术，使用 16 个锁来保护所有的散列桶，这大约能把对于锁的请求减少到原来的 1/16。分段锁的一个劣势在于：在要获取多个锁来实现 **独占访问** 时将会更加困难并且开销更高，比如在扩容时需要加锁整个容器，需要获取分段锁集合中所有的锁。

* 避免热点域

当操作请求多个变量时，锁的粒度将很难降低。虽然使用一个独立的 **计数器** 能很好地提高类似 size 和 isEmpty 方法的执行速度，但却导致更难以提升实现的可伸缩性，因为每个修改 map 的操作都需要更新这个共享的计数器。即使使用锁分段来实现散列链，那么在对计数器的访问进行同步时，也会重新导致在使用独占锁时存在的可伸缩性问题。在这种情况下，计数器也被称为 **热点域**，因为每个导致元素数量变化的操作都需要访问它。为了避免这个问题，ConcurrentHashMap 为每个分段都维护了一个独立的计数，通过每个分段的锁来维护这个值，它的 size 将对每个分段进行枚举并将每个分段中的元素数量进行相加。

* 避免使用独占锁

第三种降低竞争锁的影响的技术就是 **放弃使用独占锁**，从而使用一种友好并发的方式来管理共享状态，比如使用并发容器、读-写锁、不可变对象以及原子变量，详见下文。

# Java 对并发的支持

### 显式锁

在大多数情况下，内置锁都能很好地工作，但在功能上存在一些 **局限性**，例如无法中断一个正在等待获取锁的线程，或者无法在请求获取一个锁时无限地等待下去。Java 5.0 增加了 `ReentrantLock`，它提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作。

**可定时的与可轮询的锁获取** 模式是由 tryLock 方法实现的，使用它可以有效避免死锁的发生（**代码示例参见上文**：死锁的避免与诊断）。使用定时锁时，如果不能在指定的时间内给出结果，那么就会使程序提前结束。当使用内置锁时，在开始请求锁后，这个操作将无法取消，因此内置锁将难以实现带有时间限制的操作。此外，`Lock.lockInterruptibly` 方法能够在获得锁的同时保持对中断的响应，实现一个可取消的加锁操作。

相比 ReentrantLock 使用的保守且强硬的互斥锁，`ReadWriteLock` 提供了一种较为乐观的实现：一个资源可以被多个读操作访问，或者被一个写操作访问，但两者不能同时进行。读-写锁是一种性能优化措施，可以提高多处理器系统上被 **频繁读取** 的数据结构的性能（见下图），而在其他情况下，读-写锁的性能要比独占锁的性能要略差一些，因为它们的复杂性更高。

【缺一张读写锁的图】

### 条件队列

在并发程序中，**基于状态的条件** 可能会由于其他线程的操作而改变：一个资源池可能在几条指令之前还是空的，但现在却变为非空的，因为另一个线程可能会返回一个元素到资源池。当基于状态的前提条件不满足时，方法可以返回一个错误值，**让调用者自行处理** 失败的情况，比如 `Queue.poll()` 在队列为空时会返回 null，`Queue.remove()` 则会直接抛出 NoSuchElementException 异常。

客户代码在处理失败时，可以选择直接重试（称为忙等待或 **自旋等待**），如果状态在很长一段时间内都不会发生变化，那么使用这种方法就会 **消耗大量的 CPU 时间**。相反，调用者页可以进入休眠状态来避免消耗过多的 CPU 时间，但如果状态在刚调用完 sleep 就立即发生变化，那么将不必要地休眠一段时间。因此，客户代码必须要在二者之间进行选择：要么容忍自旋导致的 CPU 时钟周期的浪费，要么容忍由于休眠导致的低响应性。

阻塞库的方法实现了 "**轮询与休眠**" 的重试机制，从而使调用者无须在每次调用时都实现重试逻辑。如果队列为空，那么 take 将休眠并直到另一个线程在队列中放入一个元素；如果队列满了，那么 put 将休眠并直到另一个线程从队列中取走一个元素，以便有空间容纳新的数据。这种方法将前提条件的管理操作 **封装** 起来，并简化了对队列的使用，但在实现的过程中需要付出大量的努力。如果存在某种挂起线程的方法，并且这种方法能够确保当某个 **条件成真时线程立即醒来**，那么将极大地简化实现工作，这正是条件队列实现的功能。

在调用内置条件队列的 `Object.wait()` 方法时，需要持有条件队列对象上的锁。wait 方法将释放锁，阻塞当前线程，并等待直至超时，然后线程被中断或者通过一个通知被唤醒。虽然在锁、条件谓词（基于状态的前提条件）和条件队列之间的三元关系并不复杂，但使用内置的条件队列可能会造成以下几个问题：
1. 过早唤醒：一个条件队列能与多个条件谓词相关，当执行控制重新进入调用 wait 的代码时，可能有其他线程已经获取了这个锁，并修改了对象的状态。因此每当线程从 wait 中唤醒时，都必须再次测试条件谓词
2. 信号丢失：在开始 wait 之前没有检查条件谓词，导致线程在等待一个已经发过的事件；此外，多个线程可以基于不同的条件谓词在同一个条件队列上等待，如果使用 notify 可能会导致信号丢失的问题，所以在大多数情况下应该优先选择 notifyAll。虽然 notifyAll 可能比 notify 更低效，但却更容易确保类的行为是正确的。

由于每个内置锁都只能有一个相关联的条件队列，而多个线程可能在同一个条件队列上 **等待不同的条件谓词**，使得无法满足在使用 notifyAll 时所有等待线程为同一类型的需求。如果想编写一个带有多个条件谓词的并发对象，可以使用更灵活的 `Condition`。正如 Lock 比内置锁提供了更为丰富的功能，Condition 同样比内置条件队列提供了更丰富的功能：在每个锁上可存在多个等待、条件等待可以是可中断的或不可中断的、基于时限的等待，以及公平的或非公平的队列操作。

如下代码示例了使用显示的条件变量的有界缓存，分别用 notFull 和 notEmpty 表示 "非满" 与 "非空" 两个条件谓词。当缓存为空时，take 将阻塞并等待 notEmpty，此时 put 向 notEmpty 发送信号，可以解除任何在 take 中阻塞的线程。通过将两个条件谓词分开并放到两个等待线程集中，Condition 使其 **更容易满足单次通知的需求**。它能极大地减少在每次缓存操作中发生的上下文切换与锁请求的次数。

````java
public class ConditionBoundedBuffer<T> {
	protected final Lock lock = new ReentrantLock();
	private final Condition notFull = lock.newCondition();  // 条件谓词：count < items.length
	private final Condition notEmpty = lock.newCondition();  // 条件谓词：count > 0
	private final T[] items = (T[]) new Object[BUFFER_SIZE];
	private int tail, head, count;
 
	public void put(T x) throws InterruptedException {
		lock.lock();
		try {
			// 循环避免过早唤醒
			while (count == items.length) { notFull.await(); }   
			// 阻塞直到满足 notFull 条件谓词
			items[tail] = x;
			if (++tail == items.length) { tail = 0; }
			++count;
			notEmpty.signal();
		} finally {
			lock.unlock();
		}
	}
 
	public T take() throws InterruptedException {
		lock.lock();
		try {
			// 循环避免过早唤醒
			while (count == 0) { notEmpty.await(); }   
			// 阻塞直到满足 notEmpty
			T x = items[head];
			items[head] = null;
			if (++head == items.length) { head = 0; }
			--count;
			notFull.signal();
			return x;
		} finally {
			lock.unlock();
		}
	}
}
````

### AQS

`AbstractQueuedSynchronizer` 抽象的队列式同步器，是许多同步类共同的基类，是一个用于 **构建锁和同步器的框架**，使用 AQS 可以简单且高效地构造出许多同步器，比如 ReentrantLock、CountDownLatch、Semaphore 等。AQS 解决了在实现同步器时涉及的大量细节问题，基于 AQS 构建同步器可以极大地 **减少实现工作**，而且也不必处理在多个位置上发生的竞争问题。

![](/images/AQS.png)

AQS 负责管理同步器类中的状态 `state`，它是一个整数类型的状态信息，可以用于表示 **任意状态**。例如，ReentrantLock 用它来表示所有者线程已经重复获取该锁的次数，Semaphone 用它表示剩余的许可数量，FutureTask 用它来表示任务的状态（尚未开始、正在运行、已完成以及已取消）。在同步器类中还可以自行管理一些额外的状态变量，比如 ReentrantLock 保存了锁的当前所有者信息，这样就能区分某个获取操作是重入的还是竞争的。

```java
// The synchronization state.  使用 volatile 保证线程可见性
private volatile int state;

// 返回当前的同步状态
protected final int getState() { return state; }

// 设置同步状态的值
protected final void setState(int newState) { state = newState; }

// 原子地（CAS）将同步状态设置为给定值 update，如果当前同步状态的值等于期望值 expect
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### 独占访问

同时只能允许一个线程执行，例如 ReentrantLock，它的内部类分别实现了以公平（`FairSync`）或者非公平（`NonfairSync`）获取锁的方式：

```java
// 公平锁的同步对象
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() { acquire(1); }

	// 公平版本的 tryAcquire
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 和非公平锁相比，这里多了一个判断：是否有线程在等待
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```



```java
// 非公平锁的同步对象
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    // 和公平锁相比，这里会尝试使用 CAS 抢锁，成功就返回
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires); // 非公平的 tryAcquire
    }
}

/** 
  * Sync 类
  * Performs non-fair tryLock.  tryAcquire is implemented in
  * subclasses, but both need nonfair try for trylock method.
*/
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 没有判断队列中是否有线程在等待
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

总结：公平锁和非公平锁只有两处不同：

1. 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
2. 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会 **判断等待队列是否有线程处于等待状态**，如果有则不去抢锁，乖乖排到后面。

公平锁和非公平锁就这两点区别，如果这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，**都要进入到阻塞队列等待唤醒**。相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于 **饥饿** 状态。

#### 共享访问

共享访问指多个线程可同时执行，如 Semaphone、CountDownLatch、CyclicBarrier、ReadWriteLock 等。

相比 synchronized 和 ReentrantLock 同时只能允许一个线程访问某个资源，`Semaphone` 可以指定多个线程同时访问某个资源，它管理了一组虚拟的许可，它默认构造 AQS 的 state 为 permits。当执行任务的线程数量超出 permits，那么多余的线程将会被放入阻塞队列 Park,并自旋判断 state 是否大于 0。只有当 state 大于 0 的时候，阻塞的线程才能继续执行，此时先前执行任务的线程继续执行 release 方法，release 方法使得 state 的变量会加 1，那么自旋的线程便会判断成功。如此，每次只有最多不超过 permits 数量的线程能自旋成功，便 **限制了执行任务线程的数量**。

`CountDownLatch` 允许 count 个线程阻塞在一个地方，**直至所有线程的任务都执行完毕**。CountDownLatch 也是共享锁的一种实现，它默认构造 AQS 的 state 值为 count。当线程使用 countDown() 方法时,其实使用了tryReleaseShared 方法以 CAS 的操作来减少 state，直至 state 为 0 。当调用 await() 方法的时候，如果 state 不为 0，那就证明任务还没有执行完毕，await() 方法就会一直阻塞，也就是说 await() 方法之后的语句不会被执行。CountDownLatch 会自旋 CAS 判断 state == 0，如果 state == 0 的话，就会释放所有等待的线程，await() 方法之后的语句得到执行。

`CyclicBarrier` 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比CountDownLatch 更加复杂和强大。CyclicBarrier 默认的构造方法是 CyclicBarrier(int parties)，其参数表示 **屏障拦截的线程数量**，每个线程调用 await 方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。CyclicBarrier 内部通过一个 count 变量作为计数器，count 的初始值为 parties 属性的初始化值，每当一个线程到了栅栏这里了，那么就将计数器减一。如果 count 值为 0 了，表示最后一个线程到达栅栏，就尝试执行我们构造方法中输入的任务。CountDownLatch 是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而 CyclicBarrier 更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。

### 非阻塞同步机制

#### 锁的劣势

虽然使用锁可以确保数据在并发访问中的一致性，但独占锁是一项 **悲观** 技术，如果有多个线程同时请求锁，那么一些线程将被挂起并且在稍后恢复运行。当线程恢复执行时，还必须等待其他线程执行完它们的时间片以后，才能被调度执行。在挂起和恢复线程等过程中存在着很大的开销，并且通常存在着较长时间的 **中断**。

锁定还存在其他一些缺点。当一个线程正在等待锁时，它 **不能做任何其他事情**。如果一个线程在持有锁的情况下被延迟执行（例如发生了缺页错误、调度延迟，或者其他类似情况），那么所有需要这个锁的线程都无法执行下去。如果被阻塞线程的优先级较高，而持有锁的线程优先级较低，就会产生 **优先级反转**。即使高优先级的线程可以抢先执行，但仍然需要等待锁被释放，从而导致它的优先级会降至低优先级线程的级别。如果持有锁的线程被永久地阻塞（例如由于出现了无限循环，死锁，活锁或者其他的活跃性故障），所有等待这个锁的线程就永远无法执行下去。

#### CAS 与原子变量

与基于锁的方案相比，非阻塞算法使用 **底层的原子机器指令**，在设计和实现上都要复杂得多，但它们在可伸缩性和活跃性上却拥有巨大的优势。由于非阻塞算法可以使多个线程在竞争相同的数据时不会发生阻塞，因此它能在粒度更细的层次上进行协调，并且极大地减少调度开销。而且在非阻塞算法中不存在死锁和其他活跃性问题。

在针对多处理器操作而设计的处理器中提供了一些特殊指令，用于管理对共享数据的并发访问。在早期的处理器中支持原子的测试并设置（Test-and-Set），获取并递增（Fetch-and-Increment）以及交换等指令，这些指令足以实现各种互斥体，而这些互斥体又可以实现一些更复杂的并发对象。现在几乎所有的现代处理器中都包含了某种形式的 **原子读-改-写指令**，例如比较并交换（Compare-and-Swap）或者关联加载/条件存储（Load-Linked/Store-Conditional）。操作系统和 JVM 使用这些指令来实现锁和并发的数据结构。

当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，其他线程都将失败。然而，失败的线程并 **不会被挂起**，而是被告知在这次竞争中失败，并可以再次尝试。由于一个线程在竞争 CAS 时失败不会阻塞，因此它可以决定是否重新尝试，或者执行一些恢复操作，或者不执行任何操作。这种灵活性就大大减少了与锁相关的活跃性风险。

原子变量比锁的 **粒度更细，量级更轻**，它将发生竞争的范围缩小到单个变量上。在使用基于原子变量而非锁的算法中，因为不需要挂起或重新调度线程，线程在执行时不易出现延迟，并且如果遇到竞争，也更容易恢复过来。下图给出了在适度/高度竞争的情况下使用锁和原子变量的吞吐量情况。可以看出，在高度竞争的情况下，锁的性能将超过原子变量的性能，但在更真实的竞争情况下，
原子变量的性能将超过锁的性能。这是因为锁在发生竞争时会挂起线程，从而降低了 CPU 的使用率和共享内存总线上的同步通信量。另一方面，如果使用原子变量，那么发出调用的类负责对竞争进行管理，如果在遇到竞争时立即重试，会导致在激烈竞争的环境下产生更多的竞争。（这就好比在交通拥堵时，交通信号灯能够实现较高的吞吐量；而在低拥堵时，环岛能实现更高的吞吐量。）

![](/images/AtomicInteger.jpg)

可能有人会觉得原子变量的性能比锁更糟糕，但上图中的高度竞争程度其实有点 **不切实际**。任何一个真实的程序都不会除了竞争锁或原子变量，其他什么工作都不做。在实际情况中，原子变量在可伸缩性上要高于锁，因为在应对常见的竞争程度时，原子变量的效率会更高。锁和原子变量在不同竞争程度上的性能差异很好地说明了各自的优势和劣势。在中低程度的竞争下，原子变量能提供更高的可伸缩性，而在高强度的竞争下，锁能够更有效地 **避免竞争**。

#### 非阻塞算法

如果在某种算法中，一个线程的失败或挂起不会导致其他线程也失败或挂起，那么这种算法就被称为非阻塞算法。如果在算法的每个步骤中都存在某个线程能够执行下去，那么这种算法也被称为 **无锁算法**。在非阻塞算法中通常不会出现死锁和优先级反转问题（但可能会出现饥饿和活锁问题，因为在算法中会反复地重试）。

在实现相同功能的前提下，非阻塞算法通常比基于锁的算法更为复杂。创建非阻塞算法的关键在于，如何 **将原子修改的范围缩小到单个变量上**，同时还要维护数据的一致性。栈是最简单的链式数据结构：每个元素仅指向一个元素，并且每个元素也只被一个元素引用。下列代码示例了如何通过原子引用构建线程安全的 ConcurrentStack，可以看出非阻塞算法的特性：如果某项工作的完成具有不确定性，必须重新执行。

~~~java
public class ConcurrentStack<E> {
	AtomicReference<Node<E>> top = new AtomicReference<Node<E>>(); 

	public void push (E item) {
 		Node<E> newHead = new Node<E>(item);
		Node<E> oldHead;
		do {
			oldHead = top.get();
			newHead.next = oldHead;
		} while(!top.compareAndSet(oldHead, newHead));  
		/** 如果在开始插入节点时，位于栈顶的节点没有发生变化，那么 CAS 就会成功；
		如果栈顶节点发生了变化（其他线程插入或者删除了元素），
		那么 CAS 将会失败，而 push 方法会根据栈的当前状态更新节点，并且再次尝试。**/
	}

	public E pop() {
		Node<E> oldHead;
		Node<E> newHead;
		do {
			oldHead = top.get();
			if (oldHead == null) { return null; }
			newHead = oldHead.next;
		} while (!top.compareAndSet(oldHead, newHead));
		return oldHead.item;
	}

	private static class Node<E> {
		public final E item;
		public Node<E> next;

		public Node(E item) {
			this.item = item;
		}
	}
}
~~~

非阻塞算法在栈中的实现很容易，但对于一些更复杂的数据结构，例如队列、散列表或数，则要复杂得多。下列代码展示了 Michael-Scott 提出的非阻塞链表算法（ConcurrentLinkedQueue 中使用的正是该算法），在初始化时就将头节点和尾节点都指向哑节点，当插入一个新的元素时，**需要更新两个指针**。首先更新当前最后一个元素的 next 指针，将新节点链接到列表队尾，然后更新尾节点，将其指向这个新元素。如果第一个 CAS 成功，但第二个 CAS 失败，那么链表将处于不一致的状态，即使这两个 CAS 都成功了，仍有可能有另外一个线程在两个操作之间访问这个链表。

~~~java
public class LinkedQueue<E> {
	// 非阻塞链表算法中的插入算法
	private static class Node<E> {
 		final E item;
		final AtomicReference<Node<E>> next;

		public Node(E item, Node<E> next) {
			this.item = item;
			this.next = new AtomicReference<Node<E>>(next);
		}
	}

	private final Node<E> dummy = new Node<E>(null, null); 
	// 空链表 dummy 哑节点，等同于 sentinel 哨兵节点
	private final AtomicReference<Node<E>> head = new AtomicReference<Node<E>>(dummy);
	private final AtomicReference<Node<E>> tail = new AtomicReference<Node<E>>(dummy);

 	public boolean put(E item) {
 		Node<E> newNode = new Node<E>(item, null);
		while(true) {
			Node<E> curTail = tail.get();
			Node<E> tailNext = curTail.next.get();
			if (curTail == tail.get()) {  
				if (tailNext != null) {  
				// 检查链表是否处于中间状态
					tail.compareAndSet(curTail, tailNext);
					// 链表处于中间状态（另一个线程正在插入元素），帮助结束其他线程执行的插入元素操作，然后循环重试
				} else {   // 链表处于稳定状态
					if (curTail.next.compareAndSet(null, newNode)) {  
					// 如果其他线程插队，CAS 失败，会重新进入 while 循环重试
						tail.compareAndSet(curTail, newNode);
						// 第二步 CAS 如果失败，表示这一步已经由其他线程代为完成
						return true;
					}
				}
			}
		}
	}
}
~~~

实现非阻塞算法的链表的关键点在于：当队列处于稳定状态时（下图 15-5），尾节点的 next 域将为空，如果队列处于中间状态，那么 tail.next 将为非空。因此，任何线程都能通过 **检查 tail.next 来获取链表当前的状态**。而且，当链表处于中间状态时，可以通过将尾节点向前移动一个节点，从而结束其他线程正在执行的插入元素操作，并使得链表恢复为稳定状态。

【缺非阻塞算法链表过程状态图】

在使用 CAS 实现非阻塞算法时，可能会遇到 ABA 问题，即 V 的值首先由 A 变成 B，再由 B 变成 A，却仍然被认为是失败，需要重新执行算法中的某些步骤。对此可以选择同时更新两个值，一个引用和一个版本号，通过在引用上加上 "版本号" 来避免 ABA 问题。

### Java 内存模型

