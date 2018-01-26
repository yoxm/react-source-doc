# React setState的过程

我们首先从以下代码入手来观察setState函数的调用栈及setState函数的具体函数签名

```javascript
import React, { Component } from "react";

class App extends Component {
  constructor(porps) {
    super(porps);
    this.state = {
      text: "123"
    };
    this.onClick = this.onClick.bind(this);
  }

  onClick() {
    this.setState({
      text: "456"
    });
  }

  render() {
    let { text } = this.state;
    return (
      <div>
        <button onClick={this.onClick} className={"button"}>
          clickme
        </button>
        <p>{text}</p>
      </div>
    );
  }
}

export default App;

```

我们通过点击按钮，设置原本的state，将text 123 变为 456，我们观察以下具体的函数调用栈。
通过在chrome中的断点调试，我们发现setState的函数如下所示。

```javascript
Component.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

`setState`拥有一个回调函数，这个回调函数会在执行完setState后执行，所以我们可以在回调函数中拿到最新的state。进入setState内部，我们发现，setState实际上调用的是组件的updater属性的enqueueSetState方法，传入当前组件的实例， 需要设置的state，回调函数， 以及'setState‘标识。我们可以进一步进入enqueueSetState内部看一下。

```javascript
enqueueSetState: function (instance, partialState, callback) {
      var fiber = get(instance);
      callback = callback === undefined ? null : callback;
      {
        warnOnInvalidCallback(callback, 'setState');
      }
      var expirationTime = computeExpirationForFiber(fiber);
      var update = {
        expirationTime: expirationTime,
        partialState: partialState,
        callback: callback,
        isReplace: false,
        isForced: false,
        nextCallback: null,
        next: null
      };
      insertUpdateIntoFiber(fiber, update);
      scheduleWork(fiber, expirationTime);
    },
```

首先，我们通过我们传入的this, 获取到了这个实例对应的fiber对象。之后计算出fiber对象的优先级，创建此次更新的update对象，，利用`insertUpdateIntoFiber`将此次更新插入到对应的fiber对象中，之后利用`scheduleWork`进行一次任务调度，整个setState就到此为止了。但是这个react15又有何不同呢？我们可以从expirationTime来看看。

expirationTime, 顾名思义，就是失效时间，在React里表示更新任务的最晚执行时间，所谓的到期时间（ExpirationTime），是相对于调度器初始调用的起始时间而言的一个时间段；调度器初始调用后的某一段时间内，需要调度完成这项更新，这个时间段长度值就是到期时间值。在React 15的调度过程中，是使用priorityLevel这个优先级来区分。在16.x版本中使用的则是ExpirationTime的到期时间方式表示任务的优先级，可以更好的对任务进行切分，调度。

在computeExpirationForFiber函数中， 

```javascript
function computeExpirationForFiber(fiber) {
    var expirationTime = void 0;
    if (expirationContext !== NoWork) {
      // 明确的到期上下文被设置;
      expirationTime = expirationContext;
    } else if (isWorking) {
      if (isCommitting) {
        //在提交阶段发生的更新应该具有同步优先级。
        expirationTime = Sync;
      } else {
        // 呈现阶段中的更新应该与正在呈现的工作同时过期。
        expirationTime = nextRenderExpirationTime;
      }
    } else {
      // 没有明确的过期上下文，我们目前没有执行工作。 计算新的到期时间。
      if (useSyncScheduling && !(fiber.internalContextTag & AsyncUpdates)) {
        // 同步
        expirationTime = Sync;
      } else {
        // T异步
        expirationTime = computeAsyncExpiration();
      }
    }
    return expirationTime;
  }

```