# JS中异步编程方案的演化过程
在JS发展的过程中，尽管对于异步代码的**底层实现机制（eventLoop）**基本上是没有改变的。但是在**异步代码组织**上是发生了很大的变化的。

下面会概括的介绍四种异步方案的基本面貌，后面文章中会做具体分析

## 1.回调函数
在ES6诞生之前，js中异步代码就只有一种组织模式 —— 回调函数。**在异步任务发起的时候，先将任务完成之后的处理代码存在回调函数传给第三方，等到第三方将任务完成之后去调用这个回调函数**。在第三方完成任务的过程中，不阻塞主程序中代码的执行。
```js
fs.readFile('1.json',(err, data)=>{
  ...
})
```
上面代码中就是将读取文件之后的处理函数传给fs模块，等到文件读完之前由fs模块来处理这个回调函数（推到Poll Queue中等待执行）

### 1.1 回调函数的问题
#### 1.控制反转带来的信任问题
由于回调函数方案中，回调函数会被传到第三方，由第三方程序来处理，比如 jQuery.ajax 。这里就发生了**控制反转**，将回调函数的控制权由你自己的程序转移到了第三方程序。

然而**第三方程序并不总是值的信任的**，可能会出现很多的异常情况。比如：提前调用，多次调用或者调用的时候传递错误的参数。

信任危机带来的问题就是：
1. **在回调函数中不得不增加很多异常情况处理的代码**
2. **多个异步任务中，每个任务中的异常情况只能自己单独在回调函数中处理，不能一次性处理**

这样会导致回调函数中充满了if...else语句，非常难维护。
```js
ajax('xxx',(data)=>{
  if(条件1){
    ...
  }else if(条件2){
    ...
  }
  ...
})
```

#### 2.回调地狱
在有一些场景中，多个异步任务相互有数据依赖，这时多个任务必须要**串联执行**。
```js
fs.readFile('1.json',(err,data)=>{
  ...
  fs.readFile('2.json',(err,data)=>{
    ...
    fs.readFile('3.json',(err,data)=>{
      ...
    })
  })
})
```
如上述代码，需要在回调中嵌套回调。如果这样的异步任务很多的时候，嵌套的层次也越来越多，这种现象就叫做**回调地狱**。


## 2. Promise
为了解决回调函数中出现的问题，ES6中提供了Promise机制。Promise就是“承诺”的意义，**是ES6提供是一种可信任的用来管理异步回调函数的机制。**

==Promise会将异步任务转化成一个承诺，该承诺在未来总会有一个结果。如果承诺实现了，会调用实现的回调函数；如果承诺失败了，会调用失败的回调函数==

```js
function readFilePromise(name){
  return new Promise(function(resolve,reject){
    fs.readFile(name,(err,data)=>{
      if(err){
        reject(err);
      }else{
        resolve(data);
      }
    })
  })
}
readFilePromise('1.json').then((data)=>{
  ...
  return readFilePromise('2.json');
}).then((data)=>{
  ...
  return readFilePromise('3.json');
}).then((data)=>{
  ...
}).catch((err)=>{
  ...
})
```
从代码中可以看出：
1. **Promise支持链式调用**，可以有效的避免嵌套使用。比较符合人的线性思维方式
2. **Promise中错误是可以冒泡的，Promise提供了catch方法，用来捕获整个异步任务链中出现的错误。**不需要每个回调函数都有大量臃肿的错误判断逻辑

```javascript
var p = new Promise(function(resolve,reject){
  a = b+1; // a,b 都没有定义，会抛错
  resolve(a);
})
console.log('ok'); 
// ok 后面代码正常执行，并没有因为抛错中断执行

try{
  var p = new Promise(function(resolve,reject){
  	a = b+1; // a,b 都没有定义，会抛错
  	resolve(a);
	});
}catch(e){
  console.log(e);  //不执行这句代码，说明try中没有捕获错误
}
```



### 2.1 为什么说Promise是值的信任的
1. **Promise中then中传入的回调函数都是异步调用的，而且是存在微任务队列中，会在当前tick结束之前调用**。不存在调用过早或过晚的问题
2. **Promise中状态的变化是不可逆的，一旦由pending变成了fulfilled或者rejected，以后都无法改变了**。所以回调函数也只会执行一次，不存在调用多次的问题
3. **Promise本身不会对异步任务的返回值做任何处理**，resolve(...)接收到的值都会直接传给回调函数。所以如果异步任务的返回值是正确的，回调函数接收到的参数也肯定是正确的。

### 2.2 Promise的局限
1. ==Promise内部产生的错误是无法在外面进行捕获的，只能使用catch(...)方法来捕获==
2. Promise是无法中途取消的，也就是说是没有中间状态的

## 3. Generator / CO
**Generator是ES6提供的一种可以被打断执行的函数，返回值是一个迭代器，来控制函数体的执行**

Generator函数配合**CO 模块**可以==用同步代码的方式来组织异步代码，按顺序一个一个执行==。
```js
function* readFileGenerator(){
  yield readFilePromise('1.json');
  yield readFilePromise('2.json');
  yield readFilePromise('3.json');
  yield readFilePromise('4.json');
}

co(readFileGenerator); //返回一个Promise对象
```

## 4. async + await
async和await是ES7中实现的关键字，配合使用。使用形式和Generator中的 `*` 和yield关键字差不多。也可以用同步代码的方式来书写异步代码

**async + await就是 Generator+CO 的语法糖，由js内部实现**。

1. 凡是有async关键字的函数返回值都是Promise对象
2. await不能单独使用，必须使用在async函数内部
3. await后面可以是Promise对象和原始类型的值（原型类型会自动转化成Promise对象）

```js
async function readFile(){
  await readFilePromise('1.json');
  await readFilePromise('2.json');
  await readFilePromise('3.json');
  await readFilePromise('4.json');
}
```


