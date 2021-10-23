---
title: "深入分析 ClassLoader 工作机制"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Reading notes"]
date: 2020-09-22T09:19:47+01:00
draft: false
---

Java 是个强类型的编程语言，一个变量的类型在编译时就已经决定了，就是最初声明它的类型， 如果想把它当成另一个类型来使用，需要先经过显式的类型转换，转换时可能会因为类型不符而抛出 `ClassCastException`。

本篇以 [《深入分析 Java Web 技术内幕》](https://book.douban.com/subject/25953851/) 第六章 深入分析 ClassLoader 工作机制 的内容为参考，讲解 Java 中的类型，类加载机制，类加载错误分析 等。

### Java 中的类型

如开篇所说，一个 Java 对象的类型在声明的时候就被确定了，但 Java 是面对对象的语言，有丰富的接口和抽象类，一个对象被传来传去的时候，可能就丢失了它的一些类型信息，但 Java 也像 C++ 那样是拥有 RTTI (Run-Time Type Identification) **运行时类型识别** 特性的语言。在任何时刻，任何一个对象都清楚地知道自己的具体类型（可以通过反射获取）。 

不知道大家有没有想过，Java 中的对象是如何被创造出来的，创造时是否又遵循了一个怎样的规则。其实 Java 中的对象都是根据各自的说明书构建出来的，这个说明书信息就是在 [Javac 编译原理与 class 文件结构](https://hoffmanzheng.github.io/2020/java-compile/) 中生成的类的 class 字节码文件之中，这些字节码文件又被存储在永久代 (Permenent Generation Java 7 之前) / 元空间 (MetaSpace Java 8 之后) 之中。

当需要创建一个对象时，就会将这个创建任务委托给类加载器来完成，类加载器会找到指定的 class 字节码文件，根据说明书来装配出一个需要的对象。因此任何一个 Java 对象，在运行时都清楚的知道自己是由哪个类的字节码装配出来的，都可以通过 `getClass() / instanceof` 来获取自己的具体类型。 

### 类加载机制

JVM 类加载机制分为五个部分：加载、验证、准备、解析、初始化，如下所示：

![](/images/class-load-process.jpg)

| 类加载阶段 | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| 加载       | 生成 Class 对象，作为方法区这个类的各种数据入口              |
| 验证       | 确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求      |
| 准备       | 为类变量分配内存空间，并设置变量初始值，非 finanl 是0，final 会赋值 |
| 解析       | 虚拟机将常量池中的符号引用替换为直接引用的过程               |
| 初始化     | 执行类构造器方法（静态语句块）                               |

### ClassLoader 类加载器

类加载器 ClassLoader 在 JDK 源码中的 rt.jar (rt 即 runtime) 这个包中，先看下这个类都有哪几个主要的方法：

| 方法                          | 主要作用                                   |
| ----------------------------- | ------------------------------------------ |
| defineClass(byte[], int, int) | 将字节流解析成 JVM 能够识别的 Class 对象   |
| findClass(String)             | 获取要加载类的字节码（重写以自定义）       |
| resolveClass(Class<?> c)      | 链接类到 JVM（实例化时进行）               |
| loadClass(String)             | 加载某个类（不重新定义加载规则，直接调用） |

因为有 defineClass 方法的存在，构建对象实例的字节码不仅可以是编译后产生的 class 文件，它还可能是从网络上传输过来的字节流，或者是在内存中 **动态生成的增强字节码**。

#### 双亲委派加载模型

JVM 提供了三层 ClassLoader：

| ClassLoader           | function                                                     |
| --------------------- | ------------------------------------------------------------ |
| Bootstrap ClassLoader | 加载 JVM 自身工作需要的类（不遵守双亲委派原则）`JAVA_HOME\lib` |
| ExtClassLoader        | 扩展类加载器 `JAVA_HOME\lib\ext`                             |
| AppClassLoader        | 应用类加载器，加载用户路径 classpath 下的类                  |

![](/images/ClassLoader.jpg)

加载类对象时，会先委托其 parent 类加载器来加载这个类，如果 parent 类加载器加载成功，则直接返回加载结果了；如果 parent 类加载器加载失败，则再尝试使用当前的类加载器加载这个类，具体可以看 `loadClass` 方法：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                    // 如果有 parent 加载器，通过 parent 来加载
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);
                // 如果 parent 拒绝加载此类，则会有此类加载器完成加载

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

双亲委托加载机制，是出于安全考虑，JVM 核心类都是由最顶端的启动类加载器加载的，不会由下面的类加载器加载，以保证安全。比如我自己写了一个异常的 `java.lang.String` ，是无法被恶意地加载的，因为应用类加载器会委托其父加载器去加载，而在父加载器的已加载类列表中已经存在了 JDK 的 java.lang.String，此时就会直接返回，而不去加载这个恶意的类。

#### 常见加载类错误分析

| 类加载异常                  | 可能的原因                         |
| --------------------------- | ---------------------------------- |
| ClassNotFoundException      | 当前 classpath 下没有指定的文件    |
| NoClassDefFoundError        | 命令行运行程序时，类前面没有加包名 |
| UnsatisfiedLinkError        | 不小心把 JVM 中的某个 lib 删除了   |
| ClassCastException          | 强制类型转换时，类型不匹配         |
| ExceptionInInitializerError | 对象初始化时出现异常               |



