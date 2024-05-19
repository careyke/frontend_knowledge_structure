# 数组乱序

数组乱序就是将数组的排列顺序随机打乱

## 1. Math.random()方案

最常见的方案就是使用Math.random()给数组随机排序

```js
function shuffle(arr){
  arr.sort(function(){
    return Math.random() - 0.5;  //随机正序或者反序
  });
  return arr;
}
```

这个方案看似可以完美解决我们的需求，使数组随机乱序。但是其实效果不是特别好

```js
function test(){
	const times=[0, 0, 0, 0, 0];
  for(let i=0;i<10000;i++){
    const arr=[1,2,3,4,5];
    shuffle(arr);
    times[arr[0]-1]++; // 测试1，2，3，4，5在数组第一个位置的次数
  }
  return times;
}
test();
// [3189, 1210, 1661, 2222, 1718]
```

上面代码中，将数组 `[1,2,3,4,5]` 乱序10000次，测试各个数字出现在第一个位置的次数。

结果可知，这个方案乱序是不可靠的。因为**一个元素出现在各个位置的概率不是一样的**。



### 1.1 分析概率不一样的原因

上一篇文章有讲过，sort内部排序使用的是插入排序和快速排序两种，但是最终区间变小的时候都是**插入排序**。

插入排序：

```js
function insertSort(arr, left, right, compareFn){
  for(let i=left+1;i<=right;i++){
    let t = arr[i];
    let j = i-1;
    for(; j>=left && compareFn(arr[j], t) > 0; j--){
      arr[j+1] = arr[j];
    }
    arr[j+1] = t;
  }
  return;
}
```

插入排序的原理就是**从无序区间中依次选取元素插入到有序区间中，默认开始的时候第一个值是有序区间的。**

简单分析一下数组`[1,2,3]`的乱序过程

```js
var arr=[1,2,3]
shuffle(arr);
```

- 第一个插入的值是2，比较 2 和 1
  - 50%概率：[1, 2]
  - 50%概率：[2, 1]

- 第二个插入的值是3
  - 如果此时有序区间是[1,2]，首先比较3和2
    - 50%概率：[1,2,3]，插入完成
    - 50%概率：[1,3,2]，插入没有完成，要继续比较3和1
      - 50%概率：[1,3,2]
      - 50%概率：[3, 1, 2]
  - 如果此时有序区间是[2,1]，首先比较3和2
    - 50%概率：[2,1,3]，插入完成
    - 50%概率：[2,3,1]，插入没有完成，要继续比较3和1
      - 50%概率：[2,3,1]
      - 50%概率：[3,2,1]

| 结果      | 概率  |
| --------- | ----- |
| [1,2,3]   | 25%   |
| [1,3,2]   | 12.5% |
| [3, 1, 2] | 12.5% |
| [2,1,3]   | 25%   |
| [2,3,1]   | 12.5% |
| [3,2,1]   | 12.5% |

代码验证：

```js
function chanceTest(arr){
  const obj={};
  const time = 10000;
  for(let i=0;i<time;i++){
    let newArr = shuffle(arr.concat([]));
   	let key = JSON.stringify(newArr);
    if(obj[key]){
      obj[key]++;
    }else{
      obj[key] = 1;
    }
  }
  for(let k in obj){
    console.log(`${k}: ${obj[k]/time}`);
  }
}
chanceTest([1,2,3]);
// 输出结果
[3,2,1]: 0.3152
[1,2,3]: 0.369
[3,1,2]: 0.0647
[2,1,3]: 0.1241
[2,3,1]: 0.0636
[1,3,2]: 0.0634
```

这个输出结果看起来和上面分析的也不一致（不知道是不是内部算法做了修改）。但是可以证明的是，这种方案对于数组的乱序处理并不彻底。

根本原因在于：

**插入排序的时候，当元素插入有序区间的时候，如果确定了位置，就不会再和前面的元素继续比较。所以元素插入各位位置的概率并不是一致的。**



## 2. Fisher–Yates

之所以叫做这个名字是因为这个算法是由 Ronald Fisher 和 Frank Yates 首次提出的

原理很简单，就是**使每一种情况产生的概率是一样的**。有点像排列组合

假设有一个数组`[1,2,3,4,5]`，

1. 第一个位置存放5个数中的任意一个

2. 第二个位置只能存放剩下的4个数字中的一个，因为**同一个元素是不能同时存放在多个位置中的**
3. 以此类推，直到存放完。所以组合出现的概率是一样的

### 2.1 代码实现

```js
function shuffleByFisherYates(arr){
  for(let i=arr.length-1; i>=0;i--){
    const randomIndex = Math.floor(Math.random() * (i+1));
    let t = arr[i];
    arr[i] = arr[randomIndex];
    arr[randomIndex] = t;
  }
  return arr;
}
```

测试一下

```js
function chanceTest(arr){
  const obj={};
  const time = 10000;
  for(let i=0;i<time;i++){
    let newArr = shuffleByFisherYates(arr.concat([]));
   	let key = JSON.stringify(newArr);
    if(obj[key]){
      obj[key]++;
    }else{
      obj[key] = 1;
    }
  }
  for(let k in obj){
    console.log(`${k}: ${obj[k]/time}`);
  }
}



chanceTest([1,2,3]);
//输出结果
[3,1,2]: 0.1652
[2,1,3]: 0.1689
[3,2,1]: 0.1692
[2,3,1]: 0.1646
[1,2,3]: 0.1651
[1,3,2]: 0.167
```

测试结果可以看出，各种情况出现的概率基本都是一样的



## 3. 实现一个shuffle函数

```js
Array.prototype.shuffle=function(){
  const thisObj = Object(this);
  let len = 0;
  if(Number(thisObj.length) && Number(thisObj.length) > 0){
    len = Number(thisObj.length);
  }
  if(len === 0){
    throw TypeError('length is 0');
  }
  for(let i=len-1; i>=0;i--){
    const randomIndex = Math.floor(Math.random() * (i+1));
    let t = thisObj[i];
    thisObj[i] = thisObj[randomIndex];
    thisObj[randomIndex] = t;
  }
  return thisObj;
}
```



## 4. 参考文章

1. [JavaScript专题之乱序](https://github.com/mqyqingfeng/Blog/issues/51)

