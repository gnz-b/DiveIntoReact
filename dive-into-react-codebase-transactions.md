# 深入React源码：事务(transaction)
[原文链接](http://reactkungfu.com/2015/12/dive-into-react-codebase-transactions/)

阅读中型或者大型项目的源码，是提高你该编程语言技巧的不错的途径之一。选择React.js的项目是一个不错的开始。
他是由很有能力的JavaScript开发者编写的，被很好的测试过并且不会让你感到陌生，因为使用了用JavaScript编写的设计模式。

这篇文章我想着重介绍React.js中应用的优秀的模式 —— 事务。他的应用很广(和众多设计模式一样)从保持变量不变到管理横切关注点。同时他的实现也很简单，我也会进行讨论。我也会讲述他在React中是如何应用的。

## 问题

应用中的某些关注点会贯穿整个app。无论你的应用模块化做的多么好，你会在许多地方看到这种关注点。这些关注点在你的app中都是很常见的，类似授权，鉴权或是日志。

举个简单的例子 —— 描述一段代码。你可能想知道这段代码平均的运行时间 —— 也许后续有些工作，例如给开发者生成一份报告。你可以像下面的代码这么做：

```javascript
import { profilingStatsOf, updateStats } from 'profiling-database';

function makeSomethingCritical() {
  /* 初始化阶段 */
  var profilingStats = profilingStatsOf('makeSomethingCritical');
  var beforeTimestamp = new Date();

  // ... 做一些工作

  /* 后置条件 (`结束` 阶段) */
  var afterTimestamp = new Date();
  updateStats(profilingStats, afterTimestamp.getTime() - beforeTimestamp.getTime());
}
```

这里有一段代码 —— 你可能想保持你的初始条件和后置条件是完整的。

举个例子 —— 生成一些复杂的销售报表。你需要做一些复杂的计算来改变这些销售报表，输入一些新的值。但是你也需要保证钱的总数是大于0的。如果小于0了，你需要恢复之前版本的销售报告：
```javascript
var salesReport = {
  // ...
  totalDollarsEarned: 782
};

function calculateStateAfterRiskyInvestments(salesReport) {
  var savedOldReport = JSON.load(JSON.stringify(salesReport)); /* 克隆！`初始化`阶段 */

  /* 销售报表中许多字段的复杂计算... */

  /* 后置条件 */
  if(salesReport.totalDollarsEarned < 0) {
    Object.assign(salesReport, savedOldReport); /* 字段重置 (`结束` 阶段). */
  }
}
```
这种根据条件保存并且恢复的模式，会在很多函数中碰到，不仅仅一次(你会有许多发生突变的变量)。当然你可以创建一个把风险投资计算函数作为销售报表的参数的函数(也被称作高阶函数)。但是这种方式需要你强制返回计算函数计算出来的指，这些规则必须由你现在所在的模块来制定。如果你需要动态得设置初始化步骤和后置条件，而不是由模块来定义，这肯定不是好的选择。之前描述函数的例子也有同样的问题 —— 如果他需要被别的模块调用并且是动态(你可以热插拔)也会碰到同样的问题。

这种情况下事务模式可以帮到你。他允许你定义一系列动态的前置条件(初始化)和后置条件(退出步骤)，让你的函数在该事务中运行。你可以从别的函数中export出事务并且通过模块重用他，无需向别的模块暴露实现的细节。

## 事务模式 —— 概览
React中实现的事务模式有一个隐性的前置条件 —— 如果一个事务在运行，你不能调起另外一个事务。在JavaScript的异步世界里这很重要 —— 因为React.js在内部调解阶段会使用事务，增加这个前置条件对他是个聪明的选择。

事务API的主要构成：

- getTransactionWrappers —— 这是一个抽象函数，他会返回一个对象的list，该对象有上有个两个字段initialize和close，这是该对象的两个方法。

  initialize是前置条件或者是初始化 —— 你可以用他来设置关闭的场景。

  close是你的退出步骤或是后置条件 —— 你可以获取初始方法返回的值。
  
  这两个属性都是可选的。

- isInTransaction —— 判断其他事务是否在运行。
- perform —— 这里你可以调用你的函数。你可以给他提供scope和参数。它和调用函数的call方法很类似。只要你的函数或是初始化出错，全部的结束步骤都会运行。
- reinitializeTransaction —— 可选步骤，清理全部被初始化步骤存储的数据。它也可被用来准备内部数据结构(一个用来保存初始化返回值的list)。我不明白为什么React.js保留了这个如此不优雅的解决方案，可能模式就是这样。

在初始化阶段，函数执行阶段和退出阶段的异常处理还是很复杂。下面是事务如何处理的：
- 如果初始化的时候有异常，perform的function不会没执行但是其他的初始化函数都会被调用。当然那些没有报错的initialize函数对应的close函数也将都被调用。第一个产生的异常将会从事务中被抛出。
- 如果perform的函数抛出异常，close函数将会全部执行。异常将会被抛出事务。
- 如果close阶段抛出异常，剩下的close们将会被继续执行，第一个异常会被抛出。

React.js中的实现，Transaction.Mixin被暴露出来，Transaction对象会通过原型继承的方式获得这些方法。一般事务的创建过程是这样的：
```javascript
var Transaction = require('shared/utils/Transaction');

var MY_TRANSACTION_WRAPPERS = [
  { initialize: function() { /* ... */ },
    close: function() { /* ... */ } 
  },
  // ...  
];

function MyTransaction() {
  this.reinitializeTransaction();
}

Object.assign({}, 
  MyTransaction.prototype, 
  Transaction.Mixin, 
  { getTransactionWrappers: function() { return MY_TRANSACTION_WRAPPERS; } }
);

module.exports = MyTransaction;
```

下面的代码片段是你该如何使用事务：

```javascript
var MyTransaction = require('MyTransaction');

var transaction = new MyTransaction();

transaction.perform(myFunction, contextObject, 1, 2, 3, 4);
```
## 实现剖析

知道如何使用事务只是比较基本的。看一下事务的源代码是更明智的。虽然这不是我见过最整洁的代码，但是我知道他这样的逻辑并且这代码是被完整测试过的。让我们深入研究他的实现。
去除注释之后，完整的代码如下：
如你所见Transaction还包括了Mixin的模式，还有个内部使用的OBSERVED_ERROR的常数 —— 稍后会详细介绍。
```javascript
'use strict';

var invariant = require('invariant');

var Mixin = {
  reinitializeTransaction: function() {
    this.transactionWrappers = this.getTransactionWrappers();
    if (this.wrapperInitData) {
      this.wrapperInitData.length = 0;
    } else {
      this.wrapperInitData = [];
    }
    this._isInTransaction = false;
  },

  _isInTransaction: false,
  getTransactionWrappers: null,

  isInTransaction: function() {
    return !!this._isInTransaction;
  },

  perform: function(method, scope, a, b, c, d, e, f) {
    invariant(
      !this.isInTransaction(),
      'Transaction.perform(...): Cannot initialize a transaction when there ' +
      'is already an outstanding transaction.'
    );
    var errorThrown;
    var ret;
    try {
      this._isInTransaction = true;
      errorThrown = true;
      this.initializeAll(0);
      ret = method.call(scope, a, b, c, d, e, f);
      errorThrown = false;
    } finally {
      try {
        if (errorThrown) {
          try {
            this.closeAll(0);
          } catch (err) {}
        } else {
          this.closeAll(0);
        }
      } finally {
        this._isInTransaction = false;
      }
    }
    return ret;
  },

  initializeAll: function(startIndex) {
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      try {
        this.wrapperInitData[i] = Transaction.OBSERVED_ERROR;
        this.wrapperInitData[i] = wrapper.initialize ?
          wrapper.initialize.call(this) :
          null;
      } finally {
        if (this.wrapperInitData[i] === Transaction.OBSERVED_ERROR) {
          try {
            this.initializeAll(i + 1);
          } catch (err) {
          }
        }
      }
    }
  },

  closeAll: function(startIndex) {
    invariant(
      this.isInTransaction(),
      'Transaction.closeAll(): Cannot close transaction when none are open.'
    );
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      var initData = this.wrapperInitData[i];
      var errorThrown;
      try {
        errorThrown = true;
        if (initData !== Transaction.OBSERVED_ERROR && wrapper.close) {
          wrapper.close.call(this, initData);
        }
        errorThrown = false;
      } finally {
        if (errorThrown) {
          try {
            this.closeAll(i + 1);
          } catch (e) {
          }
        }
      }
    }
    this.wrapperInitData.length = 0;
  },
};

var Transaction = {
  Mixin: Mixin,
  OBSERVED_ERROR: {},
};

module.exports = Transaction;
```

这里还依赖了invariant模块。他使用如下：
```javascript
invariant(condition, notPassMessage)
```
如果conditon是false的话，会抛出一个带notPassMessage的错误。他很像c家族语言中的assert。

## perform
我们从事务的核心perform开始 —— perform方法：
```javascript
  perform: function(method, scope, a, b, c, d, e, f) {
```
有个很有趣的事情 —— React的开发者不能给perform传超过6的变量，因为他们限制了perform可以传的变量的数量为6。其实这个问题可以通过使用每个function都有的伪数组对象arguments解决，内部只要将call变成apply就可以了。但是因为React开发人员不需要，所以他们决定保持这样。
```javascript
invariant(
  !this.isInTransaction(),
  'Transaction.perform(...): Cannot initialize a transaction when there ' +
  'is already an outstanding transaction.'
);
```
首先有个invariant的检查，保证事务中一次只能有一个函数被perform。
```javascript
var errorThrown;
var ret;
try {
  this._isInTransaction = true;
  errorThrown = true;
  this.initializeAll(0);
  ret = method.call(scope, a, b, c, d, e, f);
  errorThrown = false;
}
```
这段代码第一件事就是“锁住”事务 —— 执行完这段代码之后，其他的perform就不能被调用。

然后这里有个不太优雅的方法 —— 一个局部变量errorThrown被用来决定下面两段代码是否抛出异常。想法很简单 —— 因为抛出异常的话会打断执行，如果发生异常errorThrown = false这段就不会被执行。所以后续就通过判断errorThrown是true就能保证这段代码抛出异常了。

所有的initialize被调用。之后你会看到initializeAll的递归调用，所以0被作为初始参数用来循环initialize的数组。细节将在之后解释。

根据指定的scope和参数列表perform被调用。

下面部分是他有趣的地方：
```javascript
finally {
  try {
    if (errorThrown) {
      try {
        this.closeAll(0);
      } catch (err) {}
    } else {
      this.closeAll(0);
    }
  } 
  finally {
    this._isInTransaction = false;
  }
}
```
这段丑陋双finnal代码根据initialize或者performed的时候抛出的错误来做决定。你不用关心是谁抛出的错，这决定了close是否抛出错误，因为只需要第一个抛出的错误。因此如果initialize或者performed抛出错误了，close阶段的错误就不抛出了。
如果close阶段出错了。一个类似this.initializeAll的方法会调用全部的close函数，同样也以0为初始参数。

最后一段finnally保证了无论怎样，perform完成之后，事务能够被再次使用 —— 把this._isInTransaction设置成false来“解锁”事务。

最后一行是返回performed函数的结果:
```javascript
return ret;
```

## initializeAll & closeAll
initializeAll和closeAll代码差不多都有点绕，我们这儿着重介绍initializeAll。这里的实现有个设计漏洞  —— initializeAll和closeAll是公共可见的。React的开发者被教导不要直接使用这两个方法，还是可以通过工厂方法很容易地来修复这个设计缺陷。这背后的原因不清楚 —— 可能是实现者的疏忽。
```javascript
initializeAll: function(startIndex) {
The signature looks like initializeAll is recurrent - there is startIndex passed as an argument. And indeed the recursion is used here - but not fully since it’d be inefficient (JavaScript do not support tail-call optimalization in most environments).
```
下面的部分是循环全部的initialize:
```javascript
var transactionWrappers = this.transactionWrappers;
for (var i = startIndex; i < transactionWrappers.length; i++) {
  var wrapper = transactionWrappers[i];
  try {
    this.wrapperInitData[i] = Transaction.OBSERVED_ERROR;
    this.wrapperInitData[i] = wrapper.initialize ?
                                wrapper.initialize.call(this) :
                                null;
  }
  // ... 这部分稍后讨论
}
```
为了避免总是输入this.transactionWrappers，这里有个方便的赋值。在ES2015可以更简单的替换成const { transactionWrappers } = this;并且这样能够更好的表达这么做的意图。

下面循环开始。遍历transactionWrappers list。

这里也有同样处理错误的方式，但是实现得更老道一点。因为需要忽略出错的initalize对应的close，第i个initialize返回的初始值是一个特定的OBSERVED_ERROR。他表明了该initialize抛出了异常。接着如果存在initialize则执行，否则返回的值是null。

请注意如果initialize抛出错误了，最后的赋值语句不会被执行所以返回的值仍然是Transaction.OBSERVED_ERROR。

让我们接着看剩下的部分:
```javascript
finally {
  if (this.wrapperInitData[i] === Transaction.OBSERVED_ERROR) {
    try {
      this.initializeAll(i + 1);
    } catch (err) {}
  }
}
```
这里有个检查是否第i个initialize出错了。如果是这样的话，递归调用this.initializeAll。this.initializeAll的参数是i + 1，这意味着从这个出错的initialize之后开始这个方法。之后的错误都不会被抛出，因为这里有个需求 —— 外部需要看到第一个错误。

closeAll基本一样有两点不同。首先，判断close方法是否存在是这样的:
```javascript
var initData = this.wrapperInitData[i];

if (initData !== Transaction.OBSERVED_ERROR && wrapper.close) {
  wrapper.close.call(this, initData);
}
```
这里还会检查对应的initialize是否出错。如果出错了，close也会被忽略。

错误的检查也有点不同。也有一个类似perform方法中的errorThrown参数。

最后一部分是“清理”wrapperInitData list:
```javascript
this.wrapperInitData.length = 0;
```
我不鼓励你用类似的方法。如果你碰到类似的情况，最好可以将全部的错误存到一个list里然后只返回第一个(或者你想返回全部)。也要避免过多的使用局部变量errorThrown。但是这里可能考虑到节约内存和内存泄漏 + 所以如上的实现可能更加适合类似React这种通用的库。

## reinitializeTransaction
最好有趣的部分就是reinitializeTransaction。这是一个既是initializer又是reinitializer的方法。我觉着这是一个不好的设计。提供一个工厂方法会更加好。不管怎样，这方法的名字也不好。我们看下他的实现:
```javascript
reinitializeTransaction: function() {
  this.transactionWrappers = this.getTransactionWrappers();
  if (this.wrapperInitData) {
    this.wrapperInitData.length = 0;
  } else {
    this.wrapperInitData = [];
  }
  this._isInTransaction = false;
},
```

这个方法做的是把Transaction中getTransactionWrappers方法返回的值赋给this.transactionWrappers。这十分简练。然后这里会检查this.wrapperInitData是否已经存在。如果存在则清空。如果Transaction的API被正常使用的话，这一步是多余的因为closeAll会帮你处理。但是如果this.wrapperInitData不存在，则会被初始化成空数组。同时事务也会被“解锁”。一般情况下清除事务中wrapper的初始化数据和解锁事务是不必须的 —— 这是一个双保险。但是如果事务的API因为设计缺陷使用的特别糟糕(类似直接调用initializeAll而不使用perform)，这就有帮助了。

以上是事务实现的主要部分。除了处理抛出第一个错误的代码有点不优雅，其他都很不错。使用javascript编写面向对象的代码来达到目的也很不错。我比较欣赏的是构建你自己的事务的方式 —— 这个很棒，纯javascript的方式来解决问题。如果你想选择从这个实现中学到点什么，毫无疑问就是这样模式 —— 他在很多地方都用用。

## React.js中如何使用他
React.js中许多地方用到事务。源码中定义了许多renderer —— 类似服务端的renderer，浏览器端的render等等。这些renderer都需要他们各自的invariants。举个DOM render的例子。ReactReconcileTransaction中事务wrapper的list:
```javascript
var TRANSACTION_WRAPPERS = [
  SELECTION_RESTORATION,
  EVENT_SUPPRESSION,
  ON_DOM_READY_QUEUEING,
];
```
该事务被用在React.js生命周期中的调合阶段 —— 在更新过程中中。当更新好的DOM已经就绪 —— 一种情况是componentDidUpdate，防止因为过多的变化而丢失输入框的值，整个处理过程中抑制时间冒泡(类似失去焦点)并且将React生命周期的某些部分放入队列。

如果没有类似事务的机制，维护这些将会很困难。只要提到维护选中 —— 有太多的DOM的变化会导致input框焦点的改变。有了事务之后，这就变的很简单。

## 总结
对你的应用，事务是设计模式的一个不错的补充。虽然我觉着他的实现有一点不完美，但他在React.js这种大项目中仍然应用很成功。这种机制很类似面向方面编程中的事前和事后建议。使用事务能够让你更容易的维护你的应用 —— 尤其是当你操作很多类似对象或是DOM突变这样的副作用(React.js做的那些)。
