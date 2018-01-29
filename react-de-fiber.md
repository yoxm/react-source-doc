# React的Fiber

首先观察以下Fiber的结构

```javascript
// 一个Fiber对象作用于一个组件
export type Fiber = {|
  // 标记fiber类型tag.
  tag: TypeOfWork,
  // fiber对应的function/class/module类型组件名.
  type: any,
  // fiber所在组件树的根组件FiberRoot对象
  stateNode: any,
  // 处理完当前fiber后返回的fiber，
  // 返回当前fiber所在fiber树的父级fiber实例
  return: Fiber | null,
  // fiber树结构相关链接
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,

  // 当前处理过程中的组件props对象
  pendingProps: any,
  // 缓存的之前组件props对象
  memoizedProps: any, // The props used to create the output.
  // The state used to create the output
  memoizedState: any,

  // 组件状态更新及对应回调函数的存储队列
  updateQueue: UpdateQueue<any> | null,


  // 描述当前fiber实例及其子fiber树的数位，
  // 如，AsyncUpdates特殊字表示默认以异步形式处理子树，
  // 一个fiber实例创建时，此属性继承自父级fiber，在创建时也可以修改值，
  // 但随后将不可修改。
  internalContextTag: TypeOfInternalContext,

  // 更新任务的最晚执行时间
  expirationTime: ExpirationTime,

  // fiber的版本池，即记录fiber更新过程，便于恢复
  alternate: Fiber | null,

  // Conceptual aliases
  // workInProgress : Fiber ->  alternate The alternate used for reuse happens
  // to be the same as work in progress.
|};
```

1. tag & key：同React元素的值；
2. type：描述fiber对应的React组件；
    * 对于组合组件：值为function或class组件本身；
    * 对于原生组件（div等）：值为该元素类型字符串；
3. key：调和阶段，标识fiber，以检测是否可重用该fiber实例；
4. child & sibling：组件树，对应生成fiber树，类比的关系；
5. pendingProps & memoizedProps：分别表示组件当前传入的及之前的props；
6. return：返回当前fiber所在fiber树的父级fiber实例，即当前组件的父组件对应的fiber；
7. alternate：fiber的版本池，即记录fiber更新过程，便于恢复重用；
8. workInProgress：正在处理的fiber，概念上叫法，实际上没有此属性；

## tag

首先从tag 的 typeOfWork的类型来说起。

```javascript
export type TypeOfWork = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10;

export const IndeterminateComponent = 0; // 尚不知是类组件还是函数式组件
export const FunctionalComponent = 1; // 函数式组件
export const ClassComponent = 2; // Class类组件
export const HostRoot = 3; // 组件树根组件，可以嵌套
export const HostPortal = 4; // 子树. Could be an entry point to a different renderer.
export const HostComponent = 5; // 标准组件，如地div， span等
export const HostText = 6; // 文本
export const CallComponent = 7; // 组件调用
export const CallHandlerPhase = 8; // 调用组件方法
export const ReturnComponent = 9; // placeholder（占位符）
export const Fragment = 10; // 片段
```

React会根据tag的值de不同来不同的执行任务。

## type

然后是type，他是fiber对应的function/class/module的类名。

## stateNode

stateNode对应的是FiberRoot对象，

```javascript
export type FiberRoot = {
  // fiber节点的容器元素相关信息，通常会直接传入容器元素
  containerInfo: any,
  // 当前fiber树中激活状态（正在处理）的fiber节点，
  current: Fiber,
  // 此节点剩余的任务到期时间
  remainingExpirationTime: ExpirationTime,
  // 更新是否可以提交
  isReadyForCommit: boolean,
  // 准备好提交的已处理完成的work-in-progress
  finishedWork: Fiber | null,
  // 多组件树FirberRoot对象以单链表存储链接，指向下一个需要调度的FiberRoot
  nextScheduledRoot: FiberRoot | null,
};
```

之后是return 它总是返回当前fiber所在fiber树的父级fiber实例，而其他的children,sibling,index是对应fiber树的结构，

## updateQueue

updateQueue,对应当前组件的更新队列。

## expirationTime

expirationTime:expirationTime, 顾名思义，就是失效时间，在 React 里表示更新任务的最晚执行时间，所谓的到期时间（ExpirationTime），是相对于调度器初始调用的起始时间而言的一个时间段；调度器初始调用后的某一段时间内，需要调度完成这项更新，这个时间段长度值就是到期时间值。在 React 15 的调度过程中，是使用 priorityLevel 这个优先级来区分。在 16.x 版本中使用的则是 ExpirationTime 的到期时间方式表示任务的优先级，可以更好的对任务进行切分，调度。

```javascript
// ExpirationTime的定义
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
5. 否则，调用 computeAsyncExpiration 方法重新计算此 fiber 的到期时间。

## ALTERNATE FIBER

可以理解为一个fiber版本池，用于交替记录组件更新（切分任务后变成多阶段更新）过程中fiber的更新，因为在组件更新的各阶段，更新前及更新过程中fiber状态并不一致，在需要恢复时（如，发生冲突），即可使用另一者直接回退至上一版本fiber。