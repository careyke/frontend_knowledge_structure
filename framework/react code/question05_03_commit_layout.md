# commit - layout

和前面两个阶段类似，`layout`阶段也是遍历`effectList`，然后执行操作。

```javascript
// 切换current
root.current = finishedWork; 

nextEffect = firstEffect;
do {
    if (__DEV__) {
    } else {
        try {
            commitLayoutEffects(root, lanes);
        } catch (error) {
            invariant(nextEffect !== null, 'Should be working on an effect.');
            captureCommitPhaseError(nextEffect, error);
            nextEffect = nextEffect.nextEffect;
        }
    }
} while (nextEffect !== null);

nextEffect = null;
```

## 1. commitLayoutEffects

直接`commitLayoutEffects`方法的代码。

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2437)

```javascript
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags;

    if (flags & (Update | Callback)) {
      const current = nextEffect.alternate;
      // 执行对应的生命周期钩子函数
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }

    if (enableScopeAPI) {
    } else {
      if (flags & Ref) {
        // 重新赋值Ref
        commitAttachRef(nextEffect);
      }
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

从上面的代码可以看出，`layout`阶段主要做了两个事情

1. 执行节点对应组件的生命周期函数
2. 对于需要更新`Ref`的节点，更新`Ref`

### 1.1 commitLayoutEffectOnFiber

> 当前方法的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L480)

这个方法中对于不同类型的节点会做不同的处理。

#### 1.1.1 ClassComponent

```javascript
case ClassComponent: {
    const instance = finishedWork.stateNode;
    if (finishedWork.flags & Update) {
      	// 判断是mount还是update
        if (current === null) {
            if (
                enableProfilerTimer &&
                enableProfilerCommitHooks &&
                finishedWork.mode & ProfileMode
            ) {
              // ...省略
            } else {
              	// 执行componentDidMount生命周期
                instance.componentDidMount();
            }
        } else {
            const prevProps =
                finishedWork.elementType === finishedWork.type ?
                current.memoizedProps :
                resolveDefaultProps(finishedWork.type, current.memoizedProps);
            const prevState = current.memoizedState;
            if (
                enableProfilerTimer &&
                enableProfilerCommitHooks &&
                finishedWork.mode & ProfileMode
            ) {
                // ...省略
            } else {
              	// 执行componentDidUpdate生命周期
                instance.componentDidUpdate(
                    prevProps,
                    prevState,
                  	// 这个属性表示的是getSnapshotBeforeUpdate函数的返回值
                  	// 在before mutation阶段执行getSnapshotBeforeUpdate时塞进去的
                    instance.__reactInternalSnapshotBeforeUpdate,
                );
            }
        }
    }

    const updateQueue: UpdateQueue <
        *
        , >
        | null = (finishedWork.updateQueue: any);
    if (updateQueue !== null) {
      	// 执行调用setState时传入的第二个参数
      	// 会存储在effect.callback中
        commitUpdateQueue(finishedWork, updateQueue, instance);
    }
    return;
}
```

对于`ClassComponent`做的操作：

1. 执行`componentDidMount`或`componentDidUpdate`生命周期函数
2. 执行`updateQueue`中保存的**`callback`**函数

> 这里的`callback`指的是setState的第二个参数，会保存在`effect`对象中，后面分析`update`的时候会详细解析

#### 1.1.2 FunctionComponent

```javascript
case FunctionComponent: {
    if (
        enableProfilerTimer &&
        enableProfilerCommitHooks &&
        finishedWork.mode & ProfileMode
    ) {
        // ...省略
    } else {
      	// 调用useLayoutEffect创建函数
        commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
    }
		
  	// 记录useEffect的销毁函数和创建函数
  	// 调度执行所有的useEffect
    schedulePassiveEffects(finishedWork);
    return;
}
```

对于`FunctionComponent`做的操作：

1. 调用`useLayoutEffect`的创建函数
2. 在全局变量中记录`useEffect`的销毁函数和创建函数，并且尝试调度`useEffect`

这里看一下useEffect存储的数据结构：

```javascript
export function enqueuePendingPassiveHookEffectMount(
  fiber: Fiber,
  effect: HookEffect,
): void {
  // 存储需要执行创建函数的useEffect
  pendingPassiveHookEffectsMount.push(effect, fiber);
  // ...省略
}
```

可以看出全局变量`pendingPassiveHookEffectsMount`数组中，**偶数索引存储的是`effect`，奇数索引存储的是对应的`Fiber`节点。**`pendingPassiveHookEffectsUnmount`数组中也是一样的结构。

#### 1.1.3 HostRoot

```javascript
case HostRoot: {
    const updateQueue: UpdateQueue <
        *
        , >
        | null = (finishedWork.updateQueue: any);
    if (updateQueue !== null) {
        let instance = null;
        if (finishedWork.child !== null) {
            switch (finishedWork.child.tag) {
                case HostComponent:
                    instance = getPublicInstance(finishedWork.child.stateNode);
                    break;
                case ClassComponent:
                    instance = finishedWork.child.stateNode;
                    break;
            }
        }
      	// 执行render函数的第三个参数
        commitUpdateQueue(finishedWork, updateQueue, instance);
    }
    return;
}
```

对于`HostRoot`做的操作：

1. 执行`ReactDOM.render`函数的第三个参数，这个参数会保存在`updateQueue`中

### 1.2 commitAttachRef

这个方法用来设置`Ref`的值，对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L850)

```javascript
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }
    if (typeof ref === 'function') {
      if (
        enableProfilerTimer &&
        enableProfilerCommitHooks &&
        finishedWork.mode & ProfileMode
      ) {
        try {
          startLayoutEffectTimer();
          ref(instanceToUse);
        } finally {
          recordLayoutEffectDuration(finishedWork);
        }
      } else {
        ref(instanceToUse);
      }
    } else {
      ref.current = instanceToUse;
    }
  }
}
```

从上面代码可以看出，**节点的`Ref`值取的就是对应`Fiber`节点的`stateNode`的值，所以`FunctionComponent`类型的节点给`Ref`赋值为`null`**

### 1.3 节点销毁和更新情况下`useEffect`的执行顺序

#### 1.3.1 执行的操作

1. 节点销毁情况下，执行`useEffect`的销毁函数
2. 节点更新情况下，先执行`useEffet`的销毁函数，然后执行创建函数

#### 1.3.2 执行的顺序

1. 节点销毁的情况下，**先执行父节点中`useEffect`的销毁函数，然后再执行子节点**。被删除子树遍历节点的顺序和创建`workInprogress Fiber Tree`时节点执行`beginWork`的顺序一致。
2. 节点更新的情况下，**先执行子节点中useEffect的销毁函数，然后再执行父节点**。和节点执行`completeWork`函数的顺序一致

产生这种**差异的原因**

1. 对于删除的节点来说，只在被删除子树的根节点打上了`Deletion flag`，也就是说**在`effectList`中只记录了根节点**，所以销毁的时候需要通过根节点来遍历后代所有的节点执行销毁的操作。

2. 对于更新的节点来说，每个需要更新的节点都会打上`Update flag`，而且**都会存在`effectList`中**，所以在遍历`effectList`的时候就可以执行更新的操作。`effectList`中节点的顺序是从子节点到父节点

#### 1.3.2 总结

删除节点的`useEffect`逻辑是`mutation`节点完成，更新节点的useEffect逻辑是在layout阶段完成。

所以在一次更新中，**会优先执行销毁节点的`useEffect`逻辑，执行的方向是先父后子。然后执行更新节点的`useEffect`逻辑，执行的顺序是先子后父。**

同理，对应`useLayoutEffect`和`componentWillUnmount、componentDidMount`是执行顺序也是一样的。



## 2. 切换`current tree`

在前面讲`Fiber`架构的时候讲过，`Fiber`架构使用*双缓存**的机制来更新`Fiber Tree`。`workInProgress FIber Tree`会在`commit`阶段变成`current Fiber Tree`。

看上面`layout`阶段的入口代码，**切换`Fiber Tree`的具体时机就是在`mutation`阶段之后、`layout`阶段之前。**

至于为什么放在这里执行？

代码中给了一段注释，大致的意思就是对于`ClassComponent`来说，**在执行`componentWillUnmount`的时候需要获取到更新之前的DOM节点。而在`componentDidMount`和`componentDidUpdate`中需要获取到更新之后的DOM节点。所以才在`mutation`和`layout`阶段之间来执行`Fiber Tree`切换的操作。**



> **？？？**
>
> 这里笔者有一点**疑惑**，在源代码的逻辑中，切换`Fiber Tree`的操作放在这里与否，对于`componentDidMount`中获取子元素DOM节点并没有什么影响。
>
> **`componentDidMount`中获取到的始终是更新之后的DOM节点。**



## 3. layout之后

在`layout`之后还有一些操作，我们暂时称这个阶段为`layout之后`

```javascript
// 检测commit过程中是否有调度useEffect
const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

if (rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = false;
  	// 判断是否执行useEffect创建和销毁函数的开关属性
    rootWithPendingPassiveEffects = root;
    pendingPassiveEffectsLanes = lanes;
    pendingPassiveEffectsRenderPriority = renderPriorityLevel;
} else {
    nextEffect = firstEffect;
    while (nextEffect !== null) {
        const nextNextEffect = nextEffect.nextEffect;
        nextEffect.nextEffect = null;
        if (nextEffect.flags & Deletion) {
          	// 清空被删除节点的持有属性
          	// 方便该节点执行GC
            detachFiberAfterEffects(nextEffect);
        }
        nextEffect = nextNextEffect;
    }
}

// ...省略一些逻辑

// 确保任何附加的任务被执行
ensureRootIsScheduled(root, now());


// 执行同步更新
// 比如在useLayoutEffect或者DiDMount中调用setState，就触发一次同步更新任务
// 不需要在下个tick执行
flushSyncCallbackQueue();

return null;
```

`layout之后`主要做的工作：

1. **判断`layout`阶段中是否调度了`useEffect`**，如果调度了，则为`useEffect`的执行准备一些条件。
2. **执行本次更新中产生的同步更新任务，主要是生命周期中执行`setState`触发的更新**。（后面讲更新阶段的时候会详细介绍）

下面我们来总结一下`useEffect`的执行过程。

### 3.1 总结useEffect的执行过程

在整个`commit`过程的分析中，有很多地方的逻辑都和`useEffect`有关，但是都是片段式的，没有连贯在一起。

useEffect的执行过程总共分成三个阶段：**调度、准备和执行**

#### 3.1.1 调度

```javascript
if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
    });
}
```

上面这段代码在`commit`阶段出现了很多次，但是**使用了一个全局变量`rootDoesHavePassiveEffects`来保证不会调度多次。**

使用了调度器来确保`useEffect`是**异步执行**的。

#### 3.1.2 准备

准备阶段发生在**layout之后**，由于`useEffect`是异步执行的，所以能够确保`准备阶段`发生在`执行阶段`之前

```javascript
const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

if (rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = false;
  	// 判断是否执行useEffect创建和销毁函数的开关属性
    rootWithPendingPassiveEffects = root;
    pendingPassiveEffectsLanes = lanes;
    pendingPassiveEffectsRenderPriority = renderPriorityLevel;
} else {
    // ... 省略
}
```

#### 3.1.3 执行

执行阶段的代码对应的是`flushPassiveEffects`，前面见过这个方法主要是用来执行本次更新中暂存的所有`useEffect`

```javascript
function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    return false;
  }
	// ...省略处理逻辑，前面分析过
}
```



可以看到这三个过程是环环相扣的，**通过全局变量来控制下一个阶段是否开启**。



## 4. 总结

至此，commit阶段的工作算是分析玩了，React的渲染流程也基本都已经讲完了。

**`render`阶段有两个主要的工作：**

1. 创建`workInProgress Fiber Tree`
2. 生成`effectList`



**`commit`阶段也有两个主要的工作：**

1. 执行`effectList`中描述的DOM操作
2. 执行`effectList`中节点更新之后对应的副作用



**`render`阶段的工作可以被打断，`commit`阶段的工作是同步执行完**

