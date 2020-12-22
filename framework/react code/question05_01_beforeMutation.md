# commit - before mutation

前面的文章中已经介绍了`render`阶段的过程，应该render阶段之后，`workInProgress Fiber tree`和`effectList`都已经创建完成，接下来就是进入`commit`阶段来消费这些数据。

在`performSyncWorkOnRoot`方法中，执行完`render`阶段之后，接下来进入`commit`阶段。

```javascript
function performSyncWorkOnRoot(root) {
  // ...省略
  
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  // commit阶段的入口方法
  commitRoot(root);

  ensureRootIsScheduled(root, now());

  return null;
}
```

`commit`阶段的入口函数是`commitRoot`方法，主要的流程代码是在**`commitRootImpl`**方法中。

**整个`commit`阶段可以分成三个部分**：

1. `before mutation`阶段：执行DOM操作之前
2. `mutation`阶段：执行DOM操作
3. `layout`阶段：执行DOM操作之后

除了这三个过程之外，还有一些前置工作和后置工作需要处理，称为**`before mutation之前`**和**`layout之后`**。



## 1. before mutation之前

直接看对应的方法`commitRootImpl`，从方法的开头到`if (firstEffect !== null)`这部分都属于`before mutation之前`的逻辑

> 对应的源码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1929)

```javascript
do {
    // 处理上一次更新中没有执行的useEffect
  	// 由于useEffect里面可能会产生新的更新，所以需要循环遍历，直到执行完为止
    flushPassiveEffects();
} while (rootWithPendingPassiveEffects !== null);

// root.finishedWork指的是workInProgress rootFiber
const finishedWork = root.finishedWork;
const lanes = root.finishedLanes;

if (finishedWork === null) {
    if (enableSchedulingProfiler) {
        markCommitStopped();
    }
    return null;
}
root.finishedWork = null;
root.finishedLanes = NoLanes;
root.callbackNode = null;
// 优先级相关
let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
markRootFinished(root, remainingLanes);

// 清除已完成的离散更新
if (rootsWithPendingDiscreteUpdates !== null) {
    if (
        !hasDiscreteLanes(remainingLanes) &&
        rootsWithPendingDiscreteUpdates.has(root)
    ) {
        rootsWithPendingDiscreteUpdates.delete(root);
    }
}

if (root === workInProgressRoot) {
    // 重置构建workInProgress tree相关的全局变量
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
} else {
}

// 获取effectList
let firstEffect;
if (finishedWork.flags > PerformedWork) {
  	// 判断rootFiber是否有flag,
  	// 如果有的话讲rootFiber本身也加入到effectList中
  	// completeUnitWork中处理的只是将子级的effectList收集在当前节点上，
  	// 所以rootFiber自身的flag并没有得到处理
    if (finishedWork.lastEffect !== null) {
        finishedWork.lastEffect.nextEffect = finishedWork;
        firstEffect = finishedWork.firstEffect;
    } else {
        firstEffect = finishedWork;
    }
} else {
    firstEffect = finishedWork.firstEffect;
}
```

从代码中可以得知，在`before mutation之前`主要是做一些变量赋值和状态重置的工作。主要注意以下几个点：

1. 当前更新的`commit`阶段开始之前，**会确保上一次更新中没有执行完的useEffect全部执行完**。也就是说`useEffect`的执行是一个**异步**的过程，也是可以被打断的。
2. `workInProgress`和`workInProgressRoot`全局变量会在`commit`阶段清空。
3. `workInProgress rootFiber`如果有`flag`，会在`before mutation之前`加入`effectList`

React中关于effect执行时机的[描述](https://zh-hans.reactjs.org/docs/hooks-reference.html#timing-of-effects)

> 虽然 `useEffect` 会在浏览器绘制后延迟执行，但会保证在任何新的渲染前执行。React 将在组件更新前刷新上一轮渲染的 effect。

## 2. before mutation阶段

`before mutation阶段`的代码也在`commitRootImpl`方法中

```javascript
// ...省略处理focus和blur的逻辑

// 全局变量nextEffect中保存effectList
nextEffect = firstEffect;
do {
  	try {
      	// before mutation阶段的主要方法
        commitBeforeMutationEffects();
    } catch (error) {
        invariant(nextEffect !== null, 'Should be working on an effect.');
        captureCommitPhaseError(nextEffect, error);
        nextEffect = nextEffect.nextEffect;
    }
} while (nextEffect !== null);
```

可以看到，`before mutation阶段`主要做的就是遍历effectList链表，对其中的每个`Fiber`执行`commitBeforeMutationEffects`方法

### 2.1 commitBeforeMutationEffects

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2311)

```javascript
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;

    if (!shouldFireAfterActiveInstanceBlur && focusedInstanceHandle !== null) {
      // ... 省略 处理blur和focus的逻辑
    }

    const flags = nextEffect.flags;
    if ((flags & Snapshot) !== NoFlags) {
      // 如果当前节点的flag中含有Snapshot
      // 会执行getSnapshotBeforeUpdate生命周期
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    if ((flags & Passive) !== NoFlags) {
      // 如果当前节点的flag中含有Passive，说明当前节点对应的组件使用了useEffect
      // 尝试开启一次异步调度执行本次更新中所有的useEffect
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        // 注册一个任务，调度器异步执行
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

上面代码中主要做了三件事：

1. 执行节点添加或者删除之后的`focus`和`blur`的逻辑
2. 对于更新阶段的ClassComponent，调用`getSnapshotBeforeUpdate`生命周期函数
3. **调度**本次更新中的所有`useEffect`

我们主要分析第2、3点。

#### 2.1.1 执行`getSnapshotBeforeUpdate`生命周期

代码中可以看出，对于含有`Snapshot flag`的节点才会执行`getSnapshotBeforeUpdate`生命周期，那么节点是何时加上的`Snapshot flag`的呢？

##### 2.1.1.1 添加`Snapshot flag`的时机

在`render`阶段中，创建`workInProgress Fiber`的时候，对于**`update`阶段的`ClassComponent`来说**，会执行`updateClassInstance`方法，其中有绑定`flag`的关键代码。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L1031)

```javascript
//...省略
if (shouldUpdate) {
    if (
        !hasNewLifecycles &&
        (typeof instance.UNSAFE_componentWillUpdate === 'function' ||
            typeof instance.componentWillUpdate === 'function')
    ) {
      	// componentWillUpdate是在render节点调用的
        if (typeof instance.componentWillUpdate === 'function') {
            instance.componentWillUpdate(newProps, newState, nextContext);
        }
        if (typeof instance.UNSAFE_componentWillUpdate === 'function') {
            instance.UNSAFE_componentWillUpdate(newProps, newState, nextContext);
        }
    }
    if (typeof instance.componentDidUpdate === 'function') {
        workInProgress.flags |= Update;
    }
    if (typeof instance.getSnapshotBeforeUpdate === 'function') {
        workInProgress.flags |= Snapshot;
    }
} else {
  	// 从源码中看，就算shouldUpdate放回false也是有可能执行后续的生命周期的
  	// 但是具体的场景笔者还不没有想到
    if (typeof instance.componentDidUpdate === 'function') {
        if (
            unresolvedOldProps !== current.memoizedProps ||
            oldState !== current.memoizedState
        ) {
            workInProgress.flags |= Update;
        }
    }
    if (typeof instance.getSnapshotBeforeUpdate === 'function') {
        if (
            unresolvedOldProps !== current.memoizedProps ||
            oldState !== current.memoizedState
        ) {
            workInProgress.flags |= Snapshot;
        }
    }
    workInProgress.memoizedProps = newProps;
    workInProgress.memoizedState = newState;
}
// ...省略
```

可以看出对于**需要更新**的节点，在创建`workInProgress Fiber`的过程中就会增加`Update或者Snapshot的flag`。



#### 2.1.2 调度useEffect

上面代码中，对于含有`Passive flag`的节点会将`flushPassiveEffects`方法`push`到`scheduler`中，由`scheduler`**异步调度执行**。

那么`flushPassiveEffects`方法主要做了什么工作？`Passsive flag`又是何时绑定上的？

> **注意：**
>
> 调度`useEffect`的优先级是`NormalSchedulerPriority`，说明**这个任务并不用一个高优先级的任务，是可以被打断执行的**。
>
> 所以有可能会导致前一次更新的`useEffect`，在下一次更新的`commit`阶段开始前执行。刚好呼应前面`before mutation之前`中执行上一次更新的所有`useEffect`。
>
> (这种情况笔者也没有遇到过)



##### 2.1.2.1 `flushPassiveEffects`的主要功能

`flushPassiveEffects`方法会调用`flushPassiveEffectsImpl`方法，主要是功能实现都在这个方法中。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2564)

```javascript
function flushPassiveEffectsImpl() {
  // 方法的开关
  if (rootWithPendingPassiveEffects === null) {
    return false;
  }
  // ...省略
  
  // 执行useEffect的销毁方法
  const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = [];
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    const fiber = ((unmountEffects[i + 1]: any): Fiber);
    const destroy = effect.destroy;
    effect.destroy = undefined;

    if (typeof destroy === 'function') {
      if (__DEV__) {
      } else {
        try {
          if (
            enableProfilerTimer &&
            enableProfilerCommitHooks &&
            fiber.mode & ProfileMode
          ) {
            try {
              destroy();
            } finally {
            }
          } else {
            destroy();
          }
        } catch (error) {
          invariant(fiber !== null, 'Should be working on an effect.');
          captureCommitPhaseError(fiber, error);
        }
      }
    }
  }
  // 执行useEffect的创建方法
  const mountEffects = pendingPassiveHookEffectsMount;
  pendingPassiveHookEffectsMount = [];
  for (let i = 0; i < mountEffects.length; i += 2) {
    const effect = ((mountEffects[i]: any): HookEffect);
    const fiber = ((mountEffects[i + 1]: any): Fiber);
    if (__DEV__) {
      
    } else {
      try {
        const create = effect.create;
        if (
          enableProfilerTimer &&
          enableProfilerCommitHooks &&
          fiber.mode & ProfileMode
        ) {
          try {
            startPassiveEffectTimer();
            effect.destroy = create();
          } finally {
            recordPassiveEffectDuration(fiber);
          }
        } else {
          effect.destroy = create();
        }
      } catch (error) {
        invariant(fiber !== null, 'Should be working on an effect.');
        captureCommitPhaseError(fiber, error);
      }
    }
  }
	// ...省略
  
  // 执行useEffect创建方法导致的同步任务
  flushSyncCallbackQueue();
  
  // ...省略
}
```

可以看出，`flushPassiveEffects`方法的主要功能有：

1. 执行useEffect的**销毁**函数
2. 执行useEffect的**创建**函数
3. 如果useEffect的创建函数中导致了新的**同步任务**，不用等在下个宏任务再调度执行，直接在当前宏任务执行。



> 这里还有两个点需要**注意**：
>
> 1. 当前方法的执行逻辑有一个开关变量`rootWithPendingPassiveEffects`，这个全局变量在`layout之后`才会赋值。（后面会讲到）
> 2. React内部使用了两个全局变量`pendingPassiveHookEffectsUnmount`和`pendingPassiveHookEffectsMount`来分别保存需要执行**useEffect销毁和创建函数**的节点。（这里是消费数据，后面会讲数据是如何插入的。）



##### 2.1.2.2 添加`Passive flag`的时机

和上面`ClassCompnent`类型的节点添加`Snapshot flag`的时机一样，`FunctionComponent`类型的节点添加`Passive flag`也是发生在创建`子workInProgress Fiber`的过程中。

`FunctionComponent`类型的节点生成子`ReactElement`的时候会调用`renderWithHooks`方法。

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L327)

```javascript
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  renderLanes = nextRenderLanes;
  // 将当前Fiber节点保存在全局变量中
  currentlyRenderingFiber = workInProgress;

  // ...省略

  // 调用functionComponent函数
  let children = Component(props, secondArg);

  // ...省略
  return children;
}
```

这里有两个**关键点**：

1. 将`workInProgress Fiber`保存在全局变量中，其他方法会消费这个变量。
2. 执行`FunctionComponent`函数本身生成子`ReactElement`

在调用`FunctionComponent`函数的时候，如果当前组件有`useEffect`钩子，会调用`useEffect`方法。

这里以`mount`阶段的节点来分析，调用**`mountEffect`**方法（update阶段效果大致一样的）

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L1295)

```javascript
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect | PassiveEffect, // 这里对应fiber flag中的Update和Passive
    HookPassive, // 这里是effect的flag
    create,
    deps,
  );
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
	// 使用保存workInProgress Fiber的全局变量
	// 给当前Fiber节点加上flag
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}
```

### 2.2 总结

`before mutation`阶段的**主要工作**是：

1. 处理`DOM`节点新增或删除之后的`autoFocus`、`blur`的逻辑。
2. 对应有`Snapshot flag`的节点，执行`getSnapshotBeforeUpdate`生命周期函数
3. **异步调度**本次更新的`useEffect`中的销毁函数和创建函数

