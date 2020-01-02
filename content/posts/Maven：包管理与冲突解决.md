---
title: "Maven：包管理与冲突解决"
author: "Chenghao Zheng"
tags: ["Tools"]
categories: ["Study notes"]
date: 2019-12-29T23:37:47+01:00
draft: false
---

Maven 作为一个强大的项目管理工具，可以自动管理 Java 包的传递性依赖，解决包的依赖冲突并对依赖的 scope 进行控制。 本篇对其功能和用法做简单的介绍。



### 一、包管理的必要性

* JVM 的作用就是执行一个类的字节码，当它需要加载一个新的类时，就会去 `Classpath` 中寻找这个类。包的 `全限定类名` 是包的唯一标识，当 `Classpath` 中出现（不同版本的）同名包时，JVM 会选择 `Classpath中出现最早的包` 进行加载（`Classpath Hell`）。
* 包的调用存在传递性依赖，即你所调用的包同时还调用了其他别的包，这会导致一个项目的实现需要调用相当数量的第三方包，使包的管理与更新成了一个很大的难题。（同时也使自己的代码在别人的机器上不能正常的运行）



### 二、Maven：包管理工具

#### 1. 包的约定与语义化版本

* Maven 收录了几乎所有的 Java 包保存在其远程的 [中央仓库](http://repo1.maven.org/maven2/)，并对包的 ID 给出了三个约定，分别为 `Groupid、Artifactid、version` ，用结构化的方式，把包分门别类的放到一起，实现了方便检索的目的。
* 同时，包的版本号受到了[ 语义化版本控制规范 ](https://semver.org/lang/zh-CN/)

#### 2. 自动化包管理

* 当在 `pom.xml` 中引入一个第三方包的时候，Maven 就知道去哪里下载了，并且把它的传递性依赖也同时下载至本地 `~\.m2`。你可以删除该文件夹中的包，但在 `maven 刷新` 后，maven 会为你自动下载所需要的包，这也方便了在他人机器上跑自己的代码，或是在团队项目中便捷地管理所需要的包。
* Maven 为包提供了三种 Scope：
    * compile、test、及 provided（只在编译有效，运行无效，适合运行时由他人提供 jar 包的场景）



### 三、Maven：包的冲突与解决

#### 1. 包的冲突

* 当项目的不同位置调用了 `不同版本` 的某个类库时，就可能会发生包的冲突。因为 JVM 会自动调用在 `Classpath` 中出现的早的那个版本类库，导致运行时调用了错误版本的类库产生错误，同时我们也不能很好的去管理调用类库的版本。
* 常见的包冲突异常有：
    * NoSuchMethodError - AbstractMethodError - NoClassDefFoundError - ClassNotFoundError - LinkageError 等。

#### 2. Maven 依赖树

* Maven 中有三种查看包的依赖树的方法：
    * IDE 右侧 Maven ---> Dependencies
    * Terminal --- `mvn dependency:tree` （这里可以重定向到文件，方便查看）
    * 安装 `Maven helper` 插件 ---> IDE 中查看 `pom.xml` 文件，点击下方 `Dependency Analyzer` 可以查看调用包的依赖树，并且可以直观的看到树中存在的包冲突，非常实用。

* Maven 调用包的原则

    * 绝对不允许最终的 `classpath` 出现同名不同版本的类库
    * 依赖冲突的解决：离你项目最近的胜出（Maven 不分析版本号，只依赖这个原则）
        * 相比之下，Gradle 则是选择的最高版本的类库。
        * 实际上，都需要人工介入选择项目需要调用的版本。

    * 离的一样近的情况：在 `classpath` 中先出现的胜出 （靠前声明）

#### 3. 包冲突的解决办法

* 为项目选择版本合适的类库：
    * 可以去 `Github - branch` 右侧选择 `Tags` 看不同版本的类库源代码之间的区别，为项目选择合适的版本。（特别废精力）
* 解决冲突：
    * Maven 会选择离你项目最近的依赖包（的版本）进行调用（不分析版本号） ---> 把想要调用的版本类库引入到和项目近的地方。
    * 在 `pom.xml` 对应类库下 手写 `exclusions` ，把冲突包干掉。
    * 在 `Maven helper` 中，选择不想要调用的类库，右击 `Exclude` 就可以了。



### 四、推荐阅读

* 书籍 《Maven 实战》
    * 书籍有点老，阅读指南：
    * 第五、六、七、八章为核心概念，必看。
    * 第三、四章：入门和背景案例，可以了解下，代码过时不要抄。
    * 第十章 第十二章：测试、web应用，可以瞄一眼。
    * 第十三、十四章：版本管理、灵活的构建，可以看一下下。
    * 第十七章：编写 Maven 插件，没有需要可以不看。
* [ 彰德老师的Maven讲解](https://blindpirate.github.io/2019/05/10/Maven/)