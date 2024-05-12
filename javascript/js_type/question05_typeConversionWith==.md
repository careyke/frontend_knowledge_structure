## 隐式转换 : ==号两边的数据的类型转换规则
在js中，==运算符是被人诟病最多的，原因是 **==运算符可以比较两个不同类型的值，不同类型的数据会先隐式转换成相同类型然后再比较**。

然而==运算符两边的数据类型转换的规则是比较杂乱的，所以很多时候会出现意想不到的结果。

这里就是要总结一下==运算符两边数据的隐式类型转换规则

### 1.如果两边类型相同
==两边的数据如果类型相同，直接比较值就可以了，无需类型转化

```js
'abc' == 'abc'; //true
12 == 1 //false

var o = {a:1};
a = o;
b = o;
a == b  //true  指向同一个内存就相等
```
#### 1.1 特殊的值比较
```js
NaN == NaN //false
+0 == -0  //true
```

### 2.string和number类型之间的相等比较
1. 如果 Type(x) 是数字，Type(y) 是字符串，则返回 x == ToNumber(y) 的结果
2. 如果 Type(x) 是字符串，Type(y) 是数字，则返回 ToNumber(x) == y 的结果

总结来说就是当string和number比较的时候，**string转换number**，然后再比较。
```js
23 == '23' //true
23 == '2'  //false
```

### 3.其他类型和boolean类型之间的相等比较
1. 如果 Type(x) 是布尔类型，则返回 ToNumber(x) == y 的结果
2. 如果 Type(y) 是布尔类型，则返回 x == ToNumber(y) 的结果

总结来说就是**boolean转化成number类型**然后比较。这个比较过程可能不止这一次类型转换。
```js
true == '42' //false
//相当于
1 == '42' //false

false == '42' //false
```

### 3.null和undefined之间的相等比较
1. 如果 x 为 null，y 为 undefined，则结果为 true
2. 如果 x 为 undefined，y 为 null，则结果为 true
3. 如果 x 为 null或undefined，y是除了null和undefined之外的类型的值，则结果为false

```js
null == undefined  //true

1 == null  //false

0 == undefined // false
```

### 4. 对象与非对象之间的相等比较
1. 如果 Type(x) 是字符串或数字，Type(y) 是对象，则返回 x == ToPrimitive(y) 的结果
2. 如果 Type(x) 是对象，Type(y) 是字符串或数字，则返回 ToPromitive(x) == y 的结果

总结就是**对象会先转换成基本类型**，然后再来做相等比较

```js
[13] == 13;  // true
//转换成
'13' == 13  // true
```

### 5.一些极端的例子
#### 5.1 如何让 a == 1 && a == 2 成立？
这个问题考的就是ToPrimitive操作，基本类型肯定是不成立的，只有可能是对象类型，因为对象类型ToPrimitive操作的时候会执行`valueOf或toString`，所以通过一些操作可能成立。
```js
let i = 1;
const obj = {
  valueOf:function(){
    return i++;
  }
}
obj == 1 && obj == 2; //true
```
虽然这种表达式是可以成立的，但是**很不推荐使用**。


#### 5.2 [] == ![]
```js
[] == ![]  //true
结果虽然匪夷所思，但其实也是符合规则的

//第一步 ![]是boolean类型  为false
[] == false

//第二步 []做ToPrimitive操作
'' == false

//第三步 如果有boolean 先将boolean转换成number
'' == 0

//第四部 string -> number
0 == 0  //true
```



## 其他比较运算符的隐式类型转换规则

除了==运算符之外，其他的比较运算符在比较的过程中也是有类型转化的。因为也会发生不同类型之间的比较。

这里重点介绍一下 < 运算符的类型转换规则

### 1. < 运算符的类型转换规则
1. 首先 < 两边的数值需要**先进行ToPrimitive操作**，变成基本类型。（**本质上基本类型的值之间的比较**）
2. **如果两个数值都是字符串，则按照字符串的比较规则比较大小**
3. **否则两个操作数都需要执行ToNumber操作，转换成数字之后再比较**，过程中有报错直接报错

```js
[23] < 45 //true

[23] < [0,1,2] 
转化成：
'23' < '0,1,2' //false
```

> 其他运算符的类型转换规则应该和 < 是一样的

### 2. <= 比较运算符
先看一段奇怪的代码：
```js
var a={x:12}
var b={x:13}

a == b //false
a === b //false

a < b //false
a > b // false

a <= b //true !!! 奇怪
a >= b //true !!! 奇怪
```
为什么会出现这种结果呢？

这里之所以出现这种情况是因为 **==和其他比较运算符不一样，==可以直接比较两个对象，但是其他比较运算符需要转化成基本类型之后再比较**。

上面 `a<=b` 最终比较的是 `"[object Object]" <= "[object Object]"`
