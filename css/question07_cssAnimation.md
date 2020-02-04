## CSS动画的实现方式有哪些？CSS动画 VS JS动画?
**动画的形成实际上是元素从一个关键帧的状态变换到另一个关键帧状态的过程，所以一个动画中的关键帧越多，说明这个动画越复杂，动画越丰富**。在CSS中，让元素动起来有三种方式可以实现：
1. transform：一个关键帧，看不到动画变化的过程，只能看到结果。
2. transition：两个关键帧，分别是动画的初始状态和结束状态，可以看到中间的过渡过程
3. animation：多个关键帧，适用于复杂动画

但是这三种方式是有本质上的区别的。

### 1.transform
通过修改CSS盒模型的坐标空间来实现元素的平移、旋转、缩放和倾斜。**只能看到变化之后的状态，无法看到过渡的过程**
```css
transform: none | transform-functions

//example
transform: none;
transfrom: translate(20px,20px);
transform: rotate(30deg) translate(20px,30px);
```  

#### transform-functions列表
function | 说明 | 示例 | 派生  
------- | ------- | -------  
<div style="width: 100px">translate | 平移元素，基点默认为元素的中心点,可以通过transform-origin修改 | <div style="width: 180px">translate(20px,30px) | translateX(20px),<br>translateY(2px),translateZ(0px),translate3d(0,0,0)  
scale | 缩放元素 | scale(x,y) | scaleX(),<br>scaleY()  
rotale | 以元素中心点为圆心顺时针旋转元素 | rotate(30deg) | 
skew | 倾斜元素，实际上倾斜坐标系 | skew(30deg,30deg) | skewX(30deg),<br>skewY(30px)
matrix | 设置元素的矩阵变换，不常用 | matrix(1, 2, 3, 4, 5, 6); |

### 2.transition
**transition属性用来描述某些属性在两个关键帧之间的过渡过程**。一旦transition-property指定的属性发生变化，就会动画的过渡过程。

transition动画由于需要transition-property发生变化的时候，才会产生过渡动画。所以**transition动画初始的时候无法触发，需要一个触发的条件。比如常用的:hover，:active或者js事件**

```js
transition: [transition-property] [transition-duration] [transition-timing-function] [transition-delay]

//example
transition: width 2s ease-in-out 1s;

transition: all 2s ease-out; //任何属性变化都会触发过渡动画

transition: none 1s ease-in; //没有过渡动画

transition:width 2s ease-in,height 1s esae-out; //多个属性

.demo:hover{
  width:20px;
}
```  

#### transition属性列表
属性名 | 说明 | 取值 | 示例  
------- | ------- | -------  
transition-property | 产生过渡动画的属性 | [css属性] / all / none | transition-property: width
transition-duration | 过渡动画的持续时间 | number[s] | transition-duration: 2s
transition-timing-function | 计算动画中间值的函数，定义动画的过渡过程 | ease,ease-in,ease-out,ease-in-out | transition-timing-function: ease
transition-delay | 动画的延时时间，过多久之后执行动画 | number[s] | transition-delay: 2s  

#### transition动画的缺点
1. 需要事件触发，无法直接执行动画
2. 是一次性的动画，如果需要多次触发动画就需要多次触发开始事件
3. 动画只有两个关键帧，无法使用transition来实现复杂的动画
4. 动画过程无法暂停或者重启


### 3.animation
animation是transition的真强版，用来实现复杂的动画。**animation动画中可以定义多个关键帧，相邻两个关键帧之间都会产生过渡动画**，意味着animation可以实现更加复杂的动画效果。

animation由于可定义的关键帧比较多，所以**采用keyframes关键字来定义动画的关键帧**。

```css
@keyframes animationName {
  0%{
    width:100px;
  }
  50%{
    width:200px;
  }
  100%{
    width:100px;
  }
}

animation: animationName 2s ease-in
```  

#### keyframes用法
```css
@keyframes name{
  keyframes-selector{css-style}
}
```  
属性名 | 说明 
------- | ------- 
animation name | 必需，定义动画的名称，animation中会使用
keyframes-selector | 必需，定义动画的关键帧。<br>合法的值：<br>1. 0% - 100%<br>2.from(0%)<br>3. to(100%)
css-style | 必需，动画的变化属性 

#### animation的属性列表
属性 | 说明 
------- | ------- 
<div style="width: 200px">animation-name | 用来调用@keyframes定义好的动画，与@keyframes定义的动画名称一致 
animation-duration | 一次完整动画的持续时间 
animation-timing-function | 动画中间帧的计算函数，规则动画速度曲线 
animation-delay | 动画执行之前的延时时间
animation-iteration-count | **动画的执行次数**。可选具体数值或者infinite
animation-direction | **设置动画的播放方向（keyframes定义的时间轴）**。normal(按时间轴顺序),reverse(时间轴反方向运行),alternate(轮流，即来回往复进行),alternate-reverse(动画先反运行再正方向运行，并持续交替运行)
animation-fill-mode | **控制动画结束后元素的样式**，有四个值：none(回到动画没开始时的状态)，forwards(动画结束后动画停留在结束状态)，backwords(动画回到第一帧的状态)，both(根据animation-direction轮流应用forwards和backwards规则)，注意与iteration-count不要冲突(动画执行无限次)
aniamtion-play-state | **控制元素动画的播放状态**，通过此来控制动画的暂停和继续，两个值：running(继续)，paused(暂停)

#### animation动画的优点，对比与transition动画
1. 实现简单，复杂的动画也可以简单的实现，animation可以定义多个关键帧
2. 动画可以重复执行
3. 动画可以暂停和继续播放
4. animation动画可以直接启动，不需要通过其他事件来启动。

通过这个对比可以看出，animation动画基本上是解决了transition动画的缺点。在实际的使用过程中，简单的动画建议使用transition，复杂的动画使用animation。

### 4.CSS动画 VS JS动画
#### javascript动画
js动画是使用js代码来控制CSS属性从而产生动画。在实际的使用中，常常通过使用**requestAnimationFrame**方法来优化js动画的性能。

#### js动画的优点
1. js可以实现的动画种类很多，比较灵活
2. js可以很灵活地控制动画的各个部分，动画的过程完全由开发者控制
3. js动画的兼容性更好

#### js动画的缺点
1. js定义动画得写js代码，实现起来较为复杂
2. js动画很容易造成性能问题，需要使用额外的优化代码
3. js动画对于低版本的浏览器，如果需要降级处理则需要额外写代码

#### css动画的优点
1. CSS动画写起来更简单
2. CSS动画由浏览器控制，在碰到低版本浏览器的时候，可以自动降级处理
3. CSS动画的性能良好，渲染引擎对CSS动画可以自动优化

#### css动画的缺点
1. CSS很多复杂的动画无法实现，比如滚动动画
2. CSS动画由浏览器控制，开发者对动画的控制不够灵活
3. CSS动画由于使用了一些比较新的属性，可能会有兼容性问题


#### 总结
1. 当动画的切换状态比较简单时，关键帧比较少的时候，可以采用CSS动画实现
2. 当动画需要控制的地方很多的时候，CSS就无法实现了，此时应该使用JS动画来实现
3. 当需要手动协调动画的时候，可以使用requestAnimationFrame来优化动画


### 参考文章
1. [CSS动画：animation、transition、transform、translate傻傻分不清](https://juejin.im/post/5b137e6e51882513ac201dfb)
2. [css3动画属性详解之transform、transition、animation](https://segmentfault.com/a/1190000004460780)
3. [MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Animations/Using_CSS_animations)













