# React架构演变

React是一个用于构建能够 **快速响应** 的大型web应用的javascript库。为了进一步实现快速响应这个理念，React在16.0版本发布的时候重写了底层的架构，推出了全新的**Fiber架构**，替换了之前的**Stack架构**。

在介绍Fiber架构之前，笔者先抛出这么几个问题：

1. Stack架构是怎么实现的？为什么限制了React的进一步优化？
2. Fiber架构是怎么实现的？相比于Stack架构做了哪些优化？

在回答上面两个问题之前，我们先来了解一下制约应用 `快速响应` 的因素有哪些。



## 1. 影响快速响应的因素

在我们日常开发web或者app页面的时候，通常有两类场景会影响页面快速响应：

1. 当遇到大数据量的运算或者设备性能不足的时候，会导致页面掉帧，从而产生卡顿现象
2. 发送网络请求的时候，由于数据返回需要等待一段时间，所以不能快速响应

这两类场景可以概括为：

1. **CPU的瓶颈**
2. **I/O的瓶颈**

由于Stack架构在这两个场景上表现不佳，所以在新的Fiber架构中针对这两类场景做出了优化。



## 2. Stack架构

### 2.1 Stack架构的结构

Stack架构在功能上可以将结构分成两层：

1. Reconciler(协调器) —— 对比前后两次更新产生的虚拟DOM，找出组件的变化。著名的 `diff算法` 就是在协调器中产生作用。

2. Renderer(渲染器) —— 负责将协调器找出的变化更新到真实的DOM树上，从而更新视图。



### 2.2 Reconciler

当开发者调用 `this.setState`、`this.forceUpdate` 或者 `ReactDOM.Render` 时，就会发出一次React应用的更新操作，然后应用内部就出触发协调器。

协调器的工作流程：

1. 调用函数组件或者class组件的render方法，生成新的虚拟DOM
2. 然后将新的虚拟DOM和老的虚拟DOM做比较，执行Diff算法，找出其中的变化
3. 通知**Renderer**将变化渲染到视图中

> 可以在[这里](https://zh-hans.reactjs.org/docs/codebase-overview.html#reconcilers)看到官方对于协调器的定义

**协调器与所处的平台没有关系**，它的工作就是对新旧虚拟DOM进行对比，找出其中的变化。这一过程是纯数据操作，不涉及到平台的API调用。

### 2.3 Renderer

React的一个支持跨平台使用的库，不同的平台渲染视图的方式是不同的，所以对应的Renderer也会不同。前端工程师最熟悉的浏览器环境的Renderer——RenderDOM

除此之外，React还提供了以下Renderer:

1. **ReactNative** Renderer——负责在App端渲染React组件

2. **ReactTest** Renderer —— 负责将React组件渲染成JSON对象，用于测试
3. **ReactArt** Renderer —— 负责将React组件渲染到Canvas、SVG或者VML(IE8)

渲染器的工作流程：

1. 接受Reconciler找出本次更新的变化
2. 调用对应平台的API，将变化的节点渲染到视图上

> 可以在[这里](https://zh-hans.reactjs.org/docs/codebase-overview.html#renderers)看到官方对于渲染器的定义



### 2.4 Stack架构的更新流程

每当更新产生的时候，React内部的工作流程：

1. **协调器**开始工作，找出当前节点对应的虚拟DOM产生的变化，然后将变化的传递给渲染器
2. **渲染器**开始工作，将从协调器中收到的变化渲染到真实的DOM节点上
3. 然后调用 `mountComponent` 或者 `updateComponent` 方法，开始递归更新子节点（深度优先遍历）
4. 重复步骤1、2、3

从上面步骤可以看出，在Stack架构中，**协调器和渲染器是交替工作的，整个组件树是递归更新，而且是同步执行的**，所以最后页面看起来所有的DOM都是同时更新的，但是实际代码中真实DOM的更新并不是同时的。

> 个人感觉官方之所以将之前的架构称为Stack，就是因为之前的架构中是递归调用更新而且是同步执行的，导致React的更新流程会有一个很深的调用栈，在Performance工具中看起来很突出很有特点。



看一个Stack架构中更新的例子：

<img src="./images/v15.png" alt="v15" style="zoom:50%;" />

### 2.5 Stack架构的缺点

上面的分析中可以看出来Stack架构有两大特点：

1. 整个更新过程是**递归同步执行**的
2. 在更新组件树的过程中，**协调器和渲染器是交替执行的**

但是同时也是因为这两个特点，导致Stack架构不得不被重写。

#### 2.5.1 递归同步执行

Stack中更新的过程是递归同步执行的，所以当层级很深，节点很多的时候，就会导致当前帧的时间超过16.6ms，从而导致页面掉帧，造成卡顿。

> 浏览器一帧的工作流程可以参考[这里](https://github.com/careyke/frontend_knowledge_structure/blob/master/environment/browser/render/question03_frameAndEventLoop.md)

看一个很多节点的例子：

```react
class Demo extends React.Component {
  render() {
    const items = [];
    for (let i = 0; i < 3000; i++) {
      items.push(<li key={i}>{i}</li>);
    }
    return <ul>{items}</ul>;
  }
}
```

在Performance中的显示

<img src="./images/v15_satck.jpg?" alt="v15_satck" style="zoom:50%;" />

可以看出当前react执行render的时间花了122ms，远远超过了一帧的时间，这个问题可以归类为CPU瓶颈问题，会影响React应用的快速响应。

**那如何解决这个问题呢？**

React官方给出的回答——**将递归同步的更新修改成可中断的异步更新。**在浏览器的每一帧中，预留一定的时间给JS引擎用来执行js代码，如果超出这个预留时间，就中断执行，将控制权交给渲染线程，使其在必要的时候有机会渲染UI。React在下一帧的时候再继续执行被中断的任务。

> **这种将长任务分拆到每一帧中，像蚂蚁搬家一样一次执行一小段任务的操作，被称为`时间切片`（time slice）**

后面在讲Fiber架构的时候会详细介绍。

下面看一下**时间切片**的效果，需要先开启`React`应用的`Concurrent Mode`

```react
ReactDOM.unstable_createRoot(document.getElementById('root')).render(<App />);
```

`Concurrent Mode`下的调用堆栈：

<img src="./images/v17_concurrent.jpg" alt="v17_concurrent" style="zoom:50%;" />

可以看出，在`Concurrent Mode`中，更新任务被分成了一个个小任务（Task），每个任务执行的时间都是5ms左右，所以浏览器在当前帧有时间去执行UI渲染，可以有效减少掉帧的可能。

既然解决CPU瓶颈的核心是**时间切片**，那么在Stack架构上就没有办法实现吗？答案是肯定的，理由有二

1. Stack中组件树的更新是递归实现的，**递归的过程是无法打断的**。这个困境理论上是可以解决的，可以将递归修改成循环来实现

2. Stack中协调器和渲染器是**交替执行**的，这个是Stack无法实现时间切片的最大原因。

#### 2.5.2 交替执行

上面说到**协调器和渲染器的交替执行是Stack架构不能实现时间切片的最主要原因**，下面来分析一下原因

还是采用上面的例子，模拟一下如果更新的过程中出现了中断（实际并不会中断），会出现什么效果。

<img src="./images/v15_stack_broken.png" alt="v15_stack_broken" style="zoom:50%;" />

可以看出当第一个节点更新完成之后中断更新，会导致页面出现**不完全更新**。基于这个原因，所以React团队决定重写整个架构。

> 笔者使用React v15.7验证过，真的是**交替执行**的。demo可以看[这里](https://codesandbox.io/s/stackreact-wfkwf?file=/src/App.js)



## 3. Fiber架构

React16.0之后，React为了解决Stack架构存在的问题，推出了全新的Fiber架构。

### 3.1 Fiber架构的特点

1. 实现了**时间切片**，将原来同步的更新任务变成了**可中断的异步更新**任务

2. 加入了**优先级调度**机制，高优先级的任务可以打断低优先级的任务提前更新，这是React[**将人机交互研究的结果整合到真实的 UI 中**](https://zh-hans.reactjs.org/docs/concurrent-mode-intro.html#putting-research-into-production)的体现

### 3.2 Fiber架构的结构

Fiber架构在结构上可以分成三层：

1. **Scheduler(调度器)**——负责调度更新任务的优先级，高优先级的任务优先进入Reconciler
2. **Reconciler(协调器)**——负责找出变化的组件，并将变化暂存在对应的Fiber节点中
3. **Renderer(渲染器**)——负责将变化的组件渲染到真实的DOM树上

### 3.3 Scheduler

时间切片实现的关键是判断当前帧中是否还有剩余的时间可以用来执行JS代码，所以就需要一种机制，当浏览器有空余时间的时候通知React应用。

大部分浏览器已经实现了这种机制，就是[`requestIdleCallback`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)，但是这个API目前还有很多的缺点：

1. 兼容性不好
2. 触发的频率不稳定，受很多因素的影响。比如当浏览器的页签不处于激活状态的时候，该页签中的页面触发`requestIdleCallback` 的频率会变得很低

基于上面的原因，React放弃了使用 `requestIdelCallback` 来实现时间切片，而是利用 **Task(宏任务)** 实现了功能更加完善的 `requestIdleCallback` polyfill —— Scheduler

Scheduler的功能：

1. **时间切片** —— 告诉 React 应用当前帧中是否还有空余的时间，以便应用决定是否中断当前任务。
2. **优先级调度** —— 高优先级的任务优先调用，可以打断低优先级的任务。

可以看出，Scheduler在Fiber架构中是一个十分重要的部分，后面会有专门的章节来详细介绍。

> Scheduler是一个独立的模块，并不是和React强绑定在一起的。



### 3.4 Reconciler

在Stack架构中，Reconciler是递归同步来更新组件的，而且Reconciler和Renderer是交替执行的，所以无法实现时间切片。（上面有介绍原因）

在Fiber架构中，`Reconciler`大体上的工作流程和Stack架构的`Reconciler`是一样的，而且解决了Stack中`Reconnciler`存在的问题：

1. Reconciler将更新工作**由递归修改成了可以中断的循环过程**。每次循环开始之前都会调用 `shouldYield` 来判断是否需要中断任务

   ```javascript
   /** @noinline */
   function workLoopConcurrent() {
     // Perform work until Scheduler asks us to yield
     while (workInProgress !== null && !shouldYield()) {
       performUnitOfWork(workInProgress);
     }
   }
   ```

2. **Reconciler和Renderer不再交替工作**，而是在Reconciler中对于变化的`fiber`节点打上标记，当所有的节点都完成Reconciler的工作时，才会统一交给Renderer来执行DOM操作更新视图。所以不会出现更新不完全的中间状态



### 3.5 Renderer

Fiber架构中，Renderer会根据Reconciler给fiber节点打上的标记来执行对应的DOM操作，**同步**更新视图。

也就是说，**Renderer的工作是不能被打断的，一旦开始执行就会执行完。**



### 3.6 Fiber架构的更新流程

还是以前面的乘法Demo为例子：

<img src="./images/fiber_process.png" alt="fiber_process" style="zoom:50%;" />

图中红框的步骤随时都可能被以下原因打断：

1. 当前帧没有剩余的时间
2. 当前有更高优先级的任务需要执行

`Scheduler`和`Reconciler`的过程是在内存中运行的，并不会展示在页面上，所以即使被打断，也不会展示更新不完全的视图在页面上。



## 4. 参考资料

1. 图片例子参考[卡颂](https://react.iamkasong.com/preparation/newConstructure.html#react16%E6%9E%B6%E6%9E%84)

