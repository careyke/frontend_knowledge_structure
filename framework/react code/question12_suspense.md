# 结合React源码分析Suspense的实现原理

前面的文章中我们提到过，React为了实现**快速响应**分别在`CPU`和`I/O`两个方面进行了优化。

1. CPU方面 - React提出了Fiber架构，实现了时间切片和优先级调度。具体的实现可以看前面的文章
2. I/O方面 - React实现`Suspense`以及配套的`hooks`，用来解决I/O方面的瓶颈。

这篇文章我们就详细地来介绍一下Suspense的实现原理。



## 1. I/O瓶颈

对于前端来说，I/O的瓶颈主要体现在网络请求的延迟，这个问题前端是没有办法直接来解决的。React团队从另外一个角度来优化这问题，将**人机交互研究的结果整合到真实的UI**中，在网络延迟客观存在的情况下**尽可能的让用户减少对网络延迟的感知**。

`Suspense`组件的实现是React解决I/O瓶颈重要的一步。

熟悉React开发的小伙伴应该都知道，传统的方式在React应用中发起网络请求一般会遇到下面两个问题：

1. **请求“瀑布”（waterfall）**：就是本应该并行发出的请求，由于组件的嵌套关系导致串行发出
2. **请求“竞速”（race condition）**：在某些情况下先发出的请求后响应，会导致页面数据更新出错

这两个问题在传统的方法里面虽然是可以解决的，但是解决起来相对比较麻烦，而且可能会造成其他的问题。

`Suspense`组件就可以完美的解决上面所说的两个问题。`Suspense`是React为了**将I/O请求和视图渲染更好的结合在一起**提出的一种解决方案，并不会导致数据获取和视图渲染之间会耦合在前一起。Suspense详细的定义可以看官方的[文档](https://zh-hans.reactjs.org/docs/concurrent-mode-suspense.html)

下面我们就来解析一下`Suspense`的源码实现以及它如何来解决上面提到的两个问题。



## 2. Suspense组件的渲染流程

尝试过使用`Suspense`的小伙伴应该知道，在官方给出的例子中，使用Suspense之前需要修改一下和发起请求相关的代码。类似于下面

```js
export const wrapPromise = (promise: Promise<unknown>) => {
  let status = "pending";
  let result: unknown;
  const suspender = promise.then(
    (r) => {
      status = "success";
      result = r;
    },
    (e) => {
      status = "error";
      result = e;
    }
  );
  return {
    read() {
      switch (status) {
        case "pending":
          throw suspender;
        case "success":
          return result;
        case "error":
          throw result;
      }
    },
  };
};
```

我们可以使用`wrapPromise`来包裹发起请求的`promise`，来体验`Suspense`的功能。

> 笔者在分析Suspense相关的源代码时，使用的就是这种方式

从上面代码中可以看出，`wrapPromise`最大的功能就是：**当`promise`处于`pending`状态的时候抛出错误，而且这个错误对象就是`promise`本身**。

也就是说，React内部可以捕获到这个错误，然后来控制对应的`Suspense`组件渲染`primaryChildren`还是`fallbackChildren`。

> 这里我们将Suspense组件的子节点分成两类：
>
> 1. **fallbackChildren** : Suspense`组件处于suspended状态时渲染的子节点
> 2. **primaryChildren** : Suspense`组件处于unsuspended状态时渲染的子节点



### 2.1 updateSuspenseComponent方法

首先我们来看一下`beginWork`阶段`Suspense Fiber`对应的处理方法`updateSuspenseComponent`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/120fc3afe1ff33a10134e175090f4c6240b6cb4b/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1609)

```js
function updateSuspenseComponent(current, workInProgress, renderLanes) {
  const nextProps = workInProgress.pendingProps;
  let suspenseContext: SuspenseContext = suspenseStackCursor.current;
  let showFallback = false;
  const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;

  if (
    didSuspend ||
    shouldRemainOnFallback(
      suspenseContext,
      current,
      workInProgress,
      renderLanes,
    )
  ) {
    showFallback = true;
    workInProgress.flags &= ~DidCapture; // DidCapture在这里被消耗掉了
  } else {
    if (
      current === null ||
      (current.memoizedState: null | SuspenseState) !== null
    ) {
      /**
       * 这里表明当前Suspense是mount阶段，
       * 或者当前正处于suspended状态
       */
      if (
        nextProps.fallback !== undefined &&
        nextProps.unstable_avoidThisFallback !== true
      ) {
        suspenseContext = addSubtreeSuspenseContext(
          suspenseContext,
          InvisibleParentSuspenseContext,
        );
      }
    }
  }

  suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
	// 储存一个suspenseContext，在completeWork节点消费并弹出
  pushSuspenseContext(workInProgress, suspenseContext);

  if (current === null) {
    // mount阶段
    const nextPrimaryChildren = nextProps.children;
    const nextFallbackChildren = nextProps.fallback;
    if (showFallback) {
      const fallbackFragment = mountSuspenseFallbackChildren(
        workInProgress,
        nextPrimaryChildren,
        nextFallbackChildren,
        renderLanes,
      );
      const primaryChildFragment: Fiber = (workInProgress.child: any);
      primaryChildFragment.memoizedState = mountSuspenseOffscreenState(
        renderLanes,
      );
      workInProgress.memoizedState = SUSPENDED_MARKER;
      return fallbackFragment;
    } else if (typeof nextProps.unstable_expectedLoadTime === 'number') {
      // ...省略
    } else {
      return mountSuspensePrimaryChildren(
        workInProgress,
        nextPrimaryChildren,
        renderLanes,
      );
    }
  } else {
    // update阶段
    const prevState: null | SuspenseState = current.memoizedState;
    if (prevState !== null) {
      // 当前的状态是suspended
      if (showFallback) {
        const nextFallbackChildren = nextProps.fallback;
        const nextPrimaryChildren = nextProps.children;
        const fallbackChildFragment = updateSuspenseFallbackChildren(
          current,
          workInProgress,
          nextPrimaryChildren,
          nextFallbackChildren,
          renderLanes,
        );
        const primaryChildFragment: Fiber = (workInProgress.child: any);
        const prevOffscreenState: OffscreenState | null = (current.child: any)
          .memoizedState;
        primaryChildFragment.memoizedState =
          prevOffscreenState === null
            ? mountSuspenseOffscreenState(renderLanes)
            : updateSuspenseOffscreenState(prevOffscreenState, renderLanes);
        primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
          current,
          renderLanes,
        );
        /**
         * 表示当前渲染的是fallback节点
         * 打上标记，completeWork中会消费这个数据
        */
        workInProgress.memoizedState = SUSPENDED_MARKER;
        return fallbackChildFragment;
      } else {
        const nextPrimaryChildren = nextProps.children;
        const primaryChildFragment = updateSuspensePrimaryChildren(
          current,
          workInProgress,
          nextPrimaryChildren,
          renderLanes,
        );
        workInProgress.memoizedState = null; 
        return primaryChildFragment;
      }
    } else {
      // 当前的状态是unsuspended
      if (showFallback) {
        // Timed out.
        const nextFallbackChildren = nextProps.fallback;
        const nextPrimaryChildren = nextProps.children;
        const fallbackChildFragment = updateSuspenseFallbackChildren(
          current,
          workInProgress,
          nextPrimaryChildren,
          nextFallbackChildren,
          renderLanes,
        );
        const primaryChildFragment: Fiber = (workInProgress.child: any);
        const prevOffscreenState: OffscreenState | null = (current.child: any)
          .memoizedState;
        primaryChildFragment.memoizedState =
          prevOffscreenState === null
            ? mountSuspenseOffscreenState(renderLanes)
            : updateSuspenseOffscreenState(prevOffscreenState, renderLanes);
        primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
          current,
          renderLanes,
        );
        workInProgress.memoizedState = SUSPENDED_MARKER;
        return fallbackChildFragment;
      } else {
        const nextPrimaryChildren = nextProps.children;
        const primaryChildFragment = updateSuspensePrimaryChildren(
          current,
          workInProgress,
          nextPrimaryChildren,
          renderLanes,
        );
        workInProgress.memoizedState = null;
        return primaryChildFragment;
      }
    }
  }
}
```

这个方法乍一看比较复杂，但是可以将它分成几个部分来看：

1. **判断本次渲染是否是suspended状态**，渲染对应的`fallbackChildren`
2. 根据本次渲染中**`Suspense`的渲染阶段和状态**来选择不同的处理逻辑：
   1. mount阶段
   2. update阶段
      - `Suspense`组件当前处于`suspended`状态
      - `Suspense`组件当前处于`unsuspended`状态



对应的，我们可以将Suspense组件的渲染流程分成以下**三个阶段**：

1. mount阶段 : unsuspended -> suspended
2. update阶段 : suspended -> unsuspended
3. update阶段 : unsuspended -> suspended



### 2.2 mount阶段 : unsuspended -> suspended

#### 2.2.1 尝试渲染primaryChildren

第一个执行`updateSuspenseComponent`方法时，`Suspense`默认处于`unsuspended`状态，会渲染`primaryChildren`，执行的方法是`mountSuspensePrimaryChildren`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/120fc3afe1ff33a10134e175090f4c6240b6cb4b/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1920)

```js
function mountSuspensePrimaryChildren(
  workInProgress,
  primaryChildren,
  renderLanes,
) {
  const mode = workInProgress.mode;
  const primaryChildProps: OffscreenProps = {
    mode: 'visible',
    children: primaryChildren,
  };
  // 创建了一个离屏节点Offscreent
  const primaryChildFragment = createFiberFromOffscreen(
    primaryChildProps,
    mode,
    renderLanes,
    null,
  );
  primaryChildFragment.return = workInProgress;
  workInProgress.child = primaryChildFragment;
  return primaryChildFragment;
}
```

可以看到`Suspense`的`primaryChildren`会被一个`Offscreen`节点包裹，来控制节点的显示和隐藏。

我们来看一下`Offscreen Fiber`对应的处理方法`updateOffscreenComponent`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/120fc3afe1ff33a10134e175090f4c6240b6cb4b/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L566)

```js
function updateOffscreenComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  const nextProps: OffscreenProps = workInProgress.pendingProps;
  const nextChildren = nextProps.children;

  const prevState: OffscreenState | null =
    current !== null ? current.memoizedState : null;

  if (
    nextProps.mode === 'hidden' ||
    nextProps.mode === 'unstable-defer-without-hiding'
  ) {
    // ...省略  Suspense运行的过程中并不会走到这个分支，暂时可以不关注
  } else {
    let subtreeRenderLanes;
    if (prevState !== null) {
      subtreeRenderLanes = mergeLanes(prevState.baseLanes, renderLanes);
      // 上一次隐藏，这一次不隐藏 清除判断变量
      workInProgress.memoizedState = null;
    } else {
      // 上一次不隐藏 这一次也不隐藏
      subtreeRenderLanes = renderLanes;
    }
    pushRenderLanes(workInProgress, subtreeRenderLanes);
  }

  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

可以看到，在`updateOffscreenComponent`方法中只是做了一些变量和优先级的处理，然后就会调用`reconcileChildren`继续渲染子节点，也就是我们说的`primaryChildren`。

也就是说渲染的时候会首先尝试渲染`primaryChildren`，然后在渲染`primaryChildren`的过程中，如果某个节点数据请求没有回来，`promise`处于`pending`状态，则会抛出错误。

> `Suspense`内部，网络请求不需要在`useEffect`或者`componentDidMount`阶段来发起，渲染阶段就可以发起，提前了网络请求发起的时机。
>
> 可以看看官网的例子



#### 2.2.2 捕获promise错误

在`renderRootConcurrent`和`renderRootSync`方法中都使用了`try...catch`用来捕获`render`阶段发生的异常。

这里以`renderRootConcurrent`为例来解析

```js
do {
    try {
        workLoopConcurrent();
        break;
    } catch (thrownValue) {
        handleError(root, thrownValue);
    }
} while (true);
```

捕获到错误之后会执行`handleError`方法

```js
function handleError(root, thrownValue): void {
  do {
    let erroredWork = workInProgress; // 获取抛出错误的那个workInProgress fiber节点
    try {
      // ...省略
  		
      throwException(
        root,
        erroredWork.return,
        erroredWork,
        thrownValue,
        workInProgressRootRenderLanes,
      );
      completeUnitOfWork(erroredWork);
    } catch (yetAnotherThrownValue) {
      // ...省略
    }
    return;
  } while (true);
}
```

这个方法中主要做了两个操作：

1. 调用`throwException`方法来处理错误
2. 调用`completeUnitOfWork`表示当前节点已经是叶子节点，需要进入`“归”`过程。因为当前节点对应的组件在执行的时候抛出错误，所以不会产生子节点。



##### 2.2.2.1 throwException-处理错误

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/2fe415284a961fcb6c1ed0a65b43e55f4aec3f72/packages/react-reconciler/src/ReactFiberThrow.new.js#L186)

```js
function throwException(
  root: FiberRoot,
  returnFiber: Fiber,
  sourceFiber: Fiber,
  value: mixed,
  rootRenderLanes: Lanes,
) {
  sourceFiber.flags |= Incomplete; // 表示当前节点并没有正常完成
  sourceFiber.firstEffect = sourceFiber.lastEffect = null;

  if (
    value !== null &&
    typeof value === 'object' &&
    typeof value.then === 'function'
  ) {
    // 错误对象是一个promise对象
    const wakeable: Wakeable = (value: any);

    if ((sourceFiber.mode & BlockingMode) === NoMode) {
      // legacy模式
      const currentSource = sourceFiber.alternate;
      if (currentSource) {
        sourceFiber.updateQueue = currentSource.updateQueue;
        sourceFiber.memoizedState = currentSource.memoizedState;
        sourceFiber.lanes = currentSource.lanes;
      } else {
        sourceFiber.updateQueue = null;
        sourceFiber.memoizedState = null;
      }
    }

    const hasInvisibleParentBoundary = hasSuspenseContext(
      suspenseStackCursor.current,
      (InvisibleParentSuspenseContext: SuspenseContext),
    );

    let workInProgress = returnFiber;
    // 向上寻找最近的Suspense节点
    do {
      if (
        workInProgress.tag === SuspenseComponent &&
        shouldCaptureSuspense(workInProgress, hasInvisibleParentBoundary)
      ) {
        // suspenseFiber节点的updateQueue用来保存后代节点中抛出的promise error
        const wakeables: Set<Wakeable> = (workInProgress.updateQueue: any);
        if (wakeables === null) {
          const updateQueue = (new Set(): any);
          updateQueue.add(wakeable);
          workInProgress.updateQueue = updateQueue;
        } else {
          wakeables.add(wakeable);
        }

        if ((workInProgress.mode & BlockingMode) === NoMode) {
          /**
           * legacy模式下
           * 直接在这里给Suspense Fiber加上DidCapture flag
           */
          workInProgress.flags |= DidCapture;
          sourceFiber.flags |= ForceUpdateForLegacySuspense;
          /**
           * legacy模式下，sourceFiber去掉Incomplete和生命周期相关的flag 
           * 也就是抛出异常的时候不触发生命周期
           */
          sourceFiber.flags &= ~(LifecycleEffectMask | Incomplete);

          if (sourceFiber.tag === ClassComponent) {
            const currentSourceFiber = sourceFiber.alternate;
            if (currentSourceFiber === null) {
              sourceFiber.tag = IncompleteClassComponent;
            } else {
              const update = createUpdate(NoTimestamp, SyncLane);
              update.tag = ForceUpdate;
              enqueueUpdate(sourceFiber, update);
            }
          }
          sourceFiber.lanes = mergeLanes(sourceFiber.lanes, SyncLane);
          return;
        }
				// 非legacy模式下的处理
       
        // 对promise完成之后触发的更新进行优化处理
        attachPingListener(root, wakeable, rootRenderLanes);

        workInProgress.flags |= ShouldCapture; // 非legacy模式下 增加的是ShouldCapture flag
        workInProgress.lanes = rootRenderLanes;

        return;
      }
      workInProgress = workInProgress.return;
    } while (workInProgress !== null);
  }

  // ...省略 其他类型的error的处理
}
```

这个方法是React用来处理应用内部抛出的错误，包括非`promise`类型的错误。

这里我们主要关注的是对于`promise`类型错误的处理，主要做了以下几个事情：

1. **向上寻找最近的`Suspense Fiber`节点**
2. 将`promise`对象保存在对应的`Suspense Fiber`节点中，保存在`updateQueue`中，这里`updateQueue`是一个**`Set`**数据结构
3. 给`Suspense Fiber`增加`flag`。其中`legacy`模式下增加的是`DidCapture flag`，`非legacy`模式增加的是`ShouldCapture flag`
4. 在`非legacy`模式下，通过`then`方法给当前`promise`增加回调函数，用来**对`promise`完成之后触发的更新进行优化处理**。（后面专门来讲`Suspense`中做的优化）

这里有一个细节需要注意一下，在寻找错误处理边界的时候，除了判断节点的类型之外还执行了`shouldCaptureSuspense`方法

```js
export function shouldCaptureSuspense(
  workInProgress: Fiber,
  hasInvisibleParent: boolean,
): boolean {
  const nextState: SuspenseState | null = workInProgress.memoizedState;
  if (nextState !== null) {
    // ...省略
    return false;
  }
  const props = workInProgress.memoizedProps;
  
  if (props.fallback === undefined) {
    return false;
  }
  // 常规的边界，需要捕获
  if (props.unstable_avoidThisFallback !== true) {
    return true;
  }
  
  if (hasInvisibleParent) {
    return false;
  }
  
  return true;
}
```

这个方法主要是用来**判断当前`Suspense`是否符合捕获当前`promise`的条件**。这里感觉有一些`Suspense`实验性功能相关的代码，暂时不需要去了解。

这里我们主要来分析一下第一个判断条件，这个条件乍一看在update阶段有很大的几率能够满足，但是实际上调试的时候，都会直接跳过。这是为什么呢？

这里我们需要回头看一下`updateSuspenseComponent`中的处理逻辑，针对不同的阶段，有一个通用的处理逻辑：

1. **当Suspense的状态是`unsuspended`时，会执行`suspenseWorkInProgress.memoizedState = null`**
2. **当Suspense的状态是`suspended`时，会执行`suspenseWorkInProgress.memoizedState = SUSPENDED_MARKER`**

这里实际上就是**在`suspenseWorkInProgress`中增加一个标记用来判断本次更新中`Suspense`处于什么状态**。

而且**每次应用更新的时候，执行到`Suspense`节点时默认的状态都是`unsuspended`，所以会清空`memoizedState`**，上面所说的第一个条件会一直不满足。



##### 2.2.2.2 completeUnitOfWork-进入“归”阶段

至此React内部收集到了`promise`错误并找到了对应的`Suspense`边界，而且还做了一些特殊的标记，接下来就需要想办法**向上回溯到`Suspense`组件然后重新执行`beginWork`来渲染`fallbackChildren`**。

这个向上回溯的过程和`render`阶段的`"归"过程`是一致的，所以调用`completeUnitOfWork`

```js
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    if ((completedWork.flags & Incomplete) === NoFlags) {
      // ...省略 这里的处理过程在前面讲“归”过程的时候介绍过
    } else {
      // 当前节点没有完成时
      
      const next = unwindWork(completedWork, subtreeRenderLanes);
      if (next !== null) {
        // 带有ShouldCapture的Suspense组件会走到这里
        // 触发重新执行beginWork
        next.flags &= HostEffectMask;
        workInProgress = next;
        return;
      }

      if (returnFiber !== null) {
        // 给父节点标记未完成
        returnFiber.firstEffect = returnFiber.lastEffect = null;
        returnFiber.flags |= Incomplete;
      }
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // 有兄弟节点，继续执行兄弟节点的beginWork
      workInProgress = siblingFiber;
      return;
    }
    // 否则，执行父节点的completeWork
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);

  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

在`unwindWork`这个方法中，我们需要关注的是对于`Suspense`类型节点的处理，其他类型的节点几乎都是返回`null`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/2fe415284a961fcb6c1ed0a65b43e55f4aec3f72/packages/react-reconciler/src/ReactFiberUnwindWork.new.js#L47)

```js
function unwindWork(workInProgress: Fiber, renderLanes: Lanes) {
  switch (workInProgress.tag) {
    // ...省略
    case SuspenseComponent: {
      popSuspenseContext(workInProgress);
      const flags = workInProgress.flags;
      if (flags & ShouldCapture) {
        // 对于需要捕获promise的Suspense
        // 去掉ShouldCapture 加上DidCapture
        workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
        return workInProgress;
      }
      return null;
    }
    // ...省略
    default:
      return null;
  }
}
```

对于捕获`promise`的`Suspense`节点，会返回`suspenseWorkInProgress`，从而会重新执行这个节点的`beginWork`阶段。

在`completeUnitOfWork`中做了**两个重要的操作**：

1. **对于捕获`promise`错误的`Suspense`节点，重新开始执行其`beginWork`阶段，使其渲染`fallbackChildren`**
2. 抛出`promise`错误的节点不会影响其他子树执行`beginWork`，所以其他子树中也可以在`beginWork`阶段**并发**发出网络请求，并不需要等到上层组件请求完成之后再来请求，可以解决传统React应用中的**请求“瀑布”**问题。



#### 2.2.3 渲染fallbackChildren





## 3. Suspense组件的优化处理



