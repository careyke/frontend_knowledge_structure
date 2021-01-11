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

