---
title: "四：Javac 编译原理与 class 文件结构"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Reading notes"]
date: 2020-09-05T09:19:47+01:00
draft: false
---

JVM 让 Java 成为了一次编译，到处运行，具有平台无关性的高级语言。要让 Java 代码跑在 JVM 中少不了 Javac 编译器，它把人类能理解的 Java 源代码语言翻译成 JVM 容易理解的字节码（将 .java 文件转换成 .class 文件）。某种意义上来说，Javac 编译器的存在使 Java 语言向开发者屏蔽了很多与机器相关的细节，JVM 使 Java 语言的执行和平台无关，这才成就了 Java 的繁荣。

本篇以 [《深入分析 Java Web 技术内幕》](https://book.douban.com/subject/25953851/) 第四章 Javac 编译原理 及第五章 深入 class 文件结构 的内容为参考，讲解 Javac 编译器的基本结构，Javac 的工作原理，class 文件结构等。

### Javac 编译器的基本结构

![](/images/compile.jpg)

Javac 编译器的作用就是将符合 Java 语言规范的源代码转化成符合 Java 虚拟机规范的 Java 字节码，是从一个语言规范到另一个语言规范的转化，这大概需要以下几个步骤：

| 编译过程     | 步骤                                                         |
| ------------ | ------------------------------------------------------------ |
| 词法分析过程 | 识别源代码中的语法关键词，形成 Token 流                      |
| 语法分析过程 | 检查关键词组合是否符合 Java 语言规范，形成一个抽象语法树     |
| 语义分析过程 | 把一些难懂的、复杂的语法转化成更加简单的语法，形成一个注解过后的抽象语法树 |
| 字节码生成   | 根据注解过后的抽象语法树生成字节码                           |

### 词法分析器

在 Javac 编译一个类之前，会先一个字节一个字节地读取源代码，找出其中的语法关键词，如 if、else、for、while 等，最后形成一些规范化的 Token 流，就像在人类语言中，分辨一句话中的动词、名词、标点符号等。

从 [Java I tell you-爪哇我话你知](injdk.cn) 上选择 AdoptOpenJDK 下载后，即可得到含 sun 包源码的 OpenJDK，词法相关的类大多在 `com.sun.tools.javac.parser` 包下，类结构如下图所示。两个由 Factory 生成的实现类 Scanner 和 JavacParser 负责整个词法分析的过程控制。`JavacParser` 规定了哪些词是符合 Java 语言规范的，而具体的读取和归类不同词法的操作由 `Scanner` 完成，它会逐个读取 Java 源文件的单个字符，然后解析出符合 Java 语言规范的 Token 序列。`Token` 规定了所有 Java 语言的合法关键词，`Names` 用来存储和表示解析后的词法。

![](/images/javacScanner.png)

词法分析过程是在 JavacParser 的 parseCompilationUnit 方法中完成的，源码如下：

```java
public JCTree.JCCompilationUnit parseCompilationUnit() {
    Token firstToken = token;
    JCExpression pid = null;
    JCModifiers mods = null;
    boolean consumedToplevelDoc = false;
    boolean seenImport = false;
    boolean seenPackage = false;
    List<JCAnnotation> packageAnnotations = List.nil();
    if (token.kind == MONKEYS_AT)  // 解析注解 @
    	mods = modifiersOpt();

    if (token.kind == PACKAGE) {   // 解析 package 声明
        seenPackage = true;
        if (mods != null) {
            checkNoMods(mods.flags);
            packageAnnotations = mods.annotations;
            mods = null;
    	}
        nextToken();
        pid = qualident(false);
        accept(SEMI);
    }
    ListBuffer<JCTree> defs = new ListBuffer<JCTree>();
    boolean checkForImports = true;
    boolean firstTypeDecl = true;
    while (token.kind != EOF) {
        if (token.pos > 0 && token.pos <= endPosTable.errorEndPos) {
            // error recovery
            skip(checkForImports, false, false, false);
            if (token.kind == EOF)
            	break;
    	}
        if (checkForImports && mods == null && token.kind == IMPORT) {
            seenImport = true;
            defs.append(importDeclaration());
        } else {
            Comment docComment = token.comment(CommentStyle.JAVADOC);
            if (firstTypeDecl && !seenImport && !seenPackage) {
                docComment = firstToken.comment(CommentStyle.JAVADOC);
                consumedToplevelDoc = true;
        	}
            JCTree def = typeDeclaration(mods, docComment);
            if (def instanceof JCExpressionStatement)
                def = ((JCExpressionStatement)def).expr;
                defs.append(def);
            if (def instanceof JCClassDecl)
                checkForImports = false;
            mods = null;
            firstTypeDecl = false;
        }
    }
    JCTree.JCCompilationUnit toplevel = F.at(firstToken.pos).TopLevel(packageAnnotations, pid, defs.toList());
    if (!consumedToplevelDoc)
    	attach(toplevel, firstToken.comment(CommentStyle.JAVADOC));
    if (defs.isEmpty())
    	storeEnd(toplevel, S.prevToken().endPos);
    if (keepDocComments)
    	toplevel.docComments = docComments;
    if (keepLineMap)
    	toplevel.lineMap = S.getLineMap();
    this.endPosTable.setParser(null); // remove reference to parser
    toplevel.endPositions = this.endPosTable;
    return toplevel;
}
```



### 语法分析器

### 语义分析器

### 代码生成器

