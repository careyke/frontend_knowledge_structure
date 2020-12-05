# 结合源码分析render阶段

之所以先分析render阶段，也就是`Reconciler`的工作流程，而不是先分析`Scheduler`，是因为在React当前版本中，默认情况下还是使 `LegacyRoot`，这种模式下`Scheduler`并没有起作用，而`Reconciler`和`Renderer`是不管哪种模式下工作的流程基本都是一样的。后续会话专门的章节来介绍`Scheduler`

> 所以在下面的源码分析中，凡是包含lane的变量都是和优先级调度有关的，可以先不用考虑。

`render`阶段和`commit`阶段的源码分析都是在 `legacy` 模式下进行的。

## 1. 进入render阶段之前的准备工作

上一节中介绍到，React应用在初始化的时候，会先创建一个`fiberRootNode`和`current rootFiber`。然后创建一个`update`，开启第一次更新。

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

**说明mount和update一样，都是先产生一个或多个update，然后再调用`scheduleUpdateOnFiber`开启一次更新操作。**（后面将update流程的时候会详细讲解）

进入更新操作

1. 如果是**异步更新**，则先**接入`Scheduler`**，然后调用`performConcurrentWorkOnRoot`方法进入`render`阶段；

2. 如果是**同步更新**，则直接调用`performSyncWorkOnRoot`方法进入`render`阶段

## 2. render阶段工作流程

render阶段主要是Reconciler在工作，目的是**将新的`ReactElement`节点和`current Fiber`节点来对比，创建出一个`workInProgress Fiber Tree`**。

遍历树时，一般会使用**递归**算法。但是前面讲Fiber架构的时候有提到，**Fiber架构为了实现时间切片，需要将原来递归的算法修改成循环的算法**。实际场景中，使用循环来替代递归的场景是很常见的，毕竟**递归算法除了无法中断的缺点之外还有爆栈的风险**。

> Fiber节点的数据结构中加入`return`和`sibling`指针就是为了简单的使用循环来实现递归。



render阶段的工作可以分成两个部分：

1. **”递“阶段** —— 从`workInProgress rootFiber`开始向下进行**深度优先遍历**，每遍历一个`Fiber节点`就调用一次`beginWork`方法，该方法会创建该`Fiber节点`的子节点。然后需要向下遍历并创建，直到遇到叶子节点的时候，进入”归“阶段。
2. **”归“节点** —— 对当前节点执行`completeWork`方法，执行完之后如果存在兄弟节点，则会进入兄弟节点的”递“阶段。如果不存在兄弟节点，就会进入父节点的”归“阶段。

可以看出，”递“和”归“是**交错运行**的，直到`rootFiber`的归阶段执行完，整个`render`阶段才算完成。

> React内部遍历Fiber Tree使用的算法是深度优先遍历。如果强行使用广度优先遍历，会非常麻烦，sibling指针不适合广度优先遍历

### 2.1 创建workInProgress rootFiber

因为mount的时候，render阶段开始之前并没有`workInProgress Fiber Tree`，所以需要先创建`workInProgress rootFiber`。可以看`renderRootSync`方法

```javascript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  // ...省略

  // 这里使用内部变量workInProgressRoot来保存上一次更新时的fiberRootNode
  // 如果和本次的fiberRootNode不相等，说明本次更新的是其他React应用树，需要重新创建workInProgress 		 rootFiber
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



`root Fiber`创建完成之后，在`workLoopSync`方法中，开始用循环的方式遍历树。

```javascript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  // “递”过程的主要工作
  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    // “递”过程的主要工作，调用beginWork，创建子节点，并返回child节点
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
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
2. “归”构成主要是执行`completeWork`

后面主要是解析这两个方法，看看究竟做了哪些工作。

### 2.2 ”递“过程——beginWork

