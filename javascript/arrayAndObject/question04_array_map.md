## 手动实现一个数组map方法

## 1.map方法的特点

1. map方法的 this 不能为undefined或者null
2. map方法中的this是一个对象，如果不是对象，则会先进行**ToObject()**操作
3. map方法的第一个参数一定要是函数，第二个参数是参数函数的this，可不传
4. map方法返回一个参数函数返回值组成的数组，**不会跳过空值，空值直接返回**
5. **map方法遍历的时候，也会查找原型链上的属性**



## 2 手动实现一个myMap()方法

```js
Array.prototype.myMap = function(fn, context){
  if(this == null){
    throw TypeError('myMap called on null or undefined');
  }
  if(typeof fn !== 'function'){
    throw TypeError(`${fn} is not a function`);
  }
  
  const thisObj = Object(this);
  const result=[];
  let len = 0;
  if(Number(thisObj.length) && Number(thisObj.length) > 0){
    len = Number(thisObj.length);
  }
  for(let i = 0;i<len;i++){
    if(i in thisObj){ // in 操作符会查找原型链
      const v = fn.call(context, thisObj[i], i, thisObj);
      result[i] = v;  // 这一步操作保证不跳过空值
    }
  }
  return result;
}



[1,2,3].map(v=>2*v); // [2,4,6]
[1,2,,3].map(v=>2*v); // [2, 4, empty, 6]

var likeArr={
  0:1,
  1:2,
  3:4,
  length:4
}
likeArr.__proto__={2:3}

Array.prototype.myMap.call(likeArr,v=>2*v); //[2,4,6,8]
```

