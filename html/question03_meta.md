## html中meta标签有哪些？都有什么作用？

HTML中提供了meta关键字来定义文档的**元数据**，这些元数据描述了当前html的一些特性。比如网页关键字，作者，修改日期等，这些关键字**不会渲染在页面上**，会被浏览器和搜索引擎使用。

### 特点
1. 元数据需要使用meta标签包裹起来
2. meta标签只能放在head里面
3. meta的常用属性有charset / name / http-equiv / content。**其中charset单独使用，name和http-equiv需要和content成对使用**

### charset
声明当前文档使用的**字符编码**，但该声明可以被任何一个元素的**lang**属性的值覆盖
```js
<meta charset="utf-8">
```

### name + content
通常用来定义文档级的元数据，用来**描述文档本身的属性**

#### 一般格式
```js
<meta name="元数据的属性名" content="对应的属性值" >
```

#### 常见的元数据
```js
// 网页的作者
<meta name="author" content="zhoujielun">

// 网页的描述信息
<meta name="description" content="singer">

// 定义关键字，用于SEO
<meta name="keywords" content="喜剧,悲剧">

// 网页地址
<meta name="website" content="https://www.baidu.com"/>

// 网页版权信息
<meta name="copyright" content="kechuang"/>

// 搜索引擎爬虫的索引方式,默认是all
<meta name="robots" content="all">

// 移动端视口常用设置
<meta name="viewport" content="width=device-width,initial-scale=1.0,minimum-scale=1,maximum-scale=1,user-scalable">
<!--
  width：宽度（数值 / device-width）（默认为980 像素）
  height：高度（数值 / device-height）
  initial-scale：初始的缩放比例 （范围从>0 到10）
  minimum-scale：允许用户缩放到的最小比例
  maximum-scale：允许用户缩放到的最大比例
  user-scalable：用户是否可以手动缩 (no,yes)
-->

// 定义多核浏览器的渲染方式。下面以360浏览器为例说明
<meta name="renderer" content="webkit"> //默认webkit内核
<meta name="renderer" content="ie-comp"> //默认IE兼容模式
<meta name="renderer" content="ie-stand"> //默认IE标准模式

```

### http-equiv + content
这个属性定义了能够**改变服务器行为和用户引擎行为**的元数据

#### 一般格式
```js
<meta http-equiv="元数据的属性名" content="对应的属性值" >
```

#### 常见的元数据
```js
// html文件的过期时间，一旦过期就需要重新下载
<meta http-equiv="expires" content="Fri, 12 Jan 2020 18:18:18 GMT"/>

// 间隔多少秒之后刷新本页面或者跳转到执行地址。下面表示3s之后跳转到百度
<meta http-equiv="refresh" content="3; url=https://www.baidu.com"/>

// 定义当前页面允许获取资源的服务器站点，可以有效防止XSS攻击
<meta http-equiv="Content-Security-Policy" content="script-src 'self'">

// 指定浏览器采用何种版本渲染页面。下面表示IE使用edge，chrome使用最新版本渲染
<meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />

// 设置cookie的一种方式，但是不推荐使用，即将废弃
<meta http-equiv="set-cookie" content="name=value expires=Fri, 12 Jan 2001 18:18:18 GMT,path=/"/>

// 开启DNS预解析，做性能优化的时候可以用上
<meta http-equiv="x-dns-prefetch-control" content="on" />
```

















