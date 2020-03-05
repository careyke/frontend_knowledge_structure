# 手动实现一个filter方法

## 1. filter方法的特点

1. this 不能为undefined或者null
2. this是一个对象，如果不是对象，则会先进行**ToObject()**操作
3. 第一个参数一定要是函数，第二个参数是参数函数的this，可不传
4. 返回一个数组，**空值会跳过不处理**，使参数函数的返回值为真值的数组项被添加到返回数组中
5. **遍历的时候，也会查找原型链上的属性**

## 2. 手动实现一个myFilter函数

```js
Array.prototype.myFilter = function(fn, context){
  if(this == null){
    throw TypeError('myFilter called on null or undefined');
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
      let v = thisObj[i];
      if(fn.call(context, thisObj[i], i, thisObj)){
        result.push(v); // 这一步会跳过空值
      }
    }
  }
  return result;
}


[1,2,3].myFilter((v)=>1); // [1,2,3]
[1,2,,3].myFilter((v)=>1); // [1,2,3]

var likeArr={
  0:1,
  1:2,
  3:4,
  length:4
}
likeArr.__proto__={2:3}

Array.prototype.myFilter.call(likeArr,v=>1); //[1,2,3,4]
```

