# 保存无副作用状态：useCallback、useMemo和useRef

在日常开发`React`组件的时候，我们常常会碰到一些特殊的值，有以下特点：

1. **持久化保存**
2. **值可能会随着state的变化而变化**
3. **它的修改不需要触发组件更新**

我把这类值称为**无副作用的状态**。

> 所谓无副作用，就是它的更改不会导致组件的更新

这类无副作用的状态并不适合保存在`state`中的，不仅会污染`state`，而且可能会导致组件多次更新。

举两个例子：

ClassComponent：

```react
class App extends React.Component {
  state = {
    didMount: false
  };

  componentDidMount() {
    this.setState({ didMount: true });
  }

  render() {
    return <span>1</span>;
  }
}
```

这里`didMount`放在`state`中会导致组件多更新一次



FunctionComponent：(legacy模式下)

```react
function Test() {
  const [count, setCount] = useState(0);
  const [bool, setBool] = useState(false);
  const buttonRef = useRef();

  useEffect(() => {
    buttonRef.current.addEventListener("click", () => {
      setCount(1);
      setBool(true);
    });
  }, []);

  return (
    <div>
      <button ref={buttonRef}>button</button>
    </div>
  );
}
```

在legacy模式下，点击按钮会渲染两次。

上面代码中的`didMount`和`bool`都会无副作用的状态，不应该保存在state中。

**在`ClassComponent`中，通常是直接将无副作用状态保存在组件实例`this`上**。但是`FunctionComponent`中并没有实例，为了保存无副作用的状态，React提供了三种`Hook`来保存。

1. useCallback
2. useMemo
3. useRef



## 1. useCallback

`useCallback`对应的`Hook对象`用来保存**函数类型**的无副作用状态，而且提供了状态变化的依赖项。**当依赖项变化的时候，对应的状态才会发生变化。**

> 可以类比`Vue`中的计算属性

### 1.1 mount阶段

对应的函数是`mountCallback`

```javascript
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

可以看到，`useCallback`对应`Hook`的`memoizeState`中保存了函数和依赖项。

### 1.2 update阶段

对应的函数是`updateCallback`

```javascript
function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 对比依赖项是否发生变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 没有变化，直接返回之前的值
        return prevState[0];
      }
    }
  }
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

看一下比较依赖项的方法`areHookInputsEqual`

```javascript
function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
) {
  if (prevDeps === null) {
    return false;
  }
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) { //object.is
      continue;
    }
    return false;
  }
  return true;
}
```

整体实现上还是比较简单的。



### 1.3 闭包陷阱

`useCallback`在使用中常常会遇到一个问题，就是在**函数中获取不到最新的state值**，也就是常说的闭包陷阱。

举个例子：

```react
const Test=()=>{
  const [num, setNum] = useState(0);
  
  const handleAdd=()=>{
    setNum(prev => prev+1);
  }
  
  const handlePrint=useCallback(()=>{
    console.log(num);
  },[])
  
  return (
  	<div>
    	<button onClick={handleAdd}>加1</button>
      <button onClick={handlePrint}>输出</button>
    </div>
  );
}
```

这里不管点击几次“加1”，再点击“输出”按钮，控制台打印的始终是0。

因为`useCallback`传入的函数内部使用的外面函数的临时变量，形成了闭包，导致外面函数第一次执行时的执行上下文没有被销毁，`handlePrint`每次调用的时候取到`num`的值仍然是第一次的值。

> 结合闭包的知识和`useCallback`的实现可以很好的理解

要解决这个问题，可以**将内部使用的`state`放在依赖项**中，这样每次`state`变化的时候，会使用最新的函数。

```react
const Test=()=>{
  const [num, setNum] = useState(0);
  
  const handleAdd=()=>{
    setNum(prev => prev+1);
  }
  
  const handlePrint=useCallback(()=>{
    console.log(num);
  },[num]) // 加入依赖
  
  return (
  	<div>
    	<button onClick={handleAdd}>加1</button>
      <button onClick={handlePrint}>输出</button>
    </div>
  );
}
```



## 2. useMemo

`useMemo`对应的Hook用来保存**所有类型**的无副作用状态，状态值由一个创建函数返回。和`useCallback`一样，调用`useMemo`的时候也可以提供状态变化的**依赖项**。

### 2.1 mount阶段

对应的函数是`mountMemo` 

```javascript
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

`useMemo`对应`Hook`的`memoizedState`中保存执行创建函数的返回值以及对应的依赖项

### 2.2 update阶段

对应的函数是`updateMemo`

```javascript
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 对比依赖项
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

基本和`useCallback`的逻辑是一样的。



## 3. useRef

`useRef`更像是`ClassComponent`中的`this`，其**对应`Hook`中保存的状态除非手动修改，否则不会发生变化**。

### 3.1 mount阶段 

```javascript
function mountRef<T>(initialValue: T): {|current: T|} {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}
```

### 3.2 update阶段

```javascript
function updateRef<T>(initialValue: T): {|current: T|} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState; // 返回原来的值
}
```

实现上非常简单



## 4. 总结

虽然`useCallback、useMemo和useRef`都是用来存储无副作用的状态，但是其中的区别还是很明显的。

**`useCallback`和`useMemo`通常用来保存由`state`计算所得的状态，类比Vue中的计算属性。**

**`useRef`可以用来保存任何状态，但是需要手动修改。**

