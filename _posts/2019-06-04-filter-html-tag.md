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

- HTML 文档和 HTML 元素是通过 HTML 标签进行标记的
- HTML 标签由开始标签和结束标签组成
- 开始标签是被括号包围的元素名
- 结束标签是被括号包围的斜杠和元素名
- 某些 HTML 元素没有结束标签，比如 `<br />`

> 注释：开始标签的英文翻译是 `start tag` 或 `opening tag`，结束标签的英文翻译是 `end tag` 或 `closing tag`

## HTML 转义字符

一些字符在 HTML 中拥有特殊的含义，比如小于号 (`<`) 用于定义 HTML 标签的开始。如果我们希望浏览器正确地显示这些字符，我们必须在 HTML 源码中插入<b>字符实体</b>。

字符实体有三部分：一个和号 (`&`)，一个实体名称，或者 `#` 和一个实体编号，以及一个分号 (`;`)。

要在 HTML 文档中显示小于号，我们需要这样写：`&lt;` 或者 `&#60;`

使用实体名称而不是实体编号的好处在于，名称相对来说更容易记忆。而这么做的坏处是，并不是所有的浏览器都支持最新的实体名称，然而几乎所有的浏览器对实体编号的支持都很好。

> 注意：实体对大小写敏感。    
> 详见[HTML 转义字符](http://www.w3chtml.com/html/character.html)


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
