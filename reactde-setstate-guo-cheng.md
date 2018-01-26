# React setState 的过程

这篇文章参考了http://blog.codingplayboy.com/2017/12/02/react_fiber/这篇博客,感谢他的分享，写的很好，很通透。

我们首先从以下代码入手来观察 setState 函数的调用栈及 setState 函数的具体函数签名

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

我们通过点击按钮，设置原本的 state，将 text 123 变为 456，我们观察以下具体的函数调用栈。通过在 chrome 中的断点调试，我们发现 setState 的函数如下所示。

```javascript
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, "setState");
};
```

`setState`拥有一个回调函数，这个回调函数会在执行完 setState 后执行，所以我们可以在回调函数中拿到最新的 state。进入 setState 内部，我们发现，setState 实际上调用的是组件的 updater 属性的 enqueueSetState 方法，传入当前组件的实例， 需要设置的 state，回调函数， 以及'setState‘标识。我们可以进一步进入 enqueueSetState 内部看一下。

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

首先，我们通过我们传入的 this, 获取到了这个实例对应的 fiber 对象。之后计算出 fiber 对象的优先级，创建此次更新的 update 对象，，利用`insertUpdateIntoFiber`将此次更新插入到对应的 fiber 对象中，之后利用`scheduleWork`进行一次任务调度，整个 setState 就到此为止了。但是这个 react15 又有何不同呢？我们可以从 expirationTime 来看看。

expirationTime, 顾名思义，就是失效时间，在 React 里表示更新任务的最晚执行时间，所谓的到期时间（ExpirationTime），是相对于调度器初始调用的起始时间而言的一个时间段；调度器初始调用后的某一段时间内，需要调度完成这项更新，这个时间段长度值就是到期时间值。在 React 15 的调度过程中，是使用 priorityLevel 这个优先级来区分。在 16.x 版本中使用的则是 ExpirationTime 的到期时间方式表示任务的优先级，可以更好的对任务进行切分，调度。

```javascript
export const NoWork = 0; // 没有任务等待处理
export const Sync = 1; // 同步模式，立即处理任务
export const Never = 2147483647; // Max int32: Math.pow(2, 31) - 1
const UNIT_SIZE = 10; // 过期时间单元（ms）
const MAGIC_NUMBER_OFFSET = 2; // 到期时间偏移量

// 以ExpirationTime特定单位（1单位=10ms）表示的到期执行时间
// 1 unit of expiration time represents 10ms.
export function msToExpirationTime(ms) {
  // 总是增加一个偏移量，在ms<10时与Nowork模式进行区别
  return ((ms / UNIT_SIZE) | 0) + MAGIC_NUMBER_OFFSET;
}
// 以毫秒表示的到期执行时间
export function expirationTimeToMs(expirationTime: ExpirationTime) {
  return (expirationTime - MAGIC_NUMBER_OFFSET) * UNIT_SIZE;
}
// 向上取整（整数单位到期执行时间）
// precision范围精度：弥补任务执行时间误差
function ceiling(num, precision) {
  return (((num / precision) | 0) + 1) * precision;
}

// 计算处理误差时间在内的到期时间
export function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs
) {
  return ceiling(
    currentTime + expirationInMs / UNIT_SIZE,
    bucketSizeMs / UNIT_SIZE
  );
}
```

上面代码提供的功能有：Sync：同步模式，在 UI 线程立即执行此类任务，如动画反馈等；异步模式：转换：到期时间特定单位和时间单位（ms）的相互转换；计算：计算包含允许误差在内的到期时间；在 computeExpirationForFiber 函数中，

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

1. 若当前处于任务提交阶段（更新提交至 DOM 渲染）时，设置当前 fiber 到期时间为 Sync，即同步执行模式
2. 若处于 DOM 渲染阶段时，则需要延迟此 fiber 任务，将 fiber 到期时间设置为下一次 DOM 渲染到期时间；
3. 若不在任务执行阶段，则需重新设置 fiber 到期时间
4. 若明确设置 useSyncScheduling 且 fiber.internalContextTag 值不等于 AsyncUpdates，则表明是同步模式，设置为 Sync
5. 否则，调用 computeAsyncExpiration 方法重新计算此 fiber 的到期时间，

通过这样的计算，我们得出此次任务的最晚到期时间，接下来构造了一个 update 的对象，然后调用 insertUpdateIntoFiber 方法，将此次更新插入 update 队列

```javascript
// 一个更新对应的数据结构
export type Update<State> = {
  expirationTime: ExpirationTime,
  partialState: PartialState<any, any>,
  callback: Callback | null,
  isReplace: boolean,
  isForced: boolean,
  next: Update<State> | null
};

// 更新队列，以单链表形式表示并持久化
// 调度一个更新任务时，将其添加至当前（current）fiber和work-in-progress fiber的更新队列中;
// 这两个更新队列相互独立但共享同一个持久化数据结构；
// work-in-progress更新队列通常是current fiber更新队列的子集；
// 发生调和时，更新任务从work-in-progress fiber更新队列移除，
// current fiber内的更新任务则保留，当work-in-progress中断时可以从current fiber恢复；
// 提交完更新时，work-in-progress fiber就会变成current fiber
//
export type UpdateQueue<State> = {
  // 若存在更早添加至队列的更新未被处理，
  // 则此已处理的更新并不会从队列中移除-先进先出原则
  // 所以需要维护baseState，代表第一个未处理的更新的基础状态，
  // 通常这就是队列中的第一个更新，因为在队列首部的已处理更新会被移除
  baseState: State,
  // 同理，需要维护最近的未处理的更新的到期时间，
  // 即未处理更新中到期时间值最小的
  expirationTime: ExpirationTime,
  first: Update<State> | null,
  last: Update<State> | null,
  callbackList: Array<Update<State>> | null,
  hasForceUpdate: boolean,
  isInitialized: boolean
};

// 添加更新至更新队列
export function insertUpdateIntoQueue<State>(
  queue: UpdateQueue<State>,
  update: Update<State>
) {
  // 添加更新至队列尾部
  if (queue.last === null) {
    // 队列为空
    queue.first = queue.last = update;
  } else {
    queue.last.next = update;
    queue.last = update;
  }
  if (
    queue.expirationTime === NoWork ||
    queue.expirationTime > update.expirationTime
  ) {
    // 更新最近到期时间
    queue.expirationTime = update.expirationTime;
  }
}
// 添加更新至fiber实例
export function insertUpdateIntoFiber<State>(
  fiber: Fiber,
  update: Update<State>
) {
  // 可以创建两个独立的更新队列
  // alternate主要用来保存更新过程中各版本更新队列，方便崩溃或冲突时回退
  const alternateFiber = fiber.alternate;
  let queue1 = fiber.updateQueue;
  if (queue1 === null) {
    // 更新队列不存在，则创建一个空的更新队列
    queue1 = fiber.updateQueue = createUpdateQueue(null);
  }

  let queue2;
  if (alternateFiber !== null) {
    // alternate fiber实例存在，则需要为此创建另一个更新队列
    queue2 = alternateFiber.updateQueue;
    // 如果版本池的fiber创建的队列为空，则创建一个空的更新队列。
    if (queue2 === null) {
      queue2 = alternateFiber.updateQueue = createUpdateQueue((null: any));
    }
  } else {
    queue2 = null;
  }
  // queue2和queue1相等则queue2为空。
  queue2 = queue2 !== queue1 ? queue2 : null;

  // 如果只存在一个更新队列
  if (queue2 === null) {
    insertUpdateIntoQueue(queue1, update);
    return;
  }

  // 如果任意更新队列为空，则需要将更新添加至两个更新队列
  if (queue1.last === null || queue2.last === null) {
    insertUpdateIntoQueue(queue1, update);
    insertUpdateIntoQueue(queue2, update);
    return;
  }

  // 如果2个更新队列均非空，则添加更新至第一个队列，并更新另一个队列的尾部更新项
  insertUpdateIntoQueue(queue1, update);
  queue2.last = update;
}

// 处理更新队列任务，返回新状态对象
export function processUpdateQueue<State>(
  current,
  workInProgress,
  queue,
  instance,
  props,
  renderExpirationTime
) {
  if (current !== null && current.updateQueue === queue) {
    // 克隆current fiber以创建work-in-progress fiber
    const currentQueue = queue;
    queue = workInProgress.updateQueue = {
      baseState: currentQueue.baseState,
      expirationTime: currentQueue.expirationTime,
      first: currentQueue.first,
      last: currentQueue.last,
      isInitialized: currentQueue.isInitialized,
      // These fields are no longer valid because they were already committed. Reset them.
      callbackList: null,
      hasForceUpdate: false
    };
  }

  // Reset the remaining expiration time. If we skip over any updates, we'll
  // increase this accordingly.
  queue.expirationTime = NoWork;

  let dontMutatePrevState = true;
  let update = queue.first;
  let didSkip = false;
  while (update !== null) {
    const updateExpirationTime = update.expirationTime;
    if (updateExpirationTime > renderExpirationTime) {
      // 此更新优先级不够，不处理，跳过
      if (
        queue.expirationTime === NoWork ||
        queue.expirationTime > updateExpirationTime
      ) {
        // 重新设置最近未处理更新的到期时间
        queue.expirationTime = updateExpirationTime;
      }
      update = update.next;
      continue;
    }

    // 优先级足够，处理
    let partialState;
    if (update.isReplace) {
      // 使用replaceState()直接替换状态对象方式更新时
      // 获取新状态对象
      state = getStateFromUpdate(update, instance, state, props);
      // 不需要合并至之前状态对象，标记为true
      dontMutatePrevState = true;
    } else {
      // 更新部分状态方式
      // 获取更新部分状态时的状态对象
      partialState = getStateFromUpdate(update, instance, state, props);
      if (partialState) {
        if (dontMutatePrevState) {
          // 上一次是替换状态，所以不能影响state
          state = Object.assign({}, state, partialState);
        } else {
          // 更新部分状态，直接将新状态合并至上一次状态
          state = Object.assign(state, partialState);
        }
        // 重置标记为false
        dontMutatePrevState = false;
      }
    }
    // 强制立即更新
    if (update.isForced) {
      queue.hasForceUpdate = true;
    }
    // 添加回调函数
    if (update.callback !== null) {
      // Append to list of callbacks.
      let callbackList = queue.callbackList;
      if (callbackList === null) {
        callbackList = queue.callbackList = [];
      }
      callbackList.push(update);
    }
    // 遍历下一个更新任务
    update = update.next;
  }
  // 返回最新的状态对象
  return state;
}
```

将更新队列插入完毕后，我们需要进行任务调度，主要的任务调度在 scheduleWork 中，

```javascript
function scheduleWork(fiber, expirationTime) {
  return scheduleWorkImpl(fiber, expirationTime, false);
}

function scheduleWorkImpl(fiber, expirationTime) {
  let node = fiber;
  while (node !== null) {
    // 向上遍历至根组件fiber实例，并依次更新expirationTime到期时间
    if (node.expirationTime === NoWork || node.expirationTime > expirationTime) {
      // 若fiber实例到期时间大于期望的任务到期时间，则更新fiber到期时间
      node.expirationTime = expirationTime;
    }
    // 同时更新alternate fiber的到期时间
    if (node.alternate !== null) {
      if (
        node.alternate.expirationTime === NoWork ||
        node.alternate.expirationTime > expirationTime
      ) {
        // 若alternate fiber到期时间大于期望的任务到期时间，则更新fiber到期时间
        node.alternate.expirationTime = expirationTime;
      }
    }
    // node.return为空，说明到达组件树顶部
    if (node.return === null) {
      if (node.tag === HostRoot) {
        // 确保是组件树根组件并获取FiberRoot实例
        const root = node.stateNode;
        // 请求处理任务
        requestWork(root, expirationTime);
      } else {
        return;
      }
    }
    // 获取父级组件fiber实例
    node = node.return;
  }
}
```

> React应用更新时，Fiber从当前处理节点，层层遍历至组件树根组件，然后开始处理更新，调用前面的requestIdleCallback等API执行更新处理。

主要调度逻辑实现在scheduleWork：

1. 通过fiber.return属性，从当前fiber实例层层遍历至组件树根组件；
2. 依次对每一个fiber实例进行到期时间判断，若大于传入的期望任务到期时间参数，则将其更新为传入的任务到期时间；
3. 调用requestWork方法开始处理任务，并传入获取的组件树根组件FiberRoot对象和任务到期时间；

requestWork代码实现逻辑如下

```javascript
// 当根节点发生更新时，调度器将调用requestWork方法开始任务处理过程
function requestWork(root: FiberRoot, expirationTime) {
  const remainingExpirationTime = root.remainingExpirationTime;
  if (remainingExpirationTime === NoWork ||
    expirationTime < remainingExpirationTime) {
    // 若任务剩余到期时间大于期望的任务到期时间，则需要更新
    root.remainingExpirationTime = expirationTime;
  }

  if (expirationTime === Sync) {
    // 同步
    performWork(Sync, null);
  } else {
    // 异步
    scheduleCallbackWithExpiration(expirationTime);
  }
}
```


