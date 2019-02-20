---
layout:     post
title:      "Spring Boot Redis 缓存"
subtitle:   "微服务框架（十二）"
date:       2018-09-09 10:27:58
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - Redis
    - 译
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Spring Boot集成Redis**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Spring Boot Redis缓存

　　在这篇文章中，我们将配置一个Spring Boot应用程序示例，并将其与[Redis](https://redis.io/) Cache 集成。虽然Redis是一个**开源内存数据结构存储**，用作数据库，缓存和消息代理，但本文仅演示缓存集成。

　　我们将使用[Spring Initializr](https://start.spring.io/)工具快速设置项目。

### Spring Boot Redis项目设置

　　我们将使用Spring Initializr工具快速设置项目。我们将使用3个依赖项，如下所示：
![spring boot redis cache示例](/img/in-post/post-2018-09/spring-redis-project-setup.png)

　　下载项目并解压缩。我们使用H2数据库依赖，因为我们将使用嵌入式数据库，一旦应用程序停止，该数据库将丢失所有数据。

### Maven依赖项
　　虽然我们已经使用该工具完成了自动设置，但如果您想手动设置它，我们将为此项目使用Maven构建系统，这里是我们使用的依赖项：

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>1.5.9.RELEASE</version>
  <relativePath/> <!-- lookup parent from repository -->
</parent>
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <java.version>1.8</java.version>
</properties>
<dependencies>
  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>

  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-test</artifactId>
     <scope>test</scope>
  </dependency>

  <!-- for JPA support -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <!-- for embedded database support -->
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>

</dependencies>

<build>
  <plugins>
     <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
     </plugin>
  </plugins>
</build>
```

　　确保从[maven中央仓库](https://mvnrepository.com/)使用Spring Boot的稳定版本。

### 定义模型
　　要将对象保存到Redis数据库，我们使用基本字段定义Person模型对象：

```java
package com.journaldev.rediscachedemo;

import javax.persistence.*;
import java.io.Serializable;

@Entity
public class User implements Serializable {

    private static final long serialVersionUID = 7156526077883281623L;

    @Id
    @SequenceGenerator(name = "SEQ_GEN", sequenceName = "SEQ_USER", allocationSize = 1)
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "SEQ_GEN")
    private Long id;
    private String name;
    private long followers;

    public User() {
    }

    public User(String name, long followers) {
        this.name = name;
        this.followers = followers;
    }

    //standard getters and setters

    @Override
    public String toString() {
        return String.format("User{id=%d, name='%s', followers=%d}", id, name, followers);
    }
}
```
　　这是一个标准的POJO，有getters 和 setters。

### 配置Redis缓存
　　使用Spring Boot集成了Redis相关的起步依赖后，我们可以在`application.properties`文件中定义只有三行的配置来使用本地Redis实例：

```
# Redis Config
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
```
　　另外，在Spring Boot启动类上使用注释`@EnableCaching`：

```java
package com.journaldev.rediscachedemo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class Application implements CommandLineRunner {

  private final Logger LOG = LoggerFactory.getLogger(getClass());
  private final UserRepository userRepository;

  @Autowired
  public Application(UserRepository userRepository) {
    this.userRepository = userRepository;
  }

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

  @Override
  public void run(String... strings) {
    //Populating embedded database here
    LOG.info("Saving users. Current user count is {}.", userRepository.count());
    User shubham = new User("Shubham", 2000);
    User pankaj = new User("Pankaj", 29000);
    User lewis = new User("Lewis", 550);

    userRepository.save(shubham);
    userRepository.save(pankaj);
    userRepository.save(lewis);
    LOG.info("Done saving users. Data: {}.", userRepository.findAll());
  }
}
```

　　我们已经添加了一个CommandLineRunner，因为我们想要在嵌入式H2数据库中填充一些示例数据。

### 定义存储库
　　在我们展示Redis如何工作之前，我们将为JPA相关功能定义一个Repository：

```java
package com.journaldev.rediscachedemo;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository { }
```
　　它现在没有方法调用，因为我们不需要任何方法调用。

### 定义控制器
　　**控制器**是调用Redis缓存以进行操作的位置。实际上，这是最好的地方，因为缓存与它直接相关，请求甚至不必进入具体业务逻辑的服务代码来等待缓存的结果。

　　这是控制器骨架：

```java
package com.journaldev.rediscachedemo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.web.bind.annotation.*;

@RestController
public class UserController {

  private final Logger LOG = LoggerFactory.getLogger(getClass());

  private final UserRepository userRepository;

  @Autowired
  public UserController(UserRepository userRepository) {
    this.userRepository = userRepository;
  }
   ...
}
```
　　现在，为了将某些内容放入缓存中，我们使用`@Cacheable`注释：

```java
@Cacheable(value = "users", key = "#userId", unless = "#result.followers < 12000")
@RequestMapping(value = "/{userId}", method = RequestMethod.GET)
public User getUser(@PathVariable String userId) {
  LOG.info("Getting user with ID {}.", userId);
  return userRepository.findOne(Long.valueOf(userId));
}
```
　　在上面的映射中，`getUser`方法将一个人放入名为“users”的缓存中，通过键将该对象识别为“userId”，并且仅存储具有大于12000的关注者的用户。这可确保缓存里填充的是很受欢迎、经常被查询的用户。

　　此外，我们有意在API调用中添加了一个日志语句，现在让我们从Postman做一些API调用。这些是我们的调用：
```
localhost:8090/1
localhost:8090/1
localhost:8090/2
localhost:8090/2
```
　　如果我们注意到日志，这些API调用情况的日志将是：

```
... : Getting user with ID 1.
... : Getting user with ID 1.
... : Getting user with ID 2.
```

　　注意什么？我们进行了四次API调用，但只有三个日志语句。这是因为具有ID 2的用户拥有29000个关注者，因此，它的数据被缓存。这意味着当为它进行API调用时，数据从缓存中返回，并且没有为此进行数据库调用！

### 更新缓存
　　每当更新其实际对象值时，缓存值也应更新。这可以使用`@CachePut`注解完成：

```java
@CachePut(value = "users", key = "#user.id")
@PutMapping("/update")
public User updatePersonByID(@RequestBody User user) {
  userRepository.save(user);
  return user;
}
```
　　这样，一个人再次通过他的ID识别并用结果更新。

### 清除缓存
　　如果要从实际数据库中删除某些数据，则不再需要将其保留在缓存中。我们可以使用`@CacheEvict`注解清除缓存数据：

```java
@CacheEvict(value = "users", allEntries=true)
@DeleteMapping("/{id}")
public void deleteUserByID(@PathVariable Long id) {
  LOG.info("deleting person with id {}", id);
  userRepository.delete(id);
}
```
　　在最后一个映射中，我们只是逐出缓存条目而没有做任何其他事情。

## 运行Spring Boot Redis缓存应用程序
　　我们只需使用一个命令即可运行此应用程序：

```
mvn spring-boot:run
```

## Redis缓存限制
　　尽管Redis速度非常快，但在64位系统上存储任何数据量仍然没有限制。它只能在32位系统上存储3GB的数据。更多可用内存可以带来更高的命中率，但是一旦Redis占用太多内存，会导致Redis停止。

　　当缓存大小达到内存限制时，将删除旧数据以替换新数据。

## 总结
　　在本文中，我们了解了Redis Cache为我们提供了快速数据交互的强大功能，以及我们如何将它与Spring Boot集成在一起，只需极少但功能强大的配置。

---
原文地址：[Spring Boot Redis Cache](https://www.journaldev.com/18141/spring-boot-redis-cache) written by Shubham

完整代码：[下载地址](https://www.journaldev.com/wp-content/uploads/spring/Spring-Boot-Redis-Cache.zip)
