# webpack如何实现懒加载

在现实场景中，可能会有一些组件比较复杂，其渲染的过程会阻塞后面内容的渲染，从而导致首屏的卡顿。这时可以使用懒加载的方式来**异步引入这些组件代码**，使得页面渲染不卡顿。

## 1. 懒加载的方式

1. 在CommonJS中使用`require.ensure`方法

2. ES Module中使用动态的import方法

推荐使用动态import方法来实现，但是现在很多浏览器还没有实现，需要借助babel来实现

## 2. webpack如何实现动态import

webpack实现动态import的时候是使用分包的方式来实现的。也就是说**动态import的模块会被单独打成一个chunk，然后使用到的时候动态引入这个chunk**

### 2.1 动态import语法规范

在ES2020提案中引入的import方法，来实现动态import

import()方法返回一个Promise对象，可以在条件语句中使用

```js
const main = document.querySelector('main');

import(`./section-modules/${someVariable}.js`)
  .then(module => {
    module.loadPageInto(main);
  })
  .catch(err => {
    main.textContent = err.message;
  });
```

特点：

1. 可以在条件语句中动态引入模块
2. 引入模块的方式是**异步引入**的，返回一个Promise对象

### 2.2 webpack中的实现

#### 2.2.1 源代码

```js
// import.js
export function add(a,b){
  return a+b;
}

// index.js
import React,{useState} from 'react';
import ReactDOM from 'react-dom';

function App() {
  const [asyncModule, setAsyncModule] = useState(null);
  if(!asyncModule){
    import('./import').then((module)=>{
      setAsyncModule(module);
    }).catch((err)=>{
      console.log(err);
    })
  }
  return (
    <div>
      <div>list page</div>
      <div>{asyncModule?asyncModule.add(1,2):0}</div>
    </div>
  );
}

ReactDOM.render(<App />, document.querySelector('#app'));
```

#### 2.2.2 打包之后的代码

##### 2.2.2.1 import函数的转换

```js
function (module, __webpack_exports__, __webpack_require__) {
  ...  
  function App() {
        var _useState = Object(react__WEBPACK_IMPORTED_MODULE_0__["useState"])(null),
            _useState2 = _slicedToArray(_useState, 2),
            asyncModule = _useState2[0],
            setAsyncModule = _useState2[1];

        if (!asyncModule) {          __webpack_require__.e(0).then(__webpack_require__.bind(null,"./src/list1/import.js")).then(function (module) {
                setAsyncModule(module);
            })["catch"](function (err) {
                console.log(err);
            });
        }

        return /*#__PURE__*/ react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("div", null, /*#__PURE__*/
            react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("div", null, "list page"), /*#__PURE__*/
            react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("div", null, asyncModule ? asyncModule.add(1,
                2) : 0));
    }

    react_dom__WEBPACK_IMPORTED_MODULE_1___default.a.render( /*#__PURE__*/ react__WEBPACK_IMPORTED_MODULE_0___default.a
        .createElement(App, null), document.querySelector('#app'));

    /***/
}
```

可以看出`import('./import')`会被转化成：

```js
__webpack_require__.e(0).then(__webpack_require__.bind(null,"./src/list1/import.js"))
```



##### 2.2.2.2 动态引入js文件的工具函数

也就是`__webpack_require__.e`方法

```js
__webpack_require__.e = function requireEnsure(chunkId) {
    var promises = [];
    // JSONP chunk loading for javascript
    var installedChunkData = installedChunks[chunkId];
    if (installedChunkData !== 0) { // 0 means "already installed".
        // a Promise means "currently loading".
        if (installedChunkData) {
            promises.push(installedChunkData[2]);
        } else {
            // setup Promise in chunk cache
            var promise = new Promise(function (resolve, reject) {
              // 将promise的resolve方法暂存起来
                installedChunkData = installedChunks[chunkId] = [resolve, reject];
            });
            promises.push(installedChunkData[2] = promise);

            // start chunk loading
            var script = document.createElement('script');
            var onScriptComplete;

            script.charset = 'utf-8';
            script.timeout = 120;
            if (__webpack_require__.nc) {
                script.setAttribute("nonce", __webpack_require__.nc);
            }
            script.src = jsonpScriptSrc(chunkId);

            // create error before stack unwound to get useful stacktrace later
            var error = new Error();
            onScriptComplete = function (event) {
                // avoid mem leaks in IE.
                script.onerror = script.onload = null;
                clearTimeout(timeout);
                var chunk = installedChunks[chunkId];
                if (chunk !== 0) {
                    if (chunk) {
                        var errorType = event && (event.type === 'load' ? 'missing' : event.type);
                        var realSrc = event && event.target && event.target.src;
                        error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc +
                            ')';
                        error.name = 'ChunkLoadError';
                        error.type = errorType;
                        error.request = realSrc;
                        chunk[1](error);
                    }
                    installedChunks[chunkId] = undefined;
                }
            };
            var timeout = setTimeout(function () {
                onScriptComplete({
                    type: 'timeout',
                    target: script
                });
            }, 120000);
            script.onerror = script.onload = onScriptComplete;
            document.head.appendChild(script);
        }
    }
    return Promise.all(promises);
};
```

可以看出动态引入js文件的方式是通过JSONP的方式来完成的。但是在上面 **jsonp的回调函数中并没有执行Promise的resolve方法，那是在何处执行的呢？**



##### 2.2.2.3 （*）JSONP请求返回的代码

```js
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([[0], {
    "./src/list1/import.js":
        /***/
        (function (module, __webpack_exports__, __webpack_require__) {

            "use strict";
            __webpack_require__.r(__webpack_exports__);
            /* harmony export (binding) */
            __webpack_require__.d(__webpack_exports__, "add", function () {
                return add;
            });

            function add(a, b) {
                return a + b;
            }
        })

}]);
```

由此可见，动态加载这个js文件之后，会将动态引入的module代码加入到`window['webpackJsonp']`属性中暂存起来。

可以看出这个JOSNP返回的js代码中，并没有执行回调函数，而是执行了一个push方法。

**按照正常的思维逻辑来说，应该要在jsonp返回的代码中执行resolve方法，将模块的导出值传给then中注册的回调函数。但是实际上返回的js代码中只执行了一个push方法，所以要看看这个push方法是不是被重新定义**

果然在webpack的模块系统代码的最后面，**重写了这个push方法**：

```js
var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
// 重写push方法
jsonpArray.push = webpackJsonpCallback;

jsonpArray = jsonpArray.slice();
for(var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
var parentJsonpFunction = oldJsonpFunction;
```

所以实际上会调用`webpackJsonpCallback`方法

```js
function webpackJsonpCallback(data) {
		var chunkIds = data[0];
		var moreModules = data[1];
		// add "moreModules" to the modules object,
		// then flag all "chunkIds" as loaded and fire callback
		var moduleId, chunkId, i = 0, resolves = [];
		for(;i < chunkIds.length; i++) {
			chunkId = chunkIds[i];
			if(Object.prototype.hasOwnProperty.call(installedChunks, chunkId) && installedChunks[chunkId]) {
				resolves.push(installedChunks[chunkId][0]);
			}
			installedChunks[chunkId] = 0;
		}
  	// 将动态引入模块也加到模块系统中，方便缓存
		for(moduleId in moreModules) {
			if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
				modules[moduleId] = moreModules[moduleId];
			}
		}
		if(parentJsonpFunction) parentJsonpFunction(data);

		while(resolves.length) {
			resolves.shift()(); // 执行promise的resolve方法
		}
};
```

这个方法做个几个事情：

1. 标记当前chunk为已经加载的chunk
2. 将动态引入的模块也加入到模块系统中，可以使用缓存，避免二次加载
3. 执行Promise的resolve函数，执行动态加载完的回调函数函数

##### 2.2.2.4 加载完成的模块代码如何引用

```js
__webpack_require__.e(0).then(__webpack_require__.bind(null,"./src/list1/import.js"))
```

then的回调函数调用的是`__webpack_require__`，和正常模块一样的引用方式。



### 2.3 总结

**webpack实现动态import的流程**：

1. 打包的时候，将动态import的模块单独打成一个chunk
2. 运行时在使用的时候，通过jsonp的方式动态引入这个chunk
3. 引入完成之后执行chunk中的代码，最终执行`webpackJsonpCallback`方法，**将chunk中的模块也加入到模块系统中**
4. 最后使用模块系统中提供的引入方法来引用这个模块

