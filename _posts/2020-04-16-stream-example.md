---
layout:     post
title:      "Java Stream API与Lambda表达式常用场景"
subtitle:   ""
date:       2020-04-16 21:13:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Java
    - Stream
    - Lambda
---


## Lambda表达式及方法引用

[Lambda表达式](http://blog.csdn.net/why_still_confused/article/details/78926850#t2)允许我们將函数作为一个方法的参数传递到方法体中或者將一段代码作为数据。
[方法引用](http://blog.csdn.net/why_still_confused/article/details/78926850#t4)提供了一种非常有用的语法去直接引用类或对象的方法（或构造函数）。与Lambda表达式结合使用，方法引用使语言结构看起来简洁紧凑。

> 为了更好地理解本节中的内容，可查看[Stream API与Lambda表达式的聚合操作](https://blog.csdn.net/why_still_confused/article/details/78965263)

## 常用场景
以下以该模型作为示例，描述常用场景的使用方式
```java
public class Person {
    public enum Sex {
        MALE, FEMALE
    }

    String name;
    LocalDateTime birthday;
    Sex gender;
    String emailAddress;
    //setter&getter
}
```

### List类型转换
所有成员的姓名

```java
List<String> nameList = list.stream()
        .map(Person::getName)
        .collect(Collectors.toList());
```

### List转Map
所有成员的姓名及生日
```java
Map<String, LocalDateTime> map = list.stream()
        .collect((Collectors.toMap(Person::getName, Person::getBirthday)));
```

### List过滤
所有女性成员的出生年份
```java
List<Integer> yearList = list.stream()
        .filter(p -> p.getGender() == Person.Sex.FEMALE)
        .map(y -> y.getBirthday().getYear())
        .collect(Collectors.toList());
```

### List条件判断

- anyMatch 
- allMatch 
- noneMatch

是否存在出生日期为今年的数据
```java
boolean hasNowYear = list.stream()
        .anyMatch(p -> p.getBirthday().getYear() == LocalDateTime.now().getYear());
```

### 平均值

所有男性成员的平均年龄
```java
double average = list.stream()
    .filter(p -> p.getGender() == Person.Sex.MALE)
    .mapToInt(Person::getAge)  
    .average()
    .getAsDouble();
```