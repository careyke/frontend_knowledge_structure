# 实现一个可以用for...of遍历的对象

普通对象因为没有迭代器，所以无法使用for...of遍历，一般使用for...in或者Object.keys()来遍历

但是如果我们手动给对象设置一个迭代器，对象也是可以使用for...of来遍历的

```js
// 实现一个对象迭代器
Object.prototype[Symbol.iterator] = function* objectIterator(){
  for(let key in this){
    if(obj.hasOwnProperty(key)){
      console.log('iterator')
      yield [key,this[key]];
    }
  }
}
var obj = {
  a:1,
  b:2,
  c:3
}
for(let v of obj){
  console.log(v);
}
//['a',1]
//['b',2]
//['c',3]
```

