# 从URL到页面输出中间经历了哪些过程？

要做前端的性能优化，首先要弄清楚从输出url到页面输出这中间经历了哪些过程，然后针对每一个过程进行适当的优化。

## 经历的过程

1. 请求DNS服务，对请求的域名进行解析，解析成IP的形式

2. 在传输层创建TCP连接，三次握手

3. 向请求地址发起GET请求

4. 服务器接收到请求之后返回响应报文

5. 浏览器拿到响应报文之后开始渲染页面，第一次请求一般返回的是一个html文件

   1. 浏览器解析html的响应报文，网络传输的都是字节数据，所以需要解析装换，最终生成一颗DOM Tree

   ```js
   Bytes -> Characters -> Tokens -> Nodes -> DOM
   ```

   2. 在解析DOM的过程中，如果遇到`link`标签，就会请求CSS文件，然后也进行解析，最终生成一颗CSSOM Tree

   ```js
   Bytes -> Characters -> Tokens -> Nodes -> CSSOM
   ```

   3. 在解析DOM的过程中，如果遇到`<script>`标签，会判断是否存在defer或async属性
      - 如果存在，会并行下载js文件，defer类型的js会在html解析完成之后顺序执行；async类型的js文件会在下载完之后立即执行
      - 如果不存在，则会**阻塞渲染流程（CSSOM和DOM）**，等js文件下载执行都完成之后，再开始后面的渲染流程
   4. 在这个过程中遇到媒体文件就会**并行**去下载文件
   5. DOM树和CSSOM树在渲染完成之后，会将两者合并成一颗Render Tree，每个节点包含了元素的布局和样式信息
   6. 渲染引擎根据Render Tree中的节点信息，计算每个节点的大小和在当前图层的位置
   7. 然后交给GPU进行图层绘制和图层合并



## 总结一些点

1. 在页面渲染过程中，最后三步，也就是生成Render Tree、Layout和Paint是**并行执行**的
2. DOM Tree和CSSOM Tree也是并行执行的，但是要两者都生成完成才能合成Render Tree
3. js的作用就是修改页面的结构和属性，所以js的执行会阻塞DOM Tree和CSSOM Tree的构建流程



## 参考文章

1. [知己知彼——解锁浏览器背后的运行机制](https://juejin.im/book/5b936540f265da0a9624b04b/section/5bac3a4df265da0aa81c043c)

2. [输入 URL 到页面渲染的整个流程](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc73e05188251719353031)

3. [从URL输入到页面展现到底发生什么？](https://juejin.im/post/5c7646f26fb9a049fd108380#heading-12)

