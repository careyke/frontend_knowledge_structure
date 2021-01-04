# 常用的排序算法

交替位置的方法，多处使用

```js
const exchange = (arr, i,j) => {
  let t = arr[i];
  arr[i] = arr[j];
  arr[j] = t;
}
```

粗略测试性能的方法：

```js
function prefTest(){
  const arr=[];
  for(let i=0;i<10000;i++){
    arr.push(Math.floor(Math.random()*10000));
  }
  return function(fn){
    console.time('time');
    fn(arr.concat([]));
    console.timeEnd('time');
  }
}
var pref = prefTest();
```

**算法稳定性：**

- 稳定：如果a原本在b前面，而a=b，排序之后a仍然在b的前面；
- 不稳定：如果a原本在b的前面，而a=b，排序之后a可能会出现在b的后面；



## 1. 冒泡排序

### 1.1 算法描述

1. **从起始位置开始，依次比较相邻位置的两个值，如果第一个比第二个大**，就交换位置。如此执行一遍就可以将最大值的位置放对
2. 将已经确定的位置除外，循环执行第一步，就可以将所有数组按从小到大排列起来

### 1.2 代码实现

```js
function bubbleSort(arr){
  let len = arr.length;
  for(let i=0; i<len-1; i++){
    for(let j=0; j<len-1-i; j++){
      if(arr[j] > arr[j+1]){
        exchange(arr,j,j+1);
      }
    }
  }
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
bubbleSort(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(bubbleSort);
// time: 114.116943359375ms
```

### 1.3 冒泡排序优化

上面的代码中实现的冒泡排序是正确的，但是并不是最优的。考虑下面这种情况

```js
var arr = [1,3,2,4,5,6,7,8];
// 4 5 6 7 8 的顺序是对的，不应该每次都去进行比较
```

当使用上面的算法来排序的时候，即使顺序是对的，每次仍然会去比较，其实是**徒增消耗。**

优化的手段:

**采用一个指针pos来记录每次比较中最后一次做交换操作的位置的索引。后面不做交换操作，说明后面顺序都是对的。下次遍历的时候到pos位置即可，不需要继续遍历后面的元素**

```js
function bubbleSortByPosWithFor(arr){
  const len = arr.length;
  let pos = len-1;
  for(;pos>0;){  // 这里如果使用while循环，会导致性能下降，检测过。for循环比while循环更高效
    let t = 0;
    for(let i=0;i<pos;i++){
      if(arr[i] > arr[i+1]){
        exchange(arr, i, i+1);
        t = i;
      }
    }
    pos = t;
  }
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
bubbleSortByPosWithFor(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(bubbleSortByPosWithFor);
// time: 109.612060546875ms  优化的效果不是特别明显，因为增加了一些赋值操作
```

### 1.4 算法复杂度分析

- 空间复杂度：O(1)

- 时间复杂度
  - 最好的情况：O(n)。数据刚好正序
  - 最差的情况：O(n^2)。数据刚好反序
  - 平均情况：O(n^2)
- 稳定性：稳定



## 2. 选择排序

### 2.1 算法描述

1. **每次遍历乱序区间，找到最小值的索引，然后将乱序区间的第一个索引和最小值的索引交换位置**
2. 乱序区间跨度减1，重复第一步

### 2.2 代码实现

```js
function selectSort(arr){
  const len = arr.length;
  let minIndex = 0;
  for(let i=0;i<len-1;i++){
    minIndex = i;
    for(let j = i+1;j<len;j++){
      if(arr[j] < arr[minIndex]){
        minIndex = j;
      }
    }
    exchange(arr, i, minIndex);
  }
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
selectSort(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(selectSort);
// time: 69.014892578125ms
```

相比于冒泡排序来说，**选择排序有效的减少了交换数据的操作次数，性能相对比冒泡排序要好一点**

### 2.3 算法复杂度分析

- 空间复杂度：O(1)
- 时间复杂度，任何情况下都是O(n^2)
- 稳定新：不稳定，比如[1,2,2,3,1]



## 3. 插入排序

### 3.1 算法分析

**插入排序将数组分成两个部分：有序数组[1] 和无序数组 [2,...,n]。遍历无序数组，每次将取出的元素插入有序数组的正确位置中，直到遍历完成。**

==将元素插入有序数组的操作，也需要遍历有序数组。这一步操作也是优化的关键。==

### 3.2 代码实现

```js
function insertSort(arr){
  const len = arr.length;
  for(let i=1;i<len;i++){
    let pos = i;
    for(let j=i-1;j >= 0;j--){
      if(arr[pos]<arr[j]){
        exchange(arr,pos,j);
        pos = j;
      }
    }
  }
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
insertSort(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(insertSort);
// time: 98.84521484375ms
```

### 3.3 插入排序优化

可以优化的点：

1. **插入之前先和有序数组的最后一个值比较一下大小，如果插入值比较大，则不需要遍历有序数组插入了，可以减少循环的次数**
2. 插值的方式**不采用值交换**的方式，可以**采用值覆盖**的形式

```js
function insertSortCover(arr){
  const len = arr.length;
  for(let i = 1;i<len;i++){
    let value = arr[i];
    let j;
    for(j = i-1; j>=0 && arr[j] > value; j--){  //这里使用while循环，会降低性能，for循环性能更好
      arr[j+1] = arr[j];
    }
    arr[j+1] = value;
  }
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
insertSortCover(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(insertSortCover);
// time: 33.421875ms
```

### 3.4 算法复杂度分析

- 空间复杂度：O(1)
- 时间复杂度：
  - 最佳情况：O(n)，升序的时候
  - 最差情况：O(n^2)，降序的时候
  - 平均情况：O(n^2)
- 稳定性：稳定的



## 4. 快速排序

### 4.1 算法分析

1. 首先选一个基准值p，然后将小于p的值放在p的左右，大于p的值放在p的右边
2. 然后将左右两个数组也按照步骤1进行操作
3. 重复步骤1和2，**直到数组的左边界不小于右边界，即数组的元素个数小于2**

### 4.2 代码实现

```js
function quickSort(arr){
  const len = arr.length;
  if(len<2) return arr;
  const pivot = len >>> 1;
  const pivotValue = arr[pivot];
  const leftArr=[];
  const rightArr=[];
  for(let i=0;i<len;i++){
    if(i === pivot) continue;  //不把基准值独立出来，会照成无限递归
    if(arr[i] <= pivotValue){
      leftArr.push(arr[i]);
    }else{
      rightArr.push(arr[i]);
    }
  }
  return [...quickSort(leftArr), pivotValue, ...quickSort(rightArr)];
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
quickSort(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(quickSort);
// time: 9.218017578125ms
```

### 4.3 快速排序优化

上面的实现中，由于用了新的数组来存储基准值两边的数据，所以空间复杂度比较大，约为O(n^2)。

**可以优化的点：**

1. 在当前数组中移动元素，降低空间复杂度
2. 基准值的选定也是一个可以优化的点，属于比较极致的优化。V8中的sort方法就对基准值的选取做了很多优化

```js
function quickSortByInPlace(arr){
  const len = arr.length;
  if(len<2) return arr;
  
  function quickSort(arr, left, right){
    if(left < right){
    	const pivotValue = arr[right]; //挑选right的基准值
      let p = left-1;
      for(let i=left; i<=right; i++){
        if(arr[i] <= pivotValue){
          p++;
          exchange(arr, p, i);
        }
      }
      quickSort(arr, left, p-1);
      quickSort(arr, p+1, right);
    }
  }
  
  quickSort(arr, 0, arr.length-1);
  return arr;
}

var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
quickSortByInPlace(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]


pref(quickSortByInPlace);
// time: 1.1650390625ms
```

小技巧：

==这里之所以挑选 right 为基准点，是因为遍历函数的时候是从左到右遍历的，将不大于基准值的数据都交换到了数组的左边，而right刚好是最后一个符合条件的值，所以right的值会被替换到中间，左边全部不大于它，右边全都大于它。==

其实也可以选取left为基准点，然后从右到左遍历数据，必须确保基准值是最后一个遍历，也是最后一个符合要求被移动的数据

### 4.4 算法复杂度分析

- 空间复杂度：O(logn)，因为递归的次数大概是O(logn)，每个递归的函数中空间复杂度是O(1)
- 时间复杂度：
  - 最佳情况：O(nlog(n))
  - 最差情况：O(n^2) 。当刚好正序或者反序的时候
  - 平均情况：O(nlogn)
- 稳定性：不稳定



## 5. 归并排序

### 5.1 算法分析

1. **将长度为n的数组分成两个长度为n/2的子序列，然后按其中元素的大小将两个子序列合并为一个数组。**
2. **将这两个子序列也都分别执行归并排序，直到子序列的长度为1**
3. 最终会将数组分成n个子序列来比较

### 5.2 代码实现

```js
function mergeSort(arr){
  const len = arr.length;
  if(len < 2){
    return arr;
  }
  const mid = len >>> 1;
  const arrA = arr.slice(0,mid);  // 拆分
  const arrB = arr.slice(mid);
  return merge(mergeSort(arrA),mergeSort(arrB));
}
function merge(arrA, arrB){  //合并
  let result=[];
  while(arrA.length && arrB.length){
    if(arrA[0] > arrB[0]){
      result.push(arrB.shift()); // 这里直接使用了数组api，内部会遍历数组。是一个可以优化的点，采用双指针
    }else{
      result.push(arrA.shift());
    }
  }
  if(arrA.length>0){
    result = result.concat(arrA);
  }
  if(arrB.length>0){
    result = result.concat(arrB);
  }
  return result;
}


var arr=[3,44,38,5,47,15,36,26,27,2,46,4,19,50,48];
mergeSort(arr);
// [2, 3, 4, 5, 15, 19, 26, 27, 36, 38, 44, 46, 47, 48, 50]

pref(mergeSort);
// time: 10.866943359375ms
```

### 5.3 算法复杂度分析

- 空间复杂度：O(nlogn)  空间复杂度比较高，**递归深度大概是logn，每层都需要O(n)**
- 时间复杂度：是**稳定**的，都是O(nlogn)，每层递归的时间复杂度都是
- 稳定性：稳定



## 总结

| 排序方法 | 时间复杂度 | 最差时间复杂度 | 最好时间复杂度 | 空间复杂度 | 稳定性 |
| -------- | ---------- | -------------- | :------------- | ---------- | ------ |
| 冒泡排序 | O(n^2)     | O(n^2)         | O(n)           | O(1)       | 稳定   |
| 选择排序 | O(n^2)     | O(n^2)         | O(n^2)         | O(1)       | 不稳定 |
| 插入排序 | O(n^2)     | O(n^2)         | O(n)           | O(1)       | 稳定   |
| 快速排序 | O(nlogn)   | O(n^2)         | O(nlogn)       | O(logn)    | 不稳定 |
| 归并排序 | O(nlogn)   | O(nlogn)       | O(nlogn)       | O(nlogn)   | 稳定   |

1. 冒泡排序和快速排序需要**替换运算**，其他方式可以不要
2. 快速排序的性能不一定比插入排序高，但是数据量大时，表现更好
3. for循环比while的性能更好



## 参考文章

1. [十大经典排序算法总结（JavaScript描述）](https://juejin.im/post/57dcd394a22b9d00610c5ec8#heading-19)

