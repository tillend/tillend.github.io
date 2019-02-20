---
layout:     post
title:      "Spring Boot 集成 Dubbo"
subtitle:   "微服务框架（二）"
date:       2018-07-16 12:00:00
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring Boot
    - Dubbo
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为此Spring Boot通过使用起步依赖dubbo-spring-boot-starter及自动装配的方式集成Dubbo。**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Spring Boot
　　Spring Boot旨在让开发人员使用最少的配置，尽可能快地着手开发并运行一个应用。Spring Boot框架从本质上来说就是Spring，但通过自动配置和起步依赖的方式将开发人员从繁重的配置中解放出来，专注于业务逻辑的实现。Spring Boot 的特性见[微服务框架（一）Spring Boot + Dubbo + Docker 框架特性简述](https://blog.csdn.net/why_still_confused/article/details/81045575)

### Profile运行环境配置
　　Spring Boot简化了开发人员需要的模板配置，因此我们只需要引入application.properties/application.yml即可。

> 在多环境配置下常使用`application.properties`只配置`spring.profiles.active`属性，以起选择对应环境的配置文件的作用

![这里写图片描述](/img/in-post/post-2018-07-15/application.png)

如上中，`application.properties`只需要配置`spring.profiles.active = dev`来选择配置文件，Spring Boot会自动加载config包下`application-dev.properties`的配置

> 在运行jar包或docker镜像时只需要设置`spring.profiles.active`运行变量

```
jar: `java -jar provider.jar --spring.profiles.active=dev`
docker: `-e "SPEING_PROFILES_ACTIVE=dev"`
```



## Dubbo

> Dubbo是一个由阿里巴巴开源的高性能、基于Java的RPC框架。与许多RPC系统一样，dubbo基于定义服务的思想，指定可以使用其参数和返回类型来远程调用的方法。在服务器端，Provider实现此接口并运行dubbo服务器来处理客户端调用。在客户端，Customer有一个提供与服务器相同的方法的存根。

　　[Dubbo Spring Boot](https://github.com/apache/incubator-dubbo-spring-boot-project) 项目致力于简化 Dubbo RPC 框架在 Spring Boot 应用场景的开发。同时也整合了 Spring Boot [自动装配](https://github.com/apache/incubator-dubbo-spring-boot-project/tree/master/dubbo-spring-boot-autoconfigure)及[生产就绪](https://github.com/apache/incubator-dubbo-spring-boot-project/tree/master/dubbo-spring-boot-actuator)的特性。

### 依赖引入及接口定义

Maven中引入`dubbo-spring-boot-starter`的起步依赖至`pom.xml`

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>
```

依赖关系：

|版本|Java|Spring Boot|	Dubbo|
|---|---|---|---|
|0.2.0	|1.8+	|2.0.x	|2.6.2+|
|0.1.1	|1.7+	|1.5.x|2.6.2+|

 Dubbo RPC API:
 
```java
public interface DemoService {
	String saySomething(String word);
}
```

### Dubbo Provider

1.编写Spring Boot启动类

> Dubbo的服务容器是一个 <font color="red">standalone</font> 的启动程序，不需要Web容器

```java
@SpringBootApplication
public class Starter {

	public static void main(String[] args) {
		new SpringApplicationBuilder(Starter.class)
			.web(WebApplicationType.NONE) // 非 Web 应用
			.run(args);
	}
}
```

2.实现`DemoService`接口

```java
@Service(version = "${demo.service.version}")
public class DemoServiceImpl implements DemoService {

	@Override
	public String saySomething(String word) {
		return StringUtils.isNotEmpty(word) ? word : "Say you won't let go";
	}

}
```

3.spring boot配置文件

> Spring Boot的自动装配及起步依赖使开发人员只需配置文件，而无需重复使用模板定义

application.properties/application.yml

```properties
# Spring boot application
spring.application.name = dubbo-provider-demo
server.port = 9090

# Service version
demo.service.version = 1.0.0

# Base packages to scan Dubbo Components (e.g @Service , @Reference)
dubbo.scan.basePackages  = org.spring.boot.dubbo.provider

# Dubbo Config properties
## ApplicationConfig Bean
dubbo.application.id = dubbo-provider-demo
dubbo.application.name = dubbo-provider-demo

## ProtocolConfig Bean
dubbo.protocol.id = dubbo
dubbo.protocol.name = dubbo
dubbo.protocol.port = 9900

## RegistryConfig Bean
dubbo.registry.id = my-registry
dubbo.registry.address = zookeeper://127.0.0.1:2181
```

>注：启动消费者时需启动对应的注册中心，否则服务会启动失败

### Dubbo Customer

> 使用Spring Boot框架构建Web应用时需引入`spring-boot-starter-web`的起步依赖

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
```

1.编写Spring Boot启动类（Web应用）

```java
@SpringBootApplication(scanBasePackages = "org.spring.boot.dubbo.consumer.controller")
public class Starter {

	public static void main(String[] args) {
		new SpringApplicationBuilder(Starter.class)
			.web(WebApplicationType.SERVLET).run(args);
	}
}
```

2.通过 `@Reference` 注入 `DemoService`

> 下述配置中消费者为直连生产者，未经过注册中心（`@Reference`中`url`参数若已配置则直连对应生产者，若未配置则经过配置文件中`dubbo.registry.address`参数连接注册中心，获取相应的生产者信息）

```java
@RestController
public class DemoController {

	@Reference(version = "${demo.service.version}", 
				url = "dubbo://localhost:9900")
	private DemoService demoService;

	@RequestMapping("/saySomething")
	public String sayHello(@RequestParam String name) {
		return demoService.saySomething(name);
	}
}
```

3.spring boot配置文件

 > Spring Boot的自动装配及起步依赖使开发人员只需配置文件，而无需重复使用模板定义

application.properties/application.yml

```properties
# Spring boot application
spring.application.name = dubbo-consumer-demo
server.port = 9090

# Service version
demo.service.version = 1.0.0

# Base packages to scan Dubbo Components (e.g @Service , @Reference)
dubbo.scan.basePackages  = org.spring.boot.dubbo.consumer.controller

# Dubbo Config properties
## ApplicationConfig Bean
dubbo.application.id = dubbo-consumer-demo
dubbo.application.name = dubbo-consumer-demo

## ProtocolConfig Bean
dubbo.protocol.id = dubbo
dubbo.protocol.name = dubbo
dubbo.protocol.port = 9901

## RegistryConfig Bean
dubbo.registry.id = my-registry
dubbo.registry.address = zookeeper://localhost:2181
```


---
参考资料：
 1. [Dubbo Spring Boot官方文档](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/README_CN.md)
