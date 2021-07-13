# commit - mutation

本节中我们学习`commit`的第二个阶段——`mutation`，也就是**操作真实DOM**的阶段。对应的入口代码也很简单

```javascript
// 重置nextEffect，重新获取整个effectList链表
nextEffect = firstEffect;

do {
    if (__DEV__) {
    } else {
        try {
            commitBeforeMutationEffects();
        } catch (error) {
            invariant(nextEffect !== null, 'Should be working on an effect.');
            captureCommitPhaseError(nextEffect, error);
            nextEffect = nextEffect.nextEffect;
        }
    }
} while (nextEffect !== null);
```

可以看到在`mutation`过程中，实际上也就是遍历`effectList`然后执行`commitBeforeMutationEffects`方法

## 1. commitBeforeMutationEffects

先看一下整个方法的实现

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2357)

```javascript
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags;

    // 重置文本节点
   	// 当当前节点的子节点由纯文本节点变成正常节点的时候，需要重置当前节点的子文本节点为空字符串
    if (flags & ContentReset) {
      commitResetTextContent(nextEffect);
    }

    // 对于需要修改Ref的节点
    // 重置通过currentFiber ref属性获取到的Ref，重置为null
    if (flags & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // 处理节点对应的DOM操作
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Placement: {
        // 插入DOM
        commitPlacement(nextEffect);
        nextEffect.flags &= ~Placement;
        break;
      }
      case PlacementAndUpdate: {
        // 插入DOM
        commitPlacement(nextEffect);
        nextEffect.flags &= ~Placement;

        // 更新DOM
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Hydrating: {
        // SSR
        nextEffect.flags &= ~Hydrating;
        break;
      }
      case HydratingAndUpdate: {
        // SSR
        nextEffect.flags &= ~Hydrating;

        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        // 更新DOM
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        // 删除DOM
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

上面代码可以看出，针对`effectList`中的每个节点做了如下三个操作：

1. **根据`ContentReset flag`重置子文本节点**
2. **根据`Ref flag`重置通过`currentFiber`中`ref`属性获取到的`Ref`值**
3. **执行对应的`DOM`操作**

下面我们重点分析执行的`DOM`操作，包括`Placement`、`Update`和`Deletion`，分别对应插入、更新和删除。（和SSR相关的操作先跳过）



### 2.1 Placement - 插入

插入`DOM`节点对应的函数是`commitPlacement`

> 对应的代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1205)

```javascript
function commitPlacement(finishedWork: Fiber): void {
  if (!supportsMutation) {
    return;
  }

  // 寻找第一个对应真实DOM节点的父级节点，
  // HostComponent或者HostRoot
  const parentFiber = getHostParentFiber(finishedWork); 

  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot: // RootFiber
      // rootFiber的stateNode属性就是fiberRootNode
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case FundamentalComponent:
      if (enableFundamentalAPI) {
        parent = parentStateNode.instance;
        isContainer = false;
      }
    default:
      invariant(
        false,
        'Invalid host parent fiber. This error is likely caused by a bug ' +
          'in React. Please file an issue.',
      );
  }
  // 如果父节点是由纯文本子节点变成正常子节点
	// 需要先清空父节点的textContent属性
  if (parentFiber.flags & ContentReset) {
    resetTextContent(parent);
    parentFiber.flags &= ~ContentReset;
  }

	// 获取插入位置已经存在的真实DOM节点
	// 当前节点对应的DOM节点可以插入before之前
  const before = getHostSibling(finishedWork);

  if (isContainer) {
    // 插入React挂载节点中
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    // 插入对应的父DOM节点中
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

从上面代码中可以看出，插入`DOM`节点的操作步骤有三步：

1. 获取`父级DOM`节点
2. 获取插入的位置对应的DOM节点（**后面都称为`before DOM节点`**）
3. 调用`DOM API`执行插入操作

下面会重点讲解第2、3步

#### 2.1.1 获取插入位置对应的DOM节点

获取插入位置对应的方法是`getHostSibling`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1159)

```javascript
function getHostSibling(fiber: Fiber): ?Instance {
  let node: Fiber = fiber;
  siblings: while (true) {
    while (node.sibling === null) {
      // 如果兄弟节点不存在
      // 1. 向父级方向寻找before DOM节点
      if (node.return === null || isHostParent(node.return)) {
        // 如果当前节点不存在兄弟节点，而且父节点对应真实DOM节点
        // 表示当前父DOM节点不存在子DOM节点，before DOM节点为null
        // appendChild即可，不需要插入
        return null;
      }
      node = node.return;
    }
    
    // 如果兄弟节点存在
    // 2. 向右方向寻找before DOM节点
    node.sibling.return = node.return;
    node = node.sibling;
    
    while (
      node.tag !== HostComponent &&
      node.tag !== HostText &&
      node.tag !== DehydratedFragment
    ) {
      
      if (node.flags & Placement) {
        // 如果这个节点也是本次更新新插入的节点，
        // 则需要继续寻找before DOM节点
        continue siblings;
      }
      if (node.child === null || node.tag === HostPortal) {
        // 如果以这个节点为根节点的subtree中不存在对应的DOM节点
        // 那么也需要继续向后寻找before DOM节点
        continue siblings;
      } else {
        // 如果兄弟节点没有对应的真实DOM节点
        // 3. 需要向下寻找before DOM节点
        node.child.return = node;
        node = node.child;
      }
    }
    
    
    if (!(node.flags & Placement)) {
      // 如果兄弟节点对应真实DOM节点，而且不是本次更新插入的节点
      return node.stateNode;
    }
  }
}
```

从上面代码可以看出，寻找`before DOM节点`是一个相当繁琐的过程，造成这个过程繁琐的原因就是：**`Fiber Tree` 和 `DOM Tree`中的节点并不是一一对应的**。

实际上**`Fiber Tree`中和`DOM Tree`对应的节点是`HostComponent`节点**，所以在寻找`before DOM节点`的时候，需要先找到对应的`HostComponent`节点，这个查找的过程可能需要**跨层级，跨子树**查找。

1. 首先判断当前节点是否存在兄弟节点
   - 如果存在，则**向右**寻找`before DOM节点`
   - 如果不存在，则**向上**寻找`before DOM节点`

2. 向上寻找
   - 如果节点是`HostComponent`，则表明要插入的父`DOM`节点中不存在子节点，也就是`before DOM节点`为`null`
   - 如果节点不是`HostComponent`，回到第`1`步。
3. 向右寻找
   - 如果节点不是`HostComponent`
     - 如果该节点也是本次更新插入的节点，则需要回到第`1`步继续寻找
     - 如果**以该节点为根节点的Fiber子树**没有对应真实DOM或者挂载在其他DOM节点的时候，也需要回到第`1`步继续寻找
     - 如果上面的条件不满足，则继续**向下**寻找
   - 如果节点是`HostComponent`
     - 如果是本次更新新增的节点，则需要回到第`1`步
     - 如果不是本次新增的节点，表示已经找到`before DOM节点`

上面的查找步骤中，涉及到三个方向的查找，跨多个层级多个子树，所以这个过程是一个**很耗时**的过程。

#### 2.1.2 执行插入操作

插入操作针对`挂载节点`和`常规DOM节点`做了区分操作，但其实大体的流程是一样的。这里分析一下`insertOrAppendPlacementNode`方法

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1291)

```javascript
function insertOrAppendPlacementNode(
  node: Fiber,
  before: ?Instance,
  parent: Instance,
): void {
  const {tag} = node;
  const isHost = tag === HostComponent || tag === HostText;
  if (isHost || (enableFundamentalAPI && tag === FundamentalComponent)) {
    // 如果是HostComponent,直接插入
    const stateNode = isHost ? node.stateNode : node.stateNode.instance;
    if (before) {
      insertBefore(parent, stateNode, before);
    } else {
      appendChild(parent, stateNode);
    }
  } else if (tag === HostPortal) {
  } else {
    // 如果不是HostComponent
    // 需要递归向下寻找真实DOM节点 然后插入
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNode(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNode(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
  }
}
```

这个过程中，主要的难点是需要**在当前`Fiber`子树中寻找对应的`DOM`子树。对应的DOM子树可能有一棵或者多棵，所以需要递归查找**。



### 2.2 Update - 更新

更新操作对应的方法是`commitWork`

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1490)

```javascript
function commitWork(current: Fiber | null, finishedWork: Fiber): void {
	// ...省略
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      if (
        enableProfilerTimer &&
        enableProfilerCommitHooks &&
        finishedWork.mode & ProfileMode
      ) {
        // ...省略
      } else {
        // 执行useLayoutEffect的销毁函数
        commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      }
      return;
    }
    case ClassComponent: {
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;
      if (instance != null) {
        const newProps = finishedWork.memoizedProps;
        const oldProps = current !== null ? current.memoizedProps : newProps;
        const type = finishedWork.type;
        const updatePayload: null | UpdatePayload = (finishedWork.updateQueue: any);
        finishedWork.updateQueue = null;
        if (updatePayload !== null) {
          // 根据之前对比之后获取的updateQueue，更新需要更新的DOM属性
          commitUpdate(
            instance,
            updatePayload,
            type,
            oldProps,
            newProps,
            finishedWork,
          );
        }
      }
      return;
    }
    // ... 省略其他case
}
```

从上面代码中可以看出，更新操作主要做了两件事情：

1. 对于`FunctionComponent`，执行组件中`useLayoutEffect`的销毁函数
2. 对于`HostComponent`，执行更新对应真实DOM节点的操作

#### 2.2.1 执行`useLayoutEffect`销毁函数

对应的入口是**`commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork)`**

> 对应的源代码可以看[这里]()

```javascript
function commitHookEffectListUnmount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      // 对应useLayoutEffect来说传入的tag是 HookLayout | HookHasEffect
      if ((effect.tag & tag) === tag) {
        const destroy = effect.destroy;
        effect.destroy = undefined;
        // mount的节点destroy为undefined
        if (destroy !== undefined) {
          // 执行销毁函数
          destroy();
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

**和`useEffect`不同的是，`useLayoutEffect`只会为对应的`Fiber`节点增加一个`Update flag`，并没有`Passive flag`。**

增加`flag`是时机和`useEffect`是一样的，也是在`useLayoutEffect`函数运行的时候。

```javascript
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  // HookLayout是Effect flag
  return mountEffectImpl(UpdateEffect, HookLayout, create, deps);
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}
```

#### 2.2.2 更新DOM节点

对于`HostComponent`来说，会将`render阶段`对比之后得到的需要更新的属性，更新到对应的真实DOM节点中。入口方法是`commitUpdate`。

直接看更新DOM属性的逻辑

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-dom/src/client/ReactDOMComponent.js#L335)

```javascript
function updateDOMProperties(
  domElement: Element,
  updatePayload: Array<any>,
  wasCustomComponentTag: boolean,
  isCustomComponentTag: boolean,
): void {
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];
    if (propKey === STYLE) {
      // 处理style
      setValueForStyles(domElement, propValue);
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      // 处理DANGEROUSLY_SET_INNER_HTML
      setInnerHTML(domElement, propValue);
    } else if (propKey === CHILDREN) {
      // 处理 children
      setTextContent(domElement, propValue);
    } else {
      // 处理其他属性
      setValueForProperty(domElement, propKey, propValue, isCustomComponentTag);
    }
  }
}
```



### 2.3 Deletion - 删除

删除操作的入口函数是`commitDeletion`，主要的逻辑在`unmountHostComponents`函数中

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1322)

```javascript
function unmountHostComponents(
  finishedRoot: FiberRoot,
  current: Fiber,
  renderPriorityLevel: ReactPriorityLevel,
): void {
  let node: Fiber = current;
  let currentParentIsValid = false;
  let currentParent;
  let currentParentIsContainer;

  while (true) {
    if (!currentParentIsValid) {
      let parent = node.return;
      // 查找被删除节点的父DOM节点
      findParent: while (true) {
        const parentStateNode = parent.stateNode;
        switch (parent.tag) {
          case HostComponent:
            currentParent = parentStateNode;
            currentParentIsContainer = false;
            break findParent;
          case HostRoot:
            currentParent = parentStateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
          case HostPortal:
            currentParent = parentStateNode.containerInfo;
            currentParentIsContainer = true;
            break findParent;
          case FundamentalComponent:
            if (enableFundamentalAPI) {
              currentParent = parentStateNode.instance;
              currentParentIsContainer = false;
            }
        }
        parent = parent.return;
      }
      currentParentIsValid = true;
    }

    if (node.tag === HostComponent || node.tag === HostText) {
      // 执行被删除节点删除前的操作
      commitNestedUnmounts(finishedRoot, node, renderPriorityLevel);
      // 执行DOM API删除DOM节点
      if (currentParentIsContainer) {
        removeChildFromContainer(
          ((currentParent: any): Container),
          (node.stateNode: Instance | TextInstance),
        );
      } else {
        removeChild(
          ((currentParent: any): Instance),
          (node.stateNode: Instance | TextInstance),
        );
      }
    }
    // ...省略部分分支
    else {
      // 执行节点销毁之前的操作
      commitUnmount(finishedRoot, node, renderPriorityLevel);
      // 处理子节点的销毁，“递”过程
      if (node.child !== null) {
        node.child.return = node;
        node = node.child;
        continue;
      }
    }
    if (node === current) {
      return;
    }
    // 处理兄弟节点的销毁，“归”过程
    while (node.sibling === null) {
      if (node.return === null || node.return === current) {
        return;
      }
      node = node.return;
      if (node.tag === HostPortal) {
        currentParentIsValid = false;
      }
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```

上面的代码中可以看出，**删除操作做的工作**有：

1. 获取被删除节点的**父`DOM`节点**
2. 对于**`HostComponent`**，需要执行`DOM API`删除对应的真实`DOM`节点（对**最上层**的DOM节点操作即可）
3. 对于**被销毁的节点**，需要执行节点销毁前的操作（**每个节点**都需要执行）

被删除的子树中，每个节点都需要执行一些销毁前的操作，对应的函数是`commitUnmount`。对于`HostComponent subtree`，内部同样需要对每个后代节点执行销毁前操作，`commitNestedUnmounts`方法内部也有类似递归过程。



#### 2.3.1 节点销毁前的操作

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L924)

```javascript
function commitUnmount(
  finishedRoot: FiberRoot,
  current: Fiber,
  renderPriorityLevel: ReactPriorityLevel,
): void {
  switch (current.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      const updateQueue: FunctionComponentUpdateQueue | null = (current.updateQueue: any);
      if (updateQueue !== null) {
        const lastEffect = updateQueue.lastEffect;
        if (lastEffect !== null) {
          const firstEffect = lastEffect.next;

          let effect = firstEffect;
          do {
            const {destroy, tag} = effect;
            if (destroy !== undefined) {
              if ((tag & HookPassive) !== NoHookEffect) {
                // 记录需要执行销毁函数的useEffect和fiber
                // 调度执行useEffect
                enqueuePendingPassiveHookEffectUnmount(current, effect);
              } else {
                // 处理useLayoutEffect
                if (
                  enableProfilerTimer &&
                  enableProfilerCommitHooks &&
                  current.mode & ProfileMode
                ) {
                  // ...省略
                } else {
                  // 执行useLayoutEffect的销毁函数
                  safelyCallDestroy(current, destroy);
                }
              }
            }
            effect = effect.next;
          } while (effect !== firstEffect);
        }
      }
      return;
    }
    case ClassComponent: {
      // 清除Ref
      safelyDetachRef(current);
      const instance = current.stateNode;
      // 执行willUnmount生命周期
      if (typeof instance.componentWillUnmount === 'function') {
        safelyCallComponentWillUnmount(current, instance);
      }
      return;
    }
    case HostComponent: {
      // 清除Ref
      safelyDetachRef(current);
      return;
    }
    // ...省略一些其他case
  }
}
```

- `ClassComponent`删除前的操作：

  1. 清除`Ref`

  2. 执行`componentWillUnmount`生命周期函数

- `FunctionComponent`删除前的操作：

  1. **收集需要执行销毁函数的`useEffect`和`Fiber`节点**，放在全局变量中保存，并且调度`useEffect`

  2. **执行`useLayoutEffect`的销毁函数**

- `HostComponent`删除前的操作：
  1. 清除`Ref`



> **这里会有一个误区**：`ClassComponent`组件中先清空了`Ref`然后再执行`componentWillUnmount`，那为什么还能再`willUnmount`中获取去到`Ref`呢？
>
> 这里之所以想不通是因为把这两个`Ref`弄混淆了。清空是当前`Fiber`节点`Ref`属性获取的值，但是`willUnmount`中获取的是后代节点的`Ref`，而且子树销毁的时候，`willUnmount`执行的顺序是**从父到子**的。



> **`FunctionComponent`之所以不需要清空`Ref`，是因为`FunctionComponent`通过`Ref`获取到的值都是`null`，因为对应的`Fiber`节点中`stateNode`属性值为`null`。**



## 2. 总结

在`mutation`阶段中，会遍历effectList，然后针对每个节点执行一些操作：

1. 执行对应的`DOM`操作，插入、更新和删除。
2. 执行`useLayoutEffect`的销毁函数
3. 执行`componentWillUnmount`生命周期
4. 对于**销毁**的节点，收集**需要销毁的`useEffect`和`Fiber`节点**，调度执行`useEffect`。



> **问题：**在前面beforeMutation阶段的时候已经调度了一次useEffect，这里会重复调度吗？
>
> 不会的，其中有一个全局的开关变量`rootDoesHavePassiveEffects`在控制

