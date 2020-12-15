# 结合源码分析Diff算法的实现

上一节中在讲解`beginWork`的时候讲到，在`update`阶段，根据`currentFiber`和最新的`ReactElement`创建`workInProgressFiber`的时候会使用到`Diff`算法，判断是否能够**复用**`currentFiber`。

> `Diff`算法这里的复用指的是**复用currentFiber中的大部分属性（pendingProps除外），目的是为了复用DOM节点，省去销毁和创建DOM的过程消耗**

**如果Diff算法判断不能复用，不仅要创建新的workInProgressFiber，还要销毁和创建对应的真实DOM节点**

下面会详细介绍Diff算法的实现。

## 1. Diff算法的瓶颈和React做出的应对

React文档中有提到，即使在最前沿的算法中，将前后两棵树完全对比的算法时间复杂度为`O(n^3)`，`n`表示树中的节点数。如果使用该算法，那么展示1000个节点的计算量将会达到十亿级别，所以这种做法是及其昂贵的。

为了降低算法的复杂度，React提出了一套**启发式的算法**，时间复杂度为`O(n)`。在对比的过程中，**预设了一些条件**来降低算法的复杂度：

1. **只对同层节点进行`Diff`**。如果一个节点在更新前后改变了层级，React不会去尝试复用这个节点。
2. **不同类型的节点会产生不同的子树**。如果元素由`div`变成了`p`，React会直接销毁`div整个子树`，然后新建`p整个子树`。
3. **开发者可以通过设置`key prop`来暗示哪些元素在不同的渲染中能保持稳定**。



## 2. key的作用

回想一下前面讲到的`update`阶段中创建`workInProgressFiber`节点的步骤：

1. 在`current Fiber Tree`的同层节点中，找到相对应的`currentFiber`
2. 判断`currentFiber`能否复用
   - 如果可以复用，就根据`currentFiber`来生成`workInProgressFiber`，复用大部分属性
   - 如果不能复用，就标记删除老的`Fiber`节点，根据`ReactElement`来新创建`workInProgressFiber`

在这个过程中，最重要的是第一步。

在没有引入`key prop`的时候，寻找对应的currentFiber只能通过当前节点在**数组的索引**来寻找，这种方式对于单一节点或者是更新前后节点位置不变的情况是可行。但是对于**插入节点或者节点位置移动**的情况，这种做法是非常糟糕的。举个例子

```react
// 之前
<div>
	<p>a</p>
  <span>b</span>
  <div>c</div>
</div>

// 之后
<div>
  <div>c</div>
  <p>a</p>
  <span>b</span>
</div>
```

上面例子中，由于没有引入key prop，只能按照索引的方式来找对应的`currentFiber`，结果判断不能复用，会销毁重建对应的Fiber节点和对应的真实DOM。

然而对于开发者来说，他是知道当前只是移动了一个节点的位置，是不需要销毁和重建的。React只是没有正确找到对应的`currentFiber`，所以**开发者需要一种机制来告诉React如果寻找当前节点对应的`currentFiber`，这个机制就是`key prop`。**

同样的例子加上key

```react
// 之前
<div>
	<p key='a'>a</p>
  <span key='b'>b</span>
  <div key='c'>c</div>
</div>

// 之后
<div>
  <div key='c'>c</div>
  <p key='a'>a</p>
  <span key='b'>b</span>
</div>
```

节点可以根据key值精准的找到对应的`currentFiber`，不会产生多余的销毁和重建的过程。

然而**key作为寻找`currentFiber`的唯一标识**，有两点需要注意：

1. 唯一性：同层节点中，key值需要保持唯一。
2. 稳定性：更新前后，同一节点的key值不能变化。所以不推荐使用数组索引或随机数来作为key值

**key值的作用是在`Diff`的过程中，告诉React如何精准找到与当前节点对应的currentFiber**，避免找到错误的`currentFiber`而导致多余的DOM操作。



## 3. Diff算法的具体实现

`Diff`过程发生在`update`阶段，所以可以从入口函数`reconcileChildFibers`看起。

> 对应源代码可以查看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactChildFiber.new.js#L1204)

```javascript
function reconcileChildFibers(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChild: any,
    lanes: Lanes,
): Fiber | null {
    const isObject = typeof newChild === 'object' && newChild !== null;

    if (isObject) {
      	// 单节点处理
        switch (newChild.$$typeof) {
            case REACT_ELEMENT_TYPE:
                // ...调用reconcileSingleElement处理
            // ...省略其他case
        }
    }

    // 文本节点处理
    if (typeof newChild === 'string' || typeof newChild === 'number') {
        // ...调用reconcileSingleTextNode处理
    }

    // 多节点处理
    if (isArray(newChild)) {
        // ...调用reconcileChildrenArray处理
    }

    // ...省略其他情况处理

    // 若都没有匹配到，则删除子节点
    return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

下面会分成两个方面解析`Diff`算法的实现

### 2.1 单节点Diff

单个节点的Diff实现起来比较简单，不存在同层节点移动的问题。直接看`reconcileSingleElement`方法的代码

> 对应源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactChildFiber.new.js#L1092)

```javascript
function reconcileSingleElement(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    element: ReactElement,
    lanes: Lanes,
): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    while (child !== null) {
      	// 判断key值是否相等，单个节点key值一般都是null
        if (child.key === key) {
            switch (child.tag) {
                // ...省略
                default:
                    {
                      	// 判断类型是否相同
                        if (child.elementType === element.type) {
                            deleteRemainingChildren(returnFiber, child.sibling);
                          	// 根据currentFiber来创建workInProgressFiber,复用上一次的fiber中的一些属性，包括stateNode
                            const existing = useFiber(child, element.props); 
                            existing.ref = coerceRef(returnFiber, child, element);
                            existing.return = returnFiber;
                            return existing;
                        }
                        break;
                    }
            }
            // 删除child及其所有兄弟节点
            deleteRemainingChildren(returnFiber, child);
            break;
        } else {
          	// 删除child节点，可能有其他节点key与当前节点对应
            deleteChild(returnFiber, child);
        }
        child = child.sibling;
    }
    if (element.type === REACT_FRAGMENT_TYPE) {
      // ...省略
    } else {
      // 重新创建Fiber节点
      const created = createFiberFromElement(element, returnFiber.mode, lanes);
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      created.return = returnFiber;
      return created;
    }
}
```

从上面的代码可以看出，**判断能否复用节点的条件有两个**：

1. **`key`值相同**。用来从同层的多个`currentFiber`中找出当前节点对应的`currentFiber`，如果没有找到，则表示没有对应的`currentFiber`，需要新建节点
2. **`type`**相同。判断类型是否相同

两个条件**同时满足**的时候，才能复用对应的currentFiber节点的属性。

**当节点更新前多个子节点，更新之后只有一个子节点时**，也会进入单节点`Diff`。这种情况下有个细节需要关注一下：

1. **当`key`相同，`type`不同的时候**，会执行`deleteRemainingChildren`方法将`child`及其所有兄弟节点全部标记删除。这是因为**React认为已经找到了对应的`currentFiber`节点，但是类型不同无法复用，其他节点也不需要考虑，直接删除同层所有节点**。
2. **仅当key不同的时候**，会执行`deleteChild`方式标记删除当前节点。并不会直接将其兄弟节点也删除，因为**React判断当前并没有找到对应的`currentFiber`，所以只删除当前不匹配的节点，然后遍历其兄弟节点，寻找对应的`currentFiber`**。

单节点Diff的流程：

<img src="./images/singleDiff.jpg" alt="singleDiff" style="zoom:50%;" />

### 2.2 多节点Diff





