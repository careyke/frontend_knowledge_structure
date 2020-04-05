# 网络层面的优化策略

前面分析过，页面初始加载过程中，涉及网络层面的有三个阶段：

1. DNS域名解析
2. TCP连接
3. http请求/响应

## 1.DNS解析的前端优化-预解析

DNS每次解析的过程大概消耗20-120ms的时间，所以如果应用中有多个不同的域名需要解析，那消耗的时间也是不能忽略的。可以使用DNS预解析的方式来做一些优化。

**DNS Prefetch是让具有此属性的域名不需要在请求的时候解析域名，而是先在后台进行解析，可以省略请求时的DNS解析的时间。**



DNS预解析的分类：

1. **隐式预解析**：默认情况下，浏览器会对当前html中存在的其他域名进行预解析，并且缓存结果

2. 显示预解析：如果想要对于当前html中不存在的域名进行预解析，需要使用 `link` 标签来显示引入域名

   ```html
   // 开启预解析
   <meta http-equiv="x-dns-prefetch-control" content="on">  // content = "off" 可以关掉预解析
   
   // 显示预解析
   <link rel="dns-prefetch" href="//www.zhix.net">
   ```

浏览器对网站第一次的域名DNS**解析查找流程**依次为：

1. 浏览器缓存
2. 系统缓存
3. 路由器缓存
4. ISP DNS缓存
5. 递归缓存



==注意：DNS预解析不能滥用，会导致DNS的查询次数大量增加。==



## 2. http请求过程的优化

针对于http请求的优化，主要有以下两个方向：

1. ==减少http请求的次数==
2. ==减少单次http请求所花的时间==

这两个优化的方向涉及到的是资源的合并和压缩，刚好可以在webpack构建工具中完成。

### 2.1 http协议的优化

随着http协议的迭代，内部对于http的传输过程做了很多的优化，所以推荐使用高版本的http协议，比如http2.0

http2.0的优化：

1. 二进制传输
2. **多路复用**。解决了原来浏览器只能建立大约5个TCP连接所以带来的队头阻塞的问题，现在在一个连接中可以同时传输多个请求的报文，没有了TCP连接数的限制。
3. 头部压缩和索引表

### 2.2 webpack的优化

webpack的优化分成两个方面：

1. [webpack构建速度的优化](https://github.com/careyke/frontend_knowledge_structure/blob/master/engineering/webpack/question01_speedOptimization.md)
2. [webpack打包资源体积的优化](https://github.com/careyke/frontend_knowledge_structure/blob/master/engineering/webpack/question01_volumeOptimization.md)
   1. 文件压缩
   2. 基础库分包（增加请求，减少体积），多页面应用尤为明显
   3. Tree Shaking
   4. 公共包分离（splitChunks）

### 2.3 图片资源的优化

1. 小图片使用base64格式，减少请求，webpack可以完成
2. 图片压缩
3. 雪碧图
4. 使用字体图标来代替图片图标
5. 合理使用图片格式
   - JPG：有损压缩，体积小。适用于呈现丰富多彩的图片
   - PNG: 无损压缩，质量高体积大。适用于呈现小的 Logo、颜色简单且对比强烈的图片或背景

### 2.4 静态资源的优化

静态资源的体积相对来说都是比较大的，对这些资源的请求优化是很有必要的。

==静态资源优化的方案：==

1. 部署在CDN服务器中，缩小服务器的距离，提高响应的速度

2. 使用GZIP格式传输响应报文，提交传输速度。在资源文件比较大的时候考虑使用。由于GZIP传输增加了额外的压缩和解压的过程，所以在资源文件小的时候反而会降低速度。

   ```js
   // 请求头
   accept-encoding:gzip
   ```

3. 使用HTTP缓存，提高二次请求的速度

   - 强缓存
   - 协商缓存

4. 浏览器本地存储，针对一些不变的资源。可以减少http请求的次数

   - Cookie
   - SessionStorage
   - LocalStorage
   - IndexedDB - 数据库级别的存储量

图片也是静态资源，上面的优化同样适用于图片资源



## 参考文章

1. [浅谈前端性能优化（九）——DNS解析优化](https://blog.csdn.net/zhouziyu2011/article/details/71351967)

2. [前端性能优化原理和实践](https://juejin.im/book/5b936540f265da0a9624b04b/section/5b936540f265da0aec223b5d)

