# 结合源码分析useDeferredValue

这篇文章我们来分析 Concurrent 模式中提供的另一个 Hook —— `useDeferredValue`

`useDeferredValue`的主要功能是**返回一个延迟更新的值**。

从功能上来看感觉和useTransition有一些类似，两者之间的关系后面我们也会讲到

首先来分析一下源码中是如何实现的

## 1. useDeferredValue的源码实现

在源码的实现上和前面讲的useTransition有一些类似

### 1.1 mount阶段

mount阶段对应的方法是`mountDeferredValue`

```js
function mountDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = mountState(value);
  mountEffect(() => {
    // 增加了transition上下文
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}
```

这个Hook中主要做了两件事：

1. 定义一个`state`用来保存**延时的值(deferredValue)**，初始值就是传进来`value`的值。
2. 定义一个`effect`用来更新**deferredValue**。在副作用中来更新值

**这里更新`deferredValue`时和`useTransition`中一样，先增加了`transition`上下文，也就是降低了当前触发更新的优先级，从而达到延迟更新`deferredValue`的目的**

### 1.2 updat阶段

update阶段对应的方法是`updateDeferredValue`

```js
function updateDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = updateState(value);
  updateEffect(() => {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}
```

基本没有什么变化，不需要额外分析

可以看到这个Hook的源码实现是比较简单的，功能的本质就是**延时更新一个值**。

## 2. useDeferredValue VS debounce/throttle

首先我们来看一个`useDeferredValue`经典的使用场景

> 对应的Demo在[这里](https://codesandbox.io/s/usedeferredvalue-4m6ph?file=/src/App.js)

```jsx
// index.tsx
import React, { useState } from "react";

import List from "./List";

export default function UseDeferredValueProcess() {
  const [value, setValue] = useState("");
  const deferredValue = (React as CommonObject).unstable_useDeferredValue(
    value
  );

  const handleChange = (e: CommonObject) => {
    setValue(e.target.value);
  };

  return (
    <div className="App">
      <div>
        <input onChange={handleChange} />
      </div>
      <div>
        <List value={deferredValue} />
      </div>
    </div>
  );
}


// List.tsx
import React, { FC, memo } from "react";

const getLis = (key: string) => {
  const arr = [];
  for (let i = 0; i < 10000; i++) {
    arr.push(<li key={i}>{key}</li>);
  }
  return arr;
};

interface ListProps {
  value: string;
}

const List: FC<ListProps> = (props) => {
  const { value } = props;

  return <ul>{value ? getLis(value) : null}</ul>;
};

export default memo(List);
```

当用户在输入的时候，下面列表的内容也需要更新。

但是**如果下面列表更新的时候耗费的时间比较长时，用户在输入的时候就会明显的感觉到不流畅**。

这个问题的解决思路就是就上下**更新分层**，就是用户的输入及时更新，列表数据延时更新

针对这个问题有两种解法

1. **Debounce/Throttle**: 在`legacy`模式下，开发者通常使用`debounce`或者`threttle`的方式来延迟更新并减少列表的更新次数。

2. **useDeferredValue**：在`concurrent`模式下，由于存在时间切片和优先级调度的特性，可以修改列表更新的优先级，从而来延迟更新列表并减少更新的次数，而且还可以打断更新，让高优先级的更新先渲染出来

从上面的描述中，似乎`useDeferredValue`和`debounce/throttle`的效果是类似的，那么其中的区别是什么？

1. 第一个区别是**延迟的时间**
   
   - `debounce/throttle`方案中，延迟的时间是一个固定值，不管在什么性能的机器上，延迟的时间都是一样的
   - `useDeferredValue`方案中，其延迟的时间长短取决于电脑的性能，在性能好的电脑中可能感觉不到延迟

2. 第二个区别是**延迟的更新执行时是否可以被打断**
   
   - `debounce/throttle`方案中，一旦延迟的更新开始执行，就会一直执行完成，不能被打断，而且这个过程中也会阻塞用户输入。
   
   - `useDeferredValue`方案中，**当延时的更新在执行时，如果用户继续输入，因为优先级高可以打断这个延迟的更新，所以用户的输入不会阻塞**。

## 3. useTransition 和 useDeferredValue

前面我们在分析源码的时候可以看到，`useDeferredValue`中用到了和`useTransition`相似的逻辑用来延迟更新。那么两者之间的区别是什么呢？

**使用的场景不同**

- `useTransition`着重表现的是一个更新延迟的过渡阶段，并期望开发者在过渡阶段能做点什么。所以很适合配合 Suspense 来一起使用
- `useDeferredValue`着重表现的是延迟更新一个值。至于何时开始延迟，何时结束延迟开发者不需要关心

> 注意：
> 
> 和`useTransition`一样，当前官方文档中`useDeferredValue`的API对应的React版本也是比较老的版本
