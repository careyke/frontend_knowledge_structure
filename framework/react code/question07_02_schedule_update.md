# 调度update

在分析Fiber架构的时候我们提到过，Fiber架构中实现了**时间切片**和**优先级调度**的功能。

**时间切片主要是由`Scheduler`来完成的，优先级调度主要是React内部自己来实现的。**

这一节笔者就来详细的分析一下优先级调度的实现细节。



## 1. React的优先级机制（*）

在开始分析优先级调度之前，我们首先要搞清楚`React`中的优先级机制，React内部实现了一套自己的优先级机制。

这里可能有同学会问，为什么`React`不直接使用`Scheduler`中的优先级机制呢？

因为`Scheduler`是一个独立通用的库，其优先级的划分是**粗粒度**的，然而React内部对于更新优先级的划分是**非常细粒度**的，所以React内部实现了一套自己的优先级机制。

### 1.1 lane模型

React在设计优先级模型时借鉴了**环形赛车轨道**的概念，称为`lane`模型。**使用31位二进制数表示31个不同的轨道（lane），从右向左位数越小表示轨道越靠近内圈，距离就越短，所以其所在的优先级就越高。**

在创建**更新对象（Update）**的时候，会为每个`Update`分配一个`lane`，表示当前`Update`的优先级和所在的轨道。



### 1.2 优先级（lanePriority）

`lane`是用来表示更新对象`Update`的优先级，但是在调度更新的时候是以更新任务`update`为单位来调度的，所以需要一组常量来描述`update`的优先级。

> 我们使用`update`来表示**更新任务**，用`Update`来表示**更新对象**

React在`lane模型`的基础上抽象出了16种优先级，称为**`lanePriority`**，用来表示**更新任务（update）的优先级**。

```javascript
// 优先级从高到低
export const SyncLanePriority: LanePriority = 15;
export const SyncBatchedLanePriority: LanePriority = 14;

const InputDiscreteHydrationLanePriority: LanePriority = 13;
export const InputDiscreteLanePriority: LanePriority = 12;

const InputContinuousHydrationLanePriority: LanePriority = 11;
export const InputContinuousLanePriority: LanePriority = 10;

const DefaultHydrationLanePriority: LanePriority = 9;
export const DefaultLanePriority: LanePriority = 8;

const TransitionHydrationPriority: LanePriority = 7;
export const TransitionPriority: LanePriority = 6;

const RetryLanePriority: LanePriority = 5;

const SelectiveHydrationLanePriority: LanePriority = 4;

const IdleHydrationLanePriority: LanePriority = 3;
const IdleLanePriority: LanePriority = 2;

const OffscreenLanePriority: LanePriority = 1;

export const NoLanePriority: LanePriority = 0;
```

31条轨道对应16种优先级，也就是说某些`lanePriority`会对应多条轨道，**这表示这些轨道中的`Update`是同一个优先级的，可以考虑一起批量执行。**

> 同一个`lanePriority`中不同`lane`中的`Update`并不是一定会批量执行，在某些特性的场景下会批量执行。后面会详细介绍

每种`lanePriority`对应的轨道分布可以看这里，根据变量名可以进行一一对应

```javascript
// 表示各个优先级所占有的轨道
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;

const InputContinuousHydrationLane: Lane = /*           */ 0b0000000000000000000000000100000;
const InputContinuousLanes: Lanes = /*                  */ 0b0000000000000000000000011000000;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000100000000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000001000000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000011110000000000000000000000;

export const SomeRetryLane: Lanes = /*                  */ 0b0000010000000000000000000000000;

export const SelectiveHydrationLane: Lane = /*          */ 0b0000100000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0000111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0001000000000000000000000000000;
const IdleLanes: Lanes = /*                             */ 0b0110000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

这里有一个有意思的现象，就是**优先级越低，占有的轨道就越多**。

**这个是因为优先级越低的任务越容易被打断，更新对象`Update`越容易造成积压，所以需要分配更多的轨道，用来保存Update。但是如果当前优先级中所有的`lane`都在使用的时候，那么就会对下一个同优先级的`Update`进行降级处理。**这个后面会详细讲解



### 1.3 更新任务(update)与更新对象(Update)

更新对象是用来描述状态的变化信息的一个对象，称为`Update`

更新任务指的是将对应更新对象中描述的状态变化更新到应用中的一系列操作，称为`update`。

**更新任务中还有两个很重要的概念**：

1. 优先级（lanePriority）：表示当前update的优先级
2. **renderLanes**：表示当前`update`需要执行的`lanes`，这些`lanes`中的`Update`会在本次更新任务中执行

#### 1.3.1 Update与lane

前面我们讲过，在创建`Update`的时候会分配一个`lane`，表示当前`Update`占用的轨道。

获取`lane`时调度用的是`requestUpdateLane`方法。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L401)

```javascript
export function requestUpdateLane(fiber: Fiber): Lane {
  const mode = fiber.mode;
  if ((mode & BlockingMode) === NoMode) {
    // Legacy mode
    return (SyncLane: Lane);
  } else if ((mode & ConcurrentMode) === NoMode) {
    // Blocking mode
    return getCurrentPriorityLevel() === ImmediateSchedulerPriority
      ? (SyncLane: Lane)
      : (SyncBatchedLane: Lane);
  } 
  // Concurrent mode
  
  // ... 省略部分代码
  if (currentEventWipLanes === NoLanes) {
    // 获取正在执行的lanes，也就是renderLanes
    // workInProgressRootIncludedLanes 会在执行update的过程中赋值
    currentEventWipLanes = workInProgressRootIncludedLanes;
  }
  const schedulerPriority = getCurrentPriorityLevel();
  // ... 省略部分代码
  lane = findUpdateLane(schedulerLanePriority, currentEventWipLanes);
  
  return lane;
}
```

`Legacy`和`Blocking`模式中处理都很简单，下面我们主要来分析一下`ConcurrentMode`中获取lane的流程。

1. 获取当前的renderLanes
2. 获取当前状态更新操作的优先级，也就是`SchedulerPriority`，让根据映射关系获取`lanePriority`
3. 调用`findUpdateLane`方法从当前优先级的轨道中获取`lane`

`findUpdateLane`方法的代码如下：

```javascript
export function findUpdateLane(
  lanePriority: LanePriority,
  wipLanes: Lanes,
): Lane {
  switch (lanePriority) {
    case NoLanePriority:
      break;
    case SyncLanePriority:
      return SyncLane;
    case SyncBatchedLanePriority:
      return SyncBatchedLane;
    case InputDiscreteLanePriority: {
      const lane = pickArbitraryLane(InputDiscreteLanes & ~wipLanes);
      if (lane === NoLane) {
        return findUpdateLane(InputContinuousLanePriority, wipLanes);
      }
      return lane;
    }
    case InputContinuousLanePriority: {
      const lane = pickArbitraryLane(InputContinuousLanes & ~wipLanes);
      if (lane === NoLane) {
        // Shift to the next priority level
        return findUpdateLane(DefaultLanePriority, wipLanes);
      }
      return lane;
    }
    case DefaultLanePriority: {
      let lane = pickArbitraryLane(DefaultLanes & ~wipLanes);
      if (lane === NoLane) {
        // 当前优先级的轨道已经占满，尝试降级处理
        lane = pickArbitraryLane(TransitionLanes & ~wipLanes);
        if (lane === NoLane) {
          lane = pickArbitraryLane(DefaultLanes);
        }
      }
      return lane;
    }
    case TransitionPriority: // Should be handled by findTransitionLane instead
    case RetryLanePriority: // Should be handled by findRetryLane instead
      break;
    case IdleLanePriority:
      let lane = pickArbitraryLane(IdleLanes & ~wipLanes);
      if (lane === NoLane) {
        lane = pickArbitraryLane(IdleLanes);
      }
      return lane;
    default:
      // The remaining priorities are not valid for updates
      break;
  }
}
```

上面代码中可以看出，**根据`lanePriority`获取`lane`的工作流程是这样的**：

1. 从当前`lanePriority`对应的`lanes`中剔除正在被执行的`lanes`
2. 然后在剩余闲置的`lanes`中获取**位数最小的`lane`**，作为当前`Update`的`lane`
3. 如果当前优先级没有闲置的lanes，则会采取**降级处理**，继续递归寻找。

这个有一个细节要注意一下，`workInProgressRootIncludedLanes`这个变量是在每次**`执行update`**的时候才会赋值的，然后更新任务是异步执行的，可能会导致**多个Update的lane是一样的**，这些`Update`会批量执行。

> （？？？）笔者觉得这里应该用`workInProgressRootRenderLanes`来剔除正在被执行的lanes，而不是用`workInProgressRootIncludedLanes`。
>
> 因为`workInProgressRootRenderLanes`在每次更新完成之后会清空，但是`workInProgressRootIncludedLanes`在更新完成之后并不会清空。

比如下面这个例子

```javascript
componentDidMount(){
  this.state({a:1});
	this.state({b:2});
}
```

这里产生的两个`Update`的`lane`值就是一样的。



## 2. 优先级调度

整个优先级调度的过程发生在两个方法中

1. `scheduleUpdateOnFiber`：做一些调度之前的准备工作
2. `ensureRootIsScheduled`：执行优先级调度

### 2.1 调度前准备工作—`scheduleUpdateOnFiber`

上一节中我们提到，`scheduleUpdateOnFiber`方法是调度`update`的入口方法。在这个方法会完成一些调度之前的准备工作

1. 获取`fiberRootNode`节点

2. 收集所有未执行的更新，统一管理

> `scheduleUpdateOnFiber`的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L526)

#### 2.1.1 获取fiberRootNode

在进行优先级调度之前，React应用需要收集所有的更新，统一管理起来。在React应用运行的过程中，只有`fiberRootNode`节点是保存不变的，所以把管理所有更新的对象挂载在这个节点上是一个不错的选择。

所以先要获取`fiberRootNode`，通过调用`markUpdateLaneFromFiberToRoot`来获取

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L658)

```javascript
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // Walk the parent path to the root and update the child expiration time.
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    // 将当前节点的更新信息记录在祖先节点中
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

这个方法做了两个事情：

1. 向上遍历寻找`fiberRootNode`
2. **在遍历的过程中，将当前节点的更新记录在父节点中**，在后面`beginWork`阶段可以起到**优化**的作用

之前在节点`Fiber`节点的数据结构的时候，跳过了优先级相关的属性，这里我们来介绍一下

```javascript
// 优先级调度相关

// 表示当前节点中生成的Update中lane的集合
this.lanes = NoLanes;

// 表示subtree中所有节点生成的Update中lane的集合
this.childLanes = NoLanes;
```

在创建`workInProgress tree`的时候，有两个优化的操作是根据这两个属性来判断是否进入的。

1. 判断当前节点本次是否需要更新，如果不需要则进入优化处理。

   ```javascript
   else if (!includesSomeLane(renderLanes, updateLanes)) {
         didReceiveUpdate = false;
     		// ...省略代码
         return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
   }
   ```

2. 判断当前节点的`subtree`本次是否需要更新，如果不需要，之前复用整颗`subtree`

   ```javascript
   if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
     	// 复用整颗subtree
       return null;
     } else {
       // 复用子节点
       cloneChildFibers(current, workInProgress);
       return workInProgress.child;
     }
   ```

> 优化的完成流程可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3026)



#### 2.1.2 收集所有更新

在获取到`fiberRootNode`之后，需要将**所有未执行`Update`的`lane`**记录在这个节点上，进行通过统一管理。

收集的方法是`markRootUpdated`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberLane.js#L652)

```javascript
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number,
) {
  root.pendingLanes |= updateLane;
  
  const higherPriorityLanes = updateLane - 1; // Turns 0b1000 into 0b0111
  root.suspendedLanes &= higherPriorityLanes;
  root.pingedLanes &= higherPriorityLanes;

  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  eventTimes[index] = eventTime;
}
```

这个方法主要做了两个事情：

1. 将所有未执行`Update`的`lane`记录在`root.pendingLane`中，方便获取最高优先级
2. 记录了每个`Update`的开始时间



### 2.2 优先级调度—ensureRootIsScheduled

在调度之前的准备工作完成之后，`fiberRootNode.pendingLanes`上保存了所有`Update`的`lane`值，然后会执行`ensureRootIsScheduled`方法开始优先级调度。

> `ensureRootIsScheduled`方法对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L707)

整个优先级调度的过程都是在`ensureRootIsScheduled`方法中完成，可以分成以下四个部分

1. 标记每个`lane`的过期时间，记录过期`Update`的`lane`

2. 获取所有`Update`中的最高优先级以及对应的`renderLanes`
3. 判断是否打断低优任务
4. 在`Scheduler`中注册一个任务



#### 2.2.1 记录过期Update

为了防止低优先级任务一直无法被执行，也就是常说的”饿死“状态。React给每个`lane`根据优先级都标记了一个过期时间，一旦有`lane`出现过期，会以**同步优先级**调度一个`update`，执行对应的`Update`。

对应的方法是`markStarvedLanesAsExpired`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberLane.js#L410)

```javascript
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number,
): void {
  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        // 标记lane的过期时间
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // 过期的lane记录在expiredLanes中
      root.expiredLanes |= lane;
    }

    lanes &= ~lane;
  }
}
```

这个方法主要做了两个事情：

1. 给`pendingLanes`中每个`lane`标记一个过期时间
2. 将过期的`lane`记录在`root.expiredLanes`中

> 上面代码中可以看出，`root`中出了`pendingLanes`和`expiredLanes`之外，还存储了其他类型的`lanes`。
>
> 我们暂时不用考虑其他的，只需要弄懂`pendingLanes`和`expiredLanes`的含义即可



#### 2.2.2 计算最高优先级（lanePriority）及其renderLanes

接下来会调用`getNextLanes`从`root.pendingLanes`中获取最高的优先级，这个最高优先级就是下一个`update`的优先级。

```javascript
const nextLanes = getNextLanes(
    root,
  	// workInProgressRootRenderLanes表示正在执行的lanes
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
 );
```

> `getNextLanes`对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberLane.js#L249)

```javascript
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) {
    return_highestLanePriority = NoLanePriority;
    return NoLanes;
  }

  let nextLanes = NoLanes;
  let nextLanePriority = NoLanePriority;

  const expiredLanes = root.expiredLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;

  if (expiredLanes !== NoLanes) {
    // 如果有过期任务
    nextLanes = expiredLanes;
    nextLanePriority = return_highestLanePriority = SyncLanePriority;
  } else {
    // 没有过期任务，优先处理非idleLanes
    const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
    if (nonIdlePendingLanes !== NoLanes) {
      const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
      if (nonIdleUnblockedLanes !== NoLanes) {
        // 获取nonIdleUnblockedLanes中的最高优先级的lanes
        nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
        nextLanePriority = return_highestLanePriority;
      } else {
        const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
        if (nonIdlePingedLanes !== NoLanes) {
          nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
          nextLanePriority = return_highestLanePriority;
        }
      }
    } else {
      const unblockedLanes = pendingLanes & ~suspendedLanes;
      if (unblockedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(unblockedLanes);
        nextLanePriority = return_highestLanePriority;
      } else {
        if (pingedLanes !== NoLanes) {
          nextLanes = getHighestPriorityLanes(pingedLanes);
          nextLanePriority = return_highestLanePriority;
        }
      }
    }
  }

  if (nextLanes === NoLanes) {
    return NoLanes;
  }

  nextLanes = pendingLanes & getEqualOrHigherPriorityLanes(nextLanes);

  if (
    wipLanes !== NoLanes &&
    wipLanes !== nextLanes &&
    (wipLanes & suspendedLanes) === NoLanes
  ) {
    getHighestPriorityLanes(wipLanes);
    const wipLanePriority = return_highestLanePriority;
    // 获取到的最高优先级和当前执行的优先级对比
    if (nextLanePriority <= wipLanePriority) {
      // 同优先级运行时不打断，而且不取最新的nextLanes
      return wipLanes;
    } else {
      // 高优先级任务打断
      return_highestLanePriority = nextLanePriority;
    }
  }

  // ...省略部分代码
  return nextLanes;
}
```

**获取最高优先级的流程：**

1. 判断是否有`过期lanes`，如果有最高优先级直接为`SyncLanePriority`，然后对应的`renderLanes`为`root.expiredLanes`

2. 如果没有过期`lanes`，从`root.pendingLanes`中获取最高的优先级以及对应的`renderLanes`。

   > 如果从pendingLanes中获取最高优先级可以参考[getHighestPriorityLanes](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberLane.js#L121)方法

3. 比较获取到最高优先级和正在执行update的优先级，返回较大的`lanePriority`及其对应的`renderLanes`



**相同优先级的处理**

**（*）**这里有一个细节需要**注意一下**：

当计算出的最高优先级`nextLanePriority`和`wipLanePriority`相等的时候，并没有将新的`nextLanes`作为结果返回回来，返回的还是老的`wipLanes`。

因为前面讲过，在创建`Update`的时候，会选择对应优先级中`闲置的lane`赋值给`Update`，所以`nextLanes`有可能会比`wipLanes`大，执行的`Update`会更多，如果使用`nextLanes`会导致之前已经创建`workInProgress fiber`出现状态丢失。

这时有两种解决方案可以解决：

1. **同优先级不合并**，继续使用旧的wipLanes来执行，新的`Update`要放在下一个`update`去执行，会导致需要渲染两次。
2. **同优先级合并**，放弃之前已经生成的`workInProgress Fiber`片段，采用新的`nextLanes`作为`renderLanes`重新生成，只需要渲染一次。

上面代码看起来`React`采用了第一种解决方案，笔者写了一个`demo`也验证了这一点。

> 对应的demo可以看[这里](https://codesandbox.io/s/young-platform-el2gu?file=/src/App.js)，很明显的看到渲染了两次。

也就是说，**当优先级为A的`update`正在执行的时候，此时新创建的属于A优先级的`Update`并不会合并执行，而是需要更新两次。**



第二种方案在某种**特别极端**的情况下也会发生，上面给新的`Update`获取`lane`的方法中有一段这样的代码

```javascript
	case DefaultLanePriority: {
      let lane = pickArbitraryLane(DefaultLanes & ~wipLanes);
      if (lane === NoLane) {
        // 当前优先级的轨道已经占满，尝试降级处理
        lane = pickArbitraryLane(TransitionLanes & ~wipLanes);
        if (lane === NoLane) {
          // 如果TransitionLanes中的轨道也被占满，会重新使用DefaultLanes中的轨道
          lane = pickArbitraryLane(DefaultLanes);
        }
      }
      return lane;
    }
```

当`DefaultLanePriority`和`TransitionLanePriority`中的轨道都被占满的时候，会重新使用`DefaultLanePriority`中的轨道。就相当于将最新的`Update`合并到了当前`update`，所以需要**重新生成**整个`workInProgress Fiber tree`

React中使用了全局变量`workInProgressRootUpdatedLanes`来记录`update`执行过程中新创建`Update`的`lane`

> 对应的代码片段在`scheduleUpdateOnFiber`方法中

```javascript
if (root === workInProgressRoot) {
    if (
      deferRenderPhaseUpdateToNextBatch ||
      (executionContext & RenderContext) === NoContext
    ) {
      workInProgressRootUpdatedLanes = mergeLanes(
        workInProgressRootUpdatedLanes,
        lane,
      );
    }
  }
```

然后在执行`update`的时候会判断当前的`renderLanes`是否有`workInProgressRootUpdatedLanes`中的`lane`，**如果有的话就需要放弃之前生成`workInProgressFiber tree`片段，然后重新开始生成**。

> 对应的代码片段在`performConcurrentWorkOnRoot`方法中

```javascript
if (
    includesSomeLane(
      workInProgressRootIncludedLanes,
      workInProgressRootUpdatedLanes,
    )
  ) {
    prepareFreshStack(root, NoLanes);
  }
```

当然这是一种非常极端的情况，笔者并没有例子可以实现



#### 2.2.3 判断是否打断任务

在获取到当前最高的优先级和对应的`renderLanes`之后，就需要和当前`update`的优先级作比较，判断是否打断执行。

对应的代码片段

```javascript
if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    if (existingCallbackPriority === newCallbackPriority) {
     	// 优先级相同直接返回
      return;
    }
  	// 优先级大于当前update的优先级，则打断执行
  	// 这个方法会调用Scheduler中中断任务的api
    cancelCallback(existingCallbackNode);
  }
```

**至于低优先级的任务会继续保存在`root.pendingLanes`中，每个update执行完之后会在调用一次`ensureRootIsScheduled`方法，确保所有的更新都能执行。**

> 高优先级打断低优先级任务的[demo](https://codesandbox.io/s/sameprioritynotmerge-el2gu?file=/src/HighPriority.js)



#### 2.2.4 在Scheduler中注册任务

对应的代码片段

```javascript
 let newCallbackNode;
 if (newCallbackPriority === SyncLanePriority) {
		 // Legacy模式只会命中这个分支
     newCallbackNode = scheduleSyncCallback(
         performSyncWorkOnRoot.bind(null, root),
     );
 } else if (newCallbackPriority === SyncBatchedLanePriority) {
   	// BlockingMode只会命中这个分支
     newCallbackNode = scheduleCallback(
         ImmediateSchedulerPriority,
         performSyncWorkOnRoot.bind(null, root),
     );
 } else {
   	// 将lanePriority转化两次，转化成对应的SchedulerPriority
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
```

在`Scheduler`中注册任务之后，将控制前交给`Scheduler`，由`Scheduler`来异步调度执行



## 3. 总结

从上面的分析中我们可以知道，**优先级调度中的优先级定义、比较和打断的过程都是由React自主来完成的，获取到最高的优先级之后才会在`Scheduler`中注册一个任务**。通常情况下，`Sheduler`的任务列表中只会存在一个有效的任务。

