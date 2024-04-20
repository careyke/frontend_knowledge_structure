## src和href引入资源的区别
### 定义

#### href
href表示超文本引用，指向资源的位置，将文档和资源之间建立联系。

```html
<a href="www.baidu.com" rel="stylesheet">link</a>

<link type="text/css" href="./index.css" />
```

#### src
src(source)表示引用外部资源来**替换**当前元素。在请求src资源的时候会将其**指向的资源下载下来并应用在当前文档中**。

```html
<img src="./lanqiu.jpg">

<script src="./index.js" type="text/javascript"></script>
```

### 浏览器的解析方式
#### 常见的说法
1. 浏览器对于href链接的资源，会并行下载，不会阻塞dom的构建。
2. 浏览器对于src指向的资源，会串行下载和执行，所以是会阻塞dom的构建。


但是这种说法是不正确的，浏览器对于资源的下载应该不是根据链接的种类来判断的。**img标签中src指向的图片资源是并行下载。**

```html
<div style="height: 100px;background-color:aquamarine"></div>
<img style="height: 200px" src="https://images.pexels.com/photos/3490257/pexels-photo-3490257.jpeg" />
<div style="height: 100px;background-color:royalblue"></div>
```
这里看到的现象就是两个div一开始就渲染出来，图片缓慢的渲染出来。

**script标签中的src指向的脚本资源，如果没有设置defer和async，js文件的加载和执行确实会阻塞dom的构建**。但是这应该是浏览器的策略，而不是src的功能。

```js
<div style="height: 100px;background-color:aquamarine"></div>
<script type="text/javascript" src="./index.js"></script>
<div style="height: 100px;background-color:royalblue"></div>

//index.js
const arr = [];
for (let i = 0; i < 100000000; i++) {
  arr.push(i);
  arr.splice(i % 3, i % 7, i % 5);
}
console.log(arr);
```
可以看到第二个div是在js文件下载完成并且执行之后才渲染出来的。**这也就是说明了为什么要将script标签放在body的最下面，否则会出现页面闪动的现象。**

### 总结
1. href链接资源的位置，建立资源之间的联系，浏览器会并行加载。
2. src指向的资源会下载下来，替换当前的标签，但是**浏览器并非对于所有的src资源都会同步下载和执行，script指向的外部js的下载和执行会阻塞html的解析，但是img指向的图片资源并不会**。
