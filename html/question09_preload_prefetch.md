# preload和prefetch的作用是什么？有什么区别？

`preload`和`prefetch`是浏览器提供的两种对静态资源**预下载**的方式，对于优化页面的渲染速度是很有作用的。



## 1.preload

`preload`在MDN中的定义：[传送门](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Link_types/preload)

> 关键字`preload`作为元素`<link>`的属性`rel`的值，表示用户十分有可能需要在**当前页面中加载目标资源**，所以浏览器必须预先下载和缓存对应的资源。

从上面定义可以看出，`preload`针对的是当前页面需要加载的资源，使用`preload`加载的资源会提前下载，但是并不会立即执行，而且等到使用的时候才会执行。



### 1.1 preload的使用方式

`preload`是`<link>`元素中rel属性的一个值，所以需要使用`link`标签来实现资源的预加载

```html
<link rel="preload" as="script" href='https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/js/bootstrap.min.js'>
```

对于预加载的资源来说，一般需要设置以下三个属性：

1. rel: preload或者prefetch，表示预加载的方式。必填
2. as: 表示预加载资源的类型。必填
3. href: 表示预加载资源的地址。必填