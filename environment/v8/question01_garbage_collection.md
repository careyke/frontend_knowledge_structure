# V8引擎中的垃圾回收机制

由于内存的空间大小是有限的，所以当变量没有作用的时候，需要及时去释放对应的内存资源，否则可能会导致内存空间不足的问题。

和C++不同的是，V8引擎中提供了垃圾回收的机制，是一种自动回收的机制，没有给开发者提供接口去手动控制整个垃圾回收的过程。**V8引擎来自动分配内部和内存清理**



## 1. V8能够使用的内存大小

由于V8引擎最开始是给浏览器执行js脚本来用的，所以在设置v8能够申请的最大内存的时候，并没有将值设置得很大：

1. 32位系统下，最大能使用的内存大约是700MB
2. 64为系统下，最大能使用的内存大约是1.4GB

### 1.1 为什么没有将最大的内存值设置得很大？

**v8能够使用的内存空间越大，那么就意味着能够容纳的变量也就越多，那么GC一次的时间也会更长。然而JS引擎是单线程的，而且和渲染引擎是互斥的，所以在执行GC的时候，会导致页面长时间的卡顿**。所以不宜将V8的内存值设置得很大。



## 2 V8中如何判断是否可以回收

在V8中有两个方式来判断当前的内存空间是否可以被回收：

1. 引用计数
2. 标记清除
