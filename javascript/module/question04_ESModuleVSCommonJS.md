# CommonJS和ES Module之间的区别

## 1. CommonJS VS ES Module

### 1.1 CommonJS是运行时加载依赖模块，ES Module是编译时建立引用关系

- **CommonJS中，每个模块都是一个对象**。需要在运行时先执行模块代码生成一个模块对象，然后从模块对象中引入方法。

- **ES Module中，模块不是一个对象**。引入和导出的接口会在编译期的时候建立引用关键，需要使用的时候，根据引用关系去**依赖模块的执行上下文**中动态获取值或方法。

### 1.2 CommonJS模块输出的值的拷贝，ES Module模块输出的是值的引用

CommonJS模块导出的是值的拷贝，也就是说，一旦输出值之后，模块内部这个变量发生变化并不会影响到这个输出值

```js
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  counter: counter,
  incCounter: incCounter,
};

//a.js
const {counter, incCounter} = require('./lib.js');
console.log(counter); // 3
incCounter();
console.log(counter); // 3
```

lib模块中的counter变量实际变成了4，但是输出的对象中counter属性仍是3

ES Module中，模块输出的值是一个引用。**在编译的时候，会将import和export中的接口一一对应起来，建立一种引用关系。在使用这个变量的时候，通过这种引用关系动态去依赖模块中获取对应变量的值。**

```js
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
export {counter, incCounter}

//a.js
import {counter, incCounter} from './lib.js'
console.log(counter); // 3
incCounter();
console.log(counter); // 4
```

由于是动态引用的，所以每次获取到的都是模块中最新的值

### 1.3 CommonJS只能获取模块所有接口，ES Module可以片段引入

CommonJS中动态导入模块的时候，会将依赖模块的exports属性都导入进去，即时有些方法不会使用到。

```js
var {get,set} = require('./a.js');
//相当于
var obj = require('./a.js');
var get = obj.get;
var set = obj.set;
```

ES Module中由于是建立静态引用关系，而且模块也不是一个对象。所以只会引入依赖模块中的一些片段

## 2 babel中模拟实现CommonJS和ES Module

### 2.1 CommonJS的实现

源代码：

```js
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
module.exports = {
  counter: counter,
  incCounter: incCounter,
};

//a.js
const {counter, incCounter} = require('./lib.js');
console.log(counter); // 3
incCounter();
console.log(counter); // 3
```

打包后的代码：

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
    // Return the exports of the module
    return module.exports;
  }
  // Load entry module and return exports
  return __webpack_require__(__webpack_require__.s = "./commonJS/index.js");
})
  /************************************************************************/
  ({

/***/ "./commonJS/index.js":
/*!***************************!*\
  !*** ./commonJS/index.js ***!
  \***************************/
/*! no static exports found */
/***/ (function (module, exports, __webpack_require__) {

        const { counter, incCounter } = __webpack_require__(/*! ./lib.js */ "./commonJS/lib.js");
        console.log(counter);
        incCounter();
        console.log(counter);

        /***/
}),

  /***/ "./commonJS/lib.js":
  /*!*************************!*\
    !*** ./commonJS/lib.js ***!
    \*************************/
  /*! no static exports found */
  /***/ (function (module, exports) {

        var counter = 3;
        function incCounter() {
          counter++;
        }
        module.exports = {
          counter: counter,
          incCounter: incCounter,
        };

        /***/
})

  });
```

babel中会将模块包装成一个包装函数，和node中的实现原理基本是一样的。babel内部会实现一个自己的require方法 `__webpack_require__`用来引入模块。内部同样实现了模块的管理和缓存

### 2.2 ES Module的实现

源代码：

```js
// lib.js
var counter = 3;
function incCounter() {
  counter++;
}
export {counter, incCounter}

//a.js
import {counter, incCounter} from './lib.js'
console.log(counter); // 3
incCounter();
```

babel打包之后的代码：

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
/*! no exports provided */
/***/ (function (module, __webpack_exports__, __webpack_require__) {

        "use strict";
        __webpack_require__.r(__webpack_exports__);
  /* harmony import */ var _lib_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./lib.js */ "./esModule/lib.js");
				
  			//这里变量的访问变成了访问对象的属性
        console.log(_lib_js__WEBPACK_IMPORTED_MODULE_0__["counter"]);
        Object(_lib_js__WEBPACK_IMPORTED_MODULE_0__["incCounter"])();
        console.log(_lib_js__WEBPACK_IMPORTED_MODULE_0__["counter"]);

        /***/
}),

  /***/ "./esModule/lib.js":
  /*!*************************!*\
    !*** ./esModule/lib.js ***!
    \*************************/
  /*! exports provided: counter, incCounter */
  /***/ (function (module, __webpack_exports__, __webpack_require__) {

        "use strict";
        __webpack_require__.r(__webpack_exports__);
    		// !!! 导出接口时，定义成一个getter属性
  /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "counter", function () { return counter; });
  /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "incCounter", function () { return incCounter; });
        var counter = 3;
        function incCounter() {
          counter++;
        }


        /***/
})

  });
```

上面打包的代码可以看出，babel使用的是动态模块引入的方式来模拟ES Module，类似于CommonJS。

1. 构建了模块管理系统，每个模块是一个对象
2. 包装成模块代码，生成了包装函数

前面讲到ES Module和CommonJS有一个很明显的区别就是==ES Module模块输出的是值的引用。这一点是如何模拟实现的呢？==

**在模块导出接口的时候，会调用`__webpack_require__.d()`方法，将导出的变量定义成exports对象中的一个getter属性，getter方法内部返回的是模块的真实变量，因此每次访问该属性的时候都可以得到变量最新的值。**

**在导入模块的时候，导入的是module对象中的exports属性，一个最关键的点就是，在使用变量的时候，只能通过对象属性的方式获取变量的值，而不能像之前CommonJS中一样，利用结构赋值创建两个新的变量。**

通过上面导入和导出两个方面的改造，才最终成功模拟可ES Module中输出引用值的特性。

## 3 参考文章

1. [前端模块化：CommonJS,AMD,CMD,ES6](https://juejin.im/post/5aaa37c8f265da23945f365c)

