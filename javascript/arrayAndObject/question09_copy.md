# Object的浅拷贝和深拷贝

对象类型数据的拷贝是实际编程中常常遇到的场景。对于对象的拷贝存在两种方式：浅拷贝和深拷贝

可以将对象类型的数据类比成一个树结构

- 浅拷贝：只拷贝对象的第一层子元素
- 深拷贝：对象所有层级的子元素都拷贝一份

注：下面所有方案的实现中，都只考虑普通对象和数组，没有对特殊对象做额外的处理



判断是否是对象类型：

```js
function isObject(obj){
  const type = typeof obj;
  if((type !== 'object' && type !== 'function') || obj === null){
    return false;
  }
  return true;
}
```



## 1. 浅拷贝的方案

### 1.1 手动实现一个浅拷贝

```js
function shallowCopy(obj){
  if(!isObject(obj)){
    throw TypeError('obj is not a Object');
  }
  const result = Array.isArray(obj) ? []:{};
  for(let k in obj){
    if(obj.hasOwnProperty(k)){
      result[k] = obj[k];
    }
  }
  return result;
}

var obj1 = [1,2,3];
var newObj1 = shallowCopy(obj1); //[1,2,3]
newObj1[1] = 10;
console.log(obj1); // [1,2,3]
console.log(newObj1); // [1,10,3]

var obj2 = {a:1,b:2,c:3};
var newObj2 = shallowCopy(obj2); //{a:1,b:2,c:3}
newObj2['a'] = 10;
console.log(obj2); // {a:1,b:2,c:3}
console.log(newObj2); // {a:10,b:2,c:3}
```

深拷贝和浅拷贝的区别：

```js
var object = {
  a:1,
  b:{
    c:2
  }
}
var newObj = shallowCopy(object);
newObj.b.c = 3; // 改变新对象中的数据
console.log(object);
//{a:1,b:{c:3}}  原来的对象数据也发生了改变
```

上面结果可以看出，**当对象的属性是引用类型的时候，浅拷贝只会将这个引用拷贝到新对象**。而不是在新对象中新创建一个对象，然后再拷贝内部属性，如此递归执行。

而**深拷贝则会递归拷贝内部所有的对象**

### 1.2 Object.assign()

Object.assign()方法拷贝对象的时候，执行的就是浅拷贝

```js
var obj = {a:1,b:2}
var newObj = Object.assign({},obj); //{a:1,b:2}
console.log(obj === newObj); // false

//注意
var obj2 = Object.assign(obj,{c:3,d:4});
console.log(obj2); // {a:1,b:2,c:3,d:4}
console.log(obj2 === obj); // true
```

上面结果可以看出，**Object.assign()方法拷贝的是后面对象中的属性，内部并不会自动创建一个新对象。**

### 1.3 扩展运算符...

```js
var obj = {a:1,b:2}
var newObj = {...obj};
console.log(obj === newObj); // false

var arr=[1,2,3]
var newArr=[1,2,3]
console.log(arr === newArr); // false
```

### 1.4 数组的浅拷贝

对象的浅拷贝常用的就是上面三种，但是数组的浅拷贝还有很多方法。

**Array.prototype中的返回新数组的方法都可以用来做数组的浅拷贝。**

1. Array.prototype.slice
2. Array.prototype.map
3. Array.prototype.filter
4. Array.prototype.concat
5. Array.prototype.reduce

但是要注意稀疏数组的情况，filter, reduce会跳过数组空位

```js
var arr=[1,2,3];

var newArr1 = arr.slice();

var newArr2 = arr.map(v=>v);

var newArr3 = arr.filter(v=>1);

var newArr4 = arr.concat();

var newArr5 = arr.reduce((a,v)=>{
	a.push(v);
  return a;
},[]);
```



## 2. 深拷贝

### 2.1 深拷贝的关键功能

1. **是否能够拷贝所有类型的属性**

   ```js
   function testSpecial(deepClone){
     var specialObj={
   		a:1,
     	b:null,
     	c:undefined,
     	d:{v:2}
   	}
     return deepClone(specialObj);
   }
   ```

2. **相同引用**是否能够保留

   ```js
   function testSameQuote(deepClone){
     var obj = {v:1}
     var o = {
   		a:obj,
     	b:obj
   	}
     var newO = deepClone(o);
     return newO.a === newO.b;
   }
   ```

3. **循环引用**是否能够保留

   ```js
   function testCircleQuote(deepClone){
     var a={}
   	a.b = a;
     return deepClone(a); // 不会陷入死循环，超出时间限制
   }
   ```



### 2.2 深拷贝的性能测试

提供一个方法来粗略测试深拷贝方案的性能

```js
function prefTest(){
  var deep=1000;
  var range=1000;
  const res={};
  let temp = res
  
  for(let i=0;i<deep;i++){
    for(let j=0;j<range;j++){
      temp[j] = j;
    }
    temp.data = {};
    temp = temp.data;
  }
  
  return function(fn){
    console.time('time');
    fn(res);
    console.timeEnd('time');
  }
}
var pref = prefTest();
```



### 2.3 JSON方案

先调用stringify()，然后调用parse()

```js
function deepCloneByJson(obj){
	return JSON.parse(JSON.stringify(obj));
}


testSpecial(deepCloneByJson) // {a: 1, b: null, d: {v:2}}
testSameQuote(deepCloneByJson); //false
testCircleQuote(deepCloneByJson) // Uncaught TypeError: Converting circular structure to JSON

pref(deepCloneByJson);
// time: 137.0009765625ms
```

优点：实现简单

缺点：

1. undefined类型的属性无法识别拷贝（**function类型的数据也不能拷贝，这里暂时不讨论**）
2. 无法保留相同引用
3. 无法保留循环引用



### 2.4 简单的深拷贝

可以认为深拷贝就是**递归的对每个引用类型的属性做浅拷贝**

```js
function deepCloneSimple(obj){
  if(!isObject(obj)){
    throw TypeError('obj is not a Object');
  }
  const result = Array.isArray(obj) ? []:{};
  for(let k in obj){
    if(obj.hasOwnProperty(k)){
      let v = obj[k];
      if(isObject(v)){
        result[k] = deepCloneSimple(v);
      }else{
        result[k] = v;
      }
    }
  }
  return result;
}

testSpecial(deepCloneSimple); // {a: 1, b: null, c: undefined, d: {v:2}}
testSameQuote(deepCloneSimple); // false
testCircleQuote(deepCloneSimple);  // RangeError: Maximum call stack size exceeded

pref(deepCloneSimple);
// time: 67.91796875ms
```

优点：相对于JSON方案，它可以拷贝所有类型的属性（function类型也是可以的，特殊处理就好，这里不讨论）

缺点：

1. 无法保留相同引用
2. 无法保留循环引用



### 2.5 可保存引用的深拷贝

前面的方案中，**之所以不能保存相同引用和循环引用，就是因为每次遇到引用类型的数据时，都是直接去创建一个新的对象，并没有判断这个对象是否已经出现过。**

所以如果需要保存相同引用和循环引用，就需要**创建一个映射表来保存已经出现过的对象，如果需要相同的对象，直接使用之前创建的即可。**

#### 2.5.1 使用什么数据结构来保存映射表

一般来说用来缓存映射关系的数据结构有：`{}` 、`Map`、`WeakMap`。这里最合适的是使用WeakMap。

**因为WeakMap中，`键名` 所引用的对象都是 `弱引用`，垃圾回收机制不将该引用考虑在内。因此，只要所引用的对象的其他引用都被清除，垃圾回收机制就会释放该对象所占用的内存**。

> WeakMap中key引用的对象，不算引用。如果一个对象只被WeakMap中的key引用，那么垃圾回收机制会回收这个对象

也就是说，**一旦不再需要，WeakMap 里面的键名对象和所对应的键值对会自动消失，不用手动删除引用**。

注意：

1. **WeakMap 弱引用的只是键名，而不是键值。键值依然是正常引用**
2. **WeakMap的key只能是对象**

#### 2.5.2 代码实现

```js
function deepCloneKeepQuote(obj){
  if(!isObject(obj)){
    throw TypeError('obj is not a Object');
  }
  const map = new WeakMap();
  
  function deepClone(obj){
    if(map.has(obj)) return map.get(obj);
    const constructor = obj.constructor;
    const result = new constructor();  // 保证克隆对象的原型链
    map.set(obj, result);
    
    for(let k in obj){
      if(obj.hasOwnProperty(k)){
        let v = obj[k];
        if(isObject(v)){
          result[k] = deepClone(v);
        }else{
          result[k] = v;
        }
      }
    }
    return result;
  }
  
  return deepClone(obj);
}


testSpecial(deepCloneKeepQuote); // {a: 1, b: null, c: undefined, d: {v:2}}
testSameQuote(deepCloneKeepQuote); // true
testCircleQuote(deepCloneKeepQuote);  // { b: [Circular] }

pref(deepCloneKeepQuote);
// time: 95.1279296875ms
```

优点：

1. 可以拷贝所有类型的属性值
2. 可以保留相同引用
3. 可以保留循环引用
4. 可以保留原始对象的原型链

缺点：递归调用，有爆栈的风险



### 2.6 循环实现深拷贝

**递归通常可以使用循环和栈或者队列来替代**

这里将所有对象的克隆操作都存放在队列中，然后遍历队列执行。（本质上就是二叉树的层序遍历）

```js
function deepCloneByWhile(obj){
  if(!isObject(obj)){
    throw TypeError('obj is not a Object');
  }
  
  function createObj(obj){
    const constructor = obj.constructor;
    return new constructor();
  }
  
  const map = new WeakMap();
  const targetObj = createObj(obj)
  const list = [
    {
      data: obj,
      result: targetObj
    }
  ]; // 存放对象克隆操作，每个克隆操作有object和targetObject
  
  while(list.length > 0){
    const {data,result} = list.pop();
    for(let k in data){
      if(data.hasOwnProperty(k)){
        let v = data[k];
        if(isObject(v)){
          if(map.has(v)){
            result[k] = map.get(v);
          }else{
            let target = createObj(v); // 先创建一个目标对象，复制属性的操作在后面遍历队列的时候进行
            result[k] = target; //先建立引用关系
            map.set(v, target);
            list.push({
              data:v,
              result:target
            })
          }
        }else{
          result[k] = v;
        }
      }
    }
  }
  return targetObj;
}


testSpecial(deepCloneByWhile); // {a: 1, b: null, c: undefined, d: {v:2}}
testSameQuote(deepCloneByWhile); // true
testCircleQuote(deepCloneByWhile);// { b: [Circular] }

pref(deepCloneByWhile);
// time: 98.462158203125ms
```

优点：在递归方案的基础上，将递归替换成循环，可以防止爆栈

缺点：

1. 需要使用额外的空间
2. 不是很好理解



### 2.7 总结

| 方法               | 实现思路 | 性能测试 | 功能                                                         |
| ------------------ | -------- | -------- | ------------------------------------------------------------ |
| deepCloneByJson    | JSON api | 137ms    | 1.undefined,function无法处理<br />2.无法保留相同引用和循环引用 |
| deepCloneSimple    | 递归     | 67.9ms   | 1.无法保留相同引用和循环引用<br />2. 递归爆栈                |
| deepCloneKeepQuote | 递归     | 95.1ms   | 递归爆栈                                                     |
| deepCloneByWhile   | 循环     | 98.5ms   | 引入额外空间，代码不好理解                                   |

所以：

- 如果数据量小的时候，而且不用保留相同引用和循环引用，建议使用 `deepCloneSimple` 
- 如果需要保留相同引用和循环引用的时候
  - 数据量不是特别大的时候，可以使用 `deepCloneKeepQuote`
  - 如果嵌套层次特别深，有爆栈风险，则使用 `deepCloneByWhile`



## 参考文章

1. [深拷贝的终极探索](https://segmentfault.com/a/1190000016672263#item-6)

