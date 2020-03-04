# 手动实现一个call和apply方法
都是显示绑定函数this值的方法，区别是在参数的形式：
- call调用时，原始函数的参数需要**一个个**的传进去
- apply调用时，原始函数的参数组成一个**数组或者类数组对象**传过去，**内部有一个`CreateListFromArrayLike` 方法来将参数转化成一个数组**。如果传入的是其他对象，转化之后得到的就是`[]`；如果是基本类型，就会报错

## 实现一个myCall
```js
// 使用隐式绑定的方式传入this

Function.prototype.myCall = function(context, ...args){
  if(typeof this !== 'function'){
    throw new Error('only function has call');
  }
  if(context == null){
    context = window;
  }
  const symbolKey = Symbol('fn');
  context[symbolKey] = this;
  const result = context[symbolKey](...args);
  delete context[symbolKey];
  return result;
}
```

## 实现一个myApply

```js
// 使用隐式绑定的方式传入this

Function.prototype.myApply = function(context, args){
  if(typeof this !== 'function'){
    throw new Error('only function has apply');
  }
  if(!Array.isArray(args)){ //这里简单处理，只判断是否是数组
    throw new Error('the arguments of apply is array');  
  }
  if(context == null){
    context = window;
  }
  const symbolKey = Symbol('fn');
  context[symbolKey] = this;
  const result = context[symbolKey](...args);
  delete context[symbolKey];
  return result;
}
```


