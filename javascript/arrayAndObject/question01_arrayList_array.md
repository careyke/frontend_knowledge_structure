# 类数组对象和数组的关联

## 1. 类数组对象

在定义类数组对象之前，先看《javascript权威指南》中对于类数组对象的判断方法

```js
function isArrayLike(o) {
    if (o &&                                // o is not null, undefined, etc.
        typeof o === 'object' &&            // o is an object
        isFinite(o.length) &&               // o.length is a finite number
        o.length >= 0 &&                    // o.length is non-negative
        o.length===Math.floor(o.length) &&  // o.length is an integer
        o.length < 4294967296)              // o.length < 2^32
        return true;                        // Then o is array-like
    else
        return false;                       // Otherwise it is not
}
```

根据上面的判断方法可知：

**一个拥有length属性的对象，而且length的值是一个合法的非负整数，这样的对象就是类数组对象**

常见的类数组对象有：

1. 函数中的 arguments 对象
2. 获取DOM对象的方法的返回值，比如 `document.getElementsByClassName()`



## 2. 类数组对象和组数之间的联系

假设有以下的类数组对象和数组

```js
var arr=[1,2,3];

var arrLike={
  0:1,
  1:2,
  2:3,
  length:3
}
```

**类数组对象可以和数组一样根据索引来获取值或者赋值，而且可以使用for循环来进行遍历**

```js
arr[1] //2
arrLike[1] // 2

arr[1] = 4; 
arrLike[1] = 4;

for(let i=0;i<arr.length;i++){
  console.log(arr[i]);
}
// 1 2 3

for(let i=0;i<arrLike.length;i++){
  console.log(arrLike[i]);
}
// 1 2 3
```

**数组本身也是一个属性名为所索引值的特殊对象，和类数组对象的区别就是数组是由Array类产生的，而类数组对象是由Object类产生的，所以类数组对象不能直接使用数组对象上的方法**

```js
arrLike.forEach((v)=>console.log(v));  //会报错，因为类数组原型链上没有forEach方法
```

但是可以使用call/apply的方式，将类数组对象**委托**给对象的方法来执行

```js
Array.prototype.forEach.call(arrLike,(v)=>{console.log(v)});
// 1 2 3
```



## 3. 类数组对象转化成数组的方式

由于类数组对象无法直接使用数组中的高阶方法，只能通过委托的形式去使用，比较麻烦。通常会直接将类数组对象转化成数组，然后再操作

### 3.1 借助Array的API来转化

**一切以数组为输入，并以数组为输出的 API 都可以用来将类数组对象转化成数组**

1. Array(...)方法：类数组对象作为函数参数

   ```js
   Array.apply(null, arrLike); //apply方法会将方法所有的参数构造一个数组或者类数组对象，内部再拆分
   //如果使用call的话是不行的，call会将整个类数组对象作为一个参数
   
   //相当于
   Array(1,2,3); // [1,2,3]
   ```

2. Array.from(...)：类数组对象作为函数参数

   ```js
   Array.from(arrLike); // [1,2,3]
   ```

3. Array.prototype.concat(...)：类数组对象作为函数参数

   ```js
   Array.prototype.concat.apply([], arrLike); // [1,2,3]
   //concat的参数类型是值或者数组，类数组被当成值处理，所以需用使用apply调用
   ```

4. Array.prototype.slice(...)：类数组对象作为函数this

   ```js
   Array.prototype.slice.call(arrLike); //[1,2,3]
   ```

5. Array.prototype.map(...)：类数组对象作为函数this

   ```js
   Array.prototype.map.call(arrLike,v=>v); // [1,2,3]
   ```

6. Array.prototype.filter(...)：类数组对象作为函数this

   ```js
   Array.prototype.filter.call(arrLike,v=>1); // [1,2,3]
   ```

上面6种方法**对于索引密集的类数组对象转化成数组是一样的，都是返回一个密集数组**。但是对于**索引稀疏**的类数组对象，上面方法返回是不一致的

```js
var arrLike={
  0:1,
  3:2
  length:4
}

Array.apply(null, arrLike); // [1,undefined,undefined,2]
Array.from(arrLike); // [1,undefined,undefined,2]
Array.prototype.concat.apply([], arrLike); // [1,undefined,undefined,2]

Array.prototype.slice.call(arrLike); // [1, empty × 2, 2]  稀疏数组
Array.prototype.map.call(arrLike,v=>v); // [1, empty × 2, 2] 稀疏数组
Array.prototype.filter.call(arrLike,v=>1); // [1,2] 出现不期望的过滤
```

总结：

- 当**类数组对象的索引稀疏**的时候，类似于`{0:1,4:3,length:5}`，将类数组对象作为参数的方法返回的是密集数组，使用undefined填补空位；将类数组对象作为 this 的方法返回的是稀疏数组（filter方法除外）。
- 当**类数组对象的索引密集**的时候，返回的都是密集数组
- **最好是直接使用`Array.from()`方法**



### 3.2 使用扩展运算符

在数组中使用扩展运算符的**前提条件是需要有迭代器属性**。如果没有的话会报错

```js
var arrLike={
  0:1,
  1:2,
  2:3,
  length:3
}
[...arrLike]; // 会报错
```

但是我们常见的**类数组对象 `argument` 是一个迭代器对象**，是可以用扩展运算符转化成数组的。

> 迭代器相关的可以看[深入理解迭代器和Generator](https://github.com/careyke/frontend_knowledge_structure/blob/master/javascript/iterator/question01_generator_iterator.md)

```js
function foo(){
  console.log([...arguments]);
}
foo(1,2,3);
// [1,2,3]
```



## 参考文章

1. [你真的会将类数组转化为数组吗](https://juejin.im/post/5e1d06566fb9a0301f2f1e46)

