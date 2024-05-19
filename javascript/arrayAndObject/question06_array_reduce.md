# 手动实现一个reduce方法

## 1. reduce方法的特点

1. this 不能为undefined或者null
2. this是一个对象，如果不是对象，则会先进行**ToObject()**操作
3. 第一个参数是必须是函数；**第二个函数是累加器的初始值，如果不传的话则取数组的第一个非空值（非empty）**
4. 返回的是累加器的值，遍历数组的过程中会**跳过空值**。累加器的值会传给参数函数当做第一个参数
5. **遍历的时候，也会查找原型链上的属性**



## 2. 手写一个myReduce方法

```js
Array.prototype.myReduce=function(fn, initValue){
  if(this == null){
    throw TypeError('myReduce called on null or undefined');
  }
  if(typeof fn !== 'function'){
    throw TypeError(`${fn} is not a function`);
  }
  
  let thisObj = Object(this);
  let len = 0;
  if(Number(thisObj.length) && Number(thisObj.length)>0){
    len = Number(thisObj.length);
  }
  let index = 0; //索引值记录在外面，如果初始值没有的时候，需要取数组中的第一个非空值
  
  // 设置累加器的初始值
  let result = initValue;
  if(result === undefined){
    for(; index<len; index++){
      if(index in thisObj){
        result = thisObj[index];
        index++;
        break;
      }
    }
  }
  
  for(;index < len; index++){
    if(index in thisObj){
      result = fn.call(undefined, result, thisObj[index], index, thisObj);
    }
  }
  return result;
}



[1,2,3].myReduce((sum,v)=>{return sum+v},4); //10

[1,2,,3,5,,6].myReduce((sum,v)=>{return sum+v},4); //21

var likeArr={
  0:1,
  1:2,
  3:4,
  length:4
}
likeArr.__proto__={2:3}

Array.prototype.myReduce.call(likeArr,(sum,v)=>sum+v, 4); //14
```

