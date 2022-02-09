## 手写instanceof操作符
### 1.Symbol.hasInstance
Object中有一个Symbol.hasInstance属性，指向一个内部方法。**当其他对象使用instanceof运算符，判断是否为该对象的实例时，会调用这个方法。**

foo instanceof Foo在语言内部，实际调用的是Foo[Symbol.hasInstance] (foo)

### 2.instanceof的规则
1. A instanceof B判断的原理是：**判断在 A 的原型链上是否有 B 的原型对象**。
2. B一定要是对象类型或者构造函数，否则报错
3. A可以是任意类型。但是如果A是基本类型，会直接返回false，因为基本类型没有原型链。


### 3.代码实现
```js
function myInstanceof(left,right){
  //右边数据处理
  if(!(right && (typeof right === 'object' || typeof right === 'function'))){
    throw new TypeError('right is not an object');
  }
  //左边数据处理
  if(!(left && (typeof left === 'object' || typeof left === 'function'))) return false;
  
  let proto = Object.getPrototypeOf(left);
  const rightPrototype = right.prototype;
  while(proto){
    if(proto === rightPrototype){
      return true;
    }else{
      proto = Object.getPrototypeOf(proto);
    }
  }
  return false;
}


myInstanceof(1,Number); // false
```