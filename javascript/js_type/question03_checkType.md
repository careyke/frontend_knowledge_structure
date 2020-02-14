## js中类型检测的方法有哪些？

### 1.typeof
typeof是js内部实现的判断类型的方法。但是这个方法并不能满足所有的需求。
#### 1.1对于基本类型
```js
typeof null // object  异常！！！

typeof undefined // undefined

typeof 123 // number

typeof '123' //string 

typeof true // boolean

typeof Symbol('123') // symbol

typeof 123n  // bigint
```  
从运行结果可以看出，**typeof无法准确检测null类型**，但是这是一个历史的bug。

#### 1.2 对于对象类型
**在对对象类型做类型检测的时候，期望的是可以判断当前对象具体属于哪一个内置对象，比如Function, Array, Date等等**。而不是所有的对象都返回一个object，这样检测的意义不大。

typeof在对象类型中的表现
```js
let o = {a:1};
typeof o // object

let o1 = function(){}
typeof o // function

typeof [1,2,3]  //object

typeof new Date()  //object

typeof new Number(123) //object

typeof new String('123')  //object
```  
可以看出，**对于对象类型，typeof只能判断出Function类型，其他所有对象统一返回object。**

#### 1.3 总结
1. **typeof只能判断除null之外的基本类型**
2. **typeof对于对象类型，只能判断function类型**

### 2.instanceof
#### 2.1 判断原理
A instanceof B判断的原理是：**判断在 A 的原型链上是否有 B 的原型对象。**
注意：
1. B一定要是**对象类型或者构造函数**，否则报错
2. A可以是任意类型。**但是如果A是基本类型，会直接返回false，因为基本类型没有原型链。**

```js
[1,2,3] instanceof Array //true

let f = function(){}
f instanceof Function //true

let o = {a:1}
o instanceof Object //true

1 instanceof Number //false
new Number(1) instanceof Number //true

null instanceof null // Uncaught TypeError  null不是对象
undefined instanceof undefined // Uncaught TypeError  undefined不是对象
```  
可以看出，instanceof可以判断出对象具体属于哪一个子类型，但是无法判断基本类型的数据。

#### 2.2 总结
1. **instanceof可以细分对象属于哪一个子类型。**
2. **instanceof无法判断基本类型的数据（没有装箱操作**）


### 3.Object.prototype.toString()
#### 3.1 检测原理
在对象类型的数据中，有一个内部属性[[Class]]。**这个属性表示创建这个对象的内部原生构造函数**。**对于null和undefined，虽然它们没有内建原生构造函数，但是仍然可以返回Null和Undefined**

这个属性只有一个方法可以访问，就是通过Object.prototype.toString()这个方法。

```js
Object.prototype.toString.call(1)  //[object Number]

Object.prototype.toString.call('1')  //[object String]

Object.prototype.toString.call(true)  //[object Boolean]

Object.prototype.toString.call(1n)  //[object BigInt]

Object.prototype.toString.call(Symbol(1))  //[object Symbol]

Object.prototype.toString.call(null)  //[object Null]

Object.prototype.toString.call(undefined)  //[object Undefined]

Object.prototype.toString.call({a:1})  //[object Object]

Object.prototype.toString.call([1,2])  //[object Array]

Object.prototype.toString.call(function a(){})  //[object Function]
```  
可以看出，**这个方法所有的类型都可以检测出来**。

由于[[Class]]属性只存在对象类型中，所以在对基本类型调用Object.prototype.toString()的时候，内部包含一个**装箱操作**，会将基本类型转化成其封装对象。

### 4.constructor
js中的对象可以通过constructor属性获取当前对象的构造函数。根据构造函数可以判断数据的类型。

**constructor属性并不存在与当前对象本身，而是存在于当前对象的构造函数的原型对象上**。也就是在当前对象的原型链上，因此可以获得

```js
let a = 1;
a.constructor === Number // true

let a = '1';
a.constructor === String //true

let a = true;
a.constructor === Boolean //true

let a = 1n;
a.constructor === BigInt //true

let a = Symbol(1);
a.constructor === Symbol //true

let a = null;
a.constructor // Uncaught TypeError 因为null没有原生构造函数

let a = undefined;
a.constructor // Uncaught TypeError 因为undefined没有原生构造函数

[1,2].constructor === Array  //true

(function f(){}).constructor === Function // true
```  
从运行结果可以看出，**使用constructor方法可以判断除null和undefined的所有类型**。

**其中对于基本类型来说，获取constructor的时候，有一个隐式的装箱操作。**

之所以不能判断null和undefined，原因就是这两个类型没有对应的原生构造函数。

#### 4.1 constructor方法的缺点
考虑下面的情况：
```js
function A(){}

var b = new A();

b.constructor === A  // true
```  
这种请况下，**b.constructor为A，并不是原生的内置类型。而且constructor是一个可以人为修改的属性，使用这种方式判断还是有风险的。**

#### 4.2 constructor总结
1. 除了null和undefined，其他类型都可以判断
2. 使用constructor获取类型，有时候不直观，获取不到对应的内置类型
3. constructor属性有被人为修改的风险

### 5.总结
综合上面4种方案，**最优的判断类型的方案就是使用Object.prototype.toString()这个方法**。


### 6.封装一个判断类型的方法
基本类型使用typeof（注意null），对象类型使用Object.prototype.toString()
```js
const classType = [
    'Boolean', 'Number', 'String', 'Function', 'Array', 'Date', 'RegExp', 'Object', 'Error', 'Null', 'Undefined'
];

const class2Type = (function () {
    let classObj = {}
    classType.forEach((item) => {
        classObj['[object ' + item + ']'] = item.toLowerCase(); //和typeof的返回值保持一致，都用小写
    })
    return classObj;
})()

export default function checkType(variable) {
    if (variable == null) { //兼容IE6，null 和 undefined 会被 Object.prototype.toString 识别成 [object Object]！
        return variable + '';
    }
    return typeof variable === 'object' ? class2Type[Object.prototype.toString.call(variable)] : typeof variable;
}
```  

### 7.其他特殊类型的判断方式
1. 数组可以使用：Array.isArray()
















