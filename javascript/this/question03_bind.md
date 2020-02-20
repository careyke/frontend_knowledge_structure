## 手写一个bind方法
bind方法是用来显示修改函数this值的方法。也可以用来实现函数柯里化

### bind方法的特点
1. bind返回一个新的函数
2. bind方法可以修改函数的this，并且可以预先为函数传递参数
3. new操作符修改this的优先级高于bind
4. **bind返回函数的原型链( ______proto______ )和原型对象(prototype)要继承原始函数**

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
    // 处理new操作符
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
```  












