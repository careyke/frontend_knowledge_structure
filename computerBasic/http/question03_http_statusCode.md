# 常见的http响应状态码

在http请求中，响应报文中会返回一个Status Code字段，表示该请求的状态

下面主要介绍一些具有代表性的状态码

## 1XX（临时响应）

### 100 — 继续

表示当前请求一切正常，客户端应该继续请求

### 101 — 切换协议

表示服务端应客户端的请求在切换协议



## 2XX（成功）

2开头的状态码表示请求已经被正常处理

### 200 — OK

表示请求被正常处理而且返回。是一个成功的请求

GET，POST，HEAD和TRACE的请求成功的时候，返回的状态码通常是200。但是PUT和DELETE并不是

### 204 — No Content

**表示请求被正常处理，但是响应报文里面不带响应体返回。也即是说页面不需要更新**

一般来说，当客户端不需要服务端返回数据内容更新页面的时候可以使用204状态码返回

PUT和DELETE方法发出的请求，返回的成功状态码通常是204

### 206 — Partial Content

表示该请求是一个**区域请求**，请求的区域在请求头的Range字段中设定，而在响应报文中包含由Content-Range字段指定区间的实体内容。



## 3XX（重定向）

3XX的状态码表示如果要完成请求，需要浏览器进一步的操作

### 301— 永久重定向

表示当前所请求的资源已经被分配了新的url，已经应该使用新的url来访问当前资源。如果该资源的url保存为了书签，这时应该更新书签指向的地址

### 302 — 临时重定向

表示当前资源只是在本次请求被分配了新的url，在未来还有可能发生改变，表示一直临时的情况。如果资源的url保存在书签中，这是并不会去更新该书签指向的地址

### 303 — See Other

表示当前资源存在另一个url，应该使用GET方法去访问这个url去获取资源

303和302在功能上是很相似的，但是**303状态明确告诉浏览器要使用GET方法去新的url获取资源**，但是302是没有的

```
当状态码是301 302或者303的时候，几乎所有的浏览器都会把POST改成GET请求，并删除请求报文中的请求体，然后浏览器自动再次发送请求到新的地址

虽然301和302的标准中是禁止将POST修改成GET方法的，但是实际上都是这么做的
```

### 304 — Not Modified

表示当前资源并没有发生修改，并不会返回响应体，而是让浏览器去缓存中获取资源。

304状态码和协商缓存关联在一起

### 307 — Temporary Redirect

临时重定向。和302状态码有着相同的含义

尽管302标准中规定不能将POST修改成GET方法来请求资源，但是大多数浏览器还是这样做了。

而**对于307状态码，会确保不会将POST请求修改成GET请求** 

### 308 — Permanent Redirect

永久重定向。和301状态码是一样的含义

浏览器对于301状态码，会将POST方法修改成GET方法去请求新资源

但是**对于308状态码，浏览器不会修改请求的方法**，不会将POST修改成GET方法



## 4XX（客户端出错）

4XX表示本次请求失败的原因是因为客户端发送的报文出错了

### 400 — Bad Request

表示客户端的请求报文发生了**语法错误**，需要修改之后重新发送

### 401 — Unauthorized

表示客户端缺少当前资源要求的身份凭证。响应头里面会带一个`WWW-Authenticate` 字段，里面包含如何进行验证的信息

第一次返回401的时候，会弹出一个对话窗口用于权限验证。第二次返回401的时候，则表示验证失败

### 403 — Forbidden

表示服务器拒绝了该请求，通常情况下因为没有访问的权限。但是和401不同的是，403并不会继续进行权限的验证

### 404 — Not Found

表示服务中并不存在所访问的资源，请求的url有错误

### 405 — Method Not Allowed

表示**服务器禁止使用当前方法来访问该资源**。需要注意的是GET和HEAD方法是不会被禁止的

在请求之前可以使用OPTIONS方法来获取当前资源支持的请求方法



## 5XX（服务端出错）

5XX格式的状态码表示当前请求没有完成的原因是服务端出现了异常

### 500 — Internal Server Error

表示服务器在执行请求的时候发生了错误

### 502 — Bad Gateway

表示扮演网关或者代理角色的服务器，从上游服务器接收到的响应是无效的。

这种问题客户端是无法修复的，需要路径上的代理服务器对齐进行修复

### 503 — Service Unavailable

表示**当前服务器处于一种不可接收请求的状态**。通常造成这种情况的原因是由于服务器停机维护或者超出了负载

这种情况下响应头中应该带一个`Retry-After` 字段，来告诉客户端服务器恢复可用的大概时间。

