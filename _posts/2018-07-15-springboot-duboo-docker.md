---
layout:     post
title:      "Spring Boot + Dubbo + Docker 框架特性简述"
subtitle:   "微服务框架（一）"
date:       2018-07-15 12:00:00
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring Boot
    - Dubbo
    - Docker
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。
  
　　**本文为此微服务框架的特性简述。**
  
> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## 框架简述
　　现微服务框架基于Spring Boot框架开发，具有Spring Boot完整的生态，且服务的启动**不需要服务容器**，能减少复杂性，也节省资源。
  
　　Dubbo作为**服务治理框架**，能更好地管理服务及对发生事故的服务进行追踪，使用优雅停机以停止服务时也不会影响同服务器中其他的服务；还有软负载均衡、服务发现和服务熔断等功能。
  
　　Docker作为**容器化技术**，能提供一次性的环境，也能**简化运行环境的配置**，最重要的是能很多地**隔离多个服务应用的资源**，又有**很低的虚拟化开销**。

### Spring Boot
　　Spring Boot旨在让开发人员使用最少的配置，尽可能快地着手开发并运行一个应用。Spring Boot框架从本质上来说就是Spring，但通过**自动配置**和**起步依赖**的方式将开发人员从繁重的配置中解放出来，专注于业务逻辑的实现。详见[Spring官方文档](https://spring.io/)

**Spring Boot核心**

 - 自动配置
 - 起步依赖
 - 命令行界面
 - Actuator

#### 自动配置
　　在任何传统Spring应用程序的源代码里，都有为应用程序开启特定的特性和功能的Java配置或XML配置，如下定式Spring集成Mybatis的XML配置。而Spring Boot会为这些常见的配置场景进行自动配置，为你自动配置相应的dataSource及sqlSessionFactory。

```xml
<!-- 引入配置文件 -->
<bean id="propertyConfigurer"
	class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name="location" value="classpath:jdbc.properties" />
</bean>

<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
	destroy-method="close">
	<property name="driverClassName" value="${driver}" />
	<property name="url" value="${url}" />
	<property name="username" value="${username}" />
	<property name="password" value="${password}" />
	<!-- 初始化连接大小 -->
	<property name="initialSize" value="${initialSize}"></property>
	<!-- 连接池最大数量 -->
	<property name="maxActive" value="${maxActive}"></property>
	<!-- 连接池最大空闲 -->
	<property name="maxIdle" value="${maxIdle}"></property>
	<!-- 连接池最小空闲 -->
	<property name="minIdle" value="${minIdle}"></property>
	<!-- 获取连接最大等待时间 -->
	<property name="maxWait" value="${maxWait}"></property>
</bean>

<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="dataSource" ref="dataSource" />
	<!-- 自动扫描mapping.xml文件 -->
	<property name="mapperLocations" value="classpath:mapper/*.xml"></property>
</bean>

<!-- DAO接口所在包名，Spring会自动查找其下的类 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
	<property name="basePackage" value="cn.stu.edu.lin.dao" />
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
</bean>
```

#### 起步依赖
　　Spring Boot通过起步依赖为项目的依赖管理提供帮助。如上述代码中Spring集成Mybatis的过程中，需注意相关库的Group、Artifact及version，在经过测试后方能排除依赖冲突及解决不兼容的情况。
  
　　起步依赖其实就是特殊的Maven和Gradle依赖，利用了依赖管理中的传递依赖解析特性，把经过测试的常用库聚合在一起，组成了为特定功能而定制的依赖。
![这里写图片描述](/img/in-post/post-2018-07-15/starter.png)

#### 命令行界面
　　Spring Boot CLI利用了自动配置和起步依赖，使开发人员能专注于代码。CLI会检测到代码中正在使用的特定类，自动解析合适的依赖库来支持那些类，还可以检测到当前运行的是一个Web容器，并自动引入嵌入式Web容器以供应用程序运行时使用。

#### Actuator
　　Actuator提供在运行时检视应用程序内部情况的能力。

> **Actuator能检视到应用程序内的以下细节，详见Spring Boot实战第七章：**
- Spring应用程序上下文里配置的Bean
- Spring Boot的自动配置做的决策
- 应用程序取到的环境变量、系统属性、配置属性和命令行参数
- 应用程序里线程的当前状态
- 应用程序最近处理过的HTTP请求的追踪情况
- 各种和内存用量、垃圾回收、Web请求以及数据源用量相关的指标



### Dubbo
　　Dubbo是一个分布式服务框架，以及微服务治理方案。其功能主要包括：高性能NIO通讯及多协议集成，服务动态寻址与路由，软负载均衡与容错，依赖分析与降级等。 
  
![这里写图片描述](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-architecture.jpg)

|节点	|角色说明|
|:---:|:---:|
|Provider	|暴露服务的服务提供方|
|Consumer	|调用远程服务的服务消费方|
|Registry|	服务注册与发现的注册中心|
|Monitor	|统计服务的调用次数和调用时间的监控中心|
|Container|	服务运行容器|

**调用关系说明**

 1. 服务容器负责启动，加载，运行服务提供者。 
 2. 服务提供者在启动时，向注册中心注册自己提供的服务。
 3. 服务消费者在启动时，向注册中心订阅自己所需的服务。
 4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
 5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
 6. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

> 关于Dubbo的架构及特性详细说明见[Dubbo官方文档](http://dubbo.apache.org/#/docs/user/preface/architecture.md?lang=zh-cn)

### Docker
　　Docker 使用 Google 公司推出的 Go 语言 进行开发实现，基于 Linux 内核的[cgroup](https://zh.wikipedia.org/wiki/Cgroups)，[namespace](https://en.wikipedia.org/wiki/Linux_namespaces)，以及[AUFS](https://en.wikipedia.org/wiki/Aufs)类的[Union FS](https://en.wikipedia.org/wiki/Union_mount)等技术，对进程进行封装隔离，属于[操作系统层面的虚拟化技术](https://en.wikipedia.org/wiki/Operating-system-level_virtualization)。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。
  
> 最初实现是基于 LXC，从 0.7 版本以后开始去除 LXC，转而使用自行开发的 libcontainer，从 1.11 开始，则进一步演进为使用 runC 和 containerd。

　　Docker 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。使得 Docker 技术比虚拟机技术更为轻便、快捷。

#### 镜像与容器

> 镜像（Image）就是一堆**只读层**（read-only layer）的统一视角；容器（container）的定义和镜像（image）几乎一模一样，唯一区别在于容器的最上面那一层是**可读可写**的。

　　为了使Docker镜像具有可移植性，能在Java的基础上进一步实现“一次编译，到处运行”的特点，Docker镜像是由一堆只读层组成的，而Docker容器则是在镜像的基础上增加了一层读写层。
> **容器 = 镜像 + 读写层**

![Docker镜像与容器](http://dockone.io/uploads/article/20151103/d6ad9c257d160164480b25b278f4a2ad.png)


---
参考资料：

 1. [Spring Boot实战](https://book.douban.com/subject/26857423/)
 2. [Dubbo用户文档](http://dubbo.apache.org/#/docs/user/preface/background.md?lang=zh-cn)
 3. [10张图带你深入理解Docker容器和镜像](http://dockone.io/article/783)

