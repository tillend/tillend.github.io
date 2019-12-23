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
