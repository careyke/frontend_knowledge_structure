# 什么是跨域？

## 1.同源策略

同源策略是**浏览器制定的一种规则**，为了提交网站的安全性。同源策略规定当前源下的脚本只能请求相同源的资源，而不能请求其他源下的资源。

同源的判断条件：

1. 协议相同
2. 域名相同
3. 端口相同

三个条件同时满足，就表明是相同的源。否则即是不同的源



## 2. 跨域

客户端访问不同源下的资源，即是跨域请求。

```js
// 协议不同
http://www.domain.com/a.js
https://www.domain.com/b.js

//域名不同
http://www.domain1.com/a.js
http://www.domain2.com/b.js

//端口不同
http://www.domain.com:8080/a.js
http://www.domain.com:8081/b.js
```

