---
title: "重构：改善既有代码的设计"
author: "Chenghao Zheng"
tags: ["code"]
categories: ["Reading notes"]
date: 2022-05-02T03:19:47+01:00
draft: false
---

不知道大家有没有发现，编程活动的许多方面，都很难一次做对。可能是对代码的进一步理解，亦或是用户需求的又一次变化，当前的设计可能不再适合于需求。如果容许瑕疵存在，并进一步累积，代码就会变得越来越 **复杂**。在这之后对代码进行的理解、修改或调试，就会变得益发困难起来。

所谓重构，是在 **不改变** 代码外在行为的前提下，对程序内部结构的修改，对其设计的改进。虽然重构并不是一件做起来轻松的事，它却能使软件工程拥有更长的生命力。本篇结合 [《重构 (第 2 版)》](https://book.douban.com/subject/30468597/) 从一个开发者的角度讲讲重构，包括但不限于：察觉代码的坏味道、如何安全地重构以及几种重构的场景。本篇不会事无巨细地介绍所有的重构手法，而旨在指出一些值得参考的重构思想。

### 先聊聊测试

重构并 **不总是安全** 的，正如代码的其他改动一样，重构也有可能引入新的 bug。如果当前的工程项目还没有覆盖自动测试，那笔者极力推荐在开始重构之前，先补全对于待重构代码的单元测试。

在没有自动测试的情况下，开发者不得不频繁地在做出改动后手动调试测试程序的功能。这将花费很多时间和精力，并且也不够安全。编写 **自动测试**，可以让计算机来帮我们执行规律且重复的测试动作，测试代码片段对传入参数做出的响应是否符合预期或发生变化。在每一次完成代码重构后，都可以从容地运行测试，来帮助我们检查代码的行为是否受到了影响。

笔者曾就职于国内某民营企业，其部门的项目代码竟没有任何的测试代码，开发者只能在功能上线后再进行手动测试。就算这样，线上事故也频频发生，既存在新上线的功能无法达到预期的问题，也会有老的功能受到代码修改带来的影响。其实有很多的事故，在有了充足的自动测试后，都能被及早的发现，并且可以在很大程度上避免将这些问题暴露给我们的用户。

### 代码的可读性

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

通过提炼代码，可以得到以下代码，代码的可读性得到的显著提高：

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

命名可以说是编程中最难的事之一了，好的名字能让函数、模块、变量和类清晰地表名自己的功能和用法。很多开发者经常不愿意给程序元素改名，觉得不值得费这个劲，但好的名字能节省未来用来 **猜谜** 上的大把时间。如果想不到一个好名字，说明背后很可能潜藏着更深的设计问题。

比如有以下一段看起来不是很好理解的代码：

```javascript
function price(order) {
  // price is base price - quantity discount + shipping
  return order.quantity * order.itemPrice -
    Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 + 
    Math.min(order.quantity * order.itemPrice * 0.1, 100);
}
```

对其使用提炼变量就能使代码更容易理解，同时注释也就不需要了：

```javascript
function price(order) {
  const basePrice = order.quantity * order.itemPrice;
  const quantityDiscount = Math.max(0, order.quantity - 500) * order.itemPrice *0.05;
  const shipping = Math.min(basePrice * 0.1, 100);
  return basePrice - quantityDiscount + shipping;
}
```

在函数命名时也有个小窍门：先写一句注释描述这个函数的用途，再把这句注释变成函数的名字。

#### 过长的参数列表

有时候我们可以在代码中看到一些带着很多参数的函数，**过长的参数列表** 显然并不是一个好的工程实践，通常也会困扰代码的阅读者。如果一组数据项总是结伴同行，出没于一个又一个函数，我们可以将其组织成新的数据对象，这样既可以缩短函数的参数列表，也能使所有使用该数据对象的函数都使用同样的名字来访问其中的元素，提升代码的一致性。如果可以通过查询获取另一个参数的值，也可以使用以查询取代参数来减少参数的个数。

有时我们会使用标志位 `flag` 来区分函数的行为，标记参数会影响函数内部的控制流，但它却通常无法清晰地传达自己的含义，对此可以使用移除标记参数。如果明确用一个函数来完成一项单独的任务，其含义就会清晰得多（见下述代码）。如果一个函数有多个标记参数，说明这个函数可能做得太多，应该考虑是否能用更简单的函数来组合出完整的逻辑。

```javascript
function deliveryDate(anOrder, isRush) {
  let result;
  let deliveryTime;
  if (anOrder.deliveryState === "MA" || anOrder.deliveryState === "CT") {
  	deliveryTime = isRush ? 1 : 2;
  } else if (anOrder.deliveryState === "NY" || anOrder.deliveryState === "NH") {
  	deliveryTime = 2;
    if (anOrder.deliveryState === "NH" && !isRush) {
    	deliveryTime = 3;
    }
  } else if (isRush) {
  	deliveryTime = 3;
  } else if (anOrder.deliveryState === "ME") {
  	deliveryTime = 3;
  } else {
  	deliveryTime = 4;
  }
  result = anOrder.placedOn.plusDays(2 + deliveryTime);
  if (isRush) result = result.minusDays(1);
  return result;
}

// 针对 isRush 标志位，可以在 deliveryDate 上添加两个函数
// 替换调用后，限制原函数的可见性，让人一见即知不应直接使用这个函数
function rushDeliveryDate(anOrder) {
	return deliveryDate(anOrder, true);
}
function regularDeliveryDate(anOrder) {
	return deliveryDate(anOrder, false);
}
```

### 再谈谈封装

上文有提到通过引入参数对象来减小函数的参数列表，而记录型结构有两种类型：一种需要声明合法的字段名，另一种可以随便用任何字段名。后者有哈希表、字典、数组等，使用这类结构时虽然方便但也有缺陷，那就是一条记录上 **持有什么字段** 往往不够直观。只能通过查看它的创建点和使用点来获取其维护的字段，如果这种记录只在程序的一个小范围里使用，那问题还不大，但如果其使用范围变宽，数据结构不直观就会造成更多的困扰。

虽然封装是 OOP 三点特性之一，笔者在初写 Java 的时候也时常困惑：为什么要声明一个又一个的数据类，又把其字段都用 private 封装起来，最后对外暴露 `getter, setter` 函数。这看起来有些多此一举的行为却也蕴含着封装的思想。试想一下，当想要对结构体中某个字段进行改名的时候，我们往往需要找到并同步修改所有引用这个字段的地方，但如果不够仔细，或者存在外部引用就无法保证操作的安全性。不过，在将字段的访问封装起来后，我们就可以渐进地完成对字段的改名：

1. 声明一个新字段，以及它的 getter, setter 函数
2. 将旧字段的 getter, setter 函数作为转发函数，调用新声明字段的函数
3. 渐进式地修改所有调用老字段的地方，改为调用新字段的函数
4. 在完成所有对老字段的引用的修改后，删除老的字段

类似地，在给函数重命名时，需要考虑是否能一步到位地修改其所有的调用者。如果函数还有外部的调用者（比如客户端，或者是来自其他服务的调用），也可以采用 **渐进式** 地修改函数声明：

1. 使用提炼函数将函数体提炼成一个新函数，给予新的命名
2. 在旧函数中使用内联函数，调用新的函数
3. 对外声明旧函数为废弃 `deprecated`，并告知应使用的新函数
4. 在客户端完成调用修改后，将旧函数删除

### 优化条件逻辑

条件逻辑可以提升程序的威力，同时也会引入一定的复杂度。复杂的条件逻辑是编程中最难理解的东西之一，我们可以用以卫语句取代嵌套条件表达式来清晰表达 "在主要处理逻辑之前先做检查" 的意图，也可以用以多态取代条件表达式来处理 switch 的多种情况。

条件表达式中，如果两个条件分支都属于正常行为，就应该使用形如 if...else... 的条件表达式，来表现出对两个分支同等的重视。但如果一个分支是正常行为，另一个分支是罕见或者异常的行为，就应该单独检查该条件，并在该条件为真时立刻从函数中返回，这样的单独检查常常被称为 "卫语句" `guard clauses`。卫语句可以告诉代码的阅读者，这种情况不是本函数的核心逻辑，如果它真发生了，请做一些必要的整理工作，然后退出。

```javascript
function payAmount(employee) {
  let result;
  if (employee.isSeparated) {
  	result = {amount: 0, reasonCode: "SEP"};
  }
  else {
    if (employee.isRetired) {
    	result = {amount: 0, reasonCode: "RET"};
    }
    else {
    	// logic to compute amount
      lorem.ipsum(dolor.sitAmet);
      consectetur(adipiscing).elit();
      sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
      ut.enim.ad(minim.veniam);
      result = someFinalComputation();
    }
  }
  return result;
}

// 使用卫语句后，单独检查罕见条件，并处理提前返回
function payAmount(employee) {
  if (employee.isSeparated) return {amount: 0, reasonCode: "SEP"};
  if (employee.isRetired) return {amount: 0, reasonCode: "RET"};
  // logic to compute amount
  lorem.ipsum(dolor.sitAmet);
  consectetur(adipiscing).elit();
  sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
  ut.enim.ad(minim.veniam);
  return someFinalComputation();
}
```

虽然使用条件逻辑本身的结构就足以表达不同的场景，但使用类和 **多态** 能把逻辑的拆分表述得更清晰。比如针对 switch 语句中的每种分支逻辑创建一个类，用多态来承载各个类型特有的行为，从而去除重复的分支逻辑。下面有这样一个例子，有一家评级机构，要根据航程本身的特征和船长过往的航行历史，对远洋航船的航行进行投资评级。

```javascript
function rating(voyage, history) {
  const vpf = voyageProfitFactor(voyage, history);
  const vr = voyageRisk(voyage);
  const chr = captainHistoryRisk(voyage, history);
  if (vpf * 3 > (vr + chr *2)) return "A";
  else return "B";
}
function voyageRisk(voyage) {
  let result = 1;
  if (voyage.length > 4) result += 2;
  if (voyage.length > 8) result += voyage.length -8;
  if (["china", "east-indies"].includes(voyage.zone)) result += 4;
  return Math.max(result, 0);
}
function captainHistoryRisk(voyage, history) {
  let result = 1;
  if (history.length < 5) result += 4;
  result += history.filter(v => v.profit < 0).length;
  if (voyage.zone === "china" && hasChina(history)) result -= 2;
  return Math.max(result, 0);
}
function hasChina(history) {
  return history.some(v => "china" === v.zone);
}
function voyageProfitFactor(voyage, history) {
  let result = 2;
  if (voyage.zone === "china") result += 1;
  if (voyage.zone === "east-indies") result += 1;
  if (voyage.zone === "china" && hasChina(history)) {
    result += 3;
    if (history.length > 10) result += 1;
    if (voyage.length > 12) result += 1;
    if (voyage.length > 18) result -= 1;
  }
  else {
    if (history.length > 8) result += 1;
    if (voyage.length > 14) result -= 1;
  }
  return result;
}
```

代码中有两处同样的条件逻辑，都在询问 "是否有到中国的航程" 以及 "船长是否曾去过中国"，我们可以使用继承和多态将处理 "中国因素"（会混淆视听）的逻辑从基础逻辑中分离出来。在重构后可以得到一个基本的 Rating 类，其中放着基础逻辑，不考虑与 "中国经验" 相关的复杂性：

```javascript
class Rating {
  constructor(voyage, history) {
    this.voyage = voyage;
    this.history = history;
  }
  get value() {
    const vpf = this.voyageProfitFactor;
    const vr = this.voyageRisk;
    const chr = this.captainHistoryRisk;
    if (vpf * 3 > (vr + chr *2)) return "A";
    else return "B";
  }
  get voyageRisk() {
    let result = 1;
    if (voyage.length > 4) result += 2;
    if (voyage.length > 8) result += voyage.length -8;
    if (["china", "east-indies"].includes(voyage.zone)) result += 4;
    return Math.max(result, 0);
  }
  function captainHistoryRisk(voyage, history) {
    let result = 1;
    if (history.length < 5) result += 4;
    result += history.filter(v => v.profit < 0).length;
    return Math.max(result, 0);
  }
  function voyageProfitFactor(voyage, history) {
    let result = 2;
    if (voyage.zone === "china") result += 1;
    if (voyage.zone === "east-indies") result += 1;
   	result += this.historyLengthFactor;
    result += this.voyageLengthFactor;
    return result;
  }
  get voyageLengthFactor() {
    return (voyage.length > 14) ? -1 : 0;
  }
  get historyLengthFactor() {
    reutrn (history.length > 8) ? 1 : 0;
  }
}
```

与 "中国经验" 相关的代码则清晰表述出在基本逻辑之上的一系列变体逻辑：

```javascript
class ExperienceChinaRating extends Rating {
  get captainHistoryRisk() {
    const result = super.captainHistoryRisk - 2;
    return Math.max(result, 0);
  }
  get voyageLengthFactor() {
    let result = 0;
    if (this.voyage.length > 12) result += 1;
    if (this.voyage.length > 18) result -= 1;
    return result;
  }
  get historyLengthFactor() {
  	return (this.history.length > 10) ? 1 : 0;
  }
  get voyageProfitFactor() {
    return super.voyageProfitFactor + 3;
  }
}
```

### 改善继承关系

继承作为 OOP 里最为人熟知的特性十分实用，却也经常被误用，而且常得等到你用上一段时间，才能察觉到误用所在。超类中会处理所有的通用逻辑，如果某个函数/字段在各个子类中都相同，就可以通过函数/字段上移将它上升到超类中。同样地，如果超类中的某个函数/字段只与一个（或少数几个）子类有关，那么最好将其从超类中挪走，放到真正关心它的子类中去。

如果一个字段仅仅作为 **类型码** 使用，根据其值来触发不同的行为，那么可以通过以子类取代类型码来重构，如此就可以用多态来处理条件逻辑。例如员工类已经有 "全职员工" 和 "兼职员工" 两个子类了，无法根据员工类别再创建不同的子类，就可以使用间接继承的方式：

```javascript
Class Employee {
  constructor(name, type) {
    this.validateType(type);
    this._name = name;
    this._type = type;
  }
  validateType(arg) {
    if (!["engineer", "manager", "salesman"].includes(arg)) 
      throw new Error(`Employee cannot be of type ${arg}`);
  }
  get type() {return this._type};
  set type(arg) {this._type = arg};
  get capitalizedType() {
    return this._type.chatAt(0).toUpperCase() + this._type.substr(1).toLowerCase();
  }
  toString() {
    return `${this._name} (${this.capitalizedType})`;
  }
}
```

我们可以用对象取代基本类型，使用类 EmployeeType 来取代 Employ 中的 type 属性。然后使用以子类取代类型码，把员工类别代码变成子类：

```javascript
class EmployeeType {
  constructor(aString) {
    this._value = aString;
  }
  toString() {return this._value;}
}
class Engineer extends EmployType {
  toString() {return "emgineer";}
}
class Manager extends EmployType {
  toString() {return "manager";}
}
class Salesman extends EmployeeType {
  toString() {return "salesman";}
}

// 在原先的员工类中使用工厂方法
class Employee {
  ...
  set type(arg) {this._type = Employee.createEmployeeType(arg);}
    static createEmployeeType(aString) {
      switch(aString) {
        case "engineer": return new Engineer();
        case "manager": return new Manager();
        case "salesman": return new Salesman();
        default: throw new Error(`Employee cannot be of type ${aString}`);
      }
    }
  ...
}
```

虽然继承在面向对象的语言中很容易实现，但继承也有短板。其一是继承这张牌只能打一次，只能用于处理超类在一个方向上的变化。其二是继承给类之间引入了非常紧密的关系，任何在超类上的修改，都可能破坏子类。这两个问题用 **委托** 都能解决，对于不同的变化原因，我们可以委托给不同的类。与继承相比，使用委托关系时接口更清晰、耦合更少。

```javascript
class Order {
  get daysToShip() {
    return this._warehouse.daysToShip;
  }
}
class PriorityOrder extends Order {
  get daysToShip() {
    return this._priorityPlan.daysToShip;
  }
}

// 使用委托后
class Order {
  get daysToShip() {
    return (this.priorityDelegate) 
      ? this._prorityDelegate.daysToShip
      : this._warehouse.daysToShip;
  }
}
class PriorityOrderDelegate {
  get daysToShip() {
    return this._priorityPlan.daysToShip;
  }
}
```

