## 实现柯里化函数
### 柯里化（currying）的含义
柯里化是一种降解多参数函数的方法：**把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数。**

通俗理解：
**Currying —— 只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数。**

优点：
1. 柯里化函数可以用来**重用函数参数，保存中间状态**。


### 柯里化函数实现
实现一个柯里化函数myCurry，使得下面表达式成立：
```js
function add(a,b,c,d){
  return a+b+c+d;
}
const myAdd = myCurry(add,1,2);
myAdd(3,4); //10
myAdd()(3,4); //10
myAdd(3)(4); //10
```  
实现myCurry函数：
```js
function myCurry(fn,...params){
  const len = fn.length;  //获取函数形参个数
  function currying(...args){
    const newParams = params.concat(args)
    if(newParams.length < len){
      //收集已经传递的参数只能每次调用重新传进去，如果放在params上统一管理，会出现问题
      return myCurry.call(this,fn,...newParams);  
    }else{
      return fn.apply(this,newParams);
    }
  }
  return currying;
}
```  

