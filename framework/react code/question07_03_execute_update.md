# 执行update

在执行完`调度update`阶段之后，`Scheduler`会在合适的时机调用任务的回调函数，开启**`执行update`**阶段。

`Legacy`和`BlockingMode`会执行`performSyncWorkOnRoot`方法，`ConcurrentMode`会执行`performConcurrentWorkOnRoot`方法。

可以看到在`执行update`阶段会进入我们之前分析过的`render-commit`流程，这里我们不重复分析，重点来分析一下**`Update`的具体执行过程**。

以`ClassComponent`为例



## 1. 执行Update(*)

`Update`中包含了组件最新的状态，需要在组件重新渲染的时候来执行，也就是发生在`beginWork`阶段。

执行`Update`的方法是`processUpdateQueue`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactUpdateQueue.new.js#L394)

```javascript
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
): void {
  const queue: UpdateQueue<State> = (workInProgress.updateQueue: any);

  hasForceUpdate = false;

  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;

  // Check if there are pending updates. If so, transfer them to the base queue.
  let pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    queue.shared.pending = null;

    // 剪环
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    // Append pending updates to base queue
    if (lastBaseUpdate === null) {
      firstBaseUpdate = firstPendingUpdate;
    } else {
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;

    const current = workInProgress.alternate;

    // 在current Fiber中也保存一份未执行的update，防止更新中断，数据丢失
    if (current !== null) {
      // This is always non-null on a ClassComponent or HostRoot
      const currentQueue: UpdateQueue<State> = (current.updateQueue: any);
      const currentLastBaseUpdate = currentQueue.lastBaseUpdate;
      if (currentLastBaseUpdate !== lastBaseUpdate) {
        if (currentLastBaseUpdate === null) {
          currentQueue.firstBaseUpdate = firstPendingUpdate;
        } else {
          currentLastBaseUpdate.next = firstPendingUpdate;
        }
        currentQueue.lastBaseUpdate = lastPendingUpdate;
      }
    }
  }

  // These values may change as we process the queue.
  if (firstBaseUpdate !== null) {
   	// 用来记录执行Update之后的state
    let newState = queue.baseState;
   	// 用来记录未执行Update的lane
    let newLanes = NoLanes;
		// 用来记录执行Update之后的baseState
    let newBaseState = null;
    let newFirstBaseUpdate = null;
    let newLastBaseUpdate = null;

    let update = firstBaseUpdate;
    do {
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 处理不满足条件的Update
        const clone: Update<State> = {
          eventTime: updateEventTime,
          lane: updateLane,

          tag: update.tag,
          payload: update.payload,
          callback: update.callback,

          next: null,
        };
        // 跳过执行的Update记录起来
        if (newLastBaseUpdate === null) {
          newFirstBaseUpdate = newLastBaseUpdate = clone;
          newBaseState = newState;
        } else {
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        // 记录未执行Update的lane
        newLanes = mergeLanes(newLanes, updateLane);
      } else {
        // 处理满足条件的Update

        if (newLastBaseUpdate !== null) {
          // 一个update被跳过，后面所有的都会跳过
          const clone: Update<State> = {
            eventTime: updateEventTime,
            lane: NoLane,

            tag: update.tag,
            payload: update.payload,
            callback: update.callback,

            next: null,
          };
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }

        // Process this update.
        newState = getStateFromUpdate(
          workInProgress,
          queue,
          update,
          newState,
          props,
          instance,
        );
        const callback = update.callback;
        if (callback !== null) {
          // callback需要放在commit阶段处理
          workInProgress.flags |= Callback;
          const effects = queue.effects;
          if (effects === null) {
            // effects中保存 存在callback的update
            queue.effects = [update];
          } else {
            effects.push(update);
          }
        }
      }
      update = update.next;
      if (update === null) {
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          break;
        } else {
          const lastPendingUpdate = pendingQueue;
          const firstPendingUpdate = ((lastPendingUpdate.next: any): Update<State>);
          lastPendingUpdate.next = null;
          update = firstPendingUpdate;
          queue.lastBaseUpdate = lastPendingUpdate;
          queue.shared.pending = null;
        }
      }
    } while (true);

    if (newLastBaseUpdate === null) {
      newBaseState = newState;
    }

    queue.baseState = ((newBaseState: any): State);
    queue.firstBaseUpdate = newFirstBaseUpdate;
    queue.lastBaseUpdate = newLastBaseUpdate;

    markSkippedUpdateLanes(newLanes);
    workInProgress.lanes = newLanes;
    workInProgress.memoizedState = newState;
  }
}
```

这个方法代码稍微有点长，可以分成几个部分来分析，每个部分的分界线还是很明显的。

1. 构造`Update`执行链表`UpdateList`
2. 遍历`UpdateList`执行`Update`



### 1.1 构造Update执行链表UpdateList

`pendingUpdate`指的是本次更新**新增**的`Update`，存储在`updateQueue.shared.pending`中。

`baseUpdate`指的是上次更新之后**遗留**的`Update`，存储在`updateQueue.firstBaseUpdate和updateQueue.lastBaseUpdate`中

**构造`UpdateList`实际上就是将`pendingUpdate`拼接在`baseUpdate`后面。**

```javascript
const queue: UpdateQueue < State > = (workInProgress.updateQueue: any);

hasForceUpdate = false;

let firstBaseUpdate = queue.firstBaseUpdate;
let lastBaseUpdate = queue.lastBaseUpdate;

let pendingQueue = queue.shared.pending;
if (pendingQueue !== null) {
  	// 清空pending
    queue.shared.pending = null;

    // 剪环
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    if (lastBaseUpdate === null) {
        firstBaseUpdate = firstPendingUpdate;
    } else {
        lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;

    const current = workInProgress.alternate;

    // 在current Fiber中也保存一份未执行的update，防止更新中断，Update丢失
    if (current !== null) {
        const currentQueue: UpdateQueue < State > = (current.updateQueue: any);
        const currentLastBaseUpdate = currentQueue.lastBaseUpdate;
        if (currentLastBaseUpdate !== lastBaseUpdate) {
            if (currentLastBaseUpdate === null) {
                currentQueue.firstBaseUpdate = firstPendingUpdate;
            } else {
                currentLastBaseUpdate.next = firstPendingUpdate;
            }
            currentQueue.lastBaseUpdate = lastPendingUpdate;
        }
    }
}
```

拼接的过程中有一个细节，`pendingUpdate`是一个单向环状链表，所以在拼接之前有一个**剪环**的操作。

上面代码中可以看出，在拼接的过程中还做了其他的操作：**将`pendingUpdate`存储在currentFiber中**。

那为什么需要一步这样的操作呢？

原因是为了**防止`pendingUpdate`丢失**，当`本次update`被`高优的update`打断执行的时候，`workInProgressFiber tree`会重新构建，如果不把`pendingUpdate`存储起来就会丢失。

所以React会将`pendingUpdate`在`currentFiber`中存储一份，拼接在`baseUpdate`后面，然后在组件更新的时候会**基于`currentFiber.updateQueue`克隆出`workInProgressFiber.updateQueue`**，保证`Update`不会丢失。

> 克隆的方法可以看[cloneUpdateQueue](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactUpdateQueue.new.js#L165)



### 1.2 遍历UpdateList执行Update

完成拼接之后会遍历`UpdateList`执行满足条件的`Update`，得到新的`state`。

判断`Update`是否能执行的条件是：**当前`Update`的`lane`是否在`renderLanes`中。**

```javascript
// 判断条件
if (!isSubsetOfLanes(renderLanes, updateLane))
  
// 方法实现
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}
```

整个执行过程也可以分成两个部分：

1. 处理不满足条件的`Update`
2. 处理满足条件的`Update`



#### 1.2.1 处理不满足条件的Update

```javascript
if (!isSubsetOfLanes(renderLanes, updateLane)) {
    const clone: Update < State > = {
        eventTime: updateEventTime,
        lane: updateLane,

        tag: update.tag,
        payload: update.payload,
        callback: update.callback,

        next: null,
    };
    if (newLastBaseUpdate === null) {
        newFirstBaseUpdate = newLastBaseUpdate = clone;
      	// 防止baseUpdate不正确
        newBaseState = newState;
    } else {
        newLastBaseUpdate = newLastBaseUpdate.next = clone;
    }
    newLanes = mergeLanes(newLanes, updateLane);
}
```

处理步骤：

1. 克隆`Update`
2. 放在新的`baseUpdate`链表中
3. 记录当前`Update`的`lane`



#### 1.2.2 处理满足条件的Update

```javascript
if (newLastBaseUpdate !== null) {
    // 一个Update被跳过，后面所有的都会跳过
    const clone: Update < State > = {
        eventTime: updateEventTime,
      	// 注意这里lane设置为0，是所有renderLanes的子集，后续更新时这个Update一定会执行
        lane: NoLane,
        tag: update.tag,
        payload: update.payload,
        callback: update.callback,

        next: null,
    };
    newLastBaseUpdate = newLastBaseUpdate.next = clone;
}

// 执行Update
newState = getStateFromUpdate(
    workInProgress,
    queue,
    update,
    newState,
    props,
    instance,
);
// 记录callback
const callback = update.callback;
if (callback !== null) {
    // callback需要放在commit阶段处理
    workInProgress.flags |= Callback;
    const effects = queue.effects;
    if (effects === null) {
        // effects中保存 存在callback的update
        queue.effects = [update];
    } else {
        effects.push(update);
    }
}
```

上面代码中有两个关键点：

1. 为什么满足条件的`Update`也会被克隆记录？
2. 为什么克隆的时候`lane`要设置为`Nolanes`？

其实这两个处理是为了解决同一个问题：**保证状态依赖的连续性**

当某个Update因为优先级低被跳过的时候，后面的`Update`可能会依赖这个`Update`的执行结果，所以**当出现第一个被跳过的`Update`时，其后面所有的`Update`无论优先级高低都需要被记录下来。**这也就解释了第一个问题。

下面来解释第二个问题

虽然为了保证状态的连续性，需要将`被跳过Update`后面满足条件的`Update`也记录在`newBaseUpdate`中，但是这个`Update`本身还是会被执行的，也就是说这个`Update`参与了计算本次更新的`state`。**如果不将`lane`设置为`NoLane(也就是0)`，那么下次`执行update`的时候`renderLanes`可能并不包含这个`Update`的`lane`，就会导致这个`Update`无法执行，会导致最终的`state`计算出错。**

所以说为了保证状态的连续性，这两个操作都是很有必要的。



#### 1.2.3 newState和newBaseState

上面执行过程中还有一个关键的点需要分析一下，就是`newState`和`newBaseState`的取值问题。

正常情况下，如果所有的Update都执行完的话，`newState`应该等于`newBaseState`。

但是如果出现低优先级`Update`跳过执行的情况，那么两者的取值逻辑会有所不同。

1. `newState`的取值就是**所有满足条件的`Update`执行完之后得到的值**。
2. `newBaseState`的取值是**`第一个被跳过Update`之前的`Update`执行之后得到的值**

`newBaseState`之所以会这样取值是为了保证：**每个`Update`执行前的`baseState`是正确的，就是之前所有`Update`执行后的结果**。

基于上面分析的执行规则，`被跳过Update`后面的`高优Update`也会执行，如果执行之后的结果也放到`newBaseState`上，会导致下一次`update`再执行其他`Update`的时候`baseState`不正确，从而导致最终得到的`state`出错。

但是`newState`需要实时拿到最新的执行结果，对用户的操作做出反应，保证最后的状态是对的即可。



#### 1.2.4 举个例子

这里我们举一个官方代码注释中的例子，来说明一下执行`Update`的流程

有下面几个Update，其中字母表示状态，数字表示优先级，数字越小优先级越高

```javascript
A1 - B2 - C1 - D2
```

第一次执行`update`，`renderLanes`为1，所以`A和C`会被执行，结果为：

```javascript
{
  baseState: 'A',
  UpdateS: [B2,C1,D2],
  state: 'AC'
}
// 中间过程baseState和state可能值不一样
```

第二次执行`update`，`renderLanes`为2，剩余的`Update`都会被执行，结果为：

```javascript
{
  baseState: 'ABCD',
  UpdateS: [],
  state: 'ABCD'
}
// 最终情况下，baseState和state值是一样的
```

