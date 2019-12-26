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

> 所有简单请求见[MDN 简单请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS#%E8%8B%A5%E5%B9%B2%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6%E5%9C%BA%E6%99%AF)

### 预检请求（preflight request）
与前述简单请求不同，**需预检的请求**要求必须首先使用 `OPTIONS`   方法发起一个预检请求到服务器，**以获知服务器是否允许该实际请求**。"预检请求“的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。

> 对于附带身份凭证的请求，服务器不得设置`Access-Control-Allow-Origin`的值为“*”，需设置为具体的域名，否则请求会失败。

## Nginx跨域配置
[NGINX](https://www.nginx.com/resources/wiki/)是一个免费的，开源的高性能HTTP服务器和反向代理，以及`IMAP / POP3`代理服务器。`NGINX`以其高性能，稳定性，丰富的功能集，简单的配置和低资源消耗而闻名。

通常在`nginx`下设置通用配置，以解决域名下的跨域问题
### 简单请求跨域配置
```conf
Access-Control-Allow-Origin: *.test.com
Access-Control-Allow-Credentials: true
```
### 预检请求过程
当需要支持跨域的请求不是简单请求时，需特殊处理预检请求所发起的`OPTIONS`请求

```conf
set $cors '';
if ($http_origin ~* 'https?://(localhost|www\.example\.com|m\.example\.com)') {
        set $cors 'true';
}

if ($cors = 'true') {
        add_header 'Access-Control-Allow-Origin' "$http_origin";
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With';
}

if ($request_method = 'OPTIONS') {
        return 204;
}
```

下图为，以真实请求的HTTP方法及[请求首部字段](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS#HTTP_%E8%AF%B7%E6%B1%82%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5)发起对应的预检请求

![](https://mdn.mozillademos.org/files/16753/preflight_correct.png)



---
参考资料：
1. [HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)