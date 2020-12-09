# 结合源码分析render阶段

之所以先分析render阶段，也就是`Reconciler`的工作流程，而不是先分析`Scheduler`，是因为在React当前版本中，默认情况下还是使 `LegacyRoot`，这种模式下`Scheduler`并没有起作用，而`Reconciler`和`Renderer`是不管哪种模式下工作的流程基本都是一样的。后续会话专门的章节来介绍`Scheduler`

> 所以在下面的源码分析中，凡是包含lane的变量都是和优先级调度有关的，可以先不用考虑。

`render`阶段和`commit`阶段的源码分析都是在 `legacy` 模式下进行的。

## 1. 进入render阶段之前的准备工作

上一节中介绍到，React应用在**初始化**的时候，会先创建一个`fiberRootNode`和`current rootFiber`。然后创建一个`update`，开启第一次更新。

```javascript
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  // 这里的container就是fiberRootNode
  const current = container.current;
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current);
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  // 这里创建了一个update，并且把当前应用的ReactElement Tree保存在update中
  // 表示这次更新需要渲染整个组件树
  const update = createUpdate(eventTime, lane);
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }

  enqueueUpdate(current, update);
  
  // 开启一次更新
  scheduleUpdateOnFiber(current, lane, eventTime);

  return lane;
}
```

> 源码对应[这里](https://github.com/facebook/react/blob/8e5adfbd7e605bda9c5e96c10e015b3dc0df688e/packages/react-reconciler/src/ReactFiberReconciler.new.js#L250)

**说明mount和update一样，都是先产生一个或多个update，然后再调用`scheduleUpdateOnFiber`开启一次更新操作。**（后面讲update流程的时候会详细讲解）

进入更新操作

1. 如果是**异步更新**，则先**接入`Scheduler`**，然后调用`performConcurrentWorkOnRoot`方法进入`render`阶段；

2. 如果是**同步更新**，则直接调用`performSyncWorkOnRoot`方法进入`render`阶段

## 2. render阶段工作流程

render阶段主要是Reconciler在工作，目的是**将新的`ReactElement`节点和`current Fiber`节点来对比，创建出一个`workInProgress Fiber Tree`**。

遍历树时，一般会使用**递归**算法。但是前面讲Fiber架构的时候有提到，**Fiber架构为了实现时间切片，需要将原来递归的算法修改成循环的算法**。实际场景中，使用循环来替代递归的场景是很常见的，毕竟**递归算法除了无法中断的缺点之外还有爆栈的风险**。

> Fiber节点的数据结构中加入`return`和`sibling`指针就是为了简单的使用循环来实现递归。



render阶段的工作可以分成两个部分：

1. **”递“阶段** —— 从`workInProgress rootFiber`开始向下进行**深度优先遍历**，每遍历一个`Fiber节点`就调用一次`beginWork`方法，该方法会创建该`Fiber节点`的子节点。然后继续向下遍历并创建，直到遇到叶子节点的时候，进入”归“阶段。
2. **”归“节点** —— 对当前节点执行`completeWork`方法，执行完之后如果存在兄弟节点，则会进入兄弟节点的”递“阶段。如果不存在兄弟节点，就会进入父节点的”归“阶段。

可以看出，”递“和”归“是**交错运行**的，直到`rootFiber`的归阶段执行完，整个`render`阶段才算完成。

> React内部遍历Fiber Tree使用的算法是深度优先遍历。如果强行使用广度优先遍历，会非常麻烦，sibling指针不适合广度优先遍历

### 2.1 创建workInProgress rootFiber

因为mount的时候，render阶段开始之前并没有`workInProgress Fiber Tree`，所以需要先创建`workInProgress rootFiber`。可以看`renderRootSync`方法

```javascript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  // ...省略

  // 这里使用内部变量workInProgressRoot来保存上一次更新时的fiberRootNode
  // 如果和本次的fiberRootNode不相等，说明本次更新的是其他React应用树，需要重新创建workInProgress rootFiber
  // 注意：workInProgressRoot在commit阶段会重新设置为null
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 这个方法中会创建workInProgress rootFiber
    prepareFreshStack(root, lanes);
  }

  do {
    try {
      // 开始循环创建workInProgress Fiber Tree
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  // ...省略
}

function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  // ...省略
  /**
   * 创建workInProgress树的RootFiber节点
   */
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  // ...省略
}
```

> 对应的源码见[这里](https://github.com/facebook/react/blob/8e5adfbd7e605bda9c5e96c10e015b3dc0df688e/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1501)



`root Fiber`创建完成之后，在`workLoopSync`方法中，开始用**循环**的方式遍历树。

```javascript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    // “递”过程的主要工作，调用beginWork，创建子节点，并返回child节点（也就是第一个子节点）
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // 如果是叶子节点，插入一个”归“过程
    completeUnitOfWork(unitOfWork);
  } else {
    // 如果是非叶子节点，开启下一个“递”过程
    workInProgress = next;
  }
}
```

> 源码对应[这里](https://github.com/facebook/react/blob/8e5adfbd7e605bda9c5e96c10e015b3dc0df688e/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1569)

可以看出：

1. “递”过程主要是执行`beginWork`
2. “归”过程主要是执行`completeWork`

后面主要是解析这两个方法，看看究竟做了哪些工作。

### 2.2 ”递“过程——beginWork

> 该方法的源码看[这里](https://github.com/facebook/react/blob/8e5adfbd7e605bda9c5e96c10e015b3dc0df688e/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3077)

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  
  if (current !== null) {
    // update阶段
  	// 先判断优化路径是否匹配，如果不匹配就是需要新建Fiber节点
    
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (oldProps !== newProps || hasLegacyContextChanged()) {
      // 通过didReceiveUpdate变量来判断当前节点是否需要更新
      // 判断能否复用currentFiber，不止这一个条件
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      // ... 省略
      // 调用该方法，克隆currentFiber节点，复制其中的属性
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    } else {
      // 代码简化
      didReceiveUpdate = false;
    }
  } else {
 		// mount阶段
    didReceiveUpdate = false;
  }
  workInProgress.lanes = NoLanes;

  // 根据tag不同，进入不同的创建流程
  switch (workInProgress.tag) {
    // ... 省略
    case FunctionComponent: 
      // ... 省略
    case ClassComponent: 
      // ... 省略
    case HostRoot:
      // ... 省略
    case HostComponent:
      // ... 省略
    case HostText:
      // ... 省略
    // ... 省略
}
```

#### 2.2.1 入参解读

1. `current `—— 表示当前Fiber节点对应`current Fiber Tree`上的Fiber节点，也就是`workInProgress.alternate`

2. `workInProgress` —— 表示**本次更新**中，当前组件对应的Fiber节点，也就是workInProgress Fiber Tree上的节点
3. `renderLanes` —— 优先级相关的属性，暂时可以不考虑



上面函数体中可以看出，**根据`current === null`可以判断当前组件是处于`mount`阶段还是`update`阶段**，因为对于本次新增的组件，在`current Fiber Tree`上是不存在对应的节点的，所以`currentFiber === null`

可以将beginWork的工作分成**两个部分**：

1. 组件`mount`阶段：直接创建新的子Fiber节点，没有优化流程
2. 组件`update`阶段：判断是否匹配优化路径，如果不满足则需要新创建子Fiber节点；如果匹配优化路径，则进入优化流程**复用对应的`currentFiber`**。



#### 2.2.2 更新优化（*）—— 复用currentFiber

前面讲Fiber架构的时候提到过，render阶段的主要工作是根据`current Fiber Tree`和新生成的`ReactElement Tree`来生成`workInProgress Fiber Tree`。但是如果每次更新都重新运行所有组件的render方法，生成新的ReactElement，然后生成新的Fiber节点，**对于没有任何修改的节点这是很冗余的操作**。React内部对这种情况做了相应的优化

##### 2.2.2.1 优化效果

在创建子Fiber节点过程中可以优化的节点，都会调用`bailoutOnAlreadyFinishedWork`方法

```javascript
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // ... 省略

  // 判断当前节点的子节点本次是否需要更新（反应在优先级上）
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // 如果子节点都不需要更新，本次更新直接跳过所有后代节点
    // 当前节点直接进入“归”过程
    // 相当于当前节点的所有后代节点都复用对应的currentFiber
    return null;
  } else {
    // 如果当前节点的后代节点中有节点需要在本次更新
    // 那就只克隆子Fiber节点，然后继续“递”过程
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
  }
}
```

优化的效果：**进入优化的节点，不需要调用render方法生成ReactElement，然后再生成对应的Fiber节点。而是直接复用`current Fiber Tree`中的节点。**

节点复用也分成两种情况：

1. **复用整个`currentFiber`对象**：当节点的后代节点也没有更新的时候，后代节点会直接复用对应的整个`currentFiber`对象。
2. **复用`currentFiber`中的属性**：也就是**克隆**`currentFiber`节点，克隆出来的节点也是新的节点。

> 这里需要说明的一点是：**由于在commit阶段中，都会执行`workInProgress = null`这个操作，所以每次更新的时候，都会从rootFiber开始遍历生成新的`workInProgress Fiber Tree`，在遍历的过程中插入优化的环节**。



##### 2.2.2.2 优化的前提条件：

1. `oldProps === newProps`，即更新前后的props不变，默认情况下并没有做对象的浅对比，调用render新生成的Fiber节点都不满足该条件。
2. `!includesSomeLane(renderLanes, updateLanes)=== true` ，即表示当前节点的优先级不够，不参与本次更新，后面再细讲。

当这两个条件同时满足的时候，会进入优化流程。但是**这两个条件都不是优化的必要条件，不同类型的节点有不同的逻辑判断是否进入优化流程**。

比如FunctionComponent，需要`didReceiveUpdate === false`

```javascript
function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderLanes,
) {
  // ...省略
  // 对于处于update节点的节点，current不会为null
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
	// ...省略
}
```

比如ClassComponent，需要`shouldUpdate===false`，`shouldComponentUpdate`生命周期的返回值可以影响这个变量

```javascript
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderLanes: Lanes,
) {
	// ... 省略
  const didCaptureError = (workInProgress.flags & DidCapture) !== NoFlags;

  if (!shouldUpdate && !didCaptureError) {
    // Context providers should defer to sCU for rendering
    if (hasContext) {
      invalidateContextProvider(workInProgress, Component, false);
    }

    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
	// ... 省略
}
```

#### 2.2.3 创建子fiber节点

**在创建子Fiber节点之前，先要获取当前ReactElement最新的子ReactElement**。不同类型的ReactElement生成children的方式不同，这里着重提一下ClassComponent。

ClassComponent在生成子节点的过程中，会调用对应的生命周期函数，最后调用`render`方法生成子ReactElement。这个阶段中具体调用了哪些生命周期，可以看[这里](https://github.com/facebook/react/blob/8e5adfbd7e605bda9c5e96c10e015b3dc0df688e/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L852)

最后都是调用`reconcileChildren`方法，进入创建子Fiber节点的流程。

```javascript
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes,
) {
  if (current === null) {
    // mount阶段
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // update阶段
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

