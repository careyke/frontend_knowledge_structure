# 深入理解ES Module

ES Module是es6开始js原生支持的模块化方案。在设计上是尽量静态化的

ES Module暂时并不是所有的浏览器都支持，所以使用的时候需要用babel工具来进行转化，然后才能执行

## 1. ES Module的特点

### 1.1 ES Module是一种静态的模块化方案

==ES Module方案中，模块之间的依赖关系在代码编译期就已经处理完成，会在依赖的变量之间建立一种引用关系，当代码中使用到这些变量的时候，会根据这种引用关系去依赖模块的执行上下文中找到对应的变量并返回结果。==

而之前文章中提到的其他的模块化方案中，模块之间的依赖关系都是运行时动态确定的。

```js
//a.js
export let x = 1;

//b.js
import {x} from './a.js';
console.log(x); //1
```

分析：

**b.js 通过import语句引入 a.js 中的x变量，在代码编译的时候，编译器会识别import和export关键字，将两个文件中的x变量联系起来，建立引用关系。当使用 x 变量的时候，会通过引用关系去a.js中动态获取x变量此时的值。**

### 1.2 ES Module中的模块不是一个对象

在CommonJS、AMD和CMD的实现代码中，内部都有一个模块管理系统，每个模块都是也够对象，需要使用模块系统来管理和缓存。模块之间的联系也只能在运行时建立，因为模块对象需要在运行时才能生成。

但是对于ES Module来说，**模块之间的关系编译期就可以建立，不需要在运行期动态创建模块和管理模块，所以在设计的时候，就没有把每个模块设计成一个对象。**

### 1.3 import和export导入导出的不是对象

==虽然在import关键字和export关键字后面有一个大括号，但是其实并不是一个对象，而是一个接口列表，内部的接口和模块内部的变量是一对一的关系。以此来建立引用关系。==

```js
function add(){}
export {add:add} //会报错

//只能使用
export function add(){}
```

export语句的特点：

**export关键字后面只能是变量声明语句，或者是接口列表。不能是值**

```js
export var a = 1; //ok
export {a}; //ok

export a //会报错，因为这里是直接导出1 而不是接口
```

export default语句的特点:

**export default语句后面可以是值，function或者class，但是不能使变量声明语句**

```js
var a = 1;
export default {a};  //ok, 这里{a}并不是接口列表，而是一个对象值
export default a; //ok
export default function add(){}

export default const b = 3;  //会报错
```

==因为export default语句内部会创建给一个default变量作为接口，并将default变量赋值为后面的值。所以后面不能是变量声明语句==

```js
var a = 1;
export default a; 
//等同于
export {a as default}
```

import语句的特点：

1. import语句具体提升效果，会提升到整个模块的头部

   ```js
   var a = b;
   import {b} from './b.js';
   ```

2. import后面的也是一个接口列表，和export中的是一一对应的

### 1.4 ES Module可以支持片段引入

ES Module是静态引入模块的，接口之间也是静态建立引用关系的。所以**只会引入依赖模块中的部分接口，而不会将模块的所有接口都引入进来。**

这一点和之前的模块化方案也不是一样的。**之前模块化方案中会先将依赖模块中整个导出的对象引入，然后通过结构赋值来使用其中的一些接口**

```js
var {add, get} = require('./a.js');//看起来只引入了add和get接口

//实际上等同于
var obj = require('./a.js'); //这里会将所有接口引入
var add = obj.add;
var get = obj.get;
```

ES Module中：

```js
import {add,get} from './a.js';
//这里只会将add和get变量和a.js中的add和get关联在一起，其他的接口并不会关联过里
```

## 2. ES Module中的循环引用分析

ES Module是在编译的时候建立模块之间变量的引用关系，再发生循环引用的时候也按照这个原理分析

```js
// a.mjs
import {bar} from './b';
console.log('a.mjs');
console.log(bar);
export let foo = 'foo';

// b.mjs
import {foo} from './a';
console.log('b.mjs');
console.log(foo);
export let bar = 'bar';
```

==首先在js引擎开始执行代码之前，a.mjs和b.mjs之间的变量引用关系已经建立好了==。然后执行a.mjs

1. 开始执行a.mjs的时候，发现一个import语句，**并且b.mjs没有被执行过，不存在执行上下文，js引擎会先执行b.mjs**
2. **执行b.mjs文件的时候，发现会从a模块中获取一个foo变量，此时a模块处于正在执行的状态，存在执行上下文，所以不会返回a.mjs中去执行，而是认为a.mjs中已经存在foo变量，继续执行后面的代码。**
3. 当执行到 `console.log(foo)` 的时候，会根据索引去**a.mjs的执行上下文**中获取foo变量，才发现a.mjs中并没有声明foo。所以会报错

输出为：

```js
b.mjs
ReferenceError: foo is not defined
```

**之所以会报错是因为无法在声明语句之前访问let声明的变量，要解决这个问题，可以将let声明改后var声明**

```js
// a.mjs
import {bar} from './b';
console.log('a.mjs');
console.log(bar);
export var foo = 'foo';

// b.mjs
import {foo} from './a';
console.log('b.mjs');
console.log(foo);
export var bar = 'bar';
```

输出结果

```js
b.mjs
undefined
a.mjs
bar
```

### 2.1 整个过程可以使用babel打包之后的模拟代码来辅助理解

```js
(function (modules) { // webpackBootstrap
  // The module cache
  var installedModules = {};
  /******/
  // The require function
  function __webpack_require__(moduleId) {
    /******/
    // Check if module is in cache
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }
    // Create a new module (and put it into the cache)
    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };
    /******/
    // Execute the module function
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    /******/
    // Flag the module as loaded
    module.l = true;
    /******/
    // Return the exports of the module
    return module.exports;
  }
  // define getter function for harmony exports
  __webpack_require__.d = function (exports, name, getter) {
    if (!__webpack_require__.o(exports, name)) {
      Object.defineProperty(exports, name, { enumerable: true, get: getter });
    }
  };
  /******/
  // define __esModule on exports
  __webpack_require__.r = function (exports) {
    if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
    }
    Object.defineProperty(exports, '__esModule', { value: true });
  };
  // Load entry module and return exports
  return __webpack_require__(__webpack_require__.s = "./esModule/index.js");
})
  /************************************************************************/
  ({

/***/ "./esModule/index.js":
/*!***************************!*\
  !*** ./esModule/index.js ***!
  \***************************/
/*! exports provided: foo */
/***/ (function (module, __webpack_exports__, __webpack_require__) {

        "use strict";
        __webpack_require__.r(__webpack_exports__);
/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "foo", function () { return foo; });
/* harmony import */ var _lib__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./lib */ "./esModule/lib.js");

        console.log('a.mjs');
        console.log(_lib__WEBPACK_IMPORTED_MODULE_0__["bar"]);
        var foo = 'foo';


        /***/
}),

/***/ "./esModule/lib.js":
/*!*************************!*\
  !*** ./esModule/lib.js ***!
  \*************************/
/*! exports provided: bar */
/***/ (function (module, __webpack_exports__, __webpack_require__) {

        "use strict";
        __webpack_require__.r(__webpack_exports__);
/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "bar", function () { return bar; });
/* harmony import */ var _index__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./index */ "./esModule/index.js");

        console.log('b.mjs');
        console.log(_index__WEBPACK_IMPORTED_MODULE_0__["foo"]);
        var bar = 'bar';

        /***/
})

  });
```

着重看两个文件中的exports和require的顺序