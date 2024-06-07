# Tapable源码分析

Webpack是一款基于事件流的打包构建工具，其中最为出彩的是它的插件生态。插件生态丰富，开发者自定义的成本也不高，这都得益于**Webpack内部基于Tapable实现了一套可插拔的插件机制**。

Tapable的实现类似于EventEmitter，基本**发布-订阅**的设计模式。但是它考虑的场景更多，实现起来更复杂。

## 1. Hook的分类

Tapable内部根据**任务类型、组织方式和执行方式**不同，实现了9中不同的Hook来覆盖各种常见的场景。

任务类型：任务的执行时机

- 同步（sync）
- 异步
  - 回调函数（async）
  - Promise（promise）

组织方式：多个任务以何种方式组织执行

- 熔断（bail）
- 返回值传递 （waterfall）
- 循环（loop）

执行方式：多个任务以何种方式执行

- 串行（series）
- 并行（parallel）

| Hook                     | 任务类型  | 组织方式  | 执行方式 | 说明                                                |
| ------------------------ | ----- | ----- | ---- | ------------------------------------------------- |
| SyncHook                 | 同步    | 无     | 串行   | 同步串行执行，任务之间没有关系，没有返回值                             |
| SyncBailHook             | 同步    | 熔断    | 串行   | “熔断” - 同步串行执行，一旦某个任务有返回值，终止执行。这个返回值作为hook的返回值     |
| SyncWaterfallHook        | 同步    | 返回值传递 | 串行   | 上一个任务的返回值作为下一个任务的参数。最后一个任务的返回值作为hook的返回值          |
| SyncLoopHook             | 同步    | 循环    | 串行   | 当某个任务的返回值不为undefined时，从头重新开始执行整个任务队列              |
| AsyncSeriesHook          | 同步、异步 | 无     | 串行   | 串行执行，上一个任务执行完成之后再执行下一个                            |
| AsyncSeriesBailHook      | 同步、异步 | 熔断    | 串行   | 当某个任务有**决议值**即停止执行                                |
| AsyncSeriesWaterfallHook | 同步、异步 | 返回值传递 | 串行   | 上一个任务的决议值作为下一个任务的参数                               |
| AsyncSeriesLoopHook      | 同步、异步 | 循环    | 串行   | 当某个任务的决议值不为undefined时，从头重新开始执行整个任务队列              |
| AsyncParallelHook        | 同步、异步 | 无     | 并行   | 任务并行执行，当最后一个任务完成之后才算是本次发布执行完成，才能执行对应的回调函数         |
| AsyncParallelBailHook    | 同步、异步 | 熔断    | 并行   | **按照任务的执行顺序**，当某个任务有决议值时，停止本次执行。并且该决议值作为本次发布的决议值。 |

> 这里所说的**决议值指的就是当前任务的执行结果**
> 
> - 对于同步任务来说，决议值就是函数的返回值
> - 对于异步任务来说，决议值就是异步任务执行之后得到的结果

## 2. 结合源码分析具体实现细节

整个源码大致可以分成两个部分：

1. Hook类：**发布-订阅模式的实现类**。用来存储任务队列，提供发布和订阅的api
2. HookCodeFactory类：**动态生成任务执行函数**。在发布阶段根据类型动态生成并执行各个任务

> 这里所说的任务指的就是订阅时注册的任务

这两个类都是基础类，其他具体的Hook类都是通过继承这两个基础类来实现的。

下面就以SyncHook为例子来分析源码的实现细节

> 笔者fork的源代码可以看[这里](https://github.com/careyke/tapable/tree/littleknife_v2.2.0)，里面笔者对于一些重要的地方加了注释，可以参考一下。
> 
> 笔者分析源码时写的demo可以看[这里](https://github.com/careyke/hello-tapable)，里面有**每一种类型的demo和对应的动态生成的任务执行函数**

### 2.1 Hook类

对于一个发布-订阅模式的实现类来说，最重要的有三点：

1. 任务的数据结构
2. 订阅流程
3. 发布流程

#### 2.1.1 任务的数据结构

首先可以来看一下Hook类的构造函数

```js
constructor(args = [], name = undefined) {
  // 任务回调函数的形参
  this._args = args;

  // 当前hook的名称
  this.name = name;

  // 任务队列
  this.taps = [];

  // 当前hook的拦截器，整个流程的各个环节都可以拦截
  this.interceptors = [];

  // 发布api
  this._call = CALL_DELEGATE;
  this.call = CALL_DELEGATE;
  this._callAsync = CALL_ASYNC_DELEGATE;
  this.callAsync = CALL_ASYNC_DELEGATE;
  this._promise = PROMISE_DELEGATE;
  this.promise = PROMISE_DELEGATE;

  // 当前hook所有任务的回调函数集合，数组
  // HookCodeFactory.js
  this._x = undefined;

  // 任务执行函数的编译方法
  this.compile = this.compile;

  // 订阅api
  this.tap = this.tap;
  this.tapAsync = this.tapAsync;
  this.tapPromise = this.tapPromise;
}
```

从构造函数中可以看出，Hook实例中不仅存储了任务队列和拦截器队列，还定义了任务回调函数的形参。这个形参主要是在动态创建任务执行函数的时候会使用。这个在后面创建任务执行函数的时候会讲到

除此之外，Hook中还分别定义了**三种发布方法和订阅方法**，用来针对不同的场景，这个后面也会讲到。

这里我们主要是来分析任务的数据结构，其实光看构造函数是无法分析出任务的数据结构的，需要分析订阅的过程才能分析出来。这里我们直接前置这个数据结构，方便后面流程的代码分析。

```typescript
// 任务的数据结构
interface Tap{
  name: string; // 任务的名称，唯一标识
  type: 'sync'|'async'|'promise' // 任务类型
  fn: Function; // 任务函数
  before: string | string[]; // 其他任务的名称，表示当前任务在这些任务之前执行
  stage: number; // 任务的相对顺序标识
}
```

#### 2.1.2 订阅流程

上面提到，订阅的api有三种

```js
tap(options, fn) {
  this._tap("sync", options, fn);
}

tapAsync(options, fn) {
  this._tap("async", options, fn);
}

tapPromise(options, fn) {
  this._tap("promise", options, fn);
}
```

每种方式注册的任务只有类型不同

- **对于同步的Hook来说，只能使用tap来注册任务，也就是说同步Hook的任务队列只能包含同步的任务**
- **对于异步的Hook来说，三种方式都可以使用，也就是说异步Hook的任务队列可以包含同步的任务**

```js
_tap(type, options, fn) {
  if (typeof options === "string") {
    options = {
      name: options.trim()
    };
  } else if (typeof options !== "object" || options === null) {
    throw new Error("Invalid tap options");
  }
  if (typeof options.name !== "string" || options.name === "") {
    throw new Error("Missing name for tap");
  }
  if (typeof options.context !== "undefined") {
    deprecateContext();
  }
  options = Object.assign({ type, fn }, options);
  // 执行注册时的拦截方法
  options = this._runRegisterInterceptors(options);
  // 按照相对顺序插入队列中合适的位置
  this._insert(options);
}
```

这里主要看一下`_insert`方法的实现

```js
_insert(item) {
  this._resetCompilation();
  let before;
  if (typeof item.before === "string") {
    before = new Set([item.before]);
  } else if (Array.isArray(item.before)) {
    before = new Set(item.before);
  }
  let stage = 0;
  if (typeof item.stage === "number") {
    stage = item.stage;
  }
  let i = this.taps.length;
  while (i > 0) {
    i--;
    const x = this.taps[i];
    this.taps[i + 1] = x;
    const xStage = x.stage || 0;
    if (before) {
      if (before.has(x.name)) {
        before.delete(x.name);
        continue;
      }
      if (before.size > 0) {
        continue;
      }
    }
    if (xStage > stage) {
      continue;
    }
    i++;
    break;
  }
  this.taps[i] = item;
}
```

**任务插入队列的规则**：

- 默认是插在队列的最后面
- 如果before字段有值，则插在before中所有任务的前面；如果before中有和现有任务不匹配的name，则插在队列的最前面
- state值小的在前面

整个订阅过程其实就是**将任务按照特定的顺序插入队列中对应的位置**

#### 2.1.3 发布流程

发布的api同样也有三种：

- **对于同步的Hook，只能使用`call`方法来发布**
- **对于异步的Hook，可以使用`callAsync`或者`promise`方法来发布**

发布方式不同，任务执行函数的返回结果也不同

```js
const CALL_DELEGATE = function(...args) {
    // 修改了this.call, 所以后续需要重置
    this.call = this._createCall("sync");
    return this.call(...args);
};
const CALL_ASYNC_DELEGATE = function(...args) {
    this.callAsync = this._createCall("async");
    return this.callAsync(...args);
};
const PROMISE_DELEGATE = function(...args) {
    this.promise = this._createCall("promise");
    return this.promise(...args);
};


this._call = CALL_DELEGATE;
this.call = CALL_DELEGATE;
this._callAsync = CALL_ASYNC_DELEGATE;
this.callAsync = CALL_ASYNC_DELEGATE;
this._promise = PROMISE_DELEGATE;
this.promise = PROMISE_DELEGATE;


compile(options) {
    throw new Error("Abstract: should be overridden");
}
_createCall(type) {
    // 委托代理模式
    return this.compile({
        taps: this.taps,
        interceptors: this.interceptors,
        args: this._args,
        type: type
    });
}
```

上面代码可以看出，发布的流程有两个部分：

1. 动态创建任务执行函数
2. 执行这个任务执行函数

> 这里有一个优化的细节：
> 
> **动态创建了任务执行函数之后，修改了发布方法（`this.call`）的指向。这样做可以避免每次调用发布方法时都动态创建执行函数。但是当任务队列发生变化之后，再次发布时就需要重新动态创建任务执行函数**

### 2.2 HookCodeFactory类

HookCodeFactory类的作用就是用来动态创建任务执行函数，通过`new Function()`的方式。在传统的发布-订阅模式中，任务执行函数都是事先定义好的，就算是有多个不同场景的Hook，也可以通过定义与之对应的任务执行函数来实现。其中公共的部分可以单独抽成函数来复用。

但是Tapable中的任务执行函数是根据当前任务队列来动态创建的，**以代码块作为基本的复用单元**，这样做代码的复用颗粒度更小，代码更加简洁。

```js
create(options) {
  this.init(options);
  let fn;
  // 根据不同的触发的方式有不同的处理逻辑，会生成不同的发布函数
  switch (this.options.type) {
    case "sync":
      // 同步的方式处理所有订阅者
      fn = new Function(
        this.args(),
        '"use strict";\n' +
          this.header() +
          this.contentWithInterceptors({
            onError: err => `throw ${err};\n`,
            onResult: result => `return ${result};\n`,
            resultReturns: true,
            onDone: () => "",
            rethrowIfPossible: true
          })
      );
      break;
    case "async":
      // 回调函数的方式处理异步调用
      // 发布时需要传入回调函数，在整个发布过程执行完成会触发回调函数
      fn = new Function(
        this.args({
          after: "_callback"
        }),
        '"use strict";\n' +
          this.header() +
          this.contentWithInterceptors({
            onError: err => `_callback(${err});\n`,
            onResult: result => `_callback(null, ${result});\n`,
            onDone: () => "_callback();\n"
          })
      );
      break;
    case "promise":
      // promise的方式处理异步调用
      // 发布函数返回一个promise，发布过程完成之后会改变promise的状态
      let errorHelperUsed = false;
      const content = this.contentWithInterceptors({
        onError: err => {
          errorHelperUsed = true;
          return `_error(${err});\n`;
        },
        onResult: result => `_resolve(${result});\n`,
        onDone: () => "_resolve();\n"
      });
      let code = "";
      code += '"use strict";\n';
      code += this.header();
      code += "return new Promise((function(_resolve, _reject) {\n";
      if (errorHelperUsed) {
        code += "var _sync = true;\n";
        code += "function _error(_err) {\n";
        code += "if(_sync)\n";
        code +=
          "_resolve(Promise.resolve().then((function() { throw _err; })));\n";
        code += "else\n";
        code += "_reject(_err);\n";
        code += "};\n";
      }
      code += content;
      if (errorHelperUsed) {
        code += "_sync = false;\n";
      }
      code += "}));\n";
      fn = new Function(this.args(), code);
      break;
  }
  this.deinit();
  return fn;
}
```

整个创建函数的流程分成三个部分：

1. 创建函数形参
2. 创建函数头
3. 创建函数体

上面代码可以看到，**不同类型的触发方式，任务执行完成之后的返回的结果也不一样，对应的任务完成的标志也不一样，所以针对不同的触发方式，定义了不同的任务结束代码块**。

- **onDone**：返回结束任务代码块，但是不关心决议值
- **onResult**：返回结束任务代码块，并且返回了决议值
- **onError**：返回任务异常处理代码块，提前跳出

#### 2.2.1 创建函数形参

```js
args({ before, after } = {}) {
    let allArgs = this._args;
    if (before) allArgs = [before].concat(allArgs);
    if (after) allArgs = allArgs.concat(after);
    if (allArgs.length === 0) {
        return "";
    } else {
        return allArgs.join(", ");
    }
}
```

任务执行函数的形参主要来自于Hook实例化时定义好的形参

#### 2.2.2 创建函数头

```js
header() {
  let code = "";
  if (this.needContext()) {
    // deprecated
    code += "var _context = {};\n";
  } else {
    code += "var _context;\n";
  }
  code += "var _x = this._x;\n";
  if (this.options.interceptors.length > 0) {
    code += "var _taps = this.taps;\n";
    code += "var _interceptors = this.interceptors;\n";
  }
  return code;
}
```

这里函数头主要是一些通用变量的定义

#### 2.2.3 创建函数体

前面介绍的函数形参和函数头基本上都是一些通用的代码块，不同类型的Hook中也基本都是一样的。而函数体的代码涉及不同逻辑的处理，在实现上有较大的差异。

```js
contentWithInterceptors(options) {
  if (this.options.interceptors.length > 0) {
    // ...省略 有拦截器时的处理逻辑
    return code;
  } else {
    return this.content(options);
  }
}
```

上面代码可以看到，函数体的代码是通过`this.content`方法来生成的。但是`HookCodeFactory`类中并没有实现这个方法，这是因为每种类型的Hook对应的执行流程都不一样，所以需要每个类自己实现。

`HookCodeFactory`类中封装了很多代码片段生成方法来供每个具体的Hook类使用

| 方法               | 作用                    |
| ---------------- | --------------------- |
| callTap          | 根据任务的类型，生成执行每个任务的代码片段 |
| callTapsSeries   | 生成以串行方式执行所有任务的代码片段    |
| callTapsLooping  | 生成以串行循环方式执行所有任务的代码片段  |
| callTapsParallel | 生成以并行方式执行所有任务的代码片段    |

各个Hook类通过这些方法来组织自己的任务处理流程。

> 这一块的代码能分析的点不多
> 
> 具体的可以看笔者fork的仓库，有比较详细的注释

#### 2.2.4 总结

至此，整个创建任务执行函数的流程就分析完了。

和传统的发布-订阅模式不一样，Tapable中使用的是动态拼接代码块的方式来执行任务。主要有以下**两个优点**：

1. 代码复用的颗粒度更小，代码更加简洁
2. 考虑到多个任务的组织方式和执行方式，使用传统的方法实现起来感觉更加繁琐，特别是对于异步任务的处理。使用代码块拼接的方式，虽然代码看起来难懂，但是其实处理逻辑很直观也很清晰。

这个库本质上没有这么难，和传统的`发布-订阅`库最大的不同在于作者对于**不同场景下任务执行的逻辑异同点**剖析得更加彻底，主要的区别在于：

1. 不同类型任务执行代码块不同
2. 不同类型Hook执行任务队列的方式不同
3. 不同类型Hook发布之后返回值不同

作者根据这三个点分别封装工具方法来解决，极大程度得提高了代码的复用性，也降低了任务执行函数的复杂度。
