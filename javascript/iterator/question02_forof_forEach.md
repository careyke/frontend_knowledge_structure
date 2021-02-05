# for...of循环和forEach之间的区别

最开始遍历数组的方式是通过for语句来实现的。但是这种方式遍历的是数组的下标，然后通过下标取到数组的值。

但是在很多场景中，我们并不关心数组的下标，只关注数组项的值。所以ES5中提供了高阶函数forEach来遍历数组，直接获取数组项的值。ES6中提供for..of语句也可以实现这种需求。

下面来比较两种方式的差别

## 1. 实现原理

### 1.1 forEach实现原理

实现一个forEach，窥探其底层的实现逻辑

```js
Array.prototype.myForEach = function (fn, context) {
  const len = this.length;
  for (let i = 0; i < len; i++) {
    fn.call(context, this[i], i, this); //直接调用这个回调函数
  }
}
```

可以看到，forEach就是相当于多次调用fn，分别传入不同的参数进去而已

### 1.2 for...of实现原理

之前的文章分析过，for...of底层是靠调用遍历器来实现的。

## 2. 两个方法之间的区别

### 2.1 forEach无法中断循环，for...of可以

forEach无法像for语句一样使用break，continue，return等关键字来终止循环

```js
var arr=[1,2,3];
arr.forEach((v)=>{
  console.log(v);
  if(v>1) return;  //break continue关键字会报错
})
//1
//2
//3
```

for...of 语句是可以用的

```js
var arr=[1,2,3];
for(let v of arr){
  console.log(v);
  if(v>1) break; 
}
//1
//2
//continue语句也能正常使用
```

### 2.2 forEach和for...of在异步任务中的表现

使用forEach执行多个异步任务的时候，无法保证异步任务的顺序。

```js
function handler(n){
  return new Promise(function(resolve, reject){
    setTimeout(()=>{
      resolve(n);
    },n*1000)
  })
}
var arr=[3,2,1];
async function test(){
  arr.forEach(async(v)=>{
    const a = await handler(v);
    console.log(a);
  })
}
test();
//期望的输出 3 2 1
//实际的输出 1 2 3
```

由forEach的实现原理可知，其内部是直接调用那个三次回调函数，**三个异步任务分散在三个函数中**，相当于并发三个异步任务，当然是谁先做完谁先打印出来。

使用for...of可以保证异步任务的顺序

```js
function handler(n){
  return new Promise(function(resolve, reject){
    setTimeout(()=>{
      resolve(n);
    },n*1000)
  })
}
var arr=[3,2,1];
async function test(){
  for(let v of arr){
    const a = await handler(v);
    console.log(a);
  }
}
test();
//实际输出： 3 2 1 符合期望
```

因为for...of底层是一直在调用重复调用遍历器的next方法，像while语句一样，循环体并没有被函数作用域包裹。所以**三个异步任务是在同一个函数中**的。

相当于：

```js
var arr=[3,2,1];
async function test(){
  const i = arr[Symbol.iterator]();
  let res = i.next();
  while(!res.done){
    let v = res.value;
    const a = await handler(v);
    console.log(a);
    res = i.next();
  }
}
test();
```

> 之所以forEach不能保证异步任务的顺序，就是因为多个异步不在一个函数中，也就是多个异步任务不在一个协程中，而在多个协程中。
>

使用传统的for语句也是可以保证异步任务的顺序的，因为多个异步任务在一个协程中。

```js
var arr=[3,2,1];
async function test(){
  for(let i=0;i<arr.length;i++){
  	const a = await handler(arr[i]);
    console.log(a);
  }
}
test();
```

