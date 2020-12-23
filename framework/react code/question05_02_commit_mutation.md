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

  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork); // 确保parentFilber的stateNode是一个dom节点

  // Note: these two variables *must* always be updated together.
  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot: // RootFiber
      parent = parentStateNode.containerInfo; // 获取挂载节点
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
    // eslint-disable-next-line-no-fallthrough
    default:
      invariant(
        false,
        'Invalid host parent fiber. This error is likely caused by a bug ' +
          'in React. Please file an issue.',
      );
  }
  // 更新节点的文本
  if (parentFiber.flags & ContentReset) {
    // Reset the text content of the parent before doing any insertions
    resetTextContent(parent);
    // Clear ContentReset from the effect tag
    parentFiber.flags &= ~ContentReset;
  }

  const before = getHostSibling(finishedWork);
  // We only have the top Fiber that was inserted but we need to recurse down its
  // children to find all the terminal nodes.
  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

