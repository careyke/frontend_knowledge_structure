## 手写一个bind方法
bind方法是用来显示修改函数this值的方法。也可以用来实现函数柯里化

### bind方法的特点
1. bind返回一个新的函数
2. bind方法可以修改函数的this，并且可以预先为函数传递参数
3. 当thisArg为null或者undefined时，新函数this为全局对象window
4. **new操作符修改this的优先级高于bind**
5. **bind返回函数的原型链( ______proto______ )和原型对象(prototype)要继承原始函数**

```js
function Foo(){
  this.a = 1;
}
Foo.prototype = {
  b:2
}
function protoFunc(){};
protoFunc.c = 3;
Object.setPrototypeOf(Foo,protoFunc);
var obj = {d:4}
var fn = Foo.bind(obj);

fn.__proto__ === Foo.__proto__; // true
fn.prototype === Foo.prototype; // false
var v = new fn();
v.b // 2 说明Foo.prototype在fn.prototype的原型链上

v.a // 1
obj.a //undefined 说明new的优先级高于bind
```



### 实现一个myBind

```js
Function.prototype.myBind = function(context, ...params){
  if(typeof this !== 'function'){
    throw new Error('only function has bind');
  }
  // 当context为undefined或null时，this取默认的window
  if(context == null){
    context = window;
  }
  const fn = this;
  function bind(...args){
    // 处理new操作符，这一步很关键，需要结合new操作符的内部实现来理解
    return fn.apply(this instanceof bind ? this:context, params.concat(args));
  }
  //设置__proto__
  const proto = Object.getPrototypeOf(this);
  Object.setPrototypeOf(bind, proto);
  
  //设置prototype
  bind.prototype = Object.create(this.prototype);
  return bind;
}
```



### 利用bind实现柯里化函数

```js
function curryByBind(fn,...params){
  const len = fn.length;
  function curry(...args){
    if(args.length < len){
      return curry.bind(this,...args); // 绑定以传入的参数
    }else{
      return fn.apply(this, args);
    }
  }
  return curry.bind(this,...params);
}
// 这个方案有一个问题是this取的是curryByBind的this，而不是调用curry的this，this值可能会不正确
```



### 一个经典的问题：一个函数可以链式bind多次吗？

```js
function foo(){
  console.log(this.a);
}

var obj1={a:1};
var obj2={a:2};

const foo1 = foo.bind(obj1);
foo1(); // 1

const foo2 = foo1.bind(obj2);
foo2(); // 1
```

从执行结果可以看出，**对于bind之后的函数再执行bind是不生效的**。

结合上面bing的实现代码可以很容易分析其中的原理。



