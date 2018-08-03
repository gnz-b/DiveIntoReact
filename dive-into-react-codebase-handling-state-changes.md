# 深入React源码：处理state的变化
[原文链接](http://reactkungfu.com/2016/03/dive-into-react-codebase-handling-state-changes)

state是React.js术语中最复杂的概念之一。
虽然我们在项目中通过集中式的方法（Redux或者其他的？）来处理state，但它任然是React.js中被广泛应用的特性。

使用state带来方便的同时也会带来一些问题。Rails meets React.js的作者之一，Robert Pankowecki刚开始使用React时，
在做表单验证的时候就遇到了问题。

事情是这样的：验证功能看上去很容易，但是form有个问题，用户第一次看到的输入框是无需验证的，即使该输入框的值是非法的。
这显然是个stateful的操作，所以Robert得出结论把验证消息放在state中会是一个不错的做法。
所以他实现的基本逻辑如下：

```javascript
  // ...
  changeTitle: function changeTitle (event) {
    this.setState({ title: event.target.value });
    this.validateTitle();
  },
  validateTitle: function validateTitle () {
    if(this.title.length === 0) {
      this.setState({ titleError: "Title can't be blank" });
    }
  },
  // ...
```

但是上面的并没有起作用。为什么？因为setState生效是异步的。这说明当调用setState之后变量this.state没有立即变化。在文档的“注意”章节中有描述：

```
  setState()不会立即改变this.state而是生成一个挂起的state事务。在调用完之后马上获取this.state可能会返回之前的值。
```

这不是一个怪异的情况，而是对setState的误解或是相关知识的欠缺。
Robert可以通过阅读文档来避免他的错误。
但是你不得不承认这是初学者很容易犯的错误。

以上的情况引发了队伍内部有趣的讨论。你预期从setState中获取什么？你一定能获取什么？如果你改变了输入框地值并且立即点击了提交按钮，为什么你能确保stateli将被正确地更新？我决定跟踪当你调用setState之后背后发生的变化，为了理解后续将发生什么，并开始一段有趣的React内部之旅。但是首先我们用个合适的方法来解决Robert的问题。

## 解决验证问题

我们一起看看在React 0.14.7中有什么其他选项可以解决上述的问题。

你可以使用setState内嵌的callback的能力。setState接受两个参数——第一个是state的变化值，第二个是callback，当state更新完成之后会调用callback：

```javascript
changeTitle: function changeTitle (event) {
  this.setState({ title: event.target.value }, function afterTitleChange () {
    this.validateTitle();
  });
},
// ...
```
afterTitleChange中state已经更新完了，所以你能确保this.state中取到的是最新值。


+ 你可以合并state的变更,为了合并你必须将validateTitle参数化，将两次state的更新合并成一次：

```javascript
  changeTitle: function changeTitle (event) {
    let nextState = Object.assign({},
                                  this.state,
                                  { title: event.target.value });
    this.validateTitle(nextState);
    this.setState(nextState);
  },
  validateTitle: function validateTitle (state) {
    if(state.title.length === 0) {
      state.titleError = "Title can't be blank";
    }
  }
```
Robert的问题解决了，因为没有更新state两次。当你state变得越来越大并且是嵌套的这就会变的复杂起来（这不是一个很好的实践！）。
把validateTitle变成一个纯函数——函数的返回值只依赖于传入的参数并且他不会改变其他任何东西，也是个不错的解决办法：

```javascript
  changeTitle: function changeTitle (event) {
    let nextState = Object.assign({},
                                  this.state,
                                  { title: event.target.value });
    this.setState(this.withTitleValidated(nextState));
  },
  withTitleValidated: function validateTitle (state) {
    let validation = {};
    if(state.title.length === 0) {
      validation.titleError = "Title can't be blank";
    }
    return Object.assign({}, state, validation);
  }
```
和之前的方法相比，这可能会更浪费性能。幸亏可以配置shouldComponentUpdate，如果有自定义的shouldComponentUpdate，一些state的更新可以忽略就不会触发componentDidUpdate，因此validateTitle也不会被调用。

 + 你也可以将state抽离出来，把验证放在app的其他部分（例如Redux中的reducer）。通过完全不使用state的方式来避免React层面的问题。

书中Robert采用了将state合并到一起的解决方法（上面的第二种）。这趟冒险让他更加坚信了要把state放到react的component之外，从而可以避免考虑他们。但这是有效的方法来抵制state？我们一起往下看。。

*如果你想跳过react代码一步一步的详解，你可以直接跳到回顾章节*

## React的内部构造 —— 深入兔子的洞穴

setState被定义在ReactComponent的prototype上，ReactComponent是React.Component类，使用ES2015类的定义React组件的时候都会继承React.Component。
通过React.createClass创建的组件都能使用setState，而且是同样的代码。React.createClass返回组件的prototype都是ReactClassComponent，
基本如下：

```javascript
  var ReactClassComponent = function() {};
  assign(
    ReactClassComponent.prototype,
    ReactComponent.prototype,
    ReactClassMixin
  );
```

如上，ReactComponent的prototype在这儿 —— 说明setState从这开始可以使用（只要ReactClassMixin没有做什么奇怪的操作 —— 当然他也没有）。
我们来看一下实现：

```javascript
  ReactComponent.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
    typeof partialState === 'function' ||
    partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
    'function which returns an object of state variables.'
  );
  if (__DEV__) {
    warning(
      partialState != null,
      'setState(...): You passed an undefined or null state object; ' +
      'instead, use forceUpdate().'
    );
  }
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback);
  }
};
```

除了检查非变量和有警告的issue，这做了两件事情：

+ setState被塞进update的队列来做他的工作。

+ 如果存在callback，它作为setState的第二个参数被塞进队列。晚些你会明白这么做的原因。

但是什么是updater？这名字暗示了他和更新组件有些联系。让我们看下他在ReactClass和ReactComponent哪儿被定义的：

```javascript
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
```

React.js源码重度依赖依赖注入原则。
这样允许根据不同的环境(服务端VS客户端，不同的平台)，可以替换React.js的某些部分。
ReactComponent是同构命名空间的一部分 —— 无论在React Native，浏览器上的ReactDOM或者是服务端，都有ReactComponent。
他仅包含JavaScript代码，这样可以保证他能够运行在所有可以解析ES5标准的js的设备。

所以真正的updater在哪儿被注入？在ReactCompositeComponent的renderer部分（mountComponent方法）：
```javascript
  // These should be set up in the constructor, but as a convenience for
  // simpler class abstractions, we set them up after the fact.
  inst.props = publicProps;
  inst.context = publicContext;
  inst.refs = emptyObject;
  inst.updater = ReactUpdateQueue;
```
ReactCompositeComponent类被用到各种React中（react-dom, react-native, react-art）来构建一个存在于每个React组件中不依赖环境的基础。
这是使用事务的先决条件，例如react-dom客户端中的ReactMount —— 依赖于平台的代码在这里运行，包上能够正确设置不依赖于平台内部细节的事务。

既然明白了什么是updater，我们看下enqueueSetState和enqueueCallback是如何实现的。

## 把state的更新和callback都放入队列 —— ReactUpdateQueue

在setState中会调用两个方法：enqueueSetState和enqueueCallback。如你所见，React使用ReactUpdateQueue的实例来实现这两个方法。下面看下他们的实现:

enqueueSetState:

```javascript
  enqueueSetState: function(publicInstance, partialState) {
    var internalInstance = getInternalInstanceReadyForUpdate(
      publicInstance,
      'setState'
    );

    if (!internalInstance) {
      return;
    }

    var queue =
      internalInstance._pendingStateQueue ||
      (internalInstance._pendingStateQueue = []);
    queue.push(partialState);

    enqueueUpdate(internalInstance);
  },
```

enqueueCallback:

```javascript
enqueueCallback: function(publicInstance, callback) {
  invariant(
    typeof callback === 'function',
    'enqueueCallback(...): You called `setProps`, `replaceProps`, ' +
    '`setState`, `replaceState`, or `forceUpdate` with a callback that ' +
    'isn\'t callable.'
  );
  var internalInstance = getInternalInstanceReadyForUpdate(publicInstance);

  // Previously we would throw an error if we didn't have an internal
  // instance. Since we want to make it a no-op instead, we mirror the same
  // behavior we have in other enqueue* methods.
  // We also need to ignore callbacks in componentWillMount. See
  // enqueueUpdates.
  if (!internalInstance) {
    return null;
  }

  if (internalInstance._pendingCallbacks) {
    internalInstance._pendingCallbacks.push(callback);
  } else {
    internalInstance._pendingCallbacks = [callback];
  }
  // TODO: The callback here is ignored when setState is called from
  // componentWillMount. Either fix it or disallow doing so completely in
  // favor of getInitialState. Alternatively, we can disallow
  // componentWillMount during server-side rendering.
  enqueueUpdate(internalInstance);
},
```

两个方法都引用了enqueueUpdate，我们稍后会深入研究。套路如下:

+ 首先取到内部的实例。你代码中的react组件都有一个内部的实例里面放着一些私有的方法。可以通过ReactInstanceMap中getInternalInstanceReadyForUpdate获取这些内部实例。
+ 对内部实例进行修改。当有callback的时候，callback被加入到挂起的callback队列。有state变更的时候，将state的变更插入到挂起的state队列里。
+ 调用enqueueUpdate来一把刷新这些方法产生的变化。看下他是如何实现的。

enqueueUpdate:

```javascript
function enqueueUpdate(internalInstance) {
  ReactUpdates.enqueueUpdate(internalInstance);
}
```
好吧，另外一个谜题！这很有趣，这层ReactUpdates是直接引用的而不是注入的。因为ReactUpdates是非常通用的，他的依赖都是注入的。接下来
我们看ReactUpdates是如何工作的。

## 开始更新 —— ReactUpdates 

我们看下enqueueUpdate在ReactUpdates中是如何实现的:

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

我们跳到上一部分 - ReactReconciler

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

我们跳到上一部分 - ReactReconciler。

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

## 一些确定的东西

+ 可以保证state的更新是按顺序的。这意味着先设置{ a:2 }，再设置{ a:3 }，最终结果state中a的值是3。
+ 同一context下调用两次setState，不能保证会更新DOM两次。事实上这是不可能的。
+ 不能保证调用setState之后state立即就被刷新。必须使用setState的回调函数(第二个参数)或者在组件的生命周期中使用state(componentDidUpdate)。
+ 可以确保callback的调用是按顺序的。
+ 不能保证setState中的callback执行时，state中的值正好就是第一个参数中的值。例如setState({ a:2 }, callback1), setState({ a:3 }, callback2)的时候，当运行callback1的时候，state中的a可能已经是3了。
+ 可以确保在你处理下个事件之前时，state的更新已经执行完毕了。ReactReconcileTransaction中的EVENT_SUPPRESSION wrapper会处理这个问题。

## 总结

跟踪setState如何工作，是展示react源码中如何运用事务的。洞察到像管理同步流程一样管理这些异步的流程。由里及表的了解到React是如何实现的，同时在未来可能会避免一些明显的问题，会让你成为一个更好的React程序员。这不是很酷，也是你所好奇的？


