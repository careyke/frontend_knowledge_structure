# Hooks的数据结构和更新流程

任何高层次的抽象设计落在实处都是对于数据结构和算法的设计。

在分析`Hooks`的实现之前，有必要先来了解一下`Hooks`的数据结构。

> 这里的`Hooks`指的是上一篇文章中提到`StateHooks`，在源码中称为`Hooks`



## 1. Hook的数据结构

前面我们提到过，`Hooks`是出现可以将复杂的`state`切分成多个简单的`state`，`Hooks`和`FunctionComponent`是相互独立的，所以每个`Hook`中需要维护自己的`state`。

看一下`Hook`的数据结构：

```javascript
type Update<S, A> = {|
  lane: Lane,
  action: A,
  eagerReducer: ((S, A) => S) | null,
  eagerState: S | null,
  next: Update<S, A>,
  priority?: ReactPriorityLevel,
|};

type UpdateQueue<S, A> = {|
  pending: Update<S, A> | null,
  dispatch: (A => mixed) | null,
  lastRenderedReducer: ((S, A) => S) | null,
  lastRenderedState: S | null,
|};

export type Hook = {|
  memoizedState: any,
  baseState: any,
  baseQueue: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null,
  next: Hook | null,
|};
```

> 一眼看过去，结构体中有一些对于Redux使用者来说非常熟悉的单词：`action、reducer和dispatch`，这些都是`Redux`中的概念。`Redux`的作者`Dan`在加入`React`研发团队之后，将`Redux`优秀的设计思想也加入了`Hooks`的设计中。
>
> **可以将每个Hook看成是一个简化的Redux**，这也是后面为什么会以`useReducer`为例来分析`Hook`的更新流程的原因

上面代码中包含三个数据结构，下面分别解释一下三个结构体中的字段：

1. `Update`结构体，**描述当前`Hook`的一次状态变化**，可以类比`ClassComponent`中的`Update`。

   - lane：状态变化的优先级（轨道）
   - action：状态变化的载体，在`useState`中`action`表示的就是最新的`state`值。
   - eagerReducer：`Update预执行`时调用的`reducer`，后面会详细讲解
   - eagerState：`Update预执行`产生的`state`，也是和优化有关，后面详细讲解
   - next：连接下一个`Update`
   - priority：预留字段，暂时没有使用

   

2. `UpdateQueue`结构体，描述`Hook`的待更新队列

   - pending：本次更新新增的`Update`，**单向环状链表**
   - dispatch：更新的触发器
   - lastRenderedReducer：当前Hook上次挂载时注册的`reducer`
   - lastRenderedState：当前`Hook`上次更新时计算出的`state`

   

3. `Hook`结构体

   - memoizedState：当前`Hook`对应的`state`
   - baseState：待更新队列执行前的基本状态，`Update`相互依赖时发挥作用
   - baseQueue：上次更新中遗留的未执行`Update`，也是一个**单向环状链表**
   - queue: `Hook`本次更新对应的更新队列
   - next：连接下一个`Hook`



## 2. FunctionComponent 与 Hooks

对应`ClassComponent`来说，`state`可以持久化保存在每个实例对象中。然而`FunctionComponent`是没有实例的，那么`Hooks`产生的状态在`FunctionComponent`中如何保存呢？

**在`FunctionComponent`中，`Hooks`产生的状态是以临时变量的形式存在**，每个`Hook函数`在运行之后都会返回对应的状态值，`FunctionComponent`中可以用任意的变量来保存这个值。



也就是说，`Hook`产生的`state`在`FunctionComponent`中保存的时候是没有特殊标志的，那么如何能保证状态和Hook能够一一对应呢？

**`FunctionComponent`使用`Hooks方法`的调用顺序不变来保证状态和`Hook`能够一一对应起来**，组件中挂载的Hooks会以**链表的形式**保存在对应的`Fiber`节点中，在`memoizedState`这个属性上，然后**每次组件渲染的时候会遍历整个链表，依次获取每个Hook，计算出最新的state保存在`FunctionComponent`的临时变量中**。

<img src="./images/function_memoizedState.jpg" alt="function_memoizedState" />



这个依靠调用顺序来保证一一对应的机制明显是不够可靠的，所以在用法上`Hook方法`有一些限制，不能使用在条件语句中，不能使用在`useEffect`中等。总结起来就是：**在`FunctionComponent`函数体执行的时候，所有的`Hook方法`都必须要按固定顺序执行，一旦有某个方法不执行，更新就会出错。**



## 3. Hooks的更新流程

在`beginWork`中，`FunctionComponnet`会调用`renderWithHooks`方法来执行对应的渲染函数，**当执行到钩子函数的时候会创建或者更新对应的`Hook`对象。**

所以`renderWithHooks`方法就是`Hooks`更新流程的入口函数

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L327)

```javascript
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  // 全局变量保存renderLanes
  renderLanes = nextRenderLanes;
  // 保存最新的workInProgressFiber
  currentlyRenderingFiber = workInProgress;

  // 先清空状态
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  if (__DEV__) {
  } else {
    // 根据组件所处的阶段来决定使用对应的HookDispatcher
    ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
  }

  let children = Component(props, secondArg);

  // 判断是否在组件render过程中有调用setState，如果有的话就优化处理
  if (didScheduleRenderPhaseUpdateDuringThisPass) {
    let numberOfReRenders: number = 0;
    do {
      didScheduleRenderPhaseUpdateDuringThisPass = false;
      
      numberOfReRenders += 1;

      // Start over from the beginning of the list
      currentHook = null;
      workInProgressHook = null;

      workInProgress.updateQueue = null;

      // 切换处理器，优化处理
      ReactCurrentDispatcher.current = __DEV__
        ? HooksDispatcherOnRerenderInDEV
        : HooksDispatcherOnRerender;

      children = Component(props, secondArg);
    } while (didScheduleRenderPhaseUpdateDuringThisPass);
  }

  // render函数执行完成之后立马切换HookDispatcher
  // 对应不规范的时候报错
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  // 判断是否提前return，报错
  const didRenderTooFewHooks =
    currentHook !== null && currentHook.next !== null;

  renderLanes = NoLanes;
  currentlyRenderingFiber = (null: any); // 执行完成之后清空全局变量

  // 清空全局变量
  currentHook = null;
  workInProgressHook = null;

  didScheduleRenderPhaseUpdate = false;

  invariant(
    !didRenderTooFewHooks,
    'Rendered fewer hooks than expected. This may be caused by an accidental ' +
      'early return statement.',
  );
  
  return children;
}
```

上面方法中有一个重要的全局变量`ReactCurrentDispatcher`，**这个变量用来保存当前时期对应的`HookDispatcher`，执行Hook函数的时候会调用`HookDispatcher`中对应的处理函数**

`HookDispatcher`的结构，列举两个常用的`HookDispatcher`

```javascript
const HooksDispatcherOnMount: Dispatcher = {
  readContext,

  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  // ... 省略部分
};

const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,

  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  // ... 省略部分
};
```

`renderWithHooks`方法中最重要的一个工作是**根据不同的时期，给`ReactCurrentDispatcher`赋值对应的`HookDispatcher`**。

`React Hooks`中一共存在4个不同的`HookDispatcher`，分别代表**4种不用的情况**。

1. HooksDispatcherOnMount：组件`mount`时使用，创建`Hook`对象
2. HooksDispatcherOnUpdate：组件`update`时使用，更新`Hook`对象
3. HooksDispatcherOnRerender：组件存在 **渲染时更新** 的时候使用（re-render），优化处理
4. ContextOnlyDispatcher：组件`render函数`执行完成之后使用，提供不规范使用的报错

下面我们以`useReducer`为例，来分别介绍这四种情况。

> 之所以使用`useReducer`而不是`useState`，是因为`useState`本质上是一个简化版的`useReducer`。后续会对比这两个钩子



### 3.1 创建Hook对象

在组件`mount`阶段，调用`useReducer`钩子会执行`mountReducer`方法

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L609)

```javascript
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 创建一个Hook对象
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });

	// 创建setState函数，绑定fiber节点
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```

上面代码中，主要的功能是

1. 创建一个`Hook`对象
2. 返回`Hook`对象中的`state`以及一个修改`state`的方法，`FunctionComponent`使用临时变量保存之后就可以使用

看一下`mountWorkInProgressHook`方法

```javascript
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

这里主要是想介绍一个全局变量`workInProgressHook`，这个变量用来**保存`workInProgressFiber`中当前正在执行的`Hook`对象**，对应的还有一个`currentHook`来**保存`currentFiber`中对应的当前正在执行的`Hook`对象**。

这两个变化在遍历`Hooks链表`的时候会使用上，在`render函数`执行完成之后会清空。



### 3.2 更新Hook对象

`FunctionComponent`的更新过程和`ClassComponent`的基本流程都是一样的，公用一套更新机制

1. 创建update
2. 调度update
3. 执行update

其中**调度`update`是完全一致的，创建`update`和执行`update`中的实现细节会有所不同**，下面我们分析一下不同的地方。



#### 3.2.1 创建update

前面讲过，执行Hook函数时会返回一个修改状态的方法，实际上会调用`dispatchAction`方法

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L1708)

```javascript
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber);

  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };

  // 单向环状链表
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  const alternate = fiber.alternate;
  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // 说明当前在render时执行了dispatch操作
    // render过程中有更新操作
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      // 当前节点没有遗留Update时
      // 尝试优化本次update
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        try {
          // 预更新
          const currentState: S = (queue.lastRenderedState: any);
          const eagerState = lastRenderedReducer(currentState, action);
          update.eagerReducer = lastRenderedReducer;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            return;
          }
        } catch (error) {
        } finally {
        }
      }
    }
    // 进入调度update
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}
```

上面代码中可以看到，除了和`ClassComponent`一样会`创建Update`和调用`调度update`方法之外，其中包含两个优化的操作：

1. **优化`render`过程产生的更新(update-in-render)**
2. **优化不必要的更新**

这两个优化都涉及到执行`update`阶段，等分析完执行`update`之后再来详细分析这两个优化的操作



#### 3.2.2 执行update

`FunctionComponent`执行update的流程和`ClassComponent`基本也是一样的，不同的点在于`执行Update对象`的处理上。对于`useReducer`来说，`执行Update`发生在`updateReducer`方法中。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L636)

```javascript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 从Hook链表中获取当前Hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

	// 重置reducer
  queue.lastRenderedReducer = reducer;

  const current: Hook = (currentHook: any);

  let baseQueue = current.baseQueue;

  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    // 拼接pendingUpdate和baseUpdate 但是没有剪环
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

	// baseQueue也是一个单向环状链表
  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            lane: NoLane, // 优先级最该，后续每次都需要执行
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        if (update.eagerReducer === reducer) {
          // 优化处理
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      // baseState和state取值逻辑不同
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

	// dispatch方法不变！！！
  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

上面代码中可以看出，执行`Update`的过程和`ClassComponent`也基本是一样的。

1. 从`currentFiber`中克隆一个`Hook`对象

2. 拼接`baseUpdate`和`pendingUpdate`

3. 根据`renderLanes`执行符合条件的`Update`，不符合条件的`Update`克隆副本暂存在`baseUpdate`中，下一次更新再执行。



##### 3.2.2.1 克隆Hook

这里我们来分析一下克隆Hook对象的过程，在`updateWorkInProgressHook`方法中

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberHooks.new.js#L537)

```javascript
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // update-in-render 会走到这里
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;

    currentHook = nextCurrentHook;
  } else {
    // 正常更新会走这里，从currentFiber中对应的Hook对象
    currentHook = nextCurrentHook;

    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null,
    };

    if (workInProgressHook === null) {
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

这个方法的主要作用就是**遍历`currentFiber`中挂载的`Hook`对象，然后克隆一个`Hook`副本挂载在`workInProgressFiber`中并返回**。



如果单纯看这个函数是比较迷惑的，因为看上面代码的逻辑，感觉正常情况下并不会走到克隆那段逻辑。因为在创建`workInProgressFiber`的时候，如果节点是`update阶段`，会执行以下代码

```javascript
workInProgress.memoizedState = current.memoizedState;
workInProgress.updateQueue = current.updateQueue;
```

> 对应的源代码看[createWorkInProgress](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiber.new.js#L248)

所以看起来`currentFiber`和`workInProgressFiber`的`memoizedState`是共享的，并不需要克隆`Hook`对象。

但是实际上不是这样的，在`renderWithHooks`方法中，执行render函数之前执行了**清空**的操作

```javascript
workInProgress.memoizedState = null;
workInProgress.updateQueue = null;
workInProgress.lanes = NoLanes;
```

所以需要从currentFiber中克隆Hook对象。



这里又有一个疑问，**为什么需要先清空，然后去克隆呢？使用共享的数据有什么问题吗？**

**在高优先级打断低优先级任务的场景下，使用共享的数据是有问题的**。由于是共享的数据结构，会**导致低优先级任务更新的状态会保存在`currentFiber`中，所以在执行高优任务的时候，会把低优任务会顺带执行了**，从而失去了优先级调度的意义。

所以克隆Hook是非常有必要的。



##### 3.2.2.2 dispatch保持不变，reducer可变

在`updateReducer`方法中还有两个细节：

1. **`dispatch`方法保持不变，只在`mount`阶段赋值，后续`update`阶段直接继续使用这个值**。所以**`dispatchAction`方法中的`fiber和queue`参数也是一直不变的**，就是`mount`阶段设置的值。乍一看可能会觉得有点问题，每次更新时应该使用最新的值才是合理的，但是其实仔细分析下来，也确实没有影响。（感觉是个优化操作）
2. `useReducer`中，**`reducer`是可以变化的**，每次执行的时候可以传入新的`reducer`。



### 3.3 优化update-in-render(rerender阶段)

在分析`dispatchAction`时，我们提到过其中涉及两种优化操作，这里我们来分析其中的一种 —— `update-in-render`。

**`update-in-render`表示的就是在执行`render`函数的过程中，执行更新操作。**举个例子

```react
function App(){
  const [count,setCount] = useState(0);
  if(count === 0){
    setCount(1);
  }
  return <span>{count}</span>;
}
```

这里的`setCount`就是在`render`函数执行的过程中调用。React内部对这种操作做了一些优化，**将新产生的Update在本次更新执行，从而减少一次渲染。**

优化的具体代码：

`dispatchAction`函数中：

```javascript
if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
) {
    // render过程中有更新操作
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
}
```

`renderWithHooks`函数中：

```javascript
if (didScheduleRenderPhaseUpdateDuringThisPass) {
    let numberOfReRenders: number = 0;
    do {
        didScheduleRenderPhaseUpdateDuringThisPass = false;
        numberOfReRenders += 1;
        // Start over from the beginning of the list
        currentHook = null;
        workInProgressHook = null;

        workInProgress.updateQueue = null;

        // 切换处理器
        ReactCurrentDispatcher.current = __DEV__ ?
            HooksDispatcherOnRerenderInDEV :
            HooksDispatcherOnRerender;

        children = Component(props, secondArg);
    } while (didScheduleRenderPhaseUpdateDuringThisPass);
}
```

对于这种情况，**`React`会切换`HookDispatcher`，然后再执行一次`render`函数**，进入**`rerender`**阶段。

#### 3.3.1 rerenderReducer

```javascript
function rerenderReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  queue.lastRenderedReducer = reducer;

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  const lastRenderPhaseUpdate = queue.pending;
  let newState = hook.memoizedState;
  if (lastRenderPhaseUpdate !== null) {
    queue.pending = null;
		// 没有拼接pendingUpdate和baseUpdate的操作
    const firstRenderPhaseUpdate = lastRenderPhaseUpdate.next;
    let update = firstRenderPhaseUpdate;
    // 执行rerender产生的Update
    do {
      const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== firstRenderPhaseUpdate);

    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
    if (hook.baseQueue === null) {
      hook.baseState = newState;
    }

    queue.lastRenderedState = newState;
  }
  return [newState, dispatch];
}
```

这个执行过程和update阶段执行过程有以下不同：

1. 没有拼接操作。也就是说**这个`Update`不会依赖之前产生的`Update`，只依赖当前的状态。**

2. 没有优先级判断。也就是说**这个`Update`本次更新会执行，不会被闲置。**

优化的效果：**`render`函数执行两次，但是只执行一次`update`**



### 3.4 优化不必要的更新

如果更新前后值没有什么变化，这个就属于不必要的更新。举个例子

```react
function App(){
  const [count,setCount] = useState(0);
  
  const handleClick = ()=>{
    setCount(0);
  }
  
  return <span onClick={handleClick}>{count}</span>;
}
```

点击`span`的之后组件并不会重新渲染。



#### 3.4.1 Update预执行

由于需要判断`当前Update`是否会导致状态更新，所以在调度`update`之前需要预执行这个`Update`。

对应的代码在`dispatchAction`函数中

```javascript
if (
    fiber.lanes === NoLanes &&
    (alternate === null || alternate.lanes === NoLanes)
) {
  	// 当前节点上不存在其他的更新
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
        let prevDispatcher;
        try {
            const currentState: S = (queue.lastRenderedState: any);
            const eagerState = lastRenderedReducer(currentState, action);
            update.eagerReducer = lastRenderedReducer;
            update.eagerState = eagerState;
          	// Object.is
            if (is(eagerState, currentState)) {
              	// 跳过调度update
                return;
            }
        } catch (error) {
        } finally {
        }
    }
}
scheduleUpdateOnFiber(fiber, lane, eventTime);
```

`Update`预执行的前提：**当前节点上不存在其他的Update**。如果存在其他的Update，为了保证状态依赖的连续性，无法优化，所以也就没有预执行。

**当`Update`满足预执行的条件时，而且执行的结果并没有导致状态变化，那么当前`Update`不会调度`update`**。也就是不触发更新



这里有两个点需要**注意**：

1. 虽然预执行`Update`不会调度`update`，但是这个`Update对象`还是会挂载在`Fiber`节点中，意味着**后续的更新中还是会执行这些`Update`**。
2. `预执行Update`的时候使用的是`lastRenderedReducer`，但是**`reducer`是可以更改的**。

这里之所以所有的Update仍然会挂载在Fiber节点中，是为了**防止状态更新错误**。

考虑一种情况，`如果Update A可以预执行，Update B不能预执行，B依赖A，而且组件重新渲染的时候传入了新的reducer`。此时如果`A`预执行完成之后就丢弃，那么`B`执行完成之后得到的最终状态就是错误的。

所以说，**预执行的Update并不能丢弃。**



#### 3.4.2 不重复执行reducer

由于预处理的`Update`无法丢弃，为了避免重复执行`reducer`，在`updateReducer`方法中也做了一个优化。

```javascript
if (update.eagerReducer === reducer) {
  	// reducer没有改变
    newState = ((update.eagerState: any): S);
} else {
  	// reducer发生改变
    const action = update.action;
    newState = reducer(newState, action);
}
```



### 3.5 不规范使用Hook报错

上面我们提到，在`Hook`函数必须要在r`ender函数`执行的期间来执行，否则就会导致状态对应错误。React为了防止这种使用不规范的错误，运行时抛出了相应的错误提示。

在`renderWithHooks`函数中，当`render函数`执行完成之后，会**切换`HookDispatcher`**

```javascript
ReactCurrentDispatcher.current = ContextOnlyDispatcher;
```

若出现`异步调用useReducer`的情况，会调用`ContextOnlyDispatcher`中对应的方法`throwInvalidHookError`

```javascript
function throwInvalidHookError() {
  invariant(
    false,
    'Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for' +
      ' one of the following reasons:\n' +
      '1. You might have mismatching versions of React and the renderer (such as React DOM)\n' +
      '2. You might be breaking the Rules of Hooks\n' +
      '3. You might have more than one copy of React in the same app\n' +
      'See https://reactjs.org/link/invalid-hook-call for tips about how to debug and fix this problem.',
  );
}
```

抛出错误！其他的Hook函数也是相似的逻辑。

> `render函数`指的就是`FunctionComponent`函数本身，和ClassComponent中的render方法相似，故采用了同样的名称



## 4. 总结

至此整个`Hook`的数据结构和更新过程就算是分析完了，更新的过程和`ClassComponnet`基本是一样的，主要是Hook的数据结构和后面的更新优化需要反复分析。

