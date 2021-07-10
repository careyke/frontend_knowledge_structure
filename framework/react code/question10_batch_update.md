# 批量更新的实现原理

React技术栈的小伙伴在面试的时候应该都有遇到过这样一个问题：**多个setState是同步执行还是异步执行？**

大家在网上搜索答案的时候，往往都能搜索到这样的答案：**在React事件或者React生命周期方法中直接调用多个`setState`的时候，是批量更新，其他情况下是一个个更新**

实际上这只是对于`Legacy模式`而言的，换一个更全面的说话是：**在React应用的执行上下文中调用多个`setState`会批量更新，脱离了执行上下文的`setState`会一个个同步更新**

常见的**脱离执行上下文**的情况

1. 异步执行`setState`
2. 在原生的事件回调函数中执行`setState`



在介绍批量更新的实现之前，有必要先来了解一下React应用的执行上下文。



## 1. React应用的执行上下文

**React内部中使用执行上下文变量`executionContext`来表示当前处于应用执行的哪个阶段。**

```javascript
// Describes where we are in the React execution stack
let executionContext: ExecutionContext = NoContext;
```



### 1.1 executionContext的取值

执行上下文`executionContext`的取值有以下几个：

```javascript
export const NoContext = /*             */ 0b0000000;

// 执行批量执行API
const BatchedContext = /*               */ 0b0000001;

// 执行React事件
const EventContext = /*                 */ 0b0000010;

// 执行离散型React事件
const DiscreteEventContext = /*         */ 0b0000100;

// legacy模式下第一次渲染的时候会使用
const LegacyUnbatchedContext = /*       */ 0b0001000;

// render阶段
const RenderContext = /*                */ 0b0010000;

// commit阶段
const CommitContext = /*                */ 0b0100000;
```

**在某个阶段开始之前加上这个阶段对应的执行上下文的值，当这个阶段执行结束，去掉对应的执行上下文的值。**

> **离散型React事件**执行时会同时拥有`EventContext`和`DiscreteEventContext`这两个上下文



比如执行离散React事件时，调用`discreteUpdates`方法

```javascript
export function discreteUpdates<A, B, C, D, R>(
  fn: (A, B, C) => R,
  a: A,
  b: B,
  c: C,
  d: D,
): R {
  const prevExecutionContext = executionContext;
  executionContext |= DiscreteEventContext; // 增加对应上下文

  if (decoupleUpdatePriorityFromScheduler) {
    // ...不执行代码 省略
  } else {
    try {
      return runWithPriority(
        UserBlockingSchedulerPriority,
        fn.bind(null, a, b, c, d),
      );
    } finally {
      executionContext = prevExecutionContext; // 去除对应上下文
      if (executionContext === NoContext) {
        resetRenderTimer();
        flushSyncCallbackQueue();
      }
    }
  }
}
```



## 2. 批量更新的实现原理分析

通常情况下，我们要实现**将多个方法由原来的一个个执行变成批量执行**，实现的要点有两个：

1. **同步收集** — 调用方法的时候先不执行，而是放在内存中暂存起来
2. **异步执行** — 在下一个`tick`去执行暂存起来的所有方法

在React内部中也是使用这种思路来实现批量执行的。



整个过程发生在`ensureRootIsScheduled`方法中

```javascript
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // 给当前更新标记过期时间，如果过期了，当前更新被中断会将当前更新保存在expiredLane中，在后面消费
  markStarvedLanesAsExpired(root, currentTime);

  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // This returns the priority level computed during the `getNextLanes` call.
  const newCallbackPriority = returnNextLanesPriority();

  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
      root.callbackNode = null;
      root.callbackPriority = NoLanePriority;
    }
    return;
  }

  // Check if there's an existing task. We may be able to reuse it.
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    if (existingCallbackPriority === newCallbackPriority) {
      // 这里是异步批量更新的关键操作，所有模式都是在这里判断
      // 通过比较两次任务的优先级，来判断是否合并
      return;
    }
    cancelCallback(existingCallbackNode);
  }

  // Schedule a new callback.
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    newCallbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root),
    );
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    newCallbackNode = scheduleCallback(
      ImmediateSchedulerPriority,
      performSyncWorkOnRoot.bind(null, root),
    );
  } else {
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
      newCallbackPriority,
    );
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

### 2.1 同步收集

在创建`Update`的时候，会根据当前`Update`的优先级在`fiberRootNode.pendingLanes`中占领轨道，相同优先级的`Update`占领的轨道是一样的。这实际上就是一个收集的过程

### 2.2 异步执行

当收集完成之后，会借助`Scheduler`来异步批量执行同一优先级中所有的`Update`。`Concurrent`模式中加入了优先级调度的逻辑

如此就实现了多次调用setState的**批量更新**



## 3. Legacy模式下的非批量更新

不管是`Legacy`模式还是`Concurrent`模式，批量更新的实现都是使用上面介绍的原理来实现的。只是`Legecy`模式下所有`Update`的优先级都是一样的。

不过在`Legacy`模式中，有些情况下不能使用批量更新，所以在代码中借助执行上下文做了一些特殊处理。

> 这里应该是**渐进增强**的处理，后面会抛弃这些特殊处理

特殊处理主要分成两个部分：

1. 预留同步执行的入口
2. 根据执行上下文判断是否需要同步执行



### 3.1 预留同步执行的入口

在`Legacy`模式下，异步执行更新的时候会调用`scheduleSyncCallback`方法

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/SchedulerWithReactIntegration.new.js#L144)

```javascript
export function scheduleSyncCallback(callback: SchedulerCallback) {
  // sync优先级的update 会进入这个方法
  if (syncQueue === null) {
    syncQueue = [callback];
    immediateQueueCallbackNode = Scheduler_scheduleCallback(
      Scheduler_ImmediatePriority,
      flushSyncCallbackQueueImpl,
    );
  } else {
    syncQueue.push(callback);
  }
  return fakeCallbackNode;
}
```

上面代码中可以看出，**在`Legacy`模式下默认也会使用`Scheduler`来异步调度更新，在下一个tick来执行更新**

同时也有一个**同步执行**的方法`flushSyncCallbackQueue`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/SchedulerWithReactIntegration.new.js#L168)

```javascript
export function flushSyncCallbackQueue() {
  if (immediateQueueCallbackNode !== null) {
    const node = immediateQueueCallbackNode;
    immediateQueueCallbackNode = null;
    Scheduler_cancelCallback(node);
  }
  flushSyncCallbackQueueImpl();
}

function flushSyncCallbackQueueImpl() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    isFlushingSyncQueue = true;
    let i = 0;
    if (decoupleUpdatePriorityFromScheduler) {
      // ...不执行代码 省略
    } else {
      try {
        const isSync = true;
        const queue = syncQueue;
        runWithPriority(ImmediatePriority, () => {
          for (; i < queue.length; i++) {
            let callback = queue[i];
            do {
              callback = callback(isSync);
            } while (callback !== null);
          }
        });
        syncQueue = null;
      } catch (error) {
        if (syncQueue !== null) {
          syncQueue = syncQueue.slice(i + 1);
        }
        // Resume flushing in the next tick
        Scheduler_scheduleCallback(
          Scheduler_ImmediatePriority,
          flushSyncCallbackQueue,
        );
        throw error;
      } finally {
        isFlushingSyncQueue = false;
      }
    }
  }
}
```

这里有一个细节需要**注意**一下：

React将所有**同步的更新**都收集保存在`syncQueue`中，然后提供了一个执行所有同步更新的方法`flushSyncCallbackQueue`。

> `flushSyncCallbackQueue`这个方法在很多地方都有使用，目的就是用来**同步执行所有的同步更新（sync）**

> 补充：
>
> **这里同步的更新只的就是`SyncLanePriority`优先级的`update`，其他优先级的`update`并不能被`flushSyncCallbackQueue`方法执行**



对于**同步的更新，React有两种处理方式**：

1. 默认情况下和其他优先级的更新一样，借助`Scheduler`来异步调度，在下一个`tick`执行

2. 直接调用`flushSyncCallbackQueue`，在当前`tick`执行



### 3.2 根据执行上下文判断是否需要同步执行

在`update`阶段的入口方法`scheduleUpdateOnFiber`中，有这么一段代码

```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // ...省略

  if (lane === SyncLane) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // legacy模式下第一次执行react应用
      schedulePendingInteractions(root, lane);
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      if (executionContext === NoContext) {
        // 如果没有React的上下文，直接执行更新
        // 可以看出Legacy模式中，是根据上下文来判断是都可以异步合并更新
        resetRenderTimer();
        flushSyncCallbackQueue(); // 同步执行同步更新
      }
    }
  } else {
    // ...省略
  }
}
```

在`Legacy`模式中，当收集`Update`之后，如果已经不在React执行上下文中，就会调用`flushSyncCallbackQueue`来同步执行更新。

如果将`setState`放在`setTimeout`中调用，就会脱离React执行上下文，就会同步执行。

> 同步执行的例子可以看[这里](https://codesandbox.io/s/lagecyreact-8ig9z?file=/src/App.js)



`Legacy`模式中，**React事件处理中也是使用`flushSyncCallbackQueue`来同步执行，但是执行的时机是在事件回调函数完成之后再执行，所以可以达到批量执行的效果**

> 对应的代码在 [discreteUpdates](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1188) 中



## 4. 总结

React内部对于批量更新的实现步骤如下：

1. **根据`Update`的优先级来收集`Update`，暂存起来**
2. **根据优先级调度来批量执行相同优先级的`Update`**

但是其中有两个特例：

1. 在`Legacy`模式下，如果`Update`脱离执行上下文，会**立即同步执行**
2. React事件回调中创建的**同步`Update`**，会在当前tick执行



所以：

1. **对于`Concurrent`模式来说，是否批量执行取决于`Update`的优先级。**
2. **对于`Legacy`模式来说，是否批量执行取决于当前所在的执行上下文**。因为`Legacy`模式下所有的`Update`优先级是一样的



分析一下例子的执行情况：

DEMO1：

```react
export default class BatchUpdate extends React.Component {
  state = {
    count: 0
  };

  onClick = () => {
    setTimeout(() => {
      this.setState((state) => {
        return { count: state.count + 2 };
      });
      this.setState((state) => {
        return { count: state.count + 2 };
      });
    }, 0);
  };

  render() {
    console.log("render", this.state.count);

    return (
      <div>
        <button ref={this.buttonRef} onClick={this.onClick}>
          增加2
        </button>
      </div>
    );
  }
}
```

1. 在Legacy模式下，会输出两次 “render”

2. 在Concurrent模式下，只会输出一次 “render”，优先级是相同的

> 对应的例子可以看[这里](https://codesandbox.io/s/reactcodeexamples-el2gu?file=/src/BatchUpdate.js)

