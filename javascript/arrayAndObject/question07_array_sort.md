# 分析V8中的sort方法并手动实现一个

## 1. V8中sort方法的实现规则

**参数规则：**

- 当**没有传入参数或者参数不是函数**的时候，使用默认的排序函数，按照 **ASCII 码从小到大**的顺序排列
- 当传入**一个函数参数**的时候，比如`function(a,b){}` ，通过函数的返回值决定a b的位置
  - 返回值相等，相对位置不变
  - 返回值小于0，a在b左边
  - 返回值大于0，a在b右边

**排序算法的规则：**

- 当length <= 10的时候，采用插入排序
- 当length > 10的时候，采用快速排序。快速排序中，基准值的选取决定了算法的优劣。==V8中会从第一个数、最后一个数和第三个数third中获取中位数作为基准值。==
  - 当10 < length <= 1000的时候，将数组中间位置的数作为 third
  - 当length > 1000 的时候，每隔 200 ~ 215 个元素取一个值，然后将这些值进行排序，取中间值的作为 third

###1.1 为什么小于等于10的时候，采用插入排序

虽然插入排序的时间复杂度是O(n^2)，大于快速排序的时间复杂度O(nlogn)。但是这些都只是**n趋向无穷大时候的估算结果**，而且是忽略掉常数项的。但是当n比较小的时候，常数项是不能忽略的，插入排序是有可能比快速排序更快的。

实际上也是这样的，**当n比较小的时候，插入排序比较快速排序更快。**



## 2. 手动实现一个sort方法

### 2.1 实现一个默认的比较方法

```js
function defaultCompareFn(a,b){
  // 不考虑a b为undefined和null的情况
  a = a.toString();
  b = b.toString();
  if(a === b){
    return 0;
  }else{
    return a > b ? 1: -1
  }
}
```

### 2.2 实现一个插入排序

这个插入函数需要传入数组的区间索引和比较函数

```js
function insertSort(arr, left, right, compareFn){
  for(let i=left+1;i<=right;i++){
    let t = arr[i];
    let j = i-1;
    for(; j>=0 && compareFn(arr[j], t) > 0; j--){
      arr[j+1] = arr[j];
    }
    arr[j+1] = t;
  }
  return;
}
```

### 2.3 实现一个获取third的方法

```js
function getThirdIndex(arr, left, right, compareFn){
  const len = right-left +1;
  if(len<= 1000){
    return (left+right) >>> 1;
  }
  const pickArr=[];
  const increment = 200 + (15 & (right - left));  // 与运算，不会操作15
  for(let i = left+1; i<right; i+=increment){
    pickArr.push([i,arr[i]]);
  }
  pickArr.sort(function(a,b){
    return compareFn(a[1],b[1]);
  })
  return pickArr[pickArr.length >>> 1][0];
}
```

### 2.4 实现快速排序

sort方法接收的比较函数，实际上就是用来将**数组的元素和基准值作比较**

```js
const exchange = (arr,a,b)=>{
  const t = arr[a];
  arr[a] = arr[b];
  arr[b] = t;
}
function quickSort(arr, left, right, compareFn){
	const pivot = arr[right]; // 前提是之前定的基准值必须在数组区间的最后一个位置
  let pos=left-1;
  for(let i = left;i <= right;i++){
    if(compareFn(arr[i], pivot) <=0 ){
      pos++;
      exchange(arr, i, pos);
    }
  }
  return pos;
  // 这里不做递归，由外层做递归，因为有可能是插入排序
}
```

### 2.5* 整体代码实现

```js
function sort(arr, compareFn) {

  function defaultCompareFn(a, b) {
    // 不考虑a b为undefined和null的情况
    a = a.toString();
    b = b.toString();
    if (a === b) {
      return 0;
    } else {
      return a > b ? 1 : -1
    }
  }

  const exchange = (arr, a, b) => {
    const t = arr[a];
    arr[a] = arr[b];
    arr[b] = t;
  }

  function insertSort(arr, left, right, compareFn) {
    for (let i = left + 1; i <= right; i++) {
      let t = arr[i];
      let j = i - 1;
      for (; j >= 0 && compareFn(arr[j], t) > 0; j--) {
        arr[j + 1] = arr[j];
      }
      arr[j + 1] = t;
    }
    return;
  }

  function getThirdIndex(arr, left, right, compareFn) {
    const len = right - left + 1;
    if (len <= 1000) {
      return (left + right) >>> 1;
    }
    const pickArr = [];
    const increment = 200 + (15 & (right - left));  // 与运算，不会操作15
    for (let i = left + 1; i < right; i += increment) {
      pickArr.push([i, arr[i]]);
    }
    pickArr.sort(function (a, b) {
      return compareFn(a[1], b[1]);
    })
    return pickArr[pickArr.length >>> 1][0];
  }

  function quickSort(arr, left, right, compareFn) {
    const pivot = arr[right]; // 前提是之前定的基准值必须在数组区间的最后一个位置
    let pos = left - 1;
    for (let i = left; i <= right; i++) {
      if (compareFn(arr[i], pivot) <= 0) {
        pos++;
        exchange(arr, i, pos);
      }
    }
    return pos;
    // 这里不做递归，由外层做递归，因为有可能是插入排序
  }

  function getPivotArr(left, right, third, compareFn) {
    const arr = [left, right, third];
    insertSort(arr, 0, 2, compareFn);
    return arr;
  }

  function sortHelper(arr, left, right, compareFn) {
    const len = right - left + 1;
    if (len <= 10) {
      insertSort(arr, left, right, compareFn);
      return;
    } else {
      let thirdIndex = getThirdIndex(arr, left, right, compareFn);
      let pivotArr = getPivotArr(arr[left], arr[right], arr[thirdIndex], compareFn);
      arr[left] = pivotArr[0];
      arr[right] = pivotArr[1];  // 保证pivot在数组区间的最右边
      arr[thirdIndex] = pivotArr[2];
      let sortedPivotIndex = quickSort(arr, left, right, compareFn);
      sortHelper(arr, left, sortedPivotIndex - 1, compareFn);
      sortHelper(arr, sortedPivotIndex + 1, right, compareFn);
    }
  }

  // 函数体
  if (typeof compareFn !== 'function') {
    compareFn = defaultCompareFn;
  }
  sortHelper(arr, 0, arr.length - 1, compareFn)

  return arr;
}


var arr = [3, 44, 38, 5, 47, 15, 36, 26, 27, 2, 46, 4, 19, 50, 48];
sort(arr);
// [15, 19, 2, 26, 27, 3, 36, 38, 4, 44, 46, 47, 48, 5, 50]

sort(arr,(a,b)=>a-b);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

sort(arr,(a,b)=>b-a);
// [50, 48, 47, 46, 44, 38, 36, 27, 26, 19, 15, 5, 4, 3, 2]
```

上面的实现中是按照 V8 中sort方法实现的特点来实现的。但是这里快速排序的实现和V8中是不一样的

### 2.6 V8中sort方法中的快速排序

采用的是双指针的遍历方式实现的。代码来自[神三元](https://juejin.im/post/5dbebbfa51882524c507fddb#heading-35)

（主观）和上面实现的快速排序的时间复杂度应该是一样，上面理解起来稍微简单一点

```js
const quickSort = (a, from, to) => {
  //...
  // 上面我们拿到了thirdIndex
  // 现在我们拥有三个元素，from, thirdIndex, to
  // 为了再次确保 thirdIndex 不是最值，把这三个值排序
  [a[from], a[thirdIndex], a[to - 1]] = _sort(a[from], a[thirdIndex], a[to - 1]);
  // 现在正式把 thirdIndex 作为哨兵
  let pivot = a[thirdIndex];
  // 正式进入快排
  let lowEnd = from + 1;
  let highStart = to - 1;
  // 现在正式把 thirdIndex 作为哨兵, 并且lowEnd和thirdIndex交换
  let pivot = a[thirdIndex];
  a[thirdIndex] = a[lowEnd];
  a[lowEnd] = pivot;
  
  // [lowEnd, i)的元素是和pivot相等的
  // [i, highStart) 的元素是需要处理的
  for(let i = lowEnd + 1; i < highStart; i++) {
    let element = a[i];
    let order = comparefn(element, pivot);
    if (order < 0) {
      a[i] = a[lowEnd];
      a[lowEnd] = element;
      lowEnd++;
    } else if(order > 0) {
      do{
        highStart--;
        if(highStart === i) break;
        order = comparefn(a[highStart], pivot);
      }while(order > 0)
      // 现在 a[highStart] <= pivot
      // a[i] > pivot
      // 两者交换
      a[i] = a[highStart];
      a[highStart] = element;
      if(order < 0) {
        // a[i] 和 a[lowEnd] 交换
        element = a[i];
        a[i] = a[lowEnd];
        a[lowEnd] = element;
        lowEnd++;
      }
    }
  }
  // 永远切分大区间
  if (lowEnd - from > to - highStart) {
    // 继续切分lowEnd ~ from 这个区间
    to = lowEnd;
    // 单独处理小区间
    quickSort(a, highStart, to);
  } else if(lowEnd - from <= to - highStart) {
    from = highStart;
    quickSort(a, from, lowEnd);
  }
}
```



## 参看文章

1. [JavaScript专题之解读 v8 排序源码](https://juejin.im/post/59e80dc6f265da432a7aaf15#heading-11)
2. [第十七篇: 能不能实现数组sort方法？](https://juejin.im/post/5dbebbfa51882524c507fddb#heading-35)

