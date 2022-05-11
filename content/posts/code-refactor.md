---
title: "重构：改善既有代码的设计"
author: "Chenghao Zheng"
tags: ["code"]
categories: ["Reading notes"]
date: 2022-05-02T03:19:47+01:00
draft: false
---

不知道大家有没有发现，编程活动的许多方面，都很难一次做对。可能是对代码的进一步理解，亦或是用户需求的又一次变化，当前的设计可能不再适应于需求。如果容许瑕疵存在，并进一步累积，代码就会变得越来越 **复杂**。在这之后对代码进行的理解、修改或调试，就会变得益发困难起来。

所谓重构，是在 **不改变** 代码外在行为的前提下，对程序内部结构的修改，对其设计的改进。虽然重构并不是一件做起来轻松的事，它却能使软件工程拥有更长的生命力。本篇结合 [《重构 (第 2 版)》](https://book.douban.com/subject/30468597/) 从一个开发者的角度讲讲重构，包括但不限于：察觉代码的坏味道、如何安全地重构以及几种重构的场景。本篇不会事无巨细地介绍所有的重构手法，而旨在指出一些值得参考的重构思想。

### 先聊聊测试

重构并 **不总是安全** 的，正如代码的其他改动一样，重构也有可能引入新的 bug。如果当前的工程项目还没有自动测试覆盖，那笔者极力推荐在开始重构之前，先补全对于待重构代码的单元测试。

在没有自动测试的情况下，开发者不得不频繁地在做出改动后手动调试测试程序的功能。这将花费很多时间和精力，并且也不够安全。编写 **自动测试**，可以让计算机来帮我们执行规律且重复的测试动作，测试代码片段对传入参数做出的响应是否符合预期或发生变化。在每一次完成代码重构后，都可以从容地运行测试，来帮助我们检查代码的行为是否受到了影响。

笔者曾就职于国内某民营企业，其部门的项目代码竟没有任何的测试代码，开发者只能在功能上线后再进行手动测试。就算这样，线上事故也频频发生，既存在新上线的功能无法达到预期的问题，也会有老的功能受到代码修改带来的影响。其实有很多的事故，在有了充足的自动测试后，都能被及早的发现，并且可以在很大程度上避免将这些问题暴露给我们的用户。

### 代码可读性

作为优秀的程序员，我们的目的从来不只是写出计算机可以理解的代码，而是写出人类容易理解的代码。大多数开发者都认为，代码被阅读和被修改的次数远远多于它被编写的次数，而保持代码易读、易修改是重构努力的方向之一。

#### 提炼函数

旧的项目大多都存在一些 **重复代码**，在阅读时得加倍仔细，留意它们之间细微的差异，在修改时也必须找出所有的副本，这让开发者痛苦不已。对此，通常我们需要先尝试移动语句，将相似的部分放在一起，再提炼成一个函数并给它一个具有意义的名字。

往往活得长的程序，其中的函数都比较短。可能初学者会觉得小函数的程序满是无穷无尽的委托调用，但和这种程序共处一段时间后，就能体会到它带来的好处 —— 更好的阐释力、且更易于分享。小函数让人易于理解的关键在于良好的命名，使代码的阅读者可以通过名字快速了解函数的作用。

每当感觉需要用 **注释** 来说明点什么的时候，我们也可以把需要说明的东西写进一个独立函数中，并以其用途（而非实现手法）命名。注释通常能指出代码用途和实现手法之间的语义差异。如果代码前方有一行注释，可能就是在提醒你：可以将这段代码替换成一个函数，并在注释的基础上给这个函数命名。

有的开发者会在代码复用时使用提炼函数，但更为合理的是，我们可以用提炼函数将代码的 **意图** 与实现分开。如果一段代码需要花一段时间游览才能弄明白它到底在干什么，就应该将其提炼到一个函数中，并根据它所做的事为其命名。提炼函数的示例代码如下：

```javascript
function printOwing(invoice) {
  let outstanding = 0;

  console.log("***********************");
  console.log("**** Customer Owes ****");
  console.log("***********************");
	
  // calculate outstanding
  for (const o of invoice.orders) {
  	outstanding += o.amount;
  }
  
  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);
  
  // print details
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
```

通过提炼代码，可以得到：

```javascript
function printOwing(invoice) {
  printBanner();
  const outstanding = calculateOutstanding(invoice);
  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}

function printBanner() {
  console.log("***********************");
  console.log("**** Customer Owes ****");
  console.log("***********************");
}

function calculateOutstanding(invoice) {
  let result = 0;
  for (const o of invoice.orders) {
    result += o.amount;
  }
  return result;
}

function recordDueDate(invoice) {
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);
}

function printDetails(invoice, outstanding) {
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
```

#### 函数/变量改名

命名可以说是编程中最难的事之一了，好的名字能让函数、模块、变量和类清晰地表名自己的功能和用法。很多开发者经常不愿意给程序元素改名，觉得不值得费这个劲，但好的名字能节省未来用来猜谜上的大把时间。如果想不到一个好名字，说明背后很可能潜藏着更深的设计问题。



#### 引入参数对象



### 再谈封装

#### 找到所有调用

#### 重构对外接口



### 优化条件逻辑

### 改善继承关系

### 

