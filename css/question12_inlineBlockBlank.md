## inline-block元素之间为什么会产生间隙？怎么解决？
### 1.奇怪的现象
![inlineblockblank01.jpg](./images/inlineblockblank01.jpg)

```js
//html
<div class="inline">xxxxxx</div>
<div class="inline">xxxxxx</div>
<div class="inline">xxxxxx</div>
<div class="inline">xxxxxx</div>
<div class="inline">xxxxxx</div>

//css
.inline {
  display: inline-block;
  background-color: chocolate;
}
```  
可以看到，每两个inline-block之间都产生了一个空隙。产生的原因是**因为我们在html文件中为了层级结构分明，在每一个div后面都有一个换行符。默认情况下div是块级元素，换行符并不会被解析。但是当元素变成行内元素的时候，元素之间的空白符（空格、换行符等）会被浏览器处理，根据CSS中white-space属性对元素中空白进行处理（默认是normal，合并多余空白），所以HTML中的换行符被处理成了一个空白字符，在字体大小不为0的时候，空白字符也会占据一定的宽度，所以就产生了间隙**。
**
不只是inline-block元素会解析换行符，所有的行内元素（inline,inline-block）都会解析元素之间的换行符。 **span,a等

### 2.解决方法
#### 1.去掉html中的换行符，但是会降低可读性
```html
<span>xxx</span
><span>xxx</span
><span>xxx</span>

//或者
<span>xxx</span><!--
--><span>xxx</span><!--
--><span>xxx</span>
```  

#### 2.使用负margin处理，但是对于不同字体大小空白字符的宽度不同，不推荐大规模使用
```js
span{
  background-color: chocolate;
  margin-right: -4px;
}
```  


#### 3.父元素使用font-size: 0px。这种情况下空白字符的宽度为0
```js
.contianer{
  font-size:0px
}
.container span{
  font-size:14px;
}
```  

#### 4.使用float，修改元素为浮动元素，不解析标签之间的换行符。但是会有浮动成本

```css
.container span{
  float:left
}
```  

#### 5.使用letter-spacing或者word-spacing属性，可能会有兼容性问题。还有就是和负margin方法一样，对不同字体大小需要额外处理

1. letter-spacing —— 字符间隔
2. word-spacing —— 单词间隔

```css
.contianer{
  letter-spacing: -4px;
}
.container span{
  letter-spacing: 0px;
}
```  

```css
.contianer{
  word-spacing: -4px;
}
```  

### 参考文章
1. [inline-block产生空白间隙的问题](https://www.cnblogs.com/erduyang/p/5341953.html)
2. [去除inline-block元素间间距的N种方法](https://www.zhangxinxu.com/wordpress/2012/04/inline-block-space-remove-%E5%8E%BB%E9%99%A4%E9%97%B4%E8%B7%9D/)











