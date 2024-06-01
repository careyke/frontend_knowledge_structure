# useState和useReducer

上篇文章中我们以`useReducer`为例介绍了`Hooks`的更新流程。提到了`useState`是简易版的`useReducer`

下面我们来具体看看`useState`的实现代码，比较其和`useReducer`之间的关系。

## 1. mount阶段

对应的函数是`mountState`

```javascript
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    // 这个函数只在mount阶段执行一次
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer, // reducer
    lastRenderedState: (initialState: any),
  });
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```

## 2. update阶段

对应的函数是`updateState`

```javascript
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

## 3. rerender阶段

对应的函数是`rerenderState`

```javascript
function rerenderState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return rerenderReducer(basicStateReducer, (initialState: any));
}
```

上面三个阶段的代码可以看出，`useState`的实现基本都是在复用`useReducer`的代码。

最大的不同在于：**`useReducer`使用的是开发者定义的`reducer`，`useState`使用的是固定的`reducer`**

看一下固定的`reducer`：`basicStateReducer`

```javascript
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

逻辑非常简单。
