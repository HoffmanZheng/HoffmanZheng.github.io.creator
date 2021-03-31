---
title: "Java：异常处理与日志记录"
author: "Chenghao Zheng"
tags: ["java"]
categories: ["Study notes"]
date: 2021-03-24T13:19:47+01:00
draft: false
---

在开发 Java 项目中通常依赖 Eclipse/IDEA 等集成开发工具的 Debug 调试来追踪解决 Bug，而在没有调试工具的测试、生产环境（一般也不允许远程调试）往往通过日志的轨迹 **快速定位并解决线上问题**。 但如果日志输出不好，不仅无法辅助定位问题反而可能会影响到程序的运行性能和稳定性。

本篇将结合 [阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20)、 [Log4j 2.x 官方文档](https://logging.apache.org/log4j/2.x/)、 [Takipi 日志相关文章](https://www.overops.com/) 讲解在项目中如何优雅地处理异常以及高效且有效地记录日志。

# Java 的异常处理

### 错误码

[阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20) 中有关错误码给出了以下几点指导：

1. 错误码就像康熙字典中的生僻字一样，用词似乎精准，但是字典不容易随身携带并且简单易懂。1）错误码必须能够快速知晓 **错误来源**，可快速判断是谁的问题。 2）错误码必须能够进行清晰地比对（代码中容易 equals）。 3） 错误码有利于团队快速对错误原因达到 **一致认知**  
2. 错误码分成两个部分：错误产生来源 + 四位数字编号 ，可参考手册中的附表（避免随意定义新的错误码），用于快速溯源。将来源于客户、当前系统、第三方服务的错误快速 **区分**。在无法具体确定的错误场景中，可直接使用 **宏观错误码**，比如 A0001（用户端错误）、B0001（系统执行出错）、C0001（调用第三方服务出错）  
3. 错误码不能直接输出给用户作为提示信息使用。堆栈（stack_trace）、错误信息（error_message）、错误码（ error_code）、提示信息（ user_tip）是一个有效关联并互相转义的和谐整体，但是 **请勿互相越俎代庖**。错误码之外的业务独特信息由 error_message 来承载，而不是让错误码本身涵盖过多具体业务属性  
4. 在获取第三方服务错误码时，向上抛出允许 **本系统转义**，由 C 转为 B，并且在错误信息上带上原有的第三方错误码。

### 异常处理

[阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20) 中有关异常处理给出了以下几点指导：

1. 异常设计的初衷是解决程序运行中的各种意外情况，而不是做流程或条件控制，且异常的处理效率比条件判断方式要低很多。 
2. 非受检异常如 RuntimeException、NullPointerException 等不应该通过 catch 的方式来处理，而是自行使用条件判断。
3. 尽可能 **区分异常类型**，再做对应的异常处理。定义异常时区分 unckecked / checked 异常，避免直接抛出 `new RuntimeException()` 或是 Exception / Throwable，应使用 **有业务含义** 的自定义异常。
4. 如果不想处理异常，可以将该异常抛出给它的调用者。最外层的业务使用者，必须处理异常，将其 **转化** 为用户可以理解的内容。在事务场景中，应注意在 catch 后 **手动回滚** 事务。
5. finally 块必须对资源对象、流对象进行关闭，推荐使用 `try-with-resources` 方式。在 finally 中禁止使用 return，finally 中的 return 会无情地屏蔽掉 try 中的 return 语句。
6. 方法的返回值可以为 null，不强制返回空集合，或者空对象等，必须添加 **注释充分说明什么情况下会返回 null 值**。必须考虑到远程调用失败、 序列化失败、运行时异常等场景返回 null 的情况。  
7. 防止 `NullPointerException` 是程序员的基本修养，注意 NPE 产生的场景：
   * 返回类型为基本数据类型， return 包装数据类型的对象时，自动拆箱有可能产生 NPE。
     反例： public int f() { return Integer 对象}， 如果为 null，自动解箱抛 NPE。
   * 数据库的查询结果可能为 null。
   * 集合里的元素即使 isNotEmpty，取出的数据元素也可能为 null。
   * 远程调用返回对象时，一律要求进行空指针判断，防止 NPE。
   * 对于 Session 中获取的数据，建议进行 NPE 检查，避免空指针。
   * 级联调用 obj.getA().getB().getC()；一连串调用，易产生 NPE。  

#### 统一的异常处理

对于不可避免需要处理的各种异常，代码中就会出现大量的 try... catch... finally... 代码块，不仅有大量的代码冗余，还影响代码的可读性。但异常还是要处理的，那么如何优雅地处理各种异常呢。

在写测试用例的时候体验过 JUnit 的 Assert 带给我们的丝滑，可以模仿其写一个断言类，**使用 asset 断言来代替日常的异常 throw**，在断言失败时会调用抛出异常的接口。

![](/images/asset-replace-throw.jpg)

将错误码和异常信息封装在枚举中，这样就不用每一种异常都单独定义一个异常类了。

![](/images/exception-enum.jpg)

最终的业务逻辑在判断处理异常时就会变得非常简洁了：

```java
ExceptionEnum.CLIENT_INVALID_INPUT.assertNotNull(null);
ExceptionEnum.CLIENT_INVALID_INPUT.assertNotNull(new Object());
```

至于统一的异常处理，可以参考使用下 Spring 提供的 `@ExceptionHandler` 。

# Java 的日志记录

### 日志组件的选择

纵览 Java Log 的发展历程：

1. Log4j（作者Ceki Gülcü）出来时就等到了广泛的应用（注意这里是直接使用），是 Java 日志事实上的标准，并成为了 Apache 的项目
2. Apache 要求把 Log4j 并入到 JDK，SUN 拒绝，并在 JDK1.4 版本后增加了 JUL（java.util.logging）
3. 毕竟是 JDK 自带的，JUL 也有很多人用。同时还有其他日志组件，如 SimpleLog 等。这时如果有人想换成其它日志组件，如 Log4j 换成 JUL，因为 API 完全不同，就需要改动代码。
4. Apache 见此，开发了 JCL（Jakarta Commons Logging），即 commons-logging-xx.jar。它只提供一套 **通用的日志接口 API**，并不提供日志的实现。很好的设计原则嘛，依赖抽象而非实现。这样应用程序可以在运行时选择自己想要的日志实现组件。
5. 这样看上去也挺美好的，但是 Log4j 的作者觉得 JCL 不好用，自己开发出 slf4j，它跟 JCL 类似，本身不替供日志具体实现，只对外提供接口或门面。目的就是为了替代 JCL。同时，还开发出 Logback，一个比 Log4j 拥有 **更高性能** 的组件，目的是为了替代 Log4j。
6. Apache 参考了 Logback 并做了一系列优化，推出了log4j 2

因此现在市面上常用的日志组件有：

| 日志框架        | 介绍                                                         |
| --------------- | ------------------------------------------------------------ |
| commons-logging | Apache 提供的日志门面接口，已停止更新，现在已经不太流行了    |
| slf4j           | Simple Logging Facade for Java，简单日志门面接口，不包含日志实现，允许用户根据喜好接入不同的日志实现 |
| Log4j           | Apache 的一个开源日志框架，市占率最多，但在 15 年 08 月被宣布停止维护，推荐升级到 log4j 2 |
| Log4j 2         | Log4j 的升级产品，不兼容 Log4j                               |
| Logging         | Java 自带的日志工具类，java.util.logging 包，现在基本没什么人用了 |
| Logback         | Slf4j 的原生实现框架，提供了对 Log4j 的改进，速度更快，所需内存更少 |

推荐使用门面模式的日志框架，有利于维护各个类统一的日志处理方式。[阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20) 对此的代码规范如下：

![image-20210326174344710](/images/manual-log-component.png)

在比较关注性能的地方，可以选择 Logback 或者自己实现高性能的 Logging API 更合适，在已经使用了 Log4j 的项目中，如果没有发现问题，则可以继续使用 Log4j2，如果不想有依赖则可以使用 java.util.logging 或者框架容器已经提供的日志接口。

### 日志的配置

#### 日志级别

| 日志级别 | 规范和说明                                                   |
| -------- | ------------------------------------------------------------ |
| DEBUG    | 主要输出调试性质的内容如：参数信息、调试细节、返回值等。用于开发、测试，应尽可能详尽，起到调试的作用 |
| INFO     | 记录系统关键信息，旨在保留系统正常工作期间的关键运行指标，方便日常运维工作以及错误回溯时上下文场景的复现。建议在项目完成后，在测试环境将日志级别调成 INFO，验证日志是否达到预期 |
| WARN     | 主要输出警告性质的内容，这些内容应该是可以预知且有规划的，比如某个方法参数不满足条件等 |
| ERROR    | 主要针对一些不可预知的信息，诸如：错误、异常等，比如在 catch 中抓获的网络通信、数据库连接异常。在带有错误、异常对象的数据时，需要将该对象一并输出 |

由于 INFO 及 DEBUG 日志打印量远大于 ERROR，出于性能的考虑，如果代码为核心代码，执行频率非常高，务必推敲日志设计是否合理，是否需要下调为 DEBUG 级别日志。

一般来说，WARN 级别代表可恢复的异常，此次失败 **不影响下次业务的执行**，不会短信报警。ERROR 级别会有短信甚至电话报警，意味着系统中发生了非常严重的问题，**必须马上有人处理**，比如数据库不可用，系统的关键业务流程走不下去等等。不区分问题的重要程度，只要有问题就 ERROR 记录下来是非常 **不负责任** 的行为。如果不分轻重缓急一律 ERROR 对待，就会徒增报错的频率，久而久之，运维人员对错误报警就不会那么在意，这个警报也就失去了原始的意义。

[阿里巴巴《Java 开发手册（嵩山版）》](https://developer.aliyun.com/topic/java20) 在日志级别方面的规范要求有：

> 【推荐】谨慎地记录日志。生产环境 **禁止** 输出 debug 日志；有选择地输出 info 日志；如果使用 warn 来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志。
>
> 说明：大量地输出无效日志，不利于系统性能提升，也不利于快速定位错误点。记录日志时请思考：这些日志真的有人看吗？看到这条日志你能做什么？能不能给问题排查带来好处？
>
> 【推荐】可以使用 warn 日志级别来记录用户输入参数错误的情况，避免用户投诉时，无所适从。如非必要，请不要在此场景打出 error 级别，**避免频繁报警**。
>
> 说明：注意日志输出的级别，error 级别只记录系统逻辑出错、异常或者重要的错误信息。

#### 日志的配置

[Log4j 2.x 配置文档](https://logging.apache.org/log4j/2.x/manual/configuration.html) 中详细地介绍了如何通过配置文件来管理日志行为。在项目启动的时候，Log4j 会在 classpath 中搜索配置文件，然后使用对应的配置工厂 `ConfigurationFactory` 来配置 Logger。如果没有找到配置文件，则将会使用默认的配置。与默认配置等效的配置文件会看起来像：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">   // 日志输出到 console
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>    // 日志格式
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">    // 默认根日志的级别为 error
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

在对日志配置前理解日志的工作机制是至关重要的，在没有理解 [Log4j 体系结构](https://logging.apache.org/log4j/2.x/manual/architecture.html) 的情况下去配置日志将会备受挫折。

日志可以配置多个 logger 元素，logger 元素必须有一个名字属性，也可以单独指定 additivity（默认 true），和 level（默认 ERROR）。如果想对某个 Logger 设置单独的日志级别，可以在配置文件中添加一个新的日志定义，如果想避免这个 Logger 下产生的日志重复提交给上级的根日志，可以将 `additivity` 设置为 false。

对 Log4j 2 一些特性如 Appenders，Filters，Layout 等的配置可以参考 ：[Log4j 2 官方文档](https://logging.apache.org/log4j/2.x/)

### 日志打印最佳实践

#### 记录日志的时机

在看线上日志的时候，可曾陷入过有效日志被 **大量无意义** 的日志信息淹没的日志泥潭？

回归初衷，记录的日志大致有以下三种用途：

| 功能     | 用途                                     |
| -------- | ---------------------------------------- |
| 问题追踪 | 辅助排查和定位线上问题，优化程序运行性能 |
| 状态监控 | 通过日志分析，监控系统的运行状态         |
| 安全审计 | 安全审计，发现未授权的操作               |

因此，记录日志的合适时机有以下几种：

* 编程语言提示异常：流行框架的异常报错，是质量非常高的，应当适当记录，结合业务情况使用 warn 或者 error 级别
* 业务流程预期不符：取决于开发人员判断能否容忍情形发生，常见的有：外部参数不正确，数据处理返回值不在合理范围内等
* 组件关键动作：核心数据表增删改查，核心组件运行等，建议记录 info 级别，如果日志频度高或打印量特别大，可以提炼关键点 info 记录，其余酌情考虑 debug 级别
* 系统初始化：核心模块或者组件初始化过程中往往依赖一些关键配置，根据参数不同会提供不一样的服务，务必在这里记录 info 日志，打印出参数以及启动完成态服务表述

#### 日志打印最佳实践

1. 使用门面接口定义日志对象，符合抽象编程思想方便切换。最好将日志对象定义为 `static final`，因为通过调用相同日志名称的 `getLogger` 将会返回相同的日志实例，但每次实例化一个新的日志对象是一个相当昂贵的操作

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger logger = LoggerFactory.getLogger({ClassName}.class);
```

2. 虽然 Apache 在 [Log4j 2.x 使用文档](https://logging.apache.org/log4j/2.x/manual/usage.html) 中花了较大的篇幅阐明：logger name 会在 Logger 创建的时候被指定，当 log 方法被调用时日志事件却会反映出调用方的类名，这可能会与 Logger 的创建类有所 **不同**。但在文末，其指出在日志中打印出 location information（class name, method name and line number）会产生较大的 **性能损失**，如果方法名和行数不重要的话，最好 **确保每个类拥有它自己的 Logger**，这样 logger name 就能真实反映出执行 logging 的类。根据 [Takipi](https://www.overops.com/blog/how-to-instantly-improve-your-java-logging-with-7-logback-tweaks/) 的测试结果，如果在日志中加入 Class 信息，性能会相比 Logback 的默认配置 **下降 6 成左右**，因此如果一定要打印类信息，可以考虑用类名来命名 Logger。

![log-class-pattern](/images/log-class-pattern.png)

3. 使用参数化形式 {} 占位，[] 进行参数隔离，可读性好。一看就知道 [] 里面是输出的动态参数。且不要在日志中调用对象的方法获取值，除非确保该对象肯定不为 null，否则很有可能会因为日志的问题而导致应用产生 **空指针** 异常。

```java
logger.debug("invalid Param [{}]", "\"\"");

2021-03-25 16:38:03,408 DEBUG [StaffInfoService.java:520] : invalid Param [""]
    
log.info("load student (id=[{}]), name: [{}]", id, student.getName());  // 不推荐

log.info("load student (id=[{}]), student: [{}]", id, student);  // 推荐
```

如果使用了字符串拼接的方式输出日志，即使 DEBUG 级别的日志在生产环境中不会输出到文件中，也可能带来不小的开销，在 [Log4j 2.x 性能文档](https://logging.apache.org/log4j/2.x/performance.html) 中就对比了字符串拼接和参数化信息两者的性能差异。

```java
log.debug("Entry number: [{}] is [{}]", i, entry[i]);
// 完全格式化的信息字符串在 logger 的 DEBUG 级别没有启用时不会被构建

// 如果没有这个 API，就需要三行代码来实现同样的功能
// 仍在使用不支持{}模板的 Log4j 1.x 或 Apache Commons Logging 组件的公司都会有在日志输出前加判断的编码规范
if (logger.isDebugEnabled()) {
    log.debug("Entry number: " + i + " is " + entry[i].toString());
}

// 即使在没有启用 DEBUG 的生产环境也会将变量转换为字符串进行拼接，产生性能开销
log.debug("Entry number: " + i + " is " + entry[i].toString());
```

   * 如果启用了 DEBUG，则日志信息需要在记录点被格式化。异步记录日志时，参数可能在后台线程记录前就在应用线程中被改变。这将在日志文件中展现出错误的信息。为了防止这种情况，Log4j 2, log4j 1.2, Logback 选择在将信息传递给后台线程之 **前**  就格式化信息文本。
   * 这项安全举措却产生了性能开销。下图对比了使用不同日志组件记录带参数信息时的吞吐量。JUL（java.util.logging）没有内置的异步处理器，它的 `MemoryHandler` 并没有采取上述的安全策略，而只是保留了原始参数对象的引用。因此它在单线程下非常快，然而在更多的应用线程并发记录日志时，**锁内容的开销超过了它的收益**。在吞吐量的绝对值上，Log4j 2 异步日志想比其他的日志框架表现得更出色，但注意信息格式化的开销将会随着参数数量的上升而急剧增加。

![log-parameterized-message](/images/log-parameterized-message.png)

4. 禁用标准输出 `System.out.println` 和 `System.err.println` 记录日志。因为这个只会打印到控制台，而不会记录到日志文件中，不方便管理日志。此外，标准输出不会显示类名和行号信息，很难定位日志内容和打印的位置，根本无法排查问题。

```java
// e.printStackTrace() 其实也是通过 System.err 来进行输出的
public void printStackTrace() {
    printStackTrace(System.err);
}
```

5. 建议记录下所有未被捕获的日志，其实抛出异常有开销，记录异常同样会带来一定的开销，主要原因是 `Throwable` 类的 `fillInStackTrace` 方法默认是同步的。一般使用 logger.error 都会打出异常的堆栈，如果对吞吐量有一定的要求，可以考虑覆盖该方法，去掉 `synchronized native`，直接返回实例本身。

```java
public synchronized Throwable fillInStackTrace() {
        if (stackTrace != null ||
            backtrace != null /* Out of protocol state */ ) {
            fillInStackTrace(0);
            stackTrace = UNASSIGNED_STACK;
        }
        return this;
    }
```

6. 异步日志：应用线程捕获所有需要的信息放入队列，后台线程随后会处理这些信息。只要队列足够大，应用线程应该只需花费非常少的时间在调用日志上并很快返回到业务逻辑中。

* 性能更高的 Log4j 2 Async Logger 使用了 **无锁** 的数据结构，而不是 Log4j 1.2 和 Log4j 2 Async Appender 所用的 `ArrayBlockingQueue`，这样多线程应用就不会在入队日志事件时经历阻塞。
* 下图展示了一个无锁数据结构在多线程场景下对吞吐量产生的影响：Log4j 2 Async Logger 随着线程数的提升表现出了更好的 **可伸缩性**。其他的日志组件由于承受锁的开销，在更多线程的场景下只获得了不变或更低的吞吐量。需要注意的是，图中展示出的峰值吞吐量，一旦队列满了 appender 就需要等待，会导致吞吐量出现了下降的情况。

![log4j 2-async-logger](/images/log4j2-async-logger.png)

7. 在分布式系统中，一个完整请求的日志并 **不只在一个应用** 的日志文件中，而是分散在不同服务器上不同应用节点的日志文件中。所以最好在每个子系统的日志中附上一个 UUID 标识，便于后续查询相关的日志。
8. 很多时候会在单机上部署多实例以便充分利用服务器资源，这时千万不要贪图日志监控或日志查询方便，将多个实例的日志写到同一个日志文件中。如果多个 JVM 往同一个文件里写日志，大约会使性能降低 10%。如果对同一个日志文件有大量的写需求，可以考虑 **拆分日志到不同的文件**。添加多个 Appender，不同的情况使用不同的 Logger。
9. 调用第三方服务时，所有的出参、入参、遇到的异常都是必须记录的，因为 **很难追溯第三方模块发生的问题**。
10. 异常信息应该包括两类信息：案发现场信息和 **异常堆栈信息**。尽量通过异常的日志能还原当时的情景，比如当时受影响的是哪个用户、传入的变量是什么、处理哪些核心数据引发的异常等等。有些代码在编写时就知道很难被执行到或者 **不希望被执行到**、以及一些基本不会走到的 else 块，这些地方需要记录下核心日志

```java
logger.error("inputParams:{} and errorMessage:{}", 各类参数或者对象 toString(), e.getMessage(), e);
```

