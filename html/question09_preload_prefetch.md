# preload和prefetch的作用是什么？有什么区别？

`preload`和`prefetch`是浏览器提供的两种对静态资源**预下载**的方式，对于优化页面的渲染速度是很有作用的。



## 1.preload - 立即下载

`preload`在MDN中的定义：[传送门](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Link_types/preload)

> 关键字`preload`作为元素`<link>`的属性`rel`的值，表示用户十分有可能需要在**当前页面中加载目标资源**，所以浏览器必须预先下载和缓存对应的资源。

从上面定义可以看出，`preload`针对的是当前页面需要加载的资源，使用`preload`加载的资源会提前下载，但是并不会立即执行，而且等到使用的时候才会执行。



### 1.1 preload的使用方式

`preload`是`<link>`元素中rel属性的一个值，所以需要使用`link`标签来实现资源的预加载

```html
<link rel="preload" as="script" href='https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/js/bootstrap.min.js'>
```

对于预加载的资源来说，一般需要设置以下三个属性：

1. rel: `preload`或者`prefetch`，表示预加载的方式。必填（rel的值很多，这里只考虑预加载的情况）
2. as: 表示预加载资源的类型。必填
3. href: 表示预加载资源的地址。必填

> **注意**
>
> 当预加载的是字体资源时，必须加上`crossorigin`属性



### 1.2 preload的好处

1. **能够分离资源的下载和执行**
2. **能够提高资源的下载优先级**
3. **能够支持多种资源的预下载**，比如脚本，样式，图片等等



### 1.3 preload VS defer

和preload一样，defer的script资源也会将下载和执行过程分离。不同的是，**`preload`的资源是由开发者来确定何时执行，`defer`的`script`资源是由浏览器来决定何时执行。**

除此之外，defer和preload相比还有以下缺点：

1. 只能支持`script`资源



### 1.4 preload VS 预解析操作

在分析页面渲染流程的时候我们提到过浏览器的一个优化操作，就是预解析操作。**当浏览器获取到HTML文件之后，会分析其中依赖哪些外部资源，并提前下载这个外部资源。**

> 页面渲染流程的文章可以看[这里](https://github.com/careyke/frontend_knowledge_structure/blob/master/environment/browser/rendering/question06_first_render.md)

看上去这个功能和preload基本上是一样的，可以达到同样的效果。

但是浏览器预解析操作有一个缺陷：**就是只能预下载HTML文件中引入的静态资源，对于当前页面动态加载的资源是无能为力的。但是preload可以解析这个问题。**



## 2. prefetch - 有空才下载

prefetch在MDN中的定义：[传送门](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Link_types/prefetch)

> 关键字 **`prefetch`** 作为元素`<link>`的属性 `rel`的值，是为了提示浏览器，用户**未来的浏览**有可能需要加载目标资源，所以浏览器有可能通过事先获取和缓存对应资源，优化用户体验。

从上面定义可以看出，`prefetch`针对的资源是用户下个浏览的页面需要的资源，可以在当前页面开始预下载，提高下个页面渲染的速度。

在使用上，prefetch和preload基本是一致的。



## 3. preload VS prefetch

preload 和 prefetch在使用上是有很大的不同的。

- **`preload`针对的资源是当前页面需要的资源，下载的优先级很高**
- **`prefetch`针对的资源是下个页面需要的资源，下载的优先级很低**

所以开发者是使用的时候需要区分场景，避免浪费用户的带宽资源。



## 4. 参考资料

1. [使用 Preload/Prefetch 优化你的应用](https://zhuanlan.zhihu.com/p/48521680)

