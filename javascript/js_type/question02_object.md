## js对象类型
**js中的object类型是引用类型，对象的内容保存在堆内存中，栈内存中存放的是指向堆内存中对象内容的指针。**

对象是一个存放多个键值对的集合。其中键叫做属性名，值叫做属性值。

### 1.对象的创建
js中一共有3种方式创建对象：
1. 字面量方式
```js
let obj = {a:1,b:2}
```  

2. 构造函数的方式
```js
let obj = new Object({a:1,b:2});
```  

3. Object.create(proto,[propertiesObject])的方式
```js
let obj = Object.create(Object.prototype,{a:{value:1},b:{value:2}});
```  

对比这三种创建方式可以看出，**字面量的创建方式是最优的**。

### 2.对象的属性
**对象中属性名只能是string或symbol类型，属性值可以是任何类型。**
```js
let obj = {1:2}

obj['1'] = 2;
obj[1] = 2; //之所以可以这样使用，因为内部有一个隐式类型转化将数字1转化成'1'
```  
**在堆内存中，对象中的属性值一般不会存放在对象容器内部，存在对象容器内部的是属性名，它们就像指针一样，指向属性值真正存储的位置。**

### 3.属性描述对象
**object中每一个属性都对应一个属性描述对象，用来设置当前属性本身的一些特性**。比如属性值(value)、可写(writable)、可枚举(enumerable)和可配置(configurable)。创建普通属性的时候，属性描述对象取默认值

#### 3.1 修改属性描述对象
object类型中提供defineProperty()来创建一个新属性或者修改已有属性的描述对象。
```js
let obj={};
Object.defineProperty(obj,'a',{
  value:2,
  writable:true,
  enumerable: true,
  configurable: true
})

obj.a  //2
```  

#### 3.2 writable
决定属性是否可写，也就是是否可以修改属性的值。
```js
let obj={};
Object.defineProperty(obj,'a',{
  value:2,
  writable:false
})

obj.a = 3;
obj.a // 2
```  
可以看出当writable为false时，无法修改属性的值。

#### 3.3 configurable
只要属性是可配置的，就可以使用 defineProperty(..) 方法来修改属性描述对象。放过来如果属性是不可配置的，那么就不能使用defineProperty来修改属性描述对象。
```js
let obj={};
Object.defineProperty(obj,'a',{
  value:2,
  configurable: false
})
obj.a //2

Object.defineProperty(obj,'a',{
  value:5,
  configurable: true
})  //会报错

Object.defineProperty(obj,'a',{
  writable: false
})

obj.a = 10;
obj.a //2
```  
configurable:false的属性的特点：
- configurable:false是一个**不可逆**的操作
- 可以将属性的writable由true设置成false，反向则不行
- delete该属性会失效，无法删除。

#### 3.4 enumerable
描述属性是否可以枚举。普通方式定义的变量，enumerable默认是true。通过defineProperty定义的变量，enumerable默认是false
可枚举属性的特点：
- **可以使用Object.keys()获取当前属性值**
```js
let obj={};
Object.defineProperty(obj,'a',{
  value:2,
  enumerable: true
})
Object.defineProperty(obj,'b',{
  value:2,
  enumerable: false
})
Object.keys(obj); // ['a']
```  
- **可以使用for..in遍历到该属性**
```js
let obj={};
Object.defineProperty(obj,'a',{
  value:2,
  enumerable:true
})
Object.defineProperty(obj,'b',{
  value:2,
  enumerable: false
})
for(let k in obj){
  console.log(k);
}
//a  没有b 
```  

#### 3.5 getter和setter
js中可以使用getter和setter来修改对象属性的默认读写操作。在读取属性的时候调用getter函数，在写入属性的时候调用setter函数。

**当属性中设置了getter和setter的时候，属性描述对象中的value和writable会被忽略，同时设置了会报错**

```js
let obj={};
Object.defineProperty(obj,'a',{
  get: function(){return this._a_},
  set: function(n){this._a_ = n}
})

obj.a = 3;
obj.a //3
```  

### 4.冻结属性和对象
#### 4.1 常量对象 —— 不能配置、不能写
使用writable:false 和 configurable:false 可以创建常量属性

#### 4.2 禁止扩展 —— 不能增加
使用 Object.prevent Extensions(..)可以使一个对象禁止扩展新属性
```js
var myObject = {a:2};
Object.preventExtensions( myObject );
myObject.b = 3; 
myObject.b; // undefined
```  

#### 4.3 密封 —— 不能增加、不能配置、可写
Object.seal(..) 会创建一个“密封”的对象，**这个方法实际上会在一个现有对象上调用 Object.preventExtensions(..) 并把所有现有属性标记为 configurable:false。**

密封之后不仅不能添加新属性，也不能重新配置或者删除任何现有属性(虽然可以 修改属性的值)

#### 4.4 冻结 —— 不能增加、不能配置、不能写
bject.freeze(..) 会创建一个冻结对象，**这个方法实际上会在一个现有对象上调用 Object.seal(..) 并把所有“数据访问”属性标记为 writable:false，这样就无法修改它们 的值。**

这里讲的所有冻结对象的方法都是浅层的，如果需要深层冻结，可以是使用递归的形式。


### 5.判断属性是否在对象中
1. in操作符 —— **会将原型链上对象的属性也包含进去。**
2. hasOwnProperty —— 判断属性是否在当前对象上
3. Object.key() —— 获取对象上所有**可枚举**的属性
4. Object.getOwnPropertyNames(..) —— 获取对象上**所有的**属性，无论是否可枚举

### 6.遍历对象
普通对象的遍历：只能使用for..in来遍历属性名。但是需要注意的是，**for..in会遍历原型链上对象的属性，所以需要使用hasOwnProperty()方法过滤。**

对于数组来说，遍历的方式有很多种：
1. for循环
2. forEach
3. for..of 
4. for..in




