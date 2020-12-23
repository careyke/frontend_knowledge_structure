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
2. **根据`Ref flag`重置通过`currentFiber`中`ref`属性获取到的`Ref`**
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

