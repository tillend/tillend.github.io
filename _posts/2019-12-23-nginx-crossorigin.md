---
layout:     post
title:      "Nginx跨域解决方案"
subtitle:   "CrossOrigin"
date:       2019-12-23 21:39:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Nginx
    - CrossOrigin
---

[跨域限制访问](https://blog.csdn.net/why_still_confused/article/details/103218785)，即为浏览器禁止访问其他网站的资源，是浏览器的限制。如果缺少了同源策略，网页很容易受到XSS、CSFR等攻击。


> **同源策略**是Web应用程序安全性模型中的重要概念。根据该策略，Web浏览器允许第一个网页中包含的脚本访问第二个网页中的数据，但前提是两个网页具有相同的来源。来源由[URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)，主机名(hostname) 和端口号(port) 的组合定义。此策略可防止一个页面上的恶意脚本通过该页面的DOM(Document Object Model)获得对另一网页上敏感数据的访问。

## CORS
跨域资源共享(CORS) 是一种机制，它使用额外的 HTTP 头来告诉浏览器，它允许 Web 应用服务器进行跨域访问控制，从而使跨域数据传输得以安全进行。

> 跨域资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。另外，规范要求，**对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是`GET`以外的 HTTP 请求，或者搭配某些 MIME 类型的`POST`请求）**，浏览器必须首先使用[OPTIONS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS)方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括`Cookies`和 HTTP 认证相关数据）。

### 简单请求
某些请求不会触发 CORS 预检请求，可称这样的请求为“简单请求”。

> 请注意，该术语并不属于 Fetch （其中定义了 CORS）规范。

- 使用下列方法之一：`GET`、`HEAD`、`POST`
- `Content-Type`的值仅限于下列三者之一：`text/plain`、`multipart/form-data`、`application/x-www-form-urlencoded`
- ...
