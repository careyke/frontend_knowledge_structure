# 谈谈CSRF

CSRF全称是Cross-site request forgery，也就是跨域请求伪造。

**攻击者诱导受害者进入第三方的网站，在第三方网站中，向被攻击网站发送跨域请求。利用受害者在被攻击网站已经获取的注册凭证（Cookie），绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。**

## 1. CSRF攻击的流程

1. 受害者登录网站www.a.com，并保留登录凭证（Cookie）
2. 攻击者诱导受害者访问了攻击网站www.b.com
3. 在www.b.com中请求www.a.com中的接口，并且会带上a网站的Cookie
4. www.a.com在接收到请求之后，会验证Cookie，确认时受害者的Cookie，会误以为是受害者自己发出的请求
5. www.a.com会执行请求的内容，从而完成攻击



## 2. CSRF攻击的特点

1. 攻击一般发生在第三方的网站，也就是说，基本上都是**跨域**的请求。
2. 攻击者**无法获取**被攻击网站的Cookie，只能利用请求会携带当前域的Cookie这一特点来**冒用**Cookie
3. 攻击者利用受害者在被攻击网站的登录凭证，冒充受害者提交操作；而不是直接窃取数据。

CSRF攻击通常都是跨域的请求，因为外域更容易被攻击者掌握。但是**有些场景下CSRF攻击会发生在同域中，例如一个可以发图和链接的论坛和评论区，攻击者可以在img点击的时候发动攻击请求，这时候的请求是一个同源的请求，危害会更大。**

> 这里更像是XSS攻击，注入了一段攻击脚本



## 3. CSRF攻击的种类

### 3.1 GET类型的CSRF

GET类型的CSRF一般都很简单，就是一个请求。**可以将这个请求放在`<img> <script>` 标签中，因为这些标签是可以跨域请求资源的**，当这些标签被加载的时候会自动发起请求

```html
<img src="http://bank.example/withdraw?amount=10000&for=hacker" />
```



### 3.2 POST类型的CSRF

POST类型的CSRF相对GET来说比较复杂，利用的是**form表单的提交没有跨域问题**这一特性来实现的，具体操作就是伪造一个表单，发起post请求

```html
<form action="http://bank.example/withdraw" method=POST>
    <input type="hidden" name="account" value="xiaoming" />
    <input type="hidden" name="amount" value="10000" />
    <input type="hidden" name="for" value="hacker" />
</form>
<script> document.forms[0].submit(); </script> 
```

页面加载完成之后会自动发一个post请求到被攻击网站

#### 为什么form表单提交没有跨域问题

使用ajax请求其他域的接口的时候，往往会发生跨域的问题。但是form表单提交的时候是没有跨域的，**因为form表单提交的时候只是单纯的向服务端提交数据，而不是为了拿到响应，而且提交之后页面会刷新，所以浏览器认为是安全的。**

**同源策略的本质是避免从其他域中获取内容，从而避免某些恶意网站返回恶意的内容攻击网站**。也就是说，ajax发送跨域请求的时候，该请求是已经发出去了，但是就是接收不到响应而已。



### 3.3 链接类型的CSRF

链接类型的CSRF中招的条件是需要用户**点击**这个链接。点击之后请求才会发起

这种类型通常是在论坛中发布的图片中嵌入恶意链接，或者以广告的形式诱导用户中招。

```html
<a href="http://test.com/csrf/withdraw.php?amount=1000&for=hacker" taget="_blank">
  重磅消息！！
<a/>
```



## 4. CSRF攻击的预防手段

CSRF攻击通常是发生在第三方网站中，被攻击网站是无法阻止攻击的发生的，只能在网站自身层面来防御这种攻击。

CSRF的特点：

1. 通常攻击请求都是跨域请求
2. CSRF无法获取被攻击网站的Cookie，只能是冒用

针对以上的特点，可以从以下几个方面来考虑预防手段

1. **阻止不明外域的访问**
   - 请求的同源检测
   - Samesite Cookie
2. **提交时要求带上只有本域才能获取到的信息**
   - Token验证
   - 双重Cookie验证

### 4.1 请求的同源检测

在http协议中，请求头中有两个字段来表示当前请求的来源域名

- Origin：只有主机名，没有具体路径
- Referer：一般请求下会带具体的路径

这两个请求头字段大多数情况下会在http请求中自动带上，而且不能被前端代码定义

服务端可以根据请求头中的这两个字段来判断是否是同源的页面请求的，如果不是则不处理。

**但是这种方式会有一些其他的问题**：

1. 有一些是搜索引擎中点击链接访问的，这种请求是需要正常处理的，所以需要一个**白名单**
2. CSRF中的请求可能是同源的，这时这种策略就无法正常工作了



### 4.2 Samesite Cookie

在Cookie中有一个新的属性叫做Samesite，如果Cookie在创建的时候设置了这个属性，则表示这个Cookie是一个**同站**的Cookie。Samesite属性有两个值：Strict和Lax

#### 4.2.1 Samesite = Strict

这种称为严格模式，表明这个 Cookie 在任何情况下都不可能作为**跨站请求**的 Cookie，绝无例外。比如说 [b.com](http://b.com/) 设置了如下 Cookie：

```js
Set-Cookie: foo=1; Samesite=Strict
Set-Cookie: bar=2; Samesite=Lax
Set-Cookie: baz=3
```

我们在 [a.com](http://a.com/) 下发起对 [b.com](http://b.com/) 的任意请求，foo 这个 Cookie 都不会被包含在 Cookie 请求头中，bar满足一定条件会，baz会带上。

#### 4.2.2 Samesite = Lax

这种称为宽松模式，比 Strict 放宽了点限制：**假如这个请求是这种请求（改变了当前页面或者打开了新页面）且同时是个GET请求**，则这个Cookie可以作为跨站请求的Cookie。比如说 b.com设置了如下Cookie：

```js
Set-Cookie: foo=1; Samesite=Strict
Set-Cookie: bar=2; Samesite=Lax
Set-Cookie: baz=3
```

当用户从 [a.com](http://a.com) 点击链接进入 [b.com](http://b.com) 时，foo 这个 Cookie 不会被包含在 Cookie 请求头中，但 bar 和 baz 会，也就是说用户在不同网站之间通过链接跳转是不受影响的。但假如这个请求是从 [a.com](http://a.com) 发起的对 [b.com](http://b.com) 的异步请求，或者页面跳转是通过表单的 post 提交触发的，则bar也不会发送

#### 4.2.3 总结

Samesite Cookie方法虽然能够阻止其他域的访问，但是也包含了**很大的副作用。**

- 当Samesite设置为Strict的时候，**跳转子域名或者是新标签重新打开刚登陆的网站，之前的Cookie都不会存在**。那对于需要登录的网站来说，每次新打开一个都需要重新登录，基本是无法接受的交互。
- 当Samesite设置为Lax的时候，其他网站可以通过页面跳转过来的时候可以使用Cookie，安全性就比较低了，CSRF还是可以攻击



### 4.3 Token验证

CSRF攻击的另一个特征是攻击者无法获取网站的信息，只能冒用网站的Cookie。那么我们可以在每一个请求中都带一个Token到服务端，由服务端来验证。但是**CSRF中的请求是无法获取到Token**的，所以可以有效防止CSRF攻击。

**Token验证的步骤**：

1. 用户登录页面的时候，服务器给当前用户生成一个Token，一般Token都包括随机字符串和时间戳的组合。
2. 用户拿到Token之后保存起来，每次向服务器请求的时候在**请求头中携带这个Token**
3. 服务器拿到这个Token之后，解密Token中的信息，对比Token中的关键信息和时间戳，如果信息一致而且没有过期，则该Token有效，也就是该请求有效。

Token验证能够有效的防御CSRF攻击，只要页面没有XSS漏洞泄漏这个Token，那么对于接口的CSRF的攻击无法进行

**验证码和密码**的形式也可以起到相同的效果，而且安全性更高，但是交互行为不友好。所以只有在一些安全性要求很高的场景中使用。比如微信转账



### 4.4 双Cookie验证（感觉漏洞有点多）

双Cookie验证的流程：

1.  在用户访问网站页面时，向请求域名注入一个Cookie，内容为随机字符串（例如`csrfcookie=v8g9e4ksfhw`）。
2. 在前端向后端发起请求时，取出Cookie，并添加到URL的参数中（接上例`POST https://www.a.com/comment?csrfcookie=v8g9e4ksfhw`）。
3. 后端接口验证Cookie中的字段与URL参数中的字段是否一致，不一致则拒绝

这种方式其实和Token方式的原理是一样的，都是利用攻击者无法获取Cookie这一特点。

但是这种方法有一个很明显的缺点：**就是Cookie会暴露在URL中，很不安全，所以需要依赖HTTPS协议。**



## 参考文章

1. [前端安全系列之二：如何防止CSRF攻击？](https://juejin.im/post/5bc009996fb9a05d0a055192#heading-14)

