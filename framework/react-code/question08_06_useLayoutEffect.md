# useEffect 和 useLayoutEffect

上一篇文章中我们分析`Effect`的数据结构和`useEffect`的工作流程，这一节中我们来分析`useLayoutEffect`的实现代码。

## 1. useLayoutEffect

### 1.1 mount阶段

对应的方法是`mountLayoutEffect`

```javascript
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

### 1.2 update阶段

对应的方法是`updateLayoutEffect`

```javascript
function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

从上面两个方法中可以看出，`useLayoutEffect`和`useEffect`的实现代码基本都是一样的。

有**两个地方不同**：

1. `Fiber`节点上增加的`flag`不同
2. `Effect`中增加的`tag`不同

#### 1.2.1 给Fiber增加的flag不同

执行`useLayoutEffect`给`Fiber`加上的是`Update flag`，表明`useLayoutEffect`对应的`Effect`并不是异步执行，而是**同步执行**。

之前在分析`commit`阶段的文章有提到过，`useLayoutEffect`对应Effect是在`commit`阶段同步执行的。

#### 1.2.2 Effect中的tag不同

`useLayoutEffect`对应的`Effect`的`tag属性`会包括`HookLayout tag`，表明其是由`useLayoutEffect`产生的。

## 2. useEffect VS useLayoutEffect

下面来对比总结一下`useEffect`和`useLayoutEffect`之间的不同。

| 对比维度               | useEffect                                                                      | useLayoutEffect                                              |
| ------------------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| 产生的副作用(Fiber flag) | Update \| Passive                                                              | Update                                                       |
| Effect的类型          | Passive                                                                        | Layout                                                       |
| Effect执行的方式        | 异步执行                                                                           | 同步执行                                                         |
| Effect执行的时机        | 主要在`commit-layout`阶段收集激活的Effect，异步执行destory和create函数（组件销毁时commit-mutation也会收集） | 在`commit-mutation`阶段执行destory函数，在`commit-layout`阶段执行create函数 |
