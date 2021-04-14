# 结合源码分析useTransition

在Concurrent模式中，React提供了一个新的API — useTransition。具有以下功能

1. 延迟更新 — **降低更新的优先级**
2. 配合`Suspense`**解决因为组件挂起而导致用户交互内容隐藏的问题**

常见的使用场景是在切换页面的时候，但是下一页的数据还没有加载好，切换过去会出现空白页或者`loading`状态，这种交互是不友好的。这种情况下，用户更希望的时候在当前页面停留一会儿，然后等下一页的数据加载好之后再切换过去。

`useTransition`通常会配合`Suspense`一起使用，用来解决`Suspense`使用中遇到的问题。具体是什么问题以及如何解决的，我们后面再来分析，这里我们先来分析一下`useTransition`的实现。



## 1. useTransition的源码实现

### 1.1 mount阶段



## 2. useTransition 和 Suspense

在上一节中我们分析了`Suspense`的实现原理和怎么解决传统模式下React应用中的一些痛点。

但是`Suspense`模式本身也会带来一个问题：**在请求没有完成的时候，会回退到`fallbackChildren`**。在`mount`阶段的时候这种交互是可以接受的，但是在`update`阶段，这种回退的交互是不友好的



