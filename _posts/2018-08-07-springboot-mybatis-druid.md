---
layout:     post
title:      "Spring Boot集成Mybatis及Druid"
subtitle:   "微服务框架（六）"
date:       2018-08-07 18:42:53
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - Dubbo
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Spring Boot集成Mybatis及Druid，包括mybatis-generator的使用**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Mybatis

MyBatis 是一款优秀的**持久层框架**，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(`Plain Ordinary Java Object`,普通的 Java对象)映射成数据库中的记录。

### mybatis-generator

> 使用`mybatis-generator`可快速生成对应的XMLMapper的文件映射及模型(暂无使用`Spring Boot`集成的打算)

 1. 配置对应的**数据库链接**
 2. 配置**模型**的包名及生成位置
 3. 配置**映射文件**的包名及生成位置
 4. 配置**DAO**的包名和生成位置
 5. 配置**表或视图**的参数

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE generatorConfiguration  
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"  
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">  
<generatorConfiguration>  
<!-- 数据库驱动-->  
    <classPathEntry  location="mysql-connector-java-5.1.24.jar"/>  
    <context id="DB2Tables"  targetRuntime="MyBatis3">  
        <commentGenerator>  
            <property name="suppressDate" value="true"/>  
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->  
            <property name="suppressAllComments" value="true"/>  
        </commentGenerator>  
        <!--数据库链接URL，用户名、密码 -->  
        <jdbcConnection driverClass="com.mysql.jdbc.Driver" connectionURL="jdbc:mysql://" userId="" password="">  
        </jdbcConnection>  
        <javaTypeResolver>  
            <property name="forceBigDecimals" value="false"/>  
        </javaTypeResolver>  
        <!-- 生成模型的包名和位置-->  
        <javaModelGenerator targetPackage="spring.boot.model" targetProject="src">  
            <property name="enableSubPackages" value="true"/>  
            <property name="trimStrings" value="true"/>  
        </javaModelGenerator>  
        <!-- 生成映射文件的包名和位置-->  
        <sqlMapGenerator targetPackage="mapper" targetProject="src">  
            <property name="enableSubPackages" value="true"/>  
        </sqlMapGenerator>  
        <!-- 生成DAO的包名和位置-->  
        <javaClientGenerator type="XMLMAPPER" targetPackage="spring.boot.dao" targetProject="src">  
            <property name="enableSubPackages" value="true"/>  
        </javaClientGenerator>  
        <!-- 要生成的表 tableName是数据库中的表名或视图名 domainObjectName是实体类名-->  
        <table tableName="test" domainObjectName="Test" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
    </context>  
</generatorConfiguration>  
```

### 缓存

MyBatis 包含一个非常强大的查询缓存特性,它可以非常方便地配置和定制。MyBatis 3 中的缓存实现的很多改进都已经实现了,使得它更加强大而且易于配置。
默认情况下是没有开启缓存的,除了局部的 session 缓存,可以增强变现而且处理循环依赖也是必须的。要开启**二级缓存**,你需要在你的 SQL 映射文件中添加一行:

```xml
<cache/>
```

字面上看就是这样。这个简单语句的效果如下:

- 映射语句文件中的所有 select 语句将会被缓存。
- 映射语句文件中的所有 insert,update 和 delete 语句会刷新缓存。
- 缓存会使用 Least Recently Used(LRU,最近最少使用的)算法来收回。
- 根据时间表(比如 no Flush Interval,没有刷新间隔), 缓存不会以任何时间顺序 来刷新。
- 缓存会存储列表集合或对象(无论查询方法返回什么)的 1024 个引用。
- 缓存会被视为是 read/write(可读/可写)的缓存,意味着对象检索不是共享的,而 且可以安全地被调用者修改,而不干扰其他调用者或线程所做的潜在修改。

> 详细缓存使用见[官方文档](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#cache)

### mybatis sql注意事项

注：

 1. 非预编译sql语句推荐使用`${}`而非`#{}`的占位符。
 2. mybatis XML中`sql`请使用`<`符号请使用此样式`<![CDATA[ < ]]>`
 3. 模糊搜索中应使用字符串拼接函数`CONCAT('%',#{word},'%')`


## Spring Boot集成Mybatis、Druid

 1. 引入起步依赖
 2. 添加对应的模型及Mapper
 3. 配置Druid数据源
 4. Spring Boot启动类扫描DAO

### 起步依赖

`Spring Boot`集成`mybatis`及`druid`时需引入对应的起步依赖

```xml
<!-- 数据库依赖 -->
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-autoconfigure</artifactId>
</dependency>

<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid-spring-boot-starter</artifactId>
</dependency>
```


### 添加对应的模型及Mapper

使用`mybatis-generator`自动生成对应的模型及Mapper后，只需粘贴至对应的包目录下即可

> `application.properties`中需引入对应的`mybatis`配置

```properties
mybatis.mapper-locations = classpath:mapper/*.xml
mybatis.type-aliases-package = spring.boot.service.model
```

### 配置Druid数据源

> 在环境文件配置`spring.profiles.active = dev,data`即可使用

`Druid`数据源配置(`application-data.properties`)

```properties
spring.datasource.name = mysql
spring.datasource.type = com.alibaba.druid.pool.DruidDataSource
spring.datasource.init.method = init
spring.datasource.destroy.method = close
spring.datasource.druid.filters = stat
spring.datasource.druid.driver-class-name = com.mysql.jdbc.Driver
spring.datasource.druid.url = jdbc:mysql://${db.host}:${db.port}/${db.name}
spring.datasource.druid.username = ${db.username}
spring.datasource.druid.password = ${db.password}
spring.datasource.druid.initial-size = 1
spring.datasource.druid.min-idle = 1
spring.datasource.druid.max-active = 20
spring.datasource.druid.max-wait = 60000
spring.datasource.druid.time-between-eviction-runs-millis = 3000
spring.datasource.druid.min-evictable-idle-time-millis = 300000
spring.datasource.druid.validation-query = SELECT 'x'
spring.datasource.druid.test-while-idle = true
spring.datasource.druid.test-on-borrow = false
spring.datasource.druid.test-on-return = false
spring.datasource.druid.pool-prepared-statements = false
spring.datasource.druid.max-pool-prepared-statement-per-connection-size = 20


mybatis.mapper-locations = classpath:mapper/*.xml
mybatis.type-aliases-package = spring.boot.service.model
```

> 为方便切换环境配置，上述db参数于对应环境配置文件里设置(如application-dev.properties)

```properties
#mysql db
db.host = localhost
db.name = name
db.port = 3306
db.username = username
db.password = password
```

### Spring Boot启动类扫描DAO

使用`@MapperScan`注解扫描包路径，以注册`MyBatis mapper`接口

```java
@SpringBootApplication
@EnableAspectJAutoProxy(proxyTargetClass = true)
@ComponentScan("spring.boot.service")
@MapperScan("spring.boot.service.dao")
public class Starter {

	public static void main(String[] args) {

		new SpringApplicationBuilder(Starter.class)
				.web(WebApplicationType.NONE).run(args);

	}

}
```


---
参考资料：
1. [Mybatis cache](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.htm)
