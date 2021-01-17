# 结合源码解析Scheduler的实现

前面在讲`Fiber`架构的时候提到，`React`在最新的`ConcurrentMode`中加入了**时间切片**和**优先级调度**的功能，用来进一步提高交互的流畅度。这两个功能的实现都会依赖于`Scheduler`模块。

**`Scheduler`模块的主要功能就是模拟`requestIdleCallback`在当前帧的空余时间来调度任务，顺便加入了优先级的概念。**

至于为什么不直接使用`requestIdleCallback`，前面也介绍过主要是因为`requestIdleCallback`**触发的频率很不稳定**而且还有**兼容性**问题。

## 1. MessageChannel模拟实现requestIdleCallback

在`Scheduler`源码中是使用`MessageChannel`来**异步调度任务，确保触发频率的稳定性**。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/scheduler/src/forks/SchedulerDOM.js#L546)

```javascript
// ...省略
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
// ...省略
```

当任务队列中含有未执行的任务时，会触发一个`宏任务`来异步执行这些任务。

但是如果这些任务耗时很长的话，当前帧就没有时间去执行页面渲染，会造成页面的卡顿。

`requestIdleCallback`一个最重要的特点就是不会阻塞页面的渲染，如果当前帧有空余时间才会去执行。光用`MessageChannel`并不能达到`requestIdleCallback`类似的效果，那么`Scheduler`是如何来模拟实现这种效果呢？

**`Scheduler`采用了一种很巧妙的方式来模拟实现类似的效果，就是在每帧中约定一个较短的固定时间用来执行任务队列，时间长度是`5ms`，如果超过了该时间，会重新开启一个宏任务执行后面的任务。由于每个宏任务结束之后，浏览器会尝试执行一次`UI`渲染，如果需要执行UI渲染，会在当前帧的剩余时间执行渲染。**

> 理想情况下每帧的时间是`16.7ms`

### 1.1 模拟实现的关键要点

1. 每帧中的宏任务执行完成之后，如果需要，会执行一次UI渲染
2. 每帧中利用`5ms`的时间来执行任务，时间用完之后会尝试将主线程交给UI渲染线程，未执行的任务会在下一个宏任务来执行

### 1.2 模拟实现的优点

1. 使用宏任务代替`requestIdleCallback`来调度任务，能保证触发频率的稳定性
2. 在每帧中固定执行的时间，最大程度上实现了**空闲时间调度**的效果

> `Scheduler`中是先执行任务，然后再将主线程交给UI渲染线程
>
> `requestIdleCallback`中则是先执行UI渲染线程，然后判断当前帧是否有剩余时间，如果有剩余时间再执行任务

### 1.3 为什么使用MessageChannel，不是使用`setTimeout(fn,0)`

虽然`MessageChannel`和`setTimeout`都是宏任务，但是**`MessageChannel`触发的优先级比`setTimeout`更高**。

```javascript
setTimeout(()=>{
  console.log('setTimeout');
},0)
const messageChannel = new MessageChannel();
const port = messageChannel.port1;
messageChannel.port2.onmessage = ()=>{
  console.log('messagechannel');
}

port.postMessage(null);

// 运行结果
// messagechannel
// setTimeout
```

`setTimeout(fn,0)`虽然语义上立马将`fn`插入宏任务队列，但是实际上由于其内部有一些运算逻辑，比稍微慢一点，大约是`2ms`。



## 2. 任务队列

由于任务队列是动态变化的，而且每次变化之后都需要重新计算出优先级最高的任务，所以非常适合用**小顶堆**来实现。

> 关于堆这个结构可以看[这里](https://github.com/careyke/frontend_knowledge_structure/blob/master/algorithm/heap/question01_heap.md)

实际上`Scheduler`内部维护了两个小顶堆，分别用来保存**正在执行的任务队列（taskQueue）**和**延时执行的任务队列（timerQueue）**

```javascript
var taskQueue = []; // 正在运行的任务队列
var timerQueue = []; // 延时运行的任务队列
```

延时任务到达执行时间的时候，会从`timerQueue`进入`taskQueue`。

### 2.1 任务注册

任务注册的方法是`unstable_scheduleCallback`，`React`中需要通过`Scheduler`来调度的任务最终都会调用这个方法

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/scheduler/src/forks/SchedulerDOM.js#L315)

```javascript
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  // 任务优先级
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  // 截止时间，防止任务饿死
  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };

  if (startTime > currentTime) {
    // 处理延时任务
    // timerQueue中任务的排序属性采用startTime
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      if (isHostTimeoutScheduled) {
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 处理非延时任务
    // taskQueue中任务的排序属性采用expirationTime
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    // 判断是否开始调度任务
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

从这个方法中可以获取很多信息，下面来一一分析。

#### 2.1.1 任务的数据结构

```javascript
var newTask = {
    id: taskIdCounter++, // 任务编号，递增生成
    callback, // 任务回调函数
    priorityLevel, // 任务优先级
    startTime, // 任务开始调度时间
    expirationTime, // 任务过期时间
    sortIndex: -1, // 任务排序属性
};
```

#### 2.1.2 任务的优先级

在`Scheduler`内部预定了5种优先级

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/scheduler/src/SchedulerPriorities.js#L13)

```javascript
export const NoPriority = 0;
export const ImmediatePriority = 1; // 立即执行的任务
export const UserBlockingPriority = 2; // 用户交互的任务，比如输入框，按钮点击
export const NormalPriority = 3; // 正常任务，大多数任务
export const LowPriority = 4; // 低优先级的任务
export const IdlePriority = 5; // 最低优先级的任务
```

其中`NoPriority`和`NormalPriority`处理的逻辑是一样的。

#### 2.1.3 任务的过期时间

如果任务排序的时候完全按照优先级来排序，那么在某些情况下**低优先级的任务会很长时间得不到执行**，也就是常说的**“任务饿死”**现象。

为了解决这个问题，**`Scheduler`内部给每个任务根据优先级设置了一个过期时间（expirationTime），然后在对`taskQueue`中的任务排序的时候会按照任务的过期时间来排，这样就可以防止“任务饿死”的现象。**

对应的处理逻辑在上面代码中



### 2.2 任务调度

开启任务调度的方法是`requestHostCallback`，

```javascript
function requestHostCallback(callback) {
  // 目前代码中callback都是flush方法
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 开启一次宏任务，在下一个tick中执行任务调度
    port.postMessage(null);
  }
}
```

任务调度的入口方法是`performWorkUntilDeadline`

```javascript
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    
    // 和浏览器约定，每帧只给5ms来执行任务
    deadline = currentTime + yieldInterval;
    const hasTimeRemaining = true;
    try {
      const hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
      if (!hasMoreWork) {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // 任务队列中，放到下一帧执行
        port.postMessage(null);
      }
    } catch (error) {
      port.postMessage(null);
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
  // 这个方法运行完之后会将控制权交给UI线程，所以可以重制状态
  needsPaint = false;
};
```

上面代码可以看出，每个宏任务的运行时间由`yieldInterval`决定，默认值是`5ms`。

> `Scheduler`向外暴露了一个`api`用来让用户根据当前机器的帧率来修改这个时间 —— `forceFrameRate`

#### 2.2.1 任务执行

任务执行发生在`workLoop`方法中，调用的方法路径是`performWorkUntilDeadline -> flush —> workLoop`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/scheduler/src/forks/SchedulerDOM.js#L200)

```javascript
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  // 尝试将timerQueue中达到开始时间的任务放到taskQueue中
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
     	// 当前宏任务已经达到约定的deadline
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // 执行任务的回调函数
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // 任务执行中断，下次调度
        currentTask.callback = continuationCallback;
      } else {
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
  // Return whether there's additional work
  if (currentTask !== null) {
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

上面代码中有三个关键点：

##### 2.2.1.1 延时任务切换队列

**当延时任务的开始时间小于当前时间的时候，表示任务可以执行，需要进入`taskQueue`**。这个过程发生在`advanceTimers`方法中

```javascript
function advanceTimers(currentTime) {
  let timer = peek(timerQueue);
  while (timer !== null) {
    if (timer.callback === null) {
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // 将timerQueue中的超时任务放入taskQueue中
      pop(timerQueue);
      timer.sortIndex = timer.expirationTime;
      push(taskQueue, timer);
    } else {
      return;
    }
    timer = peek(timerQueue);
  }
}
```

##### 2.2.1.2 中断调度

上面代码中，在每个任务开始执行之前，都会判断是否需要中断本次调度。中断要同时满足两个条件：

1. **当前任务还没有过期**：过期的任务必须在当前帧执行
2. **判断宏任务是否超时**：判断逻辑在`shouldYieldToHost`方法中

```javascript
function shouldYieldToHost() {
  if (
    enableIsInputPending &&
    navigator !== undefined &&
    navigator.scheduling !== undefined &&
    navigator.scheduling.isInputPending !== undefined
  ) {
    // 判断当前浏览器是否存在新的api isInputPending
    // 暂时浏览器都还不支持，enableIsInputPending也写死为false
    const scheduling = navigator.scheduling;
    const currentTime = getCurrentTime();
    if (currentTime >= deadline) {
      if (needsPaint || scheduling.isInputPending()) {
        return true;
      }
      return currentTime >= maxYieldInterval;
    } else {
      return false;
    }
  } else {
    // 判断是否达到deadline
    return getCurrentTime() >= deadline;
  }
}
```

##### 2.2.1.3 兼容任务中断

上面的中断只是执行每个任务之前判断是否需要中断，是建立在每个任务都不耗时的基础上，但是如果某个任务非常复杂，耗时很长的话，也是会导致页面卡顿的。比如React的`render`过程

`Scheduler`为了解决这个问题，允许你在任务执行的过程中打断任务的执行，然后在下一帧恢复执行。

```javascript
currentTask.callback = null;
currentPriorityLevel = currentTask.priorityLevel;
const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
// 执行任务的回调函数
const continuationCallback = callback(didUserCallbackTimeout);
currentTime = getCurrentTime();
if (typeof continuationCallback === 'function') {
    // 任务执行中断，下次调度
    currentTask.callback = continuationCallback;
} else {
    if (currentTask === peek(taskQueue)) {
        pop(taskQueue);
    }
}
```

实现的思路：**任务执行被打断的时候，重新返回一个回调函数给当前任务，表示当前任务没有执行完，依然放在在taskQueue中，下一帧调度任务的时候再执行。**

`React Fiber`中的**时间切片**就是借助这个功能来实现的。

```javascript
function workLoopConcurrent() {
  // 打断执行
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performConcurrentWorkOnRoot(root) {
  // ...省略
  if (root.callbackNode === originalCallbackNode) {
    // 恢复执行
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

> 这里的`shouldYield`方法对应Scheduler中的`shouldYieldToHost`方法

任务执行过程中是否中断取决于开发者，`Scheduler`在实现层面支持了任务的中断



## 3. Scheduler与React

在`React FIber`架构加入了`Scheduler`，在`ConcurrentMode`中完全开启了`Scheduler`的功能，实现了**时间切片**和**优先级调度**的特性。`Scheduler`在`React`中扮演的是一个大脑的角色，每一次更新都需要先经过`Scheduler`，由`Scheduler`来统一调度执行，调度的过程中会优先执行高优先级的更新任务，达到最好的用户体验。

