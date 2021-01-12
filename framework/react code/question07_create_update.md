# 创建update

在前面的文章中，我们分别分析了`Scheduler`、`Reconciler`和`Renderer`各自的工作内容。接下来的几篇文章中，我们将详细的介绍整个`React`应用`update`时的详细过程。

## 1. 状态变化

`React`应用中，产生更新的前提条件就是发生状态变化，`React`提供了几种发生状态变化的`API`：

1. `ReactDOM.render`
2. `this.setState`
3. `this.forceUpdate`
4. `useState`
5. `useReducer`

> `SSR`暂时不考虑

上面不同的方法使用的场景是不一样的，但是使用的是同一套更新机制，所以可以理解为这几种方法的**“输出”**是一样的。

每个方法内部都会创建一个**保存本次更新内容**的对象，叫做**`Update`**，然后在`render`阶段会根据`Update`来计算最新的`state`。

> 前面的文章中我们就提到过，`ReactDOM.render`初始化`React`应用和更新`React`应用流程是一样的，都会创建`Update`对象，然后再调度更新。

接下来笔者会以`ClassComponent`为例来分析整个`update`的过程，`FunctionComponent`的更新流程会在后面介绍`Hooks`的时候一起分析



## 2. 创建Update对象

下面我们以`this.setState`的执行过程为例来分析创建`Update`对象的过程。

我们在定义`ClassComponent`的时候，通常都要继承`React.Component`或者`React.PureComponent`，`setState`方法就是在父类中定义的方法。

```javascript
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  this.updater = updater || ReactNoopUpdateQueue;
}
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

> 对应的源代码看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react/src/ReactBaseClasses.js#L20)

我们可以看到，`Component`实例中保存了一个**更新器`updater`**，更新的创建过程是在`updater`中完成的。

`Component`中并没有给`updater`赋值，`ReactNoopUpdateQueue`可以理解为一个空值。真正给`updater`赋值发生在`beginWork`中，组件初始化的时候。

```javascript
function adoptClassInstance(workInProgress: Fiber, instance: any): void {
  // 更新器赋值
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;
  // 组件实例和fiber之间建立引用关系，方便更新的时候获取对应的fiber节点
  setInstance(instance, workInProgress);
}
```

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L563)

### 2.1 enqueueSetState

`this.setState`方法内部调用的是`enqueueSetState`，对应的逻辑都是在这个方法完成的。

> 对应的源代码可以看[这里](https://github.com/careyke/react/blob/765e89b908206fe62feb10240604db224f38de7d/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L195)

```javascript
enqueueSetState(inst, payload, callback) {
  	// 通过实例和fiber之间的索引关系来获取对应的fiber
    const fiber = getInstance(inst);
    const eventTime = requestEventTime();
  	// 获取本次更新的优先级
    const lane = requestUpdateLane(fiber);

  	// 创建Update
    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
        update.callback = callback;
    }

  	// 将当前更新加入Update链表
    enqueueUpdate(fiber, update);
  	// 调度更新入口，发起一次调度
    scheduleUpdateOnFiber(fiber, lane, eventTime);
},
```



### 2.2 Update的数据结构

