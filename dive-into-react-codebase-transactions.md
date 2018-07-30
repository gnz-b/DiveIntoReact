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


