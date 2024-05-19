## CSS样式权重和优先级
在实际的项目中，可能会有很多个CSS样式作用在同一个元素上，这时候就产生样式优先级的问题。**在CSS中，使用CSS样式权重来决定哪些规则的优先级更高，作用相同的多个样式规则中，优先级高的规则最终生效。**

### CSS样式权重的划分
**CSS中，根据样式权重的不同，可以分成6个不同的层次，可以使用数字来具体化每个权重的值，方便比较。**

1. !important : **∞**（正无穷）
2. 行内样式 : **1000**
3. id选择器 ：**100**
4. 类选择器、属性选择器和伪类选择器 ：**10**，伪类选择器（:hover,:focus）
5. 元素选择器和伪元素 ：**1**，伪元素选择器（::after,::before）
6. 通配符选择器 ：**0**

**复合选择器的权重是各个选择器的权重之和**。比如#id .class的权重就是100+10 = 110

### 权重规则
#### 1.!important的权重是最高，优先级也是最高，但是可以被高权重的!important覆盖
![selector01.jpg](./images/selector01.jpg)

```html
<div class="demo" id="name">
  文本的颜色
</div>

//css
#name{
  color: red;
}
.demo{
  color: aquamarine;
}
div{
  color: blue !important;
}
```
可以看到!important的优先级最高，但是这种用法的入侵比较高，会破坏选择器的权重比较规则。

##### 覆盖!important
![selector02.jpg](./images/selector02.jpg)

```html
<div class="demo" id="name">
  文本的颜色
</div>

//css
#name{
  color: red !important;
}
.demo{
  color: aquamarine;
}
div{
  color: blue !important;
}
```
图中可以看出，**权重高的选择器中的 !important 会覆盖低权重选择器中的 !important **。

##### 不推荐使用!important
在实际的项目中，**!important正确是使用场景是在需要覆盖插件或框架原本样式的情况下使用**。但是很多开发者使用这个属性只是为了偷懒，但是这样会造成更多的隐藏问题。**因为!important是没有上下文和结构的，而且会造成选择器权重分析的偏差，导致分析问题成本增加。**

#### 2.单独使用某一个等级权重的选择器，无论该选择器有多少个，其优先级也低于一个上级权重的选择器。
读起来比较拗口，举个例子帮助理解：**无论多少个class组成的选择器，都没有一个ID选择器权重高**。因为class选择器权重是10，id选择器权重是100，理论上是超过10个class组成的选择器权重应该是大于id选择器的。但是**权重的等级是越不过去的，先有等级，权重值只是为了比较方便而预设的。**

![selector03.jpg](./images/selector03.jpg)

```html
<div class="test1">
    <div class="test2">
      <div class="test3">
        <div class="test4">
          <div class="test5">
            <div class="test6">
              <div class="test7">
                <div class="test8">
                  <div class="test9" >
                    <div class="test10" id="test2">
                      <div class="test11" id="test11">权重</div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
   </div>
 </div>

//css
.test1 .test2 .test3 .test4 .test5 .test6 .test7 .test8 .test9 .test10 .test11{
  color: aquamarine;
}

#test11{
  color: coral;
}
```
可以看出，这个class选择器的权重值是110，而id选择器的权重是100。如果只看权重值，理论上是class选择器比较大，但是实际上是id选择器比较大。

#### 3.两个权重不同的选择器作用在同一个元素时，权重值高的选择器优先级更高
这里一般都会涉及到复合选择器的权重的计算。

![selector04.jpg](./images/selector04.jpg)

```html
<div class="test1">
  <div class="test2">
    <p id='name' class="test3">权重</p>
  </div>
</div>

//css
.test3{ //10
  color: aqua;
}
.test2 .test3{ //10+10=20
  color: blueviolet;
}
.test1 .test2 .test3{ //10+10+10=30
  color: chartreuse;
}
.test1 .test2 #name{ //10+10+100=120
  color: red;
}
```

#### 4. 如果两个权重值相同的选择器作用在同一个元素，后定义的选择器中的规则会覆盖前面的选择器
![selector05.jpg](./images/selector05.jpg)

```html
<div class="test1">
  <div class="test2">
    <p id='name' class="test3">权重</p>
  </div>
</div>

//css
.test2 .test3{ //10+10=20
  color: blueviolet;
}
.test1 .test3{ //10+10=20
  color: red;
}
```

#### 5.如果两个权重值相同的选择器作用在同一个元素，但是引入的方式不同，与元素近的选择器优先级更高。

![selector04.jpg](./images/selector04.jpg)

```html
<style type="text/css">
  .test1 .test3{
    color: red;
  }
</style>

<div class="test1">
  <div class="test2">
    <p id='name' class="test3">权重</p>
  </div>
</div>

//css
.test2 .test3{
  color: blueviolet;
}
```
由于内嵌方式引入的css样式距离元素更近，所以内嵌方式的选择器优先级更高。

### 使用总结
1. 尽量少用!important
2. 可以利用id选择器来增加权重
3. 减少复合选择器的复杂度，避免层层嵌套


### 参考文章
1. [你对CSS权重真的足够了解吗？](https://juejin.im/post/5afa98bf51882542c832e5ec)
2. [你应该知道的一些事情——CSS权重](https://www.w3cplus.com/css/css-specificity-things-you-should-know.html)











