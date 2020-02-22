## 实现一个new方法
### 1. new操作符的特性
1. 创建一个新的对象
2. 这个新对象会被执行[[Prototype]]连接，**关联构造函数的原型对象**
3. 这个新对象会**绑定到函数调用的this**
4. 如果**函数没有返回其他对象**，那么new表达式中的函数调用会自动返回这个新对象


### 2.实现一个myNew方法，模拟new

```js
function myNew(constructor,...args){
  if(typeof constructor !== 'function'){
    throw TypeError(`${constructor} is not a constructor`);
  }
  const object=Object.create(constructor.prototype);
  const result =  constructor.apply(object, args);
  if(result && (typeof result === 'object' || typeof result === 'function')){
    return result;
  }
  return object;
}

```  
