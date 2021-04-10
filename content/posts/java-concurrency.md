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

# 活跃性、并发的性能与测试

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

还有一项技术可以检测死锁和从死锁中恢复过来，即使用显示锁中的 **定时 tryLock** 或者可轮询的锁，
指定一个超时时限，在等待超过该时间后 tryLock 会返回一个失败信息，可以在发生死锁意外情况后释放自己持有的锁，让出系统的控制权。

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

可以看到只需改变队列的实现方式，就能对可伸缩性产生明显的影响。吞吐量的差异来源于两个队列中不同比例的串行部分，同步的 LinkedList 采用单个锁来保护整个队列的状态，并且在 offer 和 remove 等方法的调用期间都将持有这个锁；ConcurrentLinkedQueue 使用了一种更复杂的非阻塞队列算法，**减少了各个操作的串行部分**，获得了更好的吞吐量。当线程数达到一定值之后，吞吐量基本保持不变，这是由于线程增加后锁上的竞争增加，吞吐量受到上下文切换的限制。

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
* 减小锁的粒度
* 避免热点域
* 避免使用独占锁

### 并发程序的测试

# 安全性与 Java 对并发的支持

### 显式锁

### 条件队列

### AQS

### CAS

### Java 内存模型