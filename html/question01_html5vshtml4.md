## HTML5与HTML4之间的区别
HTML5是对超文本标记语言的第五次修改，定义了全新的语法标准。

### 语法
HTML5语法向下兼容了大部分HTML4的语法，但是**HTML5文档的解析不在基于SGML**，而是形成了自己的标准。

#### 1.文档类型声明
HTML5不在基于SGML，所以不再需要是用DTD来声明文档类型。而是变成下面的方式
```js
<!doctype html>
```

#### 2.文档编码
HTML5使用了charset meta来声明文档的编码方式
```js
// html4声明方式
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">

// html5
<meta charset="utf-8">
```

#### 3.HTML5支持了音频和视频
```js
<audio src="http://developer.mozilla.org/@api/deki/files/2926/=AudioTest_(1).ogg" autoplay>
  Your browser does not support the <code>audio</code> element.
</audio>

<video src="videofile.ogg" autoplay poster="posterimage.jpg">
  抱歉，您的浏览器不支持内嵌视频，不过不用担心，你可以 <a href="videofile.ogg">下载</a>
  并用你喜欢的播放器观看!
</video>
```

#### 4.HTML5增加了canvas和svg标签，用来展示丰富的图形
```js
<svg width="150" height="100" viewBox="0 0 3 2">
  <rect width="1" height="2" x="0" fill="#008d46" />
  <rect width="1" height="2" x="1" fill="#ffffff" />
  <rect width="1" height="2" x="2" fill="#d2232c" />
</svg>


// canvas定义的是画布，具体如何画需要配合api使用
<canvas id="canvas"></canvas>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
ctx.fillStyle = 'green';
ctx.fillRect(10, 10, 150, 100);
```

#### 5.HTML5增加了很多语义化的标签
```html
<header> <footer> <section> <article> <nav> <aside> <main> <figure>
```

#### 6.删除很多描述样式的标签，进一步分离结构和样式
```html
<basefont> <font> <big> <center> <strike> <tt>
```

#### 7.新增属性，这里主要针对表单元素来说明
- input标签增加多种type：tel, search, email, url, number, date, time, range, color。
- form属性可以用于input,select,textarea,button,label,feildset等表单元素上，用来指定该元素属于哪一个form，可以是form标签和表单元素的层叠关系变得相对自由。
- input,textarea等元素提供placeholder属性，用来给用户一些输入提示。
- script标签增加async defer属性，用来定义js脚本的加载方式。

#### 8.部分属性名默认具有boolean属性
```js
// 只写属性名默认为true
<input type="checkbox" checked />

// 属性名="属性名"也是true
<input type="checkbox" checked="checked">
```
类型的属性还有：selected, disabled, readonly

#### 9.HTML5提供了丰富的Web存储方式
##### 1.增加了localStorage和sessionStorage。
- localStorage : 长期存储站点的数据，站点内部每一个页面都可以访问到。（同源策略）
- sessionStorage : 会话存储，用来临时保存某一个页签或者窗口的数据，当页签或者窗口关闭的时候数据会销毁。

##### 2.引入了indexedDB和Web SQL，允许在浏览器端创建数据库表存储数据

##### 3.引入了应用程序缓存器(application cache)，可对web进行缓存，在没有网络的情况下使用，通过创建cache manifest文件,创建应用缓存，为PWA(Progressive Web App)提供了底层的技术支持


### 不一定需要全部记住，但是要有印象

### 参考
1. [HTML4和HTML5的区别](https://www.jianshu.com/p/e09d4b126384)
2. [HTML5与HTML4的区别](https://blog.csdn.net/superhoy/article/details/51637670)










