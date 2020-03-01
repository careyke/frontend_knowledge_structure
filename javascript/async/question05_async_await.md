# 深入理解async+await

async+await是ES7提供的异步的终极解决方案。**其底层也是通过协程来实现的，而且asyn函数内部实现了一个自执行器。**和Generator函数+CO模块的原理是相同，可以认为是Generator+co的语法糖。

上一篇文章深入理解Generator就是为了来理解async+await

## 1. async+await的特点

1. async函数也是一个协程，执行的过程中也是可以暂停和恢复的。await和Generator中的yield关键字一样，是协程交出线程控制权的标志
2. async函数内部实现了一个自执行器，可以和普通函数一样调用执行
3. **await关键字后面只能是Promise对象，如果不是Promise对象，内部会自动转化成Promise对象**
4. **await语句的返回值，就是后面Promise对象的决议值。**（类似于Generator中在then的回到函数中调用next方法，传入异步任务的结果）
5. **async函数返回一个Promsie对象**
   - **当async函数内部没有错误的时候，函数体执行完成Promise才算完成，Promise的状态就是fulfilled，决议值就是async函数的返回值（类似于co）**
   - **当async函数内部抛错而且没有在内部捕获的时候，Promise直接完成，状态为rejected，决议值就是抛出的这个异常。**

以上的这个特点，都可以类比Generator+CO模块来理解。

```js
var p1 = new Promise(function(resolve,reject){
  setTimeout(()=>{
    resolve(123);
  },2000)
})
var p2 = new Promise(function(resolve,reject){
  setTimeout(()=>{
    reject('error');
  },1000)
})


// 例子1
async function foo(){
  var a = await p1;
  console.log(a);
  var b = await 234;
  console.log(b);
  return 'ok';
}
var p = foo(); //直接执行
p.then((data)=>{console.log(data)});
//123 -> 234 -> 'ok'


// 例子二
async function foo(){
  var a = await p1;
  console.log(a);
  var b = await p2;
  console.log(b);
  return 'ok';
}
var p = foo(); 
p.then((data)=>{console.log(data)}).catch((err)=>{console.log(err)});
// 123 -> 'error'

// 例子三
async function foo(){
  try{
    var a = await p1;
  	console.log(a);
  	var b = await p2;
  	console.log(b);
  }catch(err){
    console.log(err);
  }
  return 'ok';
}
var p = foo(); 
p.then((data)=>{console.log(data)}).catch((err)=>{console.log(err)});
// 123 -> 'error' -> 'ok'
```

## 2. async+await的异常处理机制

由于async+await内部是基于Generator实现的，所以其异常处理也是基于Generator的异常处理进行封装的。

1. **由于async函数的返回值是一个Promise对象，所以async函数内部的错误也是无法在外面使用`try...catch`捕获的，因为已经被Promise捕获反应在了状态上。可以使用Promise的catch方法捕获**

   ```js
   async function foo(){
     throw 'error';
   }
   try{
     foo();
   }catch(e){
     console.log(e); //捕获不到错误
   }
   
   
   foo().catch((err)=>{console.log(err)}); // error
   ```

2. async函数内部的错误如果没有在内部捕获的话，会中断函数体的执行。（和Generator函数一样）

   ```js
   async function foo(){
   	throw 'error';
     console.log('ok');  // 不会执行输出
     await Promise.resolve(123);
   }
   foo().catch((err)=>{console.log(err)}); // error
   ```

3. **await语句后面的Promise如果有异常的话，在函数体上也可以捕获到，也能中断函数体的执行**。这个和Generator函数是不一样，==async中给这个Promise附加了一些操作，在Promise后面增加一个catch语句捕获住错误，然后通过throw方法抛到async函数体内。==

   ```js
   async function foo(){
   	await Promise.reject('error');
     console.log('ok');  // 不会执行输出
     await Promise.resolve(123);
   }
   foo().catch((err)=>{console.log(err)}); // error
   
   Promise.reject('error');
   //类似于
   Promise.reject('error').catch((err)=>{
     [遍历器].throw(err);  // 内部遍历器的throw方法
   })
   ```

### 2.1 async函数中如何优雅的捕获错误

由于在await后面的异步任务抛错，会导致整个async函数终止执行,这是一种很糟糕的情况。但是每个异步任务都可能会失败的，我们不得不在每个异步任务的外面包裹一个try...catch来捕获错误，防止其他异步任务无法执行

```js
async function foo(){
  try{
    var a = Promise.resolve(1);
  }catch(err){
    console.log(err)
  }
  try{
    var b = Promise.reject('error');
  }catch(err){
    console.log(err)
  }
  try{
    var c = Promise.reject('err');
  }catch(err){
    console.log(err)
  }
}
foo();
```

这种情况下才能保证每个异步任务都能执行。但是这种方式太繁琐，代码冗余难以维护。还有另外一种方法就是在**每个异步任务后面加一个catch方法，将错误捕获住，但是还得要能判断当前任务是成功还是失败。**

```js
function awiatWrap(value){
  return Promise.resolve(value).then((data)=>{
    return [undefined, data];
  }).catch((err)=>{
    return [err, undefined]
  })
}

async function foo(){
  var [err,dataA] = awiatWrap(Promise.resolve(1));
  var [err,dataB] = awiatWrap(Promise.reject('error'));
  var [err,dataC] = awiatWrap(Promise.reject('err'));
}
foo();
```

使用这种方式可以很优雅的解决错误中断执行的问题，而且还能根据值判断任务的真正状态

## 3. async+await的执行顺序问题

一道经典的考题，考察eventLoop

```js
console.log('script start')

async function async1() {
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2 end')
}
async1()

setTimeout(function() {
  console.log('setTimeout')
}, 0)

new Promise(resolve => {
  console.log('Promise')
  resolve()
})
  .then(function() {
    console.log('promise1')
  })
  .then(function() {
    console.log('promise2')
  })

console.log('script end')
```

执行顺序

```js
1. 'script start'
2. 'async2 end'
3. 'Promise'
4. 'script end'
5. 'async1 end'
6. 'promise1'
7. 'promise2'
8. 'setTimeout'
```

但是在低版本的浏览器和node中，执行顺序有一点偏差，但是现在已经统一