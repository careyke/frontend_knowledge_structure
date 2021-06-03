## html中引入css的方式有几种？

### 1.内联方式
```html
<div style="height:200px"></div>
```

### 2.嵌入方式(style标签)
```html
<style type="text/css">
  .content {
    height:50px;
    background-color:red;
  }
</style>

<div class="content"></div>
```

### 3.链接方式
```html
<link rel="stylesheet" type="text/css" href="./index.css" />
```

### 4.导入方式
```html
<style type="text/css">
  @import url("https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css");
</style>
```
使用@import来导入外部css文件


### 链接方式 vs 导入方式
1. link属于html，通过link标签中的href来链接外部css文件；而@import属于css范畴，需要写在css文件中，而且导入语句必须写在文件头部。
2. @import是css2.1才出现的规范，低版本浏览器可能有兼容问题
3. link方式导入的css文件，下载过程是并行的，不会阻塞dom树的构建；而**使用@import方式引入的文件则需要等dom树构建好之后才会去下载和执行，也就是`DOMContentLoaded`事件触发之后才会下载和执行css文件**。

所以在**导入外部的css文件的时候，尽量使用link的方式**。使用@import的方式添加样式的时机比较晚，可能会造成**页面抖动**。

