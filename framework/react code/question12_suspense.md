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
    // ...省略  Suspense运行的过程中并不会走到这个分支
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





## 3. Suspense组件的优化处理



