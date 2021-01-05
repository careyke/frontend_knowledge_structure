# 结合源码解析Scheduler的实现

前面在讲`Fiber`架构的时候提到，`React`在最新的`ConcurrentMode`中加入了**时间切片**和**优先级调度**的功能，用来进一步提高交互的流畅度。这两个功能的实现主要都是依赖于`Scheduler`模块。

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
const messageChannel = new MessageChannel();
```

