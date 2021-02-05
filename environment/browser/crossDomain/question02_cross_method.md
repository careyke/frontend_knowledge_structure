# 实现跨域请求的方法

## 1. jsonp

**jsonp的方案利用的其实就是 `<script>` 标签可以跨域访问资源的特性**，动态构建一个script标签，请求跨域的资源，然后在返回回来的js代码里面执行回调函数

```js
const script = document.createElement('script');
script.src = "http://www.domain.com/login?callback=jsonpCallback";
window.josnpCallback = function(data){
  console.log(data);
}

document.body.appendChild(script); // 开始请求

//销毁标签
document.body.removeChild(script);
```

需要注意的是：**jsonp只靠前端是无法实现的**，因为script标签返回的是一串可执行的js代码，如果需要请求回来之后执行回调函数，就需要将数据包裹在回调函数中。

```js
//需要的数据
{a:1,b:2};

//返回的数据
"jsonpCallback({a:1,b:2})"
```

### 1.1 如何回调函数防止命名冲突？

由于回调函数是定义在window对象上面的，有多个jsonp请求的时候，可能会造成命名冲突。

可以使用**时间戳**的形式解决这个问题

```js
const callbackName = `jsonpCallback${Date.now()}`;  //回调函数函数名
window[callbackName] = function(data){console.log(data);}
```



**注意不能使用Symbol类型值来定义回调函数在window中的key值。**

```js
const callbackName = Symbol('jsonpCallback');
window[callbackName] = function(data){console.log(data);}

//返回来的数据
"Symbol('jsonpCallback')({a:1,b:2})"
```

**但是在执行返回的js代码的时候，Symbol是唯一的，所以在window中并找不到对应的key值。**



### 1.2 jsonp+promise

```js
function jsonpPromise(url){
  return new Promise(function(resolve,reject){
    let script = document.createElement('script');
  	const callbackName = `jsonpCallback${Date.now()}`;
    
    window[callbackName] = function(data){
      window[callbackName] = undefined;
      document.head.removeChild(script); //清空script标签
      script = undefined;
      resolve(data);
    }
    script.src = `url?callback=${callbackName}`; //回调参数的名字传给后端
    script.onerror=function(err){
      reject(err);
    }
    //开始请求
    document.head.appendChild(script);
  });
}
```

### 1.3 jsonp的优点和缺点

优点：

1. 兼容性好

缺点：

1. **只能发送GET请求**
2. 安全性问题。jsonp是从其他的域中加载js脚本，然后直接执行，如果此脚本中包含一些恶意代码的话，可能会攻击我们的服务器。



## 2. postMessage + iframe

postMessage是html5提出的，用力解决不同源页面之间的通信的问题。

```js
otherWindow.postMessage(message, targetOrigin, [transfer]);
```

- otherWindow：**指的就是要接受消息的那个窗口**。这个目标窗口通常是以iframe或者window.open()的形式嵌套在当前窗口，使得当前窗口能够获得目标窗口的引用，从而像目标窗口发送消息

- message：指的是发送给目标窗口的消息，通常是String或者Object
- targetOrigin：**通过窗口的origin属性来指定哪些窗口能接收到消息事件，如果目标窗口的协议、主机或端口有一个和origin不相同，则消息就不会发送**。origin可以是具体的url或者`'*'`，`*`表示所以的窗口都可以接收到消息，及目标窗口肯定可以接收到消息。

### 2.1 发送消息

http://www.domain1.com/a.html

```js
<iframe id="iframe1" src="http://www.domain2.com/b.html"></iframe>
  
// js
const iframe = document.getElementById('iframe1');
// 需要在iframe加载完成才能发送消息，否则提前发送有可能监听不到消息
iframe.onload = function(){
  const data = {name:'abc'};
  iframe.contentWindow.postMessage(data,'http://www.domain2.com');
}
```

`iframe.contentWindow` 获取当前iframe内部的window对象

### 2.2 接收消息

postMessage发送出去的消息，可以在目标窗口中使用message事件监听到。**message事件的回调函数会接收到一个MessageEvent对象**，内部包含了接收到的消息和其他属性。

http://www.domain2.com/b.html

```js
window.addEventListener('message',(event)=>{
  if(event.origin === 'http://www.domain2.com'){
    console.log(event.data);
  	//发送消息给父页面
  	event.source.postMessage({name:'def'}, 'http://www.domain1.com');
  }
})
```

- data属性：表示接收的数据
- origin属性：表示调用 `postMessage` 时消息发送方窗口的 origin，协议、域名和端口
- source属性：表示发送消息的窗口对象的引用，可以使用这个属性在不同origin之间双向传递信息



## 3. CORS

CORS全称是**跨域资源共享**（Cross-Origin-Resource-Share）。它允许浏览器想跨域的服务器发请求。

CORS的方案**主要是服务端支持**，前端使用起来和普通的ajax请求基本是一样的

CORS分成简单请求和非简单请求

### 3.1 简单请求

简单请求满足以下特点：

1. 请求的方法是GET，HEAD或者POST
2. http头部信息不超过以下字段
   - Accept
   - Accept-Language
   - Content-Language
   - Last-Event-ID
   - Content-Type (application/x-www-form-urlencoded、 multipart/form-data、text/plain)

简单请求的步骤：

1. 浏览器判断是跨域请求的时候，会在请求头中带一个origin字段，表明请求的源，服务端会以这字段作为跨域的标志
2. 服务端在接收到请求之后，会判断当前origin是否是一个允许的源。
   - **如果是允许的源，响应头里面会包含 `Access-Control-*` 开头的一些字段**（详情看后面）
   - **如果是不允许的源，则响应头中没有 `Access-Control-Allow-Origin` 这个字段**，浏览器就会抛出一个异常，被XMLHttpRequest的onerror回调函数捕获。

**响应头中和CORS有关的字段：**

1. Access-Control-Allow-Origin：**这个字段是同意跨域请求时，响应头中必须包含的。表示服务器允许跨域请求的源的范围。**也可以是一个`*`，表示所有源都可以访问该服务
2. Access-Control-Allow-Credentials：可选字段。boolean类型的值，表示是否可以发送Cookie。默认情况下，Cookie不包含在CORS中，设置true则表示允许携带Cookie。（这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可）
3. Access-Control-Expose-Headers：可选字段。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。**如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。**

#### 发送Cookie

CORS请求中，如果需要携带Cookie，除了响应头中携带`Access-Control-Allow-Credentials：true`字段之外，还需要请求头中也设定`withCredentials:true`。如此才可以携带Cookie

如果需要发Cookie到后端，则`Access-Control-Allow-Origin`字段不能为`*` ，必须是明确的地址。**此外Cookie是遵循同源策略的，请求哪个源就只能携带该源下的Cookie，而不能携带其他源的Cookie。**



### 3.2 非简单请求

非简单请求的特点：**满足其中一个即可**

1. 使用特殊的请求方法，比如PUT，DELETE等
2. Content-Type为application/json

非简单请求中，在正式请求之前，会先发送一次**预检请求**

#### 3.2.1 预检请求

预检请求是一个OPTIONS请求，用来询问当前服务器资源支持哪些方法。

**预检请求的特殊请求头**：

1. Origin：必须字段，和简单请求一样，表示请求的源
2. Access-Control-Request-Method：必须字段，表示当前CORS请求会以哪些方法请求
3. Access-Control-Request-Headers：表示当前CORS请求会发送额外的请求头信息

和简单请求一样，如果返回的响应头中没有和CORS相关的字段，则表示服务拒绝这次跨域请求，浏览器就会抛出一个错误，被XMLHttpRequest对象的onerror回调函数捕获。

**如果服务器允许这个跨域请求，则在响应头中会返回和CORS相关的头信息。**除了简单请求的那些字段之外，还包含一下字段

1. Access-Control-Allow-Methods：必要字段。**返回当前服务所支持的所有请求方法**。
2. Access-Control-Allow-Headers：如果浏览器请求包括Access-Control-Request-Headers字段，则Access-Control-Allow-Headers字段是必需的。它也是一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。
3. Access-Control-Max-Age： **该字段可选，用来指定本次预检请求的有效期，单位为秒**。上面结果中，有效期是20天（1728000秒），即允许缓存该条回应1728000秒（即20天），在此期间，不用发出另一条预检请求。（缓存，防止每次请求都提前发送一个预检请求）

#### 3.2.2 主请求

预检请求通过之后，浏览器会发送一个主请求，携带需要发到后端的数据。

> 主请求和简单请求是一样的
>



## 4. Node.js中间层

**同源策略是浏览器的行为，并不是http请求的行为。所以在服务端是没有同源策略这种限制的**。也就是说，可以通过一个Node.js中间层来转发请求，然后实现跨域。

```js
// 前端

var xhr = new XMLHttpRequest();

// 前端开关：浏览器是否读写cookie
xhr.withCredentials = true;

// 访问http-proxy-middleware代理服务器
xhr.open('get', 'http://www.domain1.com:3000/login?user=admin', true);
xhr.send();
```

```js
// 后端

var express = require('express');
var proxy = require('http-proxy-middleware');
var app = express();

app.use('/', proxy({
    // 代理跨域目标接口
    target: 'http://www.domain2.com:8080',
    changeOrigin: true,

    // 修改响应头信息，实现跨域并允许带cookie
    onProxyRes: function(proxyRes, req, res) {
        res.header('Access-Control-Allow-Origin', 'http://www.domain1.com');
        res.header('Access-Control-Allow-Credentials', 'true');
    },

    // 修改响应信息中的cookie域名，可以实现将当前源下的Cookie带到其他源的服务端
    cookieDomainRewrite: 'www.domain1.com'  // 可以为false，表示不修改
}));

app.listen(3000);
console.log('Proxy server is listen at port 3000...');
```



## 5. Nginx代理层

这也是一种方案，通过配置就可以实现。暂时还没有玩过~



## 参考文章

1. [正确面对跨域，别慌](https://juejin.im/post/5a2f92c65188253e2470f16d#heading-20)
2. [跨域总结:从CORS到Ngnix](https://juejin.im/post/5e6c58b06fb9a07ce01a4199#heading-20)

