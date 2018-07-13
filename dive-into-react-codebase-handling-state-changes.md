## 开始更新 —— ReactUpdates 

我们看下enqueueUpdate在ReactUpdates端是如何实现的:

```javascript
function enqueueUpdate(component) {
  ensureInjected();

  // Various parts of our code (such as ReactCompositeComponent's
  // _renderValidatedComponent) assume that calls to render aren't nested;
  // verify that that's the case. (This is called by each top-level update
  // function, like setProps, setState, forceUpdate, etc.; creation and
  // destruction of top-level components is guarded in ReactMount.)

  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }

  dirtyComponents.push(component);
}
```

这有两个可配的部分(注入)在ReactUpdates层面需要着重介绍一下。ensureInjected揭示了他们:

```javascript
function ensureInjected() {
  invariant(
    ReactUpdates.ReactReconcileTransaction && batchingStrategy,
    'ReactUpdates: must inject a reconcile transaction class and batching ' +
    'strategy'
  );
}
```

batchingStrategy是React如何打包你更新的策略。现在这里只有一个ReactDefaultBatchingStrategy被用在这里。
ReactReconcileTransaction是不依赖于环境的，主要负责在更新之后“固定住”短暂的state —— 对于DOM来说，他是暂存更新之后可能会丢失的选中的文本，
在调和以及生命周期方法排队的时候抑制event。

enqueueUpdate的代码有些难懂。第一眼看过去感觉没有什么特别的。batchingStrategy事务中有个字段告诉你事务是否在处理中。如果不在处理中，enqueueUpdate会停止并且将自己注册为事务处理。然后，组件被加到脏组件的队列里。

所以到现在发生了什么？挂起的state队列和callback的队列更新了，组件进入了脏组件的队列。但是其实什么都没变化因为state实际尚还没被更新。

为了理解后面的，我们需要先跳到ReactDefaultBatchingStrategy的实现。他有两个wrapper

+ FLUSH_BATCHED_UPDATES —— 在事务中完成一个功能后会调用ReactUpdates中的flushBatchedUpdates。这是更新state代码的心脏。IMO最重要的代码片段是藏在事务的实现中让人很困惑 —— 我相信这是为了代码复用。

+ RESET_BATCHED_UPDATES —— 负责事务中的功能完成后对isBatchingUpdates标志进行清理。相比RESET_BATCHED_UPDATES的细节，flushBatchedUpdates就十分重要了 —— 这里是更新state逻辑真正实现的地方。我们看下他的实现

```javascript
var flushBatchedUpdates = function() {
  // ReactUpdatesFlushTransaction's wrappers will clear the dirtyComponents
  // array and perform any updates enqueued by mount-ready handlers (i.e.,
  // componentDidUpdate) but we need to check here too in order to catch
  // updates enqueued by setState callbacks and asap calls.
  while (dirtyComponents.length || asapEnqueued) {
    if (dirtyComponents.length) {
      var transaction = ReactUpdatesFlushTransaction.getPooled();
      transaction.perform(runBatchedUpdates, null, transaction);
      ReactUpdatesFlushTransaction.release(transaction);
    }

    if (asapEnqueued) {
      asapEnqueued = false;
      var queue = asapCallbackQueue;
      asapCallbackQueue = CallbackQueue.getPooled();
      queue.notifyAll();
      CallbackQueue.release(queue);
    }
  }
};
```

其中有另外一个事务(ReactUpdatesFlushTransaction)，在运行flushBatchedUpdates完之后，他负责捕捉挂起的更新。这十分复杂因为componentDidUpdate或者setState的callback会把下个需要处理的更新放入队列。

这个事务是额外的水池(这里有准备好的实例而不是匆忙的重新创建一个 —— React使用这个技巧来避免不必要的垃圾回收)这是很干净的技巧来自游戏的开发。稍后会介绍这里asap updates的概念。
这里有个runBatchedUpdates的方法被调用。这里有很多方法被调用从setState开始到结束。还没结束，让我们看一下:

```javascript
function runBatchedUpdates(transaction) {
  var len = transaction.dirtyComponentsLength;
  invariant(
    len === dirtyComponents.length,
    'Expected flush transaction\'s stored dirty-components length (%s) to ' +
    'match dirty-components array length (%s).',
    len,
    dirtyComponents.length
  );

  // Since reconciling a component higher in the owner hierarchy usually (not
  // always -- see shouldComponentUpdate()) will reconcile children, reconcile
  // them before their children by sorting the array.
  dirtyComponents.sort(mountOrderComparator);

  for (var i = 0; i < len; i++) {
    // If a component is unmounted before pending changes apply, it will still
    // be here, but we assume that it has cleared its _pendingCallbacks and
    // that performUpdateIfNecessary is a noop.
    var component = dirtyComponents[i];

    // If performUpdateIfNecessary happens to enqueue any new updates, we
    // shouldn't execute the callbacks until the next render happens, so
    // stash the callbacks first
    var callbacks = component._pendingCallbacks;
    component._pendingCallbacks = null;

    ReactReconciler.performUpdateIfNecessary(
      component,
      transaction.reconcileTransaction
    );

    if (callbacks) {
      for (var j = 0; j < callbacks.length; j++) {
        transaction.callbackQueue.enqueue(
          callbacks[j],
          component.getPublicInstance()
        );
      }
    }
  }
}
```
该方法获取所有的脏组建，然后将他们按照加载的顺序排序(因为更新顺序是从父组件到子组件)，把callback排到事务的队列里然后运行ReactReconciler中的performUpdateIfNecessary。所以runBatchedUpdates是为了按顺序的更新。

我们跳到上一部分 - ReactReconciler.

## ReactReconciler & performUpdateIfNeeded —— 最后一步

我们看下ReactReconciler performUpdateIfNecessary方法的实现

```javascript
  performUpdateIfNecessary: function(
    internalInstance,
    transaction
  ) {
    internalInstance.performUpdateIfNecessary(transaction);
  },
```

又是一个调用。幸运的是可以在ReactCompositeComponent很快的找到:

```javascript
  performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
      ReactReconciler.receiveComponent(
        this,
        this._pendingElement || this._currentElement,
        transaction,
        this._context
      );
    }

    if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      this.updateComponent(
        transaction,
        this._currentElement,
        this._currentElement,
        this._context,
        this._context
      );
    }
  },
```

这个方法可以拆成两部分:

+ ReactReconciler.receiveComponent —— 在元素层面比较你的组件。所以元素比较过程中如果他们不一样或者context发生变化，内部实例层会调用receiveComponent。他不会被覆盖，但是他通常会调用updateComponent，其中包括了检查元素是否一样的逻辑。
 So elements instance are compared and if they’re not the same or the context changed there is a receiveComponent called on the level of an internal instance. It won’t get covered here, but it usually just calls updateComponent which contains logic for checking if elements are the same.

+ 如果有任何被挂起的state， this.updateComponent会被调用。

你也许会想为什么这些对挂起state或者强制更新的检查是必须的。从setState一开始state一定是被挂起的吗？不是的。updateComponent是递归的所以你会有挂起state队列是空的需要更新的组件。
_pendingElement是为了应对子组件中的发生改变的情况。
我们看下updateComponent的实现:

```javascript
  updateComponent: function(
    transaction,
    prevParentElement,
    nextParentElement,
    prevUnmaskedContext,
    nextUnmaskedContext
  ) {
    var inst = this._instance;

    var nextContext = this._context === nextUnmaskedContext ?
      inst.context :
      this._processContext(nextUnmaskedContext);
    var nextProps;

    // Distinguish between a props update versus a simple state update
    if (prevParentElement === nextParentElement) {
      // Skip checking prop types again -- we don't read inst.props to avoid
      // warning for DOM component props in this upgrade
      nextProps = nextParentElement.props;
    } else {
      nextProps = this._processProps(nextParentElement.props);
      // An update here will schedule an update but immediately set
      // _pendingStateQueue which will ensure that any state updates gets
      // immediately reconciled instead of waiting for the next batch.

      if (inst.componentWillReceiveProps) {
        inst.componentWillReceiveProps(nextProps, nextContext);
      }
    }

    var nextState = this._processPendingState(nextProps, nextContext);

    var shouldUpdate =
      this._pendingForceUpdate ||
      !inst.shouldComponentUpdate ||
      inst.shouldComponentUpdate(nextProps, nextState, nextContext);

    if (__DEV__) {
      warning(
        typeof shouldUpdate !== 'undefined',
        '%s.shouldComponentUpdate(): Returned undefined instead of a ' +
        'boolean value. Make sure to return true or false.',
        this.getName() || 'ReactCompositeComponent'
      );
    }

    if (shouldUpdate) {
      this._pendingForceUpdate = false;
      // Will set `this.props`, `this.state` and `this.context`.
      this._performComponentUpdate(
        nextParentElement,
        nextProps,
        nextState,
        nextContext,
        transaction,
        nextUnmaskedContext
      );
    } else {
      // If it's determined that a component should not update, we still want
      // to set props and state but we shortcut the rest of the update.
      this._currentElement = nextParentElement;
      this._context = nextUnmaskedContext;
      inst.props = nextProps;
      inst.state = nextState;
      inst.context = nextContext;
    }
  },
```

这是一个很大的方法。我们一步一步过一下。

+ 首先检查context是否变化。如果context发生变化了，被处理过的context被存到变量nextContext。_processContext主要负责这个，但是这边先不展开。
+ 然后updateComponent检查props是否变化或者仅仅是state发生变化。如果props变化了，生命周期中的componentWillReceiveProps会被调用，类似nextContext的生成方式新的props会被准备好。props没变化的话，nextProps直接指向了nextParentElement中的props(没有任何改变)。
+ 接着state进行处理。这是我们感兴趣的最后一块。_processPendingState负责这块的处理。
+ 检查组件是否需要更新虚拟DOM。如果state变化了，所有产生的next values会被传到_performComponentUpdate方法中。如果没有变化，变量只会就地更新。

_processPendingState的实现:

```javascript
  _processPendingState: function(props, context) {
    var inst = this._instance;
    var queue = this._pendingStateQueue;
    var replace = this._pendingReplaceState;
    this._pendingReplaceState = false;
    this._pendingStateQueue = null;

    if (!queue) {
      return inst.state;
    }

    if (replace && queue.length === 1) {
      return queue[0];
    }

    var nextState = assign({}, replace ? queue[0] : inst.state);
    for (var i = replace ? 1 : 0; i < queue.length; i++) {
      var partial = queue[i];
      assign(
        nextState,
        typeof partial === 'function' ?
          partial.call(inst, nextState, props, context) :
          partial
      );
    }

    return nextState;
  },
```
因为设置和替换state使用同一个队列，所以有个条件逻辑判断state是否正在被替换。如果是替换，挂起的state会和替换的state合并。如果不是替换的话，挂起的state则和当前的state合并。还有一个条件判断设置state的风格 —— setState的第一个参数可以是object也可以是个function，在处理挂起的state的时候会判断一下。调用replaceState更新全部之前的state，所以被替换的state都在队列的最前面。
## ReactUpdates.asap

ReactUpdate有个最重要的特性之前还没展开，就是ReactUpdate中的asap:

```javascript
function asap(callback, context) {
  invariant(
    batchingStrategy.isBatchingUpdates,
    'ReactUpdates.asap: Can\'t enqueue an asap callback in a context where' +
    'updates are not being batched.'
  );
  asapCallbackQueue.enqueue(callback, context);
  asapEnqueued = true;
}
```

在ReactUpdates的flushBatchedUpdates中用过:

```javascript
var flushBatchedUpdates = function() {
  // ReactUpdatesFlushTransaction's wrappers will clear the dirtyComponents
  // array and perform any updates enqueued by mount-ready handlers (i.e.,
  // componentDidUpdate) but we need to check here too in order to catch
  // updates enqueued by setState callbacks and asap calls.
  while (dirtyComponents.length || asapEnqueued) {
    if (dirtyComponents.length) {
      var transaction = ReactUpdatesFlushTransaction.getPooled();
      transaction.perform(runBatchedUpdates, null, transaction);
      ReactUpdatesFlushTransaction.release(transaction);
    }

    if (asapEnqueued) {
      asapEnqueued = false;
      var queue = asapCallbackQueue;
      asapCallbackQueue = CallbackQueue.getPooled();
      queue.notifyAll();
      CallbackQueue.release(queue);
    }
  }
};
```

这个仅仅在更新react中的input元素的时候被使用，来避免一些问题。调用回掉函数的策略基本是这样工作的:在全部更新之后，即使只捕捉到一个。asap使得当前的更新结束之后立即回调的调用 —— 所以当更新被捕捉到之后，他们需要等待asap的回调运行结束。

## 概要回顾

set state是一个很长的处理,其中包括了:

+ 调用setState将挂起的state变更塞入ReactUpdateQueue队列。
+ ReactUpdateQueue给组件内部的实例增加了额外的挂起state的入口，调用ReactUpdates执行他的任务。
+ ReactUpdates使用batchingStrategy确保所有的state的变更在事务中执行 —— 保证所有的更新都被应用。
+ flushBatchedUpdates负责按顺序地原子操作更新
+ ReactUpdatesFlushTransaction保证了捕获的更新都被适当的处理。
+ runBatchedUpdates负责将各种更新按照从父组件到子组件的顺序排序，并且调用ReactReconciler来更新组件。
+ performUpdateIfNecessary 负责检查是props的变化还是state的变化，然后调用updateComponent收集全部的变化。
+ updateComponent有区分更新类型的逻辑，然后检查是否有shouldComponentUpdate方法 —— 可能会停止虚拟DOM的更新。他还会按需调用组件生命周期中的各种方法(shouldComponentUpdate, componentWillReceiveProps, componentWillUpdate, componentDidUpdate)。
+ _processPendingState是有关将state的变更应用到组件上的。他可以区分state是设置还是替换，同时也能区分setState中第一个参数的类型(object还是function)。
+ asap是被用在input元素上，用来解决在调和阶段的一些小问题 —— 当前的更新执行完之后立即调用。

什么是你确定能从setState中取到的？

[原文链接](http://reactkungfu.com/2016/03/dive-into-react-codebase-handling-state-changes)
