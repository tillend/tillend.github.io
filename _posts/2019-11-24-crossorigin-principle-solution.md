---
layout:     post
title:      "跨域的原理及解决方案"
subtitle:   "CrossOrigin"
date:       2019-11-24 21:39:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - CrossOrigin
    - Jsonp
    - Java
---

跨域限制访问，即为浏览器禁止访问其他网站的资源，是浏览器的限制。如果缺少了同源策略，网页很容易受到XSS、CSFR等攻击。


> **同源策略**是Web应用程序安全性模型中的重要概念。根据该策略，Web浏览器允许第一个网页中包含的脚本访问第二个网页中的数据，但前提是两个网页具有相同的来源。来源由[URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)，主机名(hostname) 和端口号(port) 的组合定义。此策略可防止一个页面上的恶意脚本通过该页面的DOM(Document Object Model)获得对另一网页上敏感数据的访问。

## 跨域场景
当一个请求 URL 的**协议**、**域名**、**端口**三者之间任意一个与当前页面 URL 不同即为跨域

|当前页面URL	| 请求页面URL |	是否跨域	| 原因|
|---|---|---|---|
|http://www.test.com/|	http://www.test.com/index.html|	否	|同源（协议、域名、端口号相同）|
|http://www.test.com/	|https://www.test.com/index.html	|是|	协议不同（http/https）|
|http://www.test.com/|	http://www.baidu.com/	|是	|主域名不同（test/baidu）|
|http://www.test.com/	|http://blog.test.com/	|是	|子域名不同（www/blog）|
|http://www.test.com:8080/|	http://www.test.com:8081/	|是	|端口号不同|

## 跨域解决方案
### JSONP

由于同源策略，一般来说网页无法跨域请求资源，而 HTML 的 \<script>元素是一个例外。利用 \<script>元素的这个开放策略，网页可以得到从其他来源动态产生的JSON数据，而这种使用模式就是所谓的 [JSONP](https://zh.wikipedia.org/wiki/JSONP)。用JSONP抓到的数据并不是JSON，而是任意的JavaScript，用 JavaScript 解释器运行而不是用 JSON 解析器解析。

> JSONP优点是简单兼容性好，可用于解决主流浏览器的跨域数据访问的问题。缺点是仅支持 get 方法，具有局限性；且可能会遭受 XSS 攻击。

### CORS

CORS 是跨域资源分享（Cross-Origin Resource Sharing）的缩写。它是 W3C 标准，属于跨源 AJAX 请求的根本解决方法。


- `Access-Control-Allow-Origin`  设置允许请求的域名，多个域名以逗号分隔
- `Access-Control-Allow-Methods` 设置允许请求的方法，多个方法以逗号分隔
- `Access-Control-Allow-Headers` 设置允许请求自定义的请求头字段，多个字段以逗号分隔
- `Access-Control-Allow-Credentials` 设置是否允许发送 Cookies


常用的跨域配置只需设置 `Access-Control-Allow-Origin`，推荐设置为对应域名。但当使用 Cookies 时，还需要配置`Access-Control-Allow-Credentials` 为`true`，且此时`Access-Control-Allow-Origin` 的配置不能为`*`，而必须配置为具体的域名。


## Java 跨域实现方案
### JSONP
Spring 4 ~ Spring 5 低版本中支持全局Jsonp。在 Spring 5 高版本中弃用，且标注建议使用`@CrossOrigin`
```java
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.AbstractJsonpResponseBodyAdvice;

/**
*
* 统一支持 jsonp 的配置输出
* 使用说明：@RequestMapping 中指定 produces 为 application/json;charset=UTF-8 即可
* 例子：@RequestMapping(path = "/signin", produces = "application/json;charset=UTF-8")
*
*/
@ControllerAdvice(basePackages = {"com.yy.ent.union.zone.controller"})
public class JsonpAdvice extends AbstractJsonpResponseBodyAdvice {
	public JsonpAdvice() {
		super("callback", "jsonpcb");
	}
}
```

### @CrossOrigin
使用`@CrossOrigin`注解时，推荐指定域名，避免使用`*`的默认配置，产生不安全的隐患
```java
@RestController
@CrossOrigin(origins = "*")
public class TestController {}
```

当需要使用Cookies时，要配置`allowCredentials`参数，使用时必须指定具体的域
```java
@CrossOrigin(origins = "http://www.test.com/", allowCredentials = "true")
```

---
参考资料：
1.[什么是跨域？跨域解决方法](https://blog.csdn.net/qq_38128179/article/details/84956552#commentBox)