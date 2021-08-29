---
title: "深入解析 Spring 架构与设计"
author: "Chenghao Zheng"
tags: ["component"]
categories: ["Reading notes"]
date: 2021-08-26T13:19:47+01:00
draft: false
---

[Spring](https://spring.io/) 在 Java 开发中，有着不可替代的作用和地位。Spring 的目标在于让 Java EE 的开发变得更容易，这也意味着 Spring 框架的使用也应该是容易的。相比与 EJB 模型引入的过度的复杂性，Spring 提供了 IoC 容器和 AOP 支持，来降低框架对应用的 **侵入性**。在处理与现有优秀解决方案的关系时，Spring 不会与这些第三方的解决方案发生竞争，而是致力于为应用提供使用优秀方案的集成平台，真正地把 Spring 定位在应用平台的地位，使得自己成为一个兼容并包的开放体系。

本篇结合 [《Spring技术内幕（第2版）》](https://book.douban.com/subject/10470970/) 从源代码的角度讲解 Spring 的各个主要功能模块的设计和实现原理。

### IoC 容器的实现

如果每个对象都需要靠自身实现去获取它所 **依赖** 的对象，会引起代码的高度耦合并且难以测试。很多对象并不会经常发生变化，它们之间的相互依赖关系也是比较稳定的，不会随着应用的 **运行状态** 的改变而改变，这些特性使得这些对象依赖关系的建立和维护可以交由容器来统一完成。

IoC 容器的出现使得对象的依赖注入可以从代码中 **解耦** 出来，把对依赖的 **控制权** 从具体业务对象手中转交到平台或者框架中，有效地解决了面对对象系统设计的复杂性和系统的可测试性。

#### BeanFactory 对 IoC 容器的功能定义

以 BeanFactory 的实现类 `XmlBeanFactory` 为例，它继承了 DefaultListableBeanFactory 并在其基础上增加了功能，从名字上可以猜到，它是一个可以读取以 XML 文件方式定义的 `BeanDefinition`（管理了对象之间的相互依赖关系） 的 IoC 容器。

![](/images/XmlBeanFactory.png)

`DefaultListableBeanFactory` 是一个很重要的 IoC 实现，其他的 IoC 容器比如 ApplicationContext，也是通过持有或者扩展 DefaultListableBeanFactory 来获得基本的 IoC 容器的功能的。BeanDefinition 的信息来源被封装在 `Resource` 中，对 XML 文件定义信息的具体处理又委托给了 `XmlBeanDefinitionReader` 来实现。它们间相互关系的伪代码如下：

```java
ClassPathResource res = new ClassPathResource("beans.xml");    // IoC 配置文件的抽象资源
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);  // BeanDefinition 读取器
reader.loadBeanDefinitions(res);  // 载入和注册 Bean
```

`ApplicationContext` 是一个高级形态意义的 IoC 容器，它除了提供 BeanFactory 的基本功能外，还为用户提供了附加服务，比如支持不同的信息源（国际化），支持不同的访问资源，支持应用事件。这些丰富的附加功能，使得 ApplicationContext 以一种更加面向框架的方式工作以及对上下文进行分层和实现继承。

#### IoC 容器的初始化

IoC 容器的初始化（一般不包含 Bean 依赖注入的实现）包括 BeanDefinition 的 Resouce 定位、载入和注册这三个基本的过程。值得注意的是，Spring在实现中是把这三个过程分开并使用不同的模块来完成的，这样可以让用户更加灵活地对这三个过程进行剪裁和扩展。

以 `ClassPathXmlApplicationContext` 为例，它通过继承 DefaultResourceLoader 具备了读入 Resource 的能力，在构造函数中通过 `refresh` 来启动 IoC 容器的初始化：

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    ...
    // configLocations 支持一个或多个 BeanDefinition 所在的文件路径
    // 还允许指定自己的双亲 IoC 容器
    // 在对象的初始化过程中，调用 refresh 函数，启动了 BeanDefinition 的载入过程（后文分析）
    public ClassPathXmlApplicationContext(String[] configLocations, 
        boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
            refresh();
        }
    }
    
    // clazz - 加载资源的类（基于给定路径）
    public ClassPathXmlApplicationContext(String[] paths, Class<?> clazz, 
        @Nullable ApplicationContext parent) throws BeansException {
        super(parent);
        Assert.notNull(paths, "Path array must not be null");
        Assert.notNull(clazz, "Class argument must not be null");
        this.configResources = new Resource[paths.length];
        for (int i = 0; i < paths.length; i++) {
            this.configResources[i] = new ClassPathResource(paths[i], clazz);
        }
        refresh();
    }
    ...
}
```

AbstractApplicationContext 中的 refresh 函数会触发整个 BeanDefinition 的载入过程，然后调用  AbstractRefreshableApplicationContext 实现的 `refreshBeanFactory` ，最后委托给 XmlBeanDefinitionReader 执行 `loadBeanDefinitions`，具体的流程见下图：

![](/images/spring-refreshBeanFactory.png)

通过代码看一下容器初始化的具体实现：

```java
// AbstractRefreshableApplicationContext
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {  // shutting down the previous bean factory (if any)
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // initializing a fresh bean factory
        DefaultListableBeanFactory beanFactory = createBeanFactory(); 
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        // 调用 loadBeanDefinitions 载入 BeanDefinition 的信息
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}

// loadBeanDefinitions 通过一个抽象函数把具体的实现委托给子类来完成
protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException;

// AbstractXmlApplicationContext
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 获取需要的 Rescource 集合或者配置文件集合，使用 reader 去解析文件
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```

Spring 可以处理不同形式的 BeanDefinition，由于这里使用的是XML方式的定义，所以需要使用XmlBeanDefinitionReader。接下来看一下 BeanDefinition 的具体解析和载入过程：

![](/images/spring-loadBeanDefinition.png)

AbstractBeanDefinitionReader 会委托 `ResourceLoader` 来获取 BeanDefinition 的 Resource，随后调用的 `loadBeanDefinitions(Resource res)` 在 BeanDefinitionReader 中是一个接口方法，具体的实现在 XmlBeanDefinitionReader 中。这个 Resource 对象封装了对XML文件的 IO 操作，所以读取器可以在打开 IO 流后得到 XML 的文件对象。有了这个 Document 对象以后，就可以按照 Spring 的Bean 定义规则来对这个 XML 的文档树进行解析和注册了。

### Spring AOP 的实现



### Spring MVC 与 Web 环境



### 数据库操作组件的实现



### Spring 事物处理的实现

