# 深入理解Generator及其在异步方案中的作用

之前也介绍过，ES6中还提供了一种异步解决方案 —— Generator + CO。Generator是ES6引入的一种新的函数，和传统的函数有很大的不同。

## 1. Generator函数的特点

### 1.1 Generator函数的执行过程

和传统的函数不一样，Generator函数并不是通过 `func()` 的形式调用之后就可以同步执行函数体的，Generator函数的执行有以下特点：

1. Generator函数调用之后，并不会立即执行函数体，而是返回的是一个**迭代器对象**
2. Generator函数体的执行需要依赖这个迭代器对象，迭代器中存在一个next()方法，用来开始执行函数体。
3. Generator函数体的**执行过程是可以打断和恢复**，通过 yield 关键字就可以打断执行，然后通过迭代器的next方法恢复执行

```js
function* gen(){
  yield 'a';
  yield 'b';
  yield 'c';
  return 'd';
}
var g = gen();
g.next(); // {value: 'a',done: false}
g.next(); // {value: 'b',done: false}
g.next(); // {value: 'c',done: false}
g.next(); // {value: 'd',done: false}
```

### 1.2 next()的返回值和yield语句的返回值

迭代器的next(...)方法的返回值是一个对象：`{value: ,done:boolean}`

1. done属性表示的是这个函数体是否执行完，遇到return语句或者执行到函数体最后，则表示执行完成，done为true；否则是false
2. value表示的是当前执行片段的返回值，**通常是yield或者return关键字后面表达式的值**。当done为true的时候，再执行next()方法返回值的value为undefined

```js
function* gen(){
  yield 'a';
  return 'b';
}
var g = gen();
g.next(); // {value:'a',done:false}
g.next(); // {value:'b',done:true} // 此时函数体已经执行完
g.next(); // {value: undefined, done:true}
```

**yield语句的返回值指的是整个yield语句的返回值，而不是yield关键字后面语句的返回值。**

**yield语句的返回值由下一个next()方法的参数来决定，next()方法的参数就是上一个yield语句的返回值。**第一个next方法的参数是没有作用的。

```js
function* gen(){
  var a = yield 1;
  console.log(a);
  var b = yield 2;
  console.log(b);
  return 3;
}
var g = gen();
g.next(); // {value:1,done:false}
g.next('aaa'); // {value:2,done:false}
// aaa
g.next('bbb'); // {value:3,done:true}
// bbb
```

### 1.3 Generator函数的异常捕获

Generator函数内部出错的时候，和正常的函数一样，使用try...catch语句来捕获错误。

1. 函数体内部错误可以在内部捕获也可以在外部捕获
2. **内部出错的时候，如果没有在内部捕获，即使外部捕获，Generator函数体会终止执行。**之后再执行next方法都返回{value:undefined, done: true}

```js
function* gen(){
  var a = yield 1;
  throw 'err';
  var b = yield 2;
  return 3;
}
var g = gen();
g.next(); // {value:1,done:false}
try{
  
}catch(err){
  
}
g.next(); // err
g.next(); // {value:undefined, done:true}

function* gen(){
  var a = yield 1;
  throw 'err';
  var b = yield 2;
  return 3;
}
var g = gen();
g.next(); // {value:1,done:false}
try{
  g.next();
}catch(err){
  console.log(err); //err
}
g.next(); //{value:undefined, done:true} 因为没有在内部捕获
```

Generator的迭代器对象中提供了一个throw()方法，用来**在函数体外抛出错误，在函数体内捕获。**

1. throw方法抛出的错误可以在函数体内捕获，相当于**将上一个yield语句替换成一个抛出异常的语句**。如果内部没有捕获，则函数体的执行提前结束。
2. throw()抛出的错误也可以在函数体外捕获
3. **throw()语句也会开启下一个函数体片段的执行**，但是无法第一个就使用throw开始函数体执行

```js
function* gen(){
  var a = yield 1;
  var b = yield 2;
  return 3;
}
var g = gen();
g.next(); //{value:1,done:false}
g.throw('error'); // error
g.next(); // {value:undefined, done:true} 提前终止


function* gen(){
  try{
    var a = yield 1;
  }catch(err){
  	console.log(err); 
  }
  var b = yield 2;
  return 3;
}
var g = gen();
g.next(); //{value:1,done:false}
g.throw('error'); // {value: 2, done:false} 开启下一个片段的执行
g.next(); // {value:3, done:true}
```

### 1.4 迭代器的next()、throw()和return()方法之间的差别

1. 这三个方法都能恢复下一个片段的执行。但是只有next()能在最开始开启函数体的执行。

2. 这三个方法都可以将上一个yield表达式替换成一个其他的语句。

   - next()将 yield语句**替换成一个值**

     ```js
     function* gen(){
       var a = yield 1;
       return 3;
     }
     var g = gen();
     g.next();
     g.next(20);
     //相当于将 var a = yield 1;
     //替换成 var a = 20;
     ```

   - throw()将 yield语句**替换成一个抛出错误的throw语句**

     ```js
     function* gen(){
       var a = yield 1;
       return 3;
     }
     var g = gen();
     g.next();
     g.throw('error');
     //相当于将 var a = yield 1;
     //替换成 throw 'error'
     ```

   - return()将 yield语句**替换成一个return语句**

     ```js
     function* gen(){
       var a = yield 1;
       return 3;
     }
     var g = gen();
     g.next();
     g.return('error');
     //相当于将 var a = yield 1;
     //替换成 return 'error'
     ```

### 1.5 yield* 语句

在一个Generator中调用另一个Generator函数时，使用yield* 语句会非常的方便。`yield*`会**自动去迭代这个迭代器对象**。就是for...of语句的简写。但是yield*只能用在Generator函数体中

```js
function* gen1(){
  var a = yield 1;
  var b = yield 2;
  return 3;
}
function* gen2(){
  var c = yield 'a';
  var d = yield* gen1();
  console.log(d);
  var e = yield 'b';
  return 'c';
}
var g = gen2();
g.next(); //{value: 'a',done:false}
g.next(); //{value: 1,done:false}
g.next(); //{value: 2,done:false}
g.next(); //{value: 'b',done:false} 这个返回值并不是{value:3,done:false}
// 3
g.next(); //{value: 'c',done:true}
```

从上面代码可以看出，对于yield*语句来说：

==Generator函数中的return语句并不是一个状态，也不是暂停函数的标志，就是yield*语句的返回值==

### 1.6 Generator函数与协程

协程（coroutine）是一种程序的运行方式。可以理解为“协作的线程”或者“协作的函数”。协程既可以使用单线程实现，也可以使用多线程实现。**单线程实现的时候，协程是一个特殊的子例程**；多线程实现的时候，协程就是一个特殊的线程。

**传统的子例程只有一个入口而且只能返回一次，可以理解成就是一个函数。但是协程可以有多个入口，而且可以在指定位置挂起和恢复执行。**

在JS中，**一个线程可以有多个协程。同一时间只能执行其中一个协程**。每个协程都有自己的执行上下文。

==Generator函数就是ES6对于协程的实现，可以将Generator函数理解成一个协程。使用yield关键字来让出线程控制权，使用next方法来重新获取线程控制权。==

```js
function B(){
  console.log('这里是B协程');
  return 300;
}
function* A(){
  console.log('这里是A协程');
  yield B();
  console.log('A协程结束');
}
var g = A();
g.next();
g.next();
//这里是A协程
//这里是B协程
//A协程结束
```

==可以将JS主程序理解成一个协程，这里Generator函数A是另一个协程。两个协程之间通过yield和next()来不断切换对于线程的控制权。==

**协程 VS 多线程**

1. 协程之间的切换由程序自身控制，是子程序之前的切换。切换的开销远远小于线程之间的切换
2. 协程不需要锁，因为都是同一个线程中，同一时间只执行一个协程。不存在变量冲突

##  2. Generator函数实现的异步方案

所谓异步指的是将来某个之间执行的代码，而Generator函数中，多个片段的代码都可以在将来才执行。这两者之间是不谋而合的。所以自然而然就是有使用Generator函数实现的异步方案。

使用Generator还有一个好处就是可以用同步的方式来组织异步的代码。

**Generator实现异步调用最关键的一步就是需要在异步任务结束之后，需要能够调用next()方法使Generator函数能够继续执行，执行异步任务的处理代码。**（回调函数中转移控制权）

实现这一步有两种方案：

1. Thunk函数
2. Promise

### 2.1 Generator异步方案 —— Thunk函数（偏函数）

什么是Thunk函数？

**Thunk函数的核心逻辑就是接收一定的参数，然后一个定制化的函数。**然后使用这个定制化的函数完成一些功能。**多个将原本多个参数的函数转换成单一参数的函数，而且可以保存中间状态，有利于复用。**

```js
//非Thunk版本
const isType=(type,data)=>{
  return Object.prototype.toString.call(data) === `[object ${type}]`;
}
isType('Number',123); //true

//Thunk版本
const isType = (type)=>{
  return (data)=>{
    return Object.prototype.toString.call(data) === `[object ${type}]`;
  }
}
var isNumber = isType('Number');
isNumber(123); // true
```

**Generator + Thunk的异步实现**

Thunk函数的返回值是一个函数，可以**将异步操作封装成一个Thunk函数**。利用返回值函数可以将Generator函数恢复执行

```js
function readFileThunk(name){
  return (callback)=>{
    fs.readFile(name,callback);
  }
}

function* gen(){
	var data1 = yield readFileThunk('1.json');
  var data2 = yield readFileThunk('2.json');
}
var g = gen();
g.next().value((err, data)=>{
  	console.log(data); // 1.json的内容
  	g.next(data).value((err, data)=>{
    	console.log(data);// 2.json的内容
      g.next(data);
    })
})
```

上面代码实现了Generator的异步调用，但是Generator函数的执行过程比较复杂，可以进一步优化后成一个自执行的函数。

**Thunk+自执行**

```js
function autoRunThunk(g){
  const next=(err, data)=>{
    var result = g.next();
    if(result.done) return ;
    result.value(next);
  }
  next();
}

//上面代码的执行过程可以修改为：
autoRunThunk(g);
```

### 2.2 Generator异步方案 —— Promise实现

**Generator + Promise的异步实现**

在Promise的then方法的回调函数中转移控制线程的权利，使generator恢复执行

```js
function readFilePromise(name){
  return new Promise(function(resolve,reject){
    fs.readFile(name,(err,data)=>{
      if(err) reject(err);
      resolve(data);
    })
  })
}
function* gen(){
	var data1 = yield readFilePromise('1.json');
  var data2 = yield readFilePromise('2.json');
}
var g = gen();
g.next().value.then((data)=>{
  g.next(data).value.then((data)=>{
    g.next(data);
  })
})
```

**Promise + 自执行**

```js
function autoRunPromise(g){
  const next=(data)=>{
    const result = g.next(data);
    if(result.done) return;
    result.value.then(next);
  }
  next();
}

//上面代码的执行过程可以修改为：
autoRunPromise(g);
```

### 2.3 Generator + CO模块

CO模块是Generator的自动执行器，其内部的实现原理其实就是前面介绍的两种方法。CO将他们封装成了一个模块。

```js
const co = require('co');
function readFilePromise(name){
  return new Promise(function(resolve,reject){
    fs.readFile(name,(err,data)=>{
      if(err) reject(err);
      resolve(data);
    })
  })
}
function* gen(){
	var data1 = yield readFilePromise('1.json');
  var data2 = yield readFilePromise('2.json');
}

co(gen()).then((res)=>{
  console.log(res); //这里的res指的是gen函数的返回值，是undefined
})
```

**CO模块的特点：**

1. 返回值是一个Promise对象
2. **yield关键字后面的语句只能是Thunk函数或者Promise对象。如果是其他类型的值，会先转化成Promise对象**
3. CO可以自动执行Generator函数

### 2.4 总结

**Generator异步方案的好处：可以将异步代码按照同步代码的方式组织。**

但是自从ES7提供了async+await之后，基本就不会使用这种方案了。之所以深入的理解也是为了了解async+await的实现机制 —— 也是基于Generator实现的。