# Hooks的数据结构和更新流程

任何高层次的抽象设计落在实处都是对于数据结构和算法的设计。

在分析`Hooks`的实现之前，有必要先来了解一下`Hooks`的数据结构。

> 这里的`Hooks`指的是上一篇文章中提到`StateHooks`，在源码中称为`Hooks`



## 1. Hook的数据结构

前面我们提到过吗，`Hooks`是出现可以将复杂的`state`切分成多个简单的`state`，`Hooks`和`FunctionComponent`是相互独立的，所以每个`Hook`中需要维护自己的`state`。

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

> 一眼看过去，结构体中有一些对于React使用者来说非常熟悉的单词：`action、reducer和dispatch`，这些都是`Redux`中的概念。`Redux`的作者`Dan`在加入`React`研发团队之后，将`Redux`优秀的设计思想也加入了`Hooks`的设计中。
>
> **可以将每个Hook看成是一个简化的Redux**，这也是后面为什么一会`useReducer`为例来分析`Hook`的更新流程

上面代码中包含三个数据结构，下面分别解释一下三个结构体中的字段：

1. `Update`结构体，**描述当前`Hook`的一次状态变化**，可以类比`ClassComponent`中的`Update`。

   - lane：状态变化的优先级（轨道）
   - action：状态变化的载体，在`useState`中`action`表示的就是最新的`state`值。
   - eagerReducer：尝试优化当前`Update`时调用的`reducer`，后面会详细讲解
   - eagerState：调用eagerReducer之后生成的state，也是和优化有关，后面详细讲解
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
   - baseQueue：上次更新中遗留的未执行`Update`，也是一个**单选环状链表**
   - queue: `Hook`对应的更新队列
   - next：连接下一个`Hook`



## 2. FunctionComponent 与 Hooks

对应`ClassComponent`来说，`state`可以持久化保存在每个实例对象中。然而`FunctionComponent`是没有实例的，那么`Hooks`产生的状态在`FunctionComponent`中如何保存呢？

**在`FunctionComponent`中，`Hooks`产生的状态是以临时变量的形式存在**，每个`Hook函数`在运行之后都会返回对应的状态值，`FunctionComponent`中可以用任意的变量来保存这个值。



也就是说，`Hook`产生的`state`在`FunctionComponent`中保存的时候是没有特殊标志的，那么如何能保证状态和Hook能够一一对应呢？

**`FunctionComponent`使用`Hooks方法`的调用顺序不变来保证状态和`Hook`能够一一对应起来**，组件中挂载的Hooks会以**链表的形式**保存在对应的`Fiber`节点中，在`memoizedState`这个属性上，然后**每次组件渲染的时候会遍历整个链表，依次获取每个Hook，计算出最新的state保存在`FunctionComponent`的临时变量中**。

<img src="./images/function_memoizedState.jpg" alt="function_memoizedState" />



这个依靠调用顺序来保证一一对应的机制明显是不够可靠的，所以在用法上`Hook方法`有一些限制，不能使用在条件语句中，不能使用在`useEffect`中等。总结起来就是：**在`FunctionComponent`函数体执行的时候，所有的`Hook方法`都必须要执行，一旦有某个方法不执行，更新就会出错。**



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

上面方法中有一个重要的全局变量`ReactCurrentDispatcher`，**这个变量用来保存当前时期对应的`HookDispatcher`，执行Hook函数的时候会调用`HookDispatcher`中对应的处理函数。**

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
3. HooksDispatcherOnRerender：组件存在 **渲染时更新** 的时候使用，优化处理
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

这里主要是想介绍一个全局变量`workInProgressHook`，这个变量用来**保存`workInProgressFiber`中挂载的`Hook`对象**，对应的还有一个`currentHook`来**保存`currentFiber`中挂载的`Hook`对象**。

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
    // render过程中有更新操作
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      // 尝试优化本次update
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        try {
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

1. **优化`render`过程产生的更新**
2. **优化不必要的更新**

这两个优化都涉及到执行`update`阶段，等分析完执行`update`之后再来详细分析这两个优化的操作



#### 3.2.2 执行update

