# 深入理解迭代器和Generator

**迭代器就是一种遍历对象数据的机制，任何数据结构只要有了迭代器属性，就可以利用这个迭代器来遍历这种数据结构的内部数据。**

一般来说，迭代器是以数据结构为单位来统一定义的，也就是说一般迭代器属性都是定义在构造函数的原型对象上，该类型的数据**通过原型链**来访问。



## 1. 迭代器（Iterator）的特点

### 1.1 具有迭代器的数据类型是一种可迭代的数据类型，可以使用for...of来遍历

**在实际编程中，数据的迭代器主要是用来被for...of 操作来消费。**例如Array, Set, Map等这些数据都是可迭代的数据，可以用for...of遍历。

```js
var a = [1,2,3];
for(let v of a){
  console.log(v);
}
// 1 2 3

var a = new Set([1,2,3]);
for(let v of a){
  console.log(v);
}
// 1 2 3

var a = new Map([['a',1],['b',2],['c',3]]);
for(let v of a){
  console.log(v);
}
// ['a', 1]
// ['b', 2]
// ['c', 3]
```

> **对于 for...of 来说，内部的实现上就是在调用迭代器获取每个值来实现遍历**



### 1.2 迭代器是一个函数，调用这个函数会生成一个遍历器对象，使用这个遍历器对象可以遍历对象内部的数据

数据的迭代器接口默认部署在 `Symbol.iterator` 属性上，是一个函数，调用这个函数会生成一个**遍历器**对象。

**遍历器对象的特点：**

1. 遍历器对象必须要有一个next(...)方法，用来遍历内部数据
2. 调用next方法返回值是一个形如`{value:123, done:boolean}` 的对象，value表示遍历的值，done表示是否遍历完成
3. 遍历器对象还可以有return(...) 和 throw(...) 方法，在提前结束遍历是会调用。**for...of中使用break提前结束遍历的时候内部会调用return方法。**内部抛错中断也会调用return方法（类似于钩子）

```js
var arr = [1,2,3];
typeof arr[Symbol.iterator] === 'function'; // true

var i = arr[Symbol.iterator]();
i.next(); //{value:1, done:false}
i.next(); //{value:2, done:false}
i.next(); //{value:3, done:false}
i.next(); //{value:undefined, done:true}

function myIterator(){
  let index = 0;
  const len = this.length;
  let done = false;
  return {
    next:()=>{
      console.log('next');
      if(index<len && !done){
        return {value:this[index++], done:done}
      }else{
       	done = true;
        return {value:undefined, done:done}
      }
    },
    return: ()=>{
      console.log('return');
      done = true;
      return {value: undefined, done:done}
    }
  }
}

arr[Symbol.iterator] = myIterator;

for(let v of arr){
  if(v>1){
    break;
  }
  console.log(v);
}
//next
//1
//next
//return

for(let v of arr){
  if(v>1){
    throw 'error';
  }
  console.log(v);
}
//next
//1
//next
//return
// Uncaught error 先调用return方法 再抛出错误
```

## 2. 迭代器和Generaor函数之间的关系

从上面的分析中可以看出，**迭代器和Generator函数是非常相似的**

1. 调用之后都是返回一个**遍历器对象**
2. 遍历器对象中都实现了next和return方法，而且作用都是相似的，next方法用来遍历对象，return方法用来终止遍历。
3. next方法和return方法的返回值的结构也是一样的

所以，**迭代器最完美的方式是用Generator函数来实现。但是不能说迭代器就是Generator函数，因为使用普通的函数也能实现。**

判断是否是Generator函数的方法：

```js
function* gen(){}
gen.constructor.name === 'GeneratorFunction'; // true

var arr=[1,2,3]
arr[Symbol.iterator].constructor.name === 'GeneratorFunction'; //false
```

上面代码可得，数组的迭代器不是Generator函数，也就是说，迭代器并不都是使用Generator实现的。

总结迭代器和Generator之前的关系：

- 迭代器不一定是生成器

- **生成器一定是迭代器，而且调用生成器的遍历器的迭代器得到的遍历器和生成器的遍历器是同一个对象（有点绕看代码），所以遍历器可以使用for...of操作符**

  ```js
  function* gen(){
    yield 1;
    yield 2;
    yield 3;
  }
  var g = gen();
  g[Symbol.iterator]() === g;  //true 有这层关系，所以才能用for...of遍历 遍历器得到生成器中的状态
  
  for(let v of g){
    console.log(v);
  }
  // 1
  // 2
  // 3
  ```

  > 也就是说，**生成器的遍历器的迭代器就是这个生成器**



**在理解迭代器的时候，可以使用Generator函数来理解**

使用Generator函数实现上面数组的迭代器

```js
var arr = [1,2,3];
function* myIterator(){
  yield 1;
  yield 2;
  yield 3;
}
arr[Symbol.iterator] = myIterator;

for(let v of arr){
  console.log(v);
}
// 1
// 2
// 3
```

注意：

**for...of 内部调用next遍历的时候，返回值中如果done属性是true的话，那个值不会被使用。也就是说，遍历的是yield关键字后面的状态，return语句中的值不会被使用**



## 3. 调用迭代器的使用场景

1. **for...of 运算符**

2. 数组和Set的解构赋值(...) 

3. 数组和Set的扩展运算符(...)
4. yield* 操作符

可以使用上面自己实现的迭代器验证