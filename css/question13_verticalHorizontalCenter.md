## 水平居中、垂直居中和垂直水平居中的方式
实际工作中，居中布局的场景随处可见。

### 1.水平居中
#### 子元素是行内元素（inline,inline-block）
```js
// 父元素设置
text-align: center;
```

#### 宽度确定的块级元素
1. margin：0 auto
2. 绝对定位+负margin
```css
.mid{
  position: absolute;
  left:50%;
  width:200px;
  margin-left:-100px;
}
```
3. 绝对定位 + margin: auto
```css
.mid{
  position: absolute;
  left: 0px;
  right: 0px;
  width: 100px;
  margin: 0 auto;
}
```


#### 宽度未知的块级元素
1. display: table + margin: 0 auto;
```css
.mid{
  display: table;
  margin: 0 auto;
}
```
2. 绝对定位 + transform
```css
.mid{
  position:absolute;
  left:50%;
  transform: translateX(-50%);
}
```
3. flex布局(作用在父元素)
```css
.father{
  display:flex;
  justify-content:center
}
```

### 2.垂直居中
#### 子元素的行内元素（inline,inline-block）
1. 父元素有确定的高度，使用line-height + vertical-align
```css
.father{
  height: 100px;
  line-height: 100px;
}
.son{
  vertical-align: middle;
}
```
如果子元素是inline元素，不需要设置vertical-align，因为inline元素在行框中垂直居中。如果子元素是inline-block，就需要设置vertical-align: middle。
2. 使用vertical-align + 伪元素，父元素宽度确不确定都可以。
```css
.father::after{
  display: inline-block;
  content: '';
  height: 100%;
  vertical-align: middle;
}
.son{
  vertical-align: middle
}
```

#### 2.高度固定的块级元素
1. 绝对定位 + 负margin
```css
.mid{
  position: absolute;
  top:50%;
  height:200px;
  margin-top:-100px;
}
```
2. 绝对定位 + margin: auto 
```css
.mid{
  position: absolute;
  top: 0px;
  bottom: 0px;
  height: 100px;
  margin: auto 0;
}
```
注意：**和水平居中对比可知，单独设置margin: auto 0 ;并不能使用块级元素垂直居中。**

#### 3.高度不固定的块级元素
1. 绝对定位 + transform
```css
.mid{
position:absolute;
top:50%;
transform: translateY(-50%);
}
```
2. flex布局（作用在父元素）
```css
.father{
  display:flex;
  align-items: center;
}
```

### 3.垂直水平居中
根据上面列出的垂直居中和水平居中，可以组合出一些垂直水平居中的方法
#### 子元素是行内元素（inline，inline-block）
1. 父元素有确定的高度：text-align + vertical + line-height
```css
.father{
height: 100px;
line-height: 100px;
text-align: center;
}
.son{
vertical-align: middle;
}
```
2. 父元素的高度不确定：text-align + vertical + 伪元素
```css
.father{
text-align: center;
}
.father::after{
display:inline-block;
content: '';
height: 100%;
vertical-align: middle;
}
.son{
vertical-align: middle;
}
```

#### 宽高确定的块级元素
可以根据上面的方法组合而成，一共可以组合出来6种。这里就说两种
1. 绝对定位 + 负margin
```css
.son{
width:100px;
height: 100px;
position: absolute;
top:50%;
left: 50%;
margin-left: -50px;
margin-top: -50px;
}
```
2. 绝对定位 + margin:auto
```css
.son{
width:100px;
height: 100px;
position: absolute;
top:0;
left: 0;
bottom: 0;
right: 0;
margin: auto;
}
```

#### 宽高都不确定的块级元素
1. flex布局（父元素）
```css
.father{
display: flex;
align-items: center;
justify-content:center;
}
```

2. 绝对定位 + transform
```css
.son{
position: absolute;
top: 50%;
left: 50%;
transform: translate(-50%,-50%);
}
```

### 参考文章
1. [https://segmentfault.com/a/1190000014116655](https://segmentfault.com/a/1190000014116655)





















