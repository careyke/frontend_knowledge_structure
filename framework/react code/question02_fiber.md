# Fiber架构的实现原理

对于开发者来说，**React最直观的作用就是将 `ReactElement Tree` 转化成 `DOM Tree`**，然而如果每次更新都是全量的重新创建新的DOM节点来更新，显然性能会非常差，所以需要使用**增量更新**。要实现增量的对比，就需要一个**载体**来记录和描述当前节点的信息。当产生更新的时候，新生成的载体和旧的载体进行对比，收集增量的变化。

在Stack架构中，这个载体叫做**虚拟DOM节点**。而在Fiber架构中，这个载体叫做 **Fiber 节点**，每个`ReactElement`都会生成一个对应的`Fiber`节点。

> 本系列文章中的源码都是出自 **`React17.0.1`** 版本



## 1. Fiber节点的数据结构

可以这里看到[Fiber节点的数据结构](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiber.new.js#L117)

```javascript
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 基本属性
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // 作用在Fiber Tree中的属性，用例描述自己的位置
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  this.flags = NoFlags;
  this.subtreeTag = NoSubtreeEffect;
  this.deletions = null;
  this.nextEffect = null;

  this.firstEffect = null;
  this.lastEffect = null;

  // 优先级调度相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 双缓存属性，指向另一次更新时改节点对应的Fiber节点
  this.alternate = null;
}
```

可以看出，Fiber节点作为Fiber架构的最小工作单位，其中包含的属性是非常多的。下面根据作用不同分成几类来进行解释

### 1.1 Fiber节点的基本属性

Fiber节点作为连接 `ReactElement` 和真实DOM节点的载体，其中包含了双方的信息，确保能够一一对应。

```javascript
// 大部分情况下表示当前节点对应的React Element的类型，FunctionComponent、ClassComponent或者HostComponent...
// 也有特殊情况，比如RootFiber节点没有对应的ReactElement
this.tag = tag;

// React Element中的key属性
this.key = key;

// 大部分情况下和type是一样的，某些情况不同，比如使用React.Mome()包裹的组件
this.elementType = null;

// 表示对应ReactElemet的构造方法。对于FunctionComponent，指的是函数本身；对于ClassComponent，指的是对应的Class
// 对于HostComponent 指的是对应的真实DOM的tagName
this.type = null;

// 表示当前React Element对应的真实节点。对于FunctionComponent，值为null；对于ClassComponent，指的是对应ReactElement的实例
// 对于HostComponent，指的是对应的真实的DOM节点
this.stateNode = null;
```

**Fiber节点的tag属性是根据type属性来生成的。不同tag的Fiber节点对应的工作内容是不同的**。React中定义了很多种类的tag，可以查看[这里](https://github.com/facebook/react/blob/1fb18e22ae/packages/react-reconciler/src/ReactWorkTags.js)

这里介绍4种常用的类型：

1. `ClassComponent` —— 表示当前Fiber节点对应的是使用Class关键字创建的组件生成的ReactElement。
2. `FunctionComponent` —— 表示当前Fiber节点对应的是使用Function关键字创建的组件生成的ReactElement。
3. `HostComponent` —— **表示当前Fiber节点对应的是生成真实DOM节点的ReactElement，DOM节点的类型是对应type属性**。
4. `HostRoot` —— **表示当前Fiber节点是RootFiber节点，也就是当前Fiber树的根节点，没有对应的ReactElement节点**。

### 1.2 Fiber节点的位置属性

每个ReactElement所对应的Fiber节点最终也会组成一颗Fiber树。Fiber通过以下属性来确定自己在树中的位置。

```javascript
// 当前Fiber节点的父节点
this.return = null;

// 当前Fiber节点的第一个子节点
this.child = null;

// 当前Fiber节点的右边兄弟节点
this.sibling = null;

// 当前Fiber节点在兄弟节点中的位置索引
this.index = 0;
```

举个例子，如下的组件结构

```react
<App>
	<p></p>
  <div>
  	<span>littleknife</span>
  </div>
</App>
```

对应的Fiber树结构：

<img src="./images/fiberTree.png" alt="fiberTree" style="zoom:50%;" />

> 这里子节点指向父节点的指针名字为什么叫做`return`，而没有叫做`parent`或者`father`？是因为Fiber节点作为一个工作单元，`return`表示当前节点执行完 `completeWork` （后面章节会介绍）之后返回的下一个节点。子节点及其兄弟节点完成工作之后都会返回其父节点，所以用`return`表示父节点

### 1.3 Fiber节点的工作单元属性

作为Fiber架构中的最小工作单元，每个ReactElement的更新都会导致对应Fiber节点的更新，

