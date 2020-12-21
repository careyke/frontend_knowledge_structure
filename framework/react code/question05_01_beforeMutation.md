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

从代码中可以得知，在`before mutation之前`主要是做一些变量赋值和状态重置的工作。主要注意的以下几个点：

1. 当前更新的`commit`阶段开始之前，**会确保之前更新中没有执行完的useEffect全部执行完**。也就是说useEffect的执行是一个**异步**的过程，也是可以被打断的。后面会详细讲
2. `workInProgress`和`workInProgressRoot`全局变量会在`commit`阶段清空。
3. `workInProgress rootFiber`如果有`flag`，会在`before mutation之前`加入`effectList`



## 2. before mutation阶段

