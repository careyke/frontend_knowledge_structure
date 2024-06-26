# 谈谈复合图层

在浏览器渲染页面的最后一个步骤中，浏览器会在多个普通图层中绘制元素的像素信息，然后将所有的层按照一定的规则合并成一个图层，称为**复合图层**。（这里的规则应该就是层叠作用上下文中的层叠顺序）

**一个正常的页面一般来说只会产生一个复合图层，当发生重排或者重绘的时候，浏览器最终都是以整个复合图层为单位来进行绘制的。**

所以在做性能优化的时候，可以将某些频繁变化的元素放在单独的复合图层中，可以避免影响其他的元素重新绘制。

## 1. 如何形成一个复合图层

形成复合图层也就是业界常说的硬件加速。

1. `translate3D `或者 `translateZ`

2. `opacity`属性/过渡动画（需要动画执行的过程中才会创建合成层，动画没有开始或结束后元素还会回到之前的状态）
3. `will-chang`属性（这个比较偏僻），一般配合opacity与translate使用。作用是提前告诉浏览器要变化，这样浏览器会开始做一些优化工作（这个最好用完后就释放）

4. `video/iframe/canvas/webgl` 等元素

**可以`Chrome源码调试 -> More Tools -> Rendering -> Layer borders`中看到，黄色的就是复合图层信息**



## 2. 复合图层的优点（*）

1. 合成层的位图，会交由 GPU 合成，比 CPU 处理要快
2. 当需要 repaint 时，只需要 repaint 本身，不会影响到其他的层
3. **对于 transform 和 opacity 效果，不会触发 layout 和 paint **

最大的优点：**形成复合图层之后，和其他的复合图层是相对独立的，内部元素发生重排或者重绘的时候，不会影响其他复合层的元素**



## 3. 复合图层的一些注意点

1. 创建复合图层是一个相对比较占资源的操作。所以在创建复合图层的时候，要考虑是否会得不偿失。
2. 层爆炸现象。也就是页面中不可预期的产生了很多复合层，导致大量占用内存。**产生这种情况的原因可能是在给某个元素添加复合图层的时候，这个元素的层级比较低，那么在这个元素的后面其它元素（层级比这个元素高的，或者相同的，并且releative或absolute属性相同的）， 会默认变为复合层渲染**

**所以在创建复合图层的时候，最好给元素加上一个z-index属性，提高元素的层级。避免创建多余的复合图层。**



## 4. absolute和复合图层的区别

absolute虽然脱离了文档流，但是并不能创建一个单独的复合图层，也就是说还是在默认的复合图层中的。所以，**就算absolute中信息改变时不会改变普通文档流中 render 树， 但是，浏览器最终绘制时，是整个复合层绘制的，所以absolute中信息的改变，仍然会影响整个复合层的绘制。**

所以如果复合层中的节点很多的时候，绘制整个复合层也是相当耗费资源的，所以这时就要考虑给元素创建新的复合图层。



## 参考文章

1. [从浏览器多进程到JS单线程，JS运行机制最全面的一次梳理](https://juejin.im/post/5a6547d0f265da3e283a1df7#heading-16)

2. [浏览器渲染流程&Composite（渲染层合并）简单总结](https://segmentfault.com/a/1190000014520786)

