# 数组去重

## 1. 去重的边界情况

去重操作必定会涉及到值的相等比较。js中提供了三种相等比较的方式：`==`、`===` 和`Object.is()`

**去重操作中数据之间的比较一般不会使用==操作符，因为会涉及到隐式类型转化的操作，影响去重的效果。**

===和Object.is()都是严格相等的比较，但是在一些特殊值的处理中，两者会有不同。

1. NaN和NaN的比较。===判断为false； Object.is()判断为true
2. +0和-0的比较。===判断为true； Object.is()判断为false

去重操作有时会有**对象去重**的操作，即将看起来一模一样的对象进行去重，比如`{a:1}`和`{a:1}`。但是其实是两个不同的对象

所以==去重操作的边界情况==是：

1. NaN和NaN
2. +0和-0
3. {a:1}和{a:1}

所以可以组成一个测试数组：

```js
var boundryTestArray=[NaN, NaN, +0, -0, {a:1}, {a:1}, undefined, null];
```

## 2. 去重方法的性能测试

生成一个大的随机数组，然后使用这个数组来测试性能

```js
function perfTest(){
	const arr=[];
	for(let i=0;i<100000;i++){
		arr.push(Math.floor(Math.random()*100000));
	}
  return function(fn){
    console.time('test');
    fn(arr.concat([])); // 防止sort方法修改原数组
    console.timeEnd('test');
  }
}
var pref=perfTest();
```

## 3. 去重的方法分析

### 3.1 indexOf方法

判断数字在新数组中是否存在

#### 3.1.1 for循环+indexOf

```js
function uniqueByIndexOf(arr){
  const result=[];
  for(let i=0;i<arr.length;i++){
    const v = arr[i]
    if(result.indexOf(v) === -1){
      result.push(v);
    }
  }
  return result;
}
```

边界测试：

```js
uniqueByIndexOf(boundryTestArray);
// [NaN,NaN,0, {a:1}, {a:1}, undefined, null]
// NaN不去重；对象不去重；+0和-0去重
```

**可见indexOf()内部使用的是===来比较数据**

性能测试：

```js
pref(uniqueByIndexOf);
// test: 2837.582763671875ms
```

#### 3.1.2 filter + indexOf

```js
function uniqueByFilterIndexOf(arr){
  return arr.filter((v,i)=>{
    return arr.indexOf(v) === i;
  })
}
```

边界测试：

```js
uniqueByFilterIndexOf(boundryTestArray);
// [0, {a:1}, {a:1}, undefined, null]
// NaN会被忽略；对象不去重；+0，-0去重
```

**NaN被忽略的原因是，indexOf(NaN)始终返回-1，因为 NaN !== NaN**

性能测试：

```js
pref(uniqueByFilterIndexOf);
// test: 3728.66796875ms
```

### 3.2 includes()方法

ES6中Array类中新增的方法

```js
function uniqueByIncludes(arr){
  const result=[];
  arr.forEach((v)=>{
    if(!result.includes(v)){
      result.push(v);
    }
  })
  return result;
}
```

边界测试：

```js
uniqueByIncludes(boundryTestArray);
//[NaN, 0, {a:1}, {a:1}, undefined, null]
// NaN去重；对象不去重；+0和-0去重
```

**Includes()方法内部对NaN进行了特殊处理，认为NaN 等于 NaN。修复了indexOf()方法无法判断是否含有NaN的错误**

性能测试：

```js
pref(uniqueByIncludes);
// test: 4265.844970703125ms
```

### 3.3 sort()方法

原理就是先排序然后去重，这里按ASCII顺序排序，不给sort传参数

```js
function uniqueBySort(arr){
  const result=[];
  arr.sort();
  arr.forEach((v)=>{
    let len = result.length;
    if(len<1){
      result.push(v);
    }else{
      if(result[len-1] !== v){
        result.push(v);
      }
    }
  })
  return result;
}


// sort + reduce
function uniqueBySortReduce(arr){
  return arr.sort().reduce((result,v)=>{
    let len = result.length;
    if(len ===0 || result[len-1] !== v){
      result.push(v);
    }
    return result;
  },[])
}


// sort + reduce + Object.is()
function uniqueBySortReduceIs(arr){
  return arr.sort().reduce((result,v)=>{
    let len = result.length;
    if(len ===0 || !Object.is(result[len-1], v)){
      result.push(v);
    }
    return result;
  },[])
}
```

边界测试：

```js
uniqueBySort(boundryTestArray);
// [0, NaN, NaN, {a:1}, {a:1}, null, undefined]

uniqueBySortReduce(boundryTestArray);
// [0, NaN, NaN, {a:1}, {a:1}, null, undefined]
// NaN不去重；对象不去重；+0和-0去重

uniqueBySortReduceIs(boundryTestArray);
// [0, -0, NaN, {a:1}, {a:1}, null, undefined]
// NaN去重；对象不去重；+0和-0不去重
```

性能测试：

```js
pref(uniqueBySort);
// test: 47.037109375ms

pref(uniqueBySortReduce);
// test: 46.18798828125ms

pref(uniqueBySortReduceIs);
// test: 47.35302734375ms
```

==注意异常情况：==

**使用sort方式排序的时候，会认为 `1` 和 `'1'` 是相等的，因为内部会先给元素做ToString或者ToNumber的操作，然后再比较。**

```js
[1,'1','2',1].sort()
// [1,'1',1,'2']

[1,'1','2',1].sort((a,b)=>a-b);
// [1,'1',1,'2']
```

如果使用上面的方法对象该数组去重，会发现 `1` 没有去重，还是有两个1，==因为每个数只和上一个数比较==

```js
uniqueBySort([1,'1','2',1]);
// [1, '1', 1, '2']

uniqueBySortReduce([1,'1','2',1]);
// [1, '1', 1, '2']
```

 **所以当一个数组中有多个类型的数据的时候，要慎用sort()方式去重。**

### 3.4 Object键值对

前面的去重方法中，都是**将唯一值记录在数组中，然后通过去数组查找是否存在当前值来判断是否唯一**。这是就需要使用到indexOf或者includes方法，这些方法内部都是**遍历数组**去查找，所以是比较耗时的。

Object键值对的方案是**使用一个object将唯一值作为key对保存在object中，然后直接根据这个object判断当前值是否唯一**

==但是使用object也存在一下问题==

1. **object中的key都是字符串的，内部会涉及到隐式类型转换，所以无法分辨 `1` 和`'1'`**
2. object中的key如果是对象类型，会**执行toString()转化成字符串**，所以很多对象都会转化成 `[object Object]`，无法对这些对象去重
3. object中可能存在一些只读的属性，会产生冲突

所以不能直接将数组子项的值直接当做key存在object中，要适当做一些转化

```js
function uniqueByObject(arr){
  const cache={};
  return arr.reduce((result,v)=>{
    let key = typeof v + JSON.stringify(v);  // 转化value成可用的key值
    if(!cache[key]){
      cache[key] = 1;
      result.push(v);
    }
    return result;
  },[])
}
```

边界检测：

```js
uniqueByObject(boundryTestArray);
// [0, NaN, {a:1}, null, undefined]
// NaN去重；对象去重；+0，-0去重
```

**这个方案可以将对象去重，是必然的效果，有好处也有坏处**

性能检测：

```js
pref(uniqueByObject);
//test: 81.544189453125ms
```

### 3.5 Map方案

由于object中的key只能是字符串，所以需要对特殊类型做一些转化才能存储进去。然后使用Map来存储映射表的话就没有这些问题。

**Map中的key可以是任何类型的值**，可以解决object方案中存在的问题。

```js
function uniqueByMap(arr){
  const map=new Map();
  return arr.reduce((result,v)=>{
    if(!map.has(v)){
      map.set(v,1);
      result.push(v);
    }
    return result;
  },[])
}
```

边界检测：

```js
uniqueByMap(boundryTestArray);
// [0, NaN, {a:1}, {a:1}, null, undefined]
// NaN去重；对象不去重；+0和-0去重
```

性能测试：

```js
pref(uniqueByMap);
// test: 11.6259765625ms
```

### 3.6 Set方案

ES6中提供了一种特殊的对象类型Set，感觉就是为去重而生的

```js
function uniqueBySet(arr){
  return [...new Set(arr)];
}
```

边界测试：

```js
uniqueBySet(boundryTestArray);
//[0, NaN, {a:1}, {a:1}, null, undefined]
// NaN去重；对象不去重；+0和-0去重
```

性能测试：

```js
pref(uniqueBySet);
// test: 9.616943359375ms
```

## 3. 总结

| 方案               | 边界处理                         | 性能测试 | 是否有风险                       |
| ------------------ | -------------------------------- | -------- | -------------------------------- |
| indexOf + for      | NaN和对象不去重，+0,-0去重       | 2837ms   |                                  |
| indexOf + filter   | NaN被忽略，对象不去重，+0,-0去重 | 3728ms   | NaN会被忽略，需要改造            |
| Includes           | NaN和0去重，对象不去重           | 4265ms   |                                  |
| sort方法           | NaN和对象不去重，0去重           | 47ms     | 排序有很大隐患，会导致去重不正确 |
| sort + Object.is() | NaN去重，对象和==0不去重==       | 47ms     | 排序有很大隐患，会导致去重不正确 |
| Object键值对       | NaN，对象和0都去重               | 81ms     |                                  |
| Map                | NaN和0去重，对象不去重           | 11ms     |                                  |
| Set                | NaN和0去重，对象不去重           | 9ms      |                                  |

==从表格结果可以看出==：

1. **对象只有使用Object键值对的方法才能去重，+0和-0只有使用Object.is来比较才不去重**
2. indexOf和includes方式，内部需要**多次遍历数组**，性能相对比较差，不推荐使用
3. sort的方式，面对**有多种数据类型的数组**，存在风险。要考虑清楚
4. 推荐使用ES6中的Map和Set方式，性能和效果都很好

## 4. 参考文章

1. [JavaScript专题之数组去重](https://juejin.im/post/5949d85f61ff4b006c0de98b#heading-7)

2. [解锁多种JavaScript数组去重姿势](https://juejin.im/post/5b0284ac51882542ad774c45#heading-6)

