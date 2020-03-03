# 深入理解CommonJS在node中的实现

CommonJS是==同步引入==的模块化方案，只适用于服务端。

## 1. Node中模块化的特点

1. **每个文件就是一个模块**
2. 每个模块上下文中都存在require方法，module对象和exports对象
   - require方法用来引入依赖模块的exports属性
   - module对象表示模块本身，**每个模块就是一个对象**
   - exports对象表示该模块导出的内容，exports是module中的一个属性
3. require方法的参数可以是模块标识，或者相对路径、绝对路径
4. 依赖模块的代码会按照引入的顺序，同步执行（没有缓存的情况下）

## 2. Node中如何来实现CommonJS规范

**Node内部将引入模块的过程分成三个步骤：**

1. 路径分析
2. 文件定位
3. 编译执行

**node中的模块分为核心模块和文件模块：**

- 核心模块是node内置的模块。**node源代码在编译的时候就已经将核心模块编译进了二进制执行文件中**，node进程启动的时候就已经存在了内存中，所以这部分模块没有文件定位和编译执行的过程，在路径分析中优先判断，加载速度最快。
- 用户编写的模块称为文件模块。文件模块是动态加载的，需要执行路径分析、文件定位、和编译执行这三个步骤。加载速度稍慢

### 2.1 路径分析和文件定位

路径分析和文件定位这两个步骤是为了准确找到模块文件。通过传入require(...)方法的模块标识符来进行查找。

1. **优先从模块缓存中加载**，node内部会缓存引用模块的结果，提高二次引用的效率
2. 如果缓存中没有匹配，则优先**判断是否是核心模块**
3. 如果不是核心模块，就判断是否是**路径形式的文件模块**
   - 如果是，则生成真实路径，根据真实路径来找到对应的文件
   - 如果不是，那就是**包文件模块**，即node_modules中的模块。需要根据模板路径的查找方式进行查找，找到对应的包之后，根据package.json中的main属性（默认是index.js）来找到具体的文件。

模块查找的匹配优先级：**缓存 > 核心模块 > 文件模块**

### 2.2 编译执行

在Node中每一个模块都是一个对象，**在定位到确切的文件之后，node内部会生成一个module对象**

```js
//node中的module对象
function Module(id, parent){
  this.id= id;
  this.exports={}
  this.parent=parent;
  if(parent && parent.children){
    parent.children.push(this); //建立双向索引
  }
  this.filename=null;
  this.loaded=false;
  this.children=[];
}
```

生成module对象之后，node会加载对应的文件，然后编译执行。对于不同类型的文件，加载的方式不一样，这里列举.js和.json文件

1. .js文件：**通过 fs 模块同步读取文件，然后编译执行**
2. .json文件：通过 fs 模块同步读取文件，然后使用 `JSON.parse()` 解析返回结果（返回的也是一个对象）

#### 2.1 分析js文件的编译执行过程

1. 通过 fs 模块读取到了js文件之后，**在编译期，node会对文件内容进行头尾包装**，一个正常的文件会包装成下面这样：

   ```js
   (function(exports,require,module,__filename,__dirname){
     ... //真实内容
   })
   //从上面代码可以知道为什么模块上下文中存在require和exports这些对象了。
   ```

2. 包装之后**通过vm原生模块中的 `runInThisContext` 方法（类似于eval方法）在执行这个字符串，得到一个function对象**。
3. **将当前模块对象module，引入模块的require方法，以及在路径分析中得到的完整文件路径和目录传到function对象中执行。**
4. 模块执行成功之后，会**以模块完整路径为作为索引将整个模块对象缓存在内部的Module._cache对象上**，提高二次引用的速度。

至此整个模块引入的过程就完成了，require方法会返回当前模块的exports属性

## 3. 为什么模块导出的时候，推荐使用module.exports而不是exports

上面的分析可知，exports和module都是传入到包装函数的变量，其中exports变量指向的是module对象中的exports属性

**require方法中返回的也是module对象中的exports属性**

所以有些情况下会出现使用exports导出无效的现象

```js
var a = 1;
var b = 2;

// 第一种方式
export.a = a;
export.b = b;
//可以使用

// 第二种方式
module.exports={
  a:a,
  b:b
}
//可以使用

// 第三种方式
exports={
  a:a,
  b:b
}
// 无效，取到的是空对象{}
```

上面第三种方式不行就是因为：赋值操作改变的是形参exports，并没有改变module对象中的exports属性

## 4. CommonJS中的循环引用的规则

**CommonJS中是引入一个模块的时候，其中引入的是module对象中的exports属性，这个属性默认值是`{}`，和模块代码是否执行完没有关系。**所以在循环引用的时候，仍然是遵循此规则分析

```js
// a.js
exports.done = false;
var b = require('./b.js');
console.log('在 a.js 之中，b.done = %j', b.done);
exports.done = true;
console.log('a.js 执行完毕');

// b.js
exports.done = false;
var a = require('./a.js');
console.log('在 b.js 之中，a.done = %j', a.done);
exports.done = true;
console.log('b.js 执行完毕');

```

当执行a.js的时候

1. 在require方法执行之前，`moduelA.exports={done.false}`，然后去执行b.js
2. **在执行b.js的时候，碰到`require('./a.js')`语句，此时a模块对象已经存在，直接从moduleA.exports中获取值，所以`a={done:false}`**
3. 接着执行b.js中剩余的代码，打印信息，并且得到`moduleB.exports={done:true}`
4. 然后返回a.js执行剩下的内容，得到`moduleA.exports={done:true}`

打印顺序为：

```js
在 b.js 之中，a.done = false
b.js 执行完毕
在 b.js 之中，b.done = true
a.js 执行完毕
```

