---
layout:     post
title:      "HTML 标签、转义字符及相应的 Java 过滤方法"
subtitle:   "工具"
date:       2019-06-04 21:41:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - HTML
    - JavaScript
    - Java
---

## HTML 标签
[HTML 标签](http://www.w3chtml.com/html/primary.html)

## HTML 转义字符
[HTML 转义字符](http://www.w3chtml.com/html/character.html)


## JavaScript 转义符

|转义序列|字符|
|---|---|
|\b|退格|
|\f|换页|
|\n|换行|
|\r|回车|
|\t|横向跳格(Ctrl-I)
|\'|单引号|
|\"|双引号|
|\\\ |反斜杠|



## Java 过滤标签及转义字符
#### HTML 标签过滤

正则表达式过滤
```java
String txtcontent = content.replaceAll("</?[^>]+>", "");
```

#### HTML 转义字符过滤
`org.apache.commons.lang3.StringEscapeUtils`
```java
String txtcontent = StringEscapeUtils.unescapeHtml4(content);
```


对于更复杂的需求，可考虑选用`Jsoup`提取相应的数据

> [Jsoup](https://jsoup.org) 是一个用于处理 HTML 的 Java 库。它提供了一些非常方便的 API，通过使用最好的 DOM，CSS 和类 jquery 的方法，以提取和操作数据。

#### JavaScript 转义符过滤
过滤换行符
```java
String txtcontent = content.replaceAll("\n", "");
```

过滤所有
```java
String txtcontent = content.replaceAll("\\s*", "");
```

---
参考资料：
1. [HTML 转义字符](http://www.w3chtml.com/html/character.html)
2. [Jsoup](https://jsoup.org)
3. [jsoup Cookbook](https://www.open-open.com/jsoup/)
