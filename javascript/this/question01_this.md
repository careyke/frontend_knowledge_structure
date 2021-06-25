# 谈谈this机制？
## 1.this是什么？
在前面执行上下文的文章中，我们就已经分析了：**this是执行上下文中的一个属性，用来存储代码执行需要的一些数据。**

**由于函数的执行上下文是动态创建的，所以this也是函数执行之前动态确定的。**
```js
var a = 1;
var obj = {
  a:2,
  fn:function foo(){
    console.log(this.a)
  }
}
obj.fn();  //2

var foo = obj.fn;
foo();  // 1
```
可以看出，函数不同的调用方式，其中的this是不同的。



## 2.this值的绑定规则

**函数的this值是函数执行之前，创建执行上下文的时候绑定的**，所以和函数的**调用位置**和**调用方式**有关。和函数声明的位置是无关的。

this值绑定的规则主要有4条

### 2.1 默认绑定
**当函数直接使用标识符调用，没有任何的修饰信息时，函数中的this绑定会采用默认规则。即将全局对象window（浏览器）绑定在this上。**

```js
var a = 2;
function foo(){
  console.log(this.a); // a是window的属性
}
foo(); // 2
```

**如果函数是严格模式下执行(函数自身代码处于严格模式下)，则不能将全局对象绑定到this上，那么this就是undefined。**
```js
var a = 2;
function foo(){
  "use strict";
  console.log(this.a); // a是window的属性
}
foo(); // TypeError: Cannot read property 'a' of undefined
```
**严格模式下调用 foo() 则不影响默认绑定**
```js
var a = 2;
function foo(){
  console.log(this.a); // a是window的属性
}
(function fn(){
  "use strict";
  foo(); // 2
})()
```



### 2.2 隐式绑定

**当函数通过某个对象属性的形式来调用的时候，采用的就是隐式规则。函数中的this绑定的是这个对象。**

```js
var a = 2；
function fn(){
  console.log(this.a);
}
var obj={
  foo:fn,
  a: 3
}
obj.fn();  //3
```



### 2.3 显示绑定

**当函数使用bind、call或者apply方法的时候，采用的是显示绑定。**
```js
var a = 2;
var obj={
  a:3
}
function fn(){
  console.log(this.a);
}
var foo1 = fn.bind(obj);
foo1(); // 3;

fn.call(obj); //3

fn.apply(obj); //3
```
关于bind、call和apply方法之间的区别会在后面文章解释

#### 2.3.1 当显示绑定的值是null或undefined时
```js
var a = 2;
var obj={
  a:3
}
function fn(){
  console.log(this.a);
}
fn.call(obj); //3

fn.call(null); //2

fn.call(undefined); //2
```
可以看出，**当显示绑定传入的值是null或者undefined的时候，函数this采用的是默认的绑定规则，即绑定全局对象window**



### 2.4 new操作符绑定

**在js中，new操作符通常是用来实例化类的。但是其实本质上就是通过new操作符来调用函数，其中也包含着this的绑定操作。**

new操作符内部具体的操作步骤，后面会有专门文章解释。

```js
function Foo(n){
  this.a = n;
}
var f = new Foo(3);
console.log(f); // {a:3} 修改了this

console.log(window.a); // undefined
```



## 绑定规则的优先级

**从高到低**排序：

1. new操作符
2. 显示绑定
3. 隐式绑定
4. 默认绑定









