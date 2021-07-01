## 数组平铺

**数组平铺就是给数组降维，减少数组的嵌套层级**。最常见的需求就是将多维数组转化成一维数组

## 1. 数组平铺的各种方案

测试数组：

```js
var flatTestArray=[1,2,3,[4,5,6,['7','8',9,[{a:1},{b:2}],10]]]
```

### 1.1 JSON.stringify() + JSON.parse()

这种方法是一种”黑科技“，并不是直接就能想到的方法。

**先将数组转化成 Json 字符串（toString()不行），然后利用字符串的特性来剔除掉数组符号 `[` 和 `]` ，然后再转化回数组。**

#### 1.1.1 一个错误

在很多文章中，将字符串转化成数组的方法采用的是 split 方法。在平铺的例子中，这是一种非常错误的做法，**虽然达到的平铺的目的，但是会改变数据子项的类型。**

**使用 `str.split(',')` 得到的数组，每个子项都是字符串类型的。**

```js
function flatByJsonSplit(arr){
  let str = JSON.stringify(arr);
  str = str.replace(/(\[|\])/g, '');
  return str.split(',');
}

flatByJsonSplit(flatTestArray);
// ["1", "2", "3", "4", "5", "6", ""7"", ""8"", "9", "{"a":1}", "{"b":2}", "10"]
// 明显的错误，数据类型都改变了
```



**只能使用`JSON.parse()` 将JSON字符串转化成数组，才能恢复原来的数据类型**

```js
function flatByJson(arr){
  let str = JSON.stringify(arr);
  str = '[' + str.replace(/(\[|\])/g, '') + ']';
  return JSON.parse(str);
}

flatByJson(flatTestArray);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
```

#### 1.1.2 这种方案的弊端

这种方式是一种非常规的解决方案，它能够解决一些问题。但是也一些**限制**

1. **JSON.stringify()无法正确解析undefined和function类型的数据**

   ```js
   var arr=[1,2,[undefined,3],4,function foo(){}]
   flatByJson(arr);
   // [1, 2, null, 3, 4, null]
   ```

2. 在使用正则表达式去掉中括号的时候，**可能会去掉不该去掉的中括号**

   ```js
   var arr=[1,2,[{a:[3,4]},7],5,6];
   flatByJson(arr);
   // 会直接报错，因为{a:[3,4]}变成了{a:3,4}
   ```

3. 这种方案只能将多维数组变成一维数据，并**不能控制展开的层级数**



### 1.2 递归

```js
function flatByRecursion(arr){
  let result=[];
 	arr.forEach((v)=>{
    if(Array.isArray(v)){
      result = result.concat(flatByRecursion(v));
    }else{
      result.push(v);
    }
  })
  return result;
}

flatByRecursion(flatTestArray);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
```

#### 1.2.1 使用reduce更优雅的实现

```js
function flatByRecursionReduce(arr){
  return arr.reduce((result,v)=>{
    return result.concat(Array.isArray(v) ? flatByRecursionReduce(v) : v);
  },[]);
}

flatByRecursionReduce(flatTestArray);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
```

递归方法总归是有爆栈的隐患，但是在这种场景中基本不会爆栈。

> 这里是利用`concat`来实现展开层级



### 1.3 使用栈的思想来

利用栈的思想，将递归改成循环来实现。

将整个数组当成是一个栈，然后从栈顶开始取数据

- **如果取出的是一个数组，则将数组中所有元素推入栈中**
- 如果取出的不是数组，则保存起来

然后继续遍历栈，以此循环，直到栈中没有数据

```js
function flatByStack(arr){
  const result=[];
  const stack = arr.slice();
  while(stack.length > 0){
    const v = stack.shift();
    if(Array.isArray(v)){
      stack.unshift(...v);
    }else{
      result.push(v);
    }
  }
  return result;
}

flatByStack(flatTestArray);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
```



### 1.4 使用Generator来实现

Generator函数中存在`yield*`，**这个操作符用来执行后面对象的迭代器的遍历器，类似于 for...of 操作**。所以刚好可以用来打平数组。

```js
function flatByGenerator(arr){
  function* flatGen(arr){
    for(let v of arr){  // 这里不能使用forEach，会导致yield在其他函数中
      if(Array.isArray(v)){
        yield* flatGen(v); // 内部其实也需要递归
      }else{
        yield v;
      }
    }
  }
  return [...flatGen(arr)];
}

flatByGenerator(flatTestArray);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
```



### 1.5 Array.prototype.flat()

ES6中提供了一个flat方法，用来平铺数组

```js
flatTestArray.flat();
// [1, 2, 3, 4, 5, 6, Array(5)] 只展开一级

flatTestArray.flat(2);
//  [1, 2, 3, 4, 5, 6, "7", "8", 9, Array(2), 10]

flatTestArray.flat(Infinity);
// [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]

flatTestArray.flat(null);
// [1, 2, 3, Array(4)]

flatTestArray.flat(0);
flatTestArray.flat(-1);
// [1, 2, 3, Array(4)]

flatTestArray.flat('1');
// [1, 2, 3, 4, 5, 6, Array(5)]

flatTestArray.flat('1w');
// [1, 2, 3, Array(4)]

[1,2,[3,,4,5],6].flat(Infinity);
// [1, 2, 3, 4, 5, 6]
```

flat方法的特点：

1. 不传参数时，默认平铺一层
2. 参数小于1，或者参数转化成非法数字的时候，则不平铺
3. 参数传入 `Infinity` 关键字的时候，则所有层级都展开
4. flat会跳过空值



## 2. 手动实现一个myFlat()方法

根据上面的特点，手动实现一个 flat() 方法

```js
Array.prototype.myFlat=function(num = 1){
  if(!Number(num) || Number(num) < 0) return this; //参数判断
  const flat=(arr, n)=>{
    if(n>0){
      return arr.reduce((result, v)=>{
        return result.concat(Array.isArray(v) ? flat(v, n-1) : v);
      },[])
    }else{
      return arr;
    }
  }
  return flat(this, num);
}

//也可以使用Generator实现
Array.prototype.myFlatBtGenerator=function(num = 1){
  if(!Number(num) || Number(num) < 0) return this; //参数判断
  
  function* flat(arr, n){
    for(let v of arr){
      if(Array.isArray(v) && n>0){
        yield* flat(v, n-1);
      }else{
        yield v;
      }
    }
  }
  return [...flat(this,num)];
}


flatTestArray.myflat(); // [1, 2, 3, 4, 5, 6, Array(5)]
flatTestArray.myFlat(2); // [1, 2, 3, 4, 5, 6, "7", "8", 9, Array(2), 10]
flatTestArray.myFlat(Infinity); // [1, 2, 3, 4, 5, 6, "7", "8", 9, {a:1}, {b:2}, 10]
flatTestArray.myFlat(null); // [1, 2, 3, Array(4)]
flatTestArray.myFlat(0); // [1, 2, 3, Array(4)]
flatTestArray.myFlat(-1); // [1, 2, 3, Array(4)]
flatTestArray.myFlat('1'); // [1, 2, 3, 4, 5, 6, Array(5)]
flatTestArray.myFlat('1w'); // [1, 2, 3, Array(4)]

[1,2,[3,,4,5],6].myFlat(Infinity); // [1, 2, 3, 4, 5, 6]  reduce函数会跳过数组的空位
```



##  3. 参考文章

1. [面试官连环追问：数组拍平（扁平化） flat 方法实现](https://juejin.im/post/5dff18a4e51d455804256d31#heading-2)

