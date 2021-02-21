# 结合源码分析Context的实现

`Context`是`React`提供的一种**组件共享数据**的解决方案。避免数据通过`props`一层层往下传递的繁琐操作。

**当`Context`中保存的值发生变化的时候，所有依赖这个`Context`的组件都会更新。**

那么Context内部是如何实现的呢？下面我们就结合源码来详细分析Context的实现

> 这里我们只会分析新型的`Context`，老版的`Context`会直接跳过



## 1. Context相关的数据结构

React提供了`createContext` API来创建`Context`对象

> 对应的方法可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react/src/ReactContext.js#L14)

```javascript
export function createContext<T>(
  defaultValue: T,
  calculateChangedBits: ?(a: T, b: T) => number,
): ReactContext<T> {
  if (calculateChangedBits === undefined) {
    calculateChangedBits = null;
  } else {
  }

  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    _calculateChangedBits: calculateChangedBits,
    _currentValue: defaultValue,
    
    // 这里是为了并发渲染预留的字段，暂时不需要了解
    _currentValue2: defaultValue,
    _threadCount: 0,
    
    // These are circular
    Provider: (null: any),
    Consumer: (null: any),
  };

  context.Provider = {
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
  };

  if (__DEV__) {
    // ...省略
  } else {
    context.Consumer = context;
  }

  return context;
}
```

上面代码中有几个细节需要注意一下：

1. `Provider`对象中可以通过`_context`属性获取`context`对象，`Consumer`本身就是`context`对象
2. 创建`context`对象的时候，预留了一个参数`calculateChangedBits`，这个方法可以**让开发者自己来判断context的值是否发生变化**。（暂时还没有开放出来）



## 2. Provider

在使用`context`的时候，通常是使用`context`提供的两个组件`Provider`和`Consumer`来实现数据的共享。

这里的设计模式和发布-订阅的模型很类似。

- **`Provider`类似于发布者**。负责对比数据有没有发生变化，如果发生变化则需要通知对应的订阅者。
- **`Consumer`类似于订阅者**。表示当前这个组件订阅对应context的值，如果context的值发生变化，组件需要更新。



这里我们先来看看在`beginWork`阶段中，对于`Provider`类型的节点是如何处理的。

> 对应的代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L2756)

```javascript
function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  const providerType: ReactProviderType<any> = workInProgress.type;
  // 获取context对象
  const context: ReactContext<any> = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;

  const newValue = newProps.value;

  // 记录context的嵌套层级，感觉是开发时期用的，暂时不讨论
  pushProvider(workInProgress, newValue);

  if (oldProps !== null) {
    // update阶段
    const oldValue = oldProps.value;
    // 判断context的值是否发生改变
    const changedBits = calculateChangedBits(context, newValue, oldValue);
    if (changedBits === 0) {
      // 没有改变
      if (
        oldProps.children === newProps.children &&
        !hasLegacyContextChanged() // 旧版的Context，不考虑
      ) {
        // 如果children是相同的，可以直接进入优化处理
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes,
        );
      }
    } else {
      // 如果context发生变化，需要向下遍历寻找对应的Consumer，打上更新标记
      propagateContextChange(workInProgress, context, changedBits, renderLanes);
    }
  }

  // 根据子ReactElement生成对应的Fiber节点
  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

从上面代码中可以看出，对于`Provider`类型的节点的处理分成两个阶段：

1. **mount阶段**：直接根据`子ReactElement`来生成对应的`Fiber`节点

2. **update阶段**：判断context的值是否发生变化

   - **如果值没有发生变化**，而且`children`也没有发生变化，会进入优化处理，复用`current Fiber Tree`中的节点

     > 在`render`阶段的文章中介绍过：
     >
     > 这里对比`children`使用的是**全等`===`**，如果执行了`render`函数，生成的`children`必要是一个新对象
     >
     > 可以看看`babel`转换之后的`React`代码，线上[试试](https://www.babeljs.cn/repl#?browsers=&build=&builtIns=false&spec=false&loose=false&code_lz=JYWwDg9gTgLgBAJQKYEMDG8BmUIjgcilQ3wG4AoctAGxQGc64ARJECOJADxiQDsATRsnQwAdAGFckXnxgBvcgEg0EXnRhQArhmgAKMDjB0AlArjm4iuprBIo-wyYoW4AX3LnFRAXd2mPLnBEMJpQvHAAPPzAAG4AfBF0YCi8CQD0SSnp0fEB5u6uQA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=es2015%2Creact&prettier=false&targets=&version=7.6.2&externalPlugins=)

   - **如果值发生了变化**，则需要**向下遍历**寻找当前`context`的订阅者，打上更新标记，本次一起更新。

下面我们重点来分析`update`阶段的实现代码。

