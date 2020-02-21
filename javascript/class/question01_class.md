## 谈谈JS中的类的本质是什么
### 类是什么？
类是一种**代码的组织结构形式** —— 一种在软件中对真实世界中问题领域的建模方法。

在面向对象的编程语言中，比如Java，类是代码的基本组织形式，是无处不在的。

类的特点：
1. 封装
2. 继承
3. 多态

### JS中的类
在ES6之间，js中是没有class的概念的。但是js一直在模拟类的用法，其中new关键字的出现就是js模拟类的产物。还有就是原型链的出现也是为了模拟类的实例化和继承。

但是**js中是不存在类的，js中的类都是靠函数模拟得来的**。ES6中出现的class只是一种语法糖，其底层的实现还是使用函数来完成的。（后面文章会将如何用function模拟class）

### 总结
1. **js中不存在真正的类**
2. **js中为了使用类的设计模式而利用函数来模拟了类**
3. **js中类的本质其实就是函数**

### babel实现Class
ES6 class
```js
class Animal{
  constructor(name){
  	this.name = name;
  }
  sayHi(){
  	console.log('Hi');
  }
}
```  

babel转化之后的代码
```js
"use strict";

function _instanceof(left, right) { 
  if (right != null && typeof Symbol !== "undefined" && right[Symbol.hasInstance]) { 
    return !!right[Symbol.hasInstance](left); 
  } else { 
    return left instanceof right; 
  } 
}

function _classCallCheck(instance, Constructor) { if (!_instanceof(instance, Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var Animal =
/*#__PURE__*/
function () {
  function Animal(name) {
    _classCallCheck(this, Animal);

    this.name = name;
  }

  _createClass(Animal, [{
    key: "sayHi",
    value: function sayHi() {
      console.log('Hi');
    }
  }]);

  return Animal;
}();
```  
可以看出class的内部实现就是使用的function来模拟的。

