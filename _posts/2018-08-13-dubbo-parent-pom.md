---
layout:     post
title:      "Spring Boot 通用Dubbo Parent POM"
subtitle:   "微服务框架（九）"
date:       2018-08-13 20:01:11
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - Dubbo
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为通用Dubbo Maven POM的集成，只需继承Parent POM即可使用**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## 通用Dubbo POM

为了封装项目间通用的依赖及配置，Maven使用`POM Reference`来解决这个问题，通过定义Parent POM供子类POM继承，即可实现“一处申明，多处使用”的目的。

> Maven POM 中可继承元素见[POM Reference](http://maven.apache.org/pom.html) 或 [Maven POM元素继承](https://www.cnblogs.com/maxiaofang/p/5944362.html)

dubbo通用POM通过以下配置引用
```xml
<parent>
    <groupId>com.test</groupId>
    <artifactId>dubbo.common.pom</artifactId>
    <version>${version}</version>
</parent>
```

## 自定义的Maven属性

### maven参数

```xml
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
<maven.compiler.source>1.8</maven.compiler.source>
<maven.compiler.target>1.8</maven.compiler.target>
<nexus.url>N/A</nexus.url>
```

### dubbo依赖版本

```xml
<dubbo.version>2.6.2</dubbo.version>
<disruptor.version>3.3.6</disruptor.version>
<netty.version>4.0.35.Final</netty.version>
<spring.version>5.0.7.RELEASE</spring.version>
<springboot.version>2.0.3.RELEASE</springboot.version>
<dubbo.springboot.version>0.2.0</dubbo.springboot.version>
<kryo.version>4.0.1</kryo.version>
<kryo-serializers.version>0.42</kryo-serializers.version>
```


## 项目的部署配置

```xml
<distributionManagement>
	<repository>
		<id>maven-releases</id>
		<name>Nexus Release Repository</name>
		<url>http://${nexus.url}/repository/maven-releases/</url>
	</repository>
	<snapshotRepository>
		<id>maven-snapshots</id>
		<name>Nexus Snapshot Repository</name>
		<url>http://${nexus.url}/repository/maven-snapshots/</url>
	</snapshotRepository>
</distributionManagement>
```


## 项目的依赖

`dubbo`通用的项目依赖，主要包括`dubbo`、`spring boot`、`mybatis`等起步依赖及`dubbo`相关的性能调优依赖

### 项目必须继承的依赖

| groupId  | artifactId  | version  | scope(默认为compile) | description |
|---|---|---|---|---|---|
| junit  | junit  | 4.12  | test | 单元测试 |
| com.alibaba.boot  | dubbo-spring-boot-starter  | 2.6.2  |  | dubbo-spring-boot启动器|
| org.springframework  | spring-context  | 5.0.7.RELEASE  |  | spring上下文 |
| org.springframework.boot  | spring-boot-starter-log4j2  | 2.0.3.RELEASE  |  | spring-boot log4j2 |
|com.lmax  | disruptor | 3.3.6  |  | log4j2日志异步化 |
|io.netty  | netty-all | 4.0.35.Final  |  | dubbo集成netty4 |
| org.springframework.boot  | spring-boot-starter-web  | 2.0.3.RELEASE  |  | spring-boot web |
| org.springframework.boot  | org.spring-boot-actuator  | 2.0.3.RELEASE  |  | spring-boot actuator |
| org.springframework.boot  | spring-boot-starter-test  | 2.0.3.RELEASE  |  | spring-boot test |
| org.springframework.boot  | spring-boot-starter-aop  | 2.0.3.RELEASE  |  | spring-boot aop |

```xml
<dependencies>
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
	<scope>test</scope>
</dependency>

<dependency>
	<groupId>com.alibaba.boot</groupId>
	<artifactId>dubbo-spring-boot-starter</artifactId>
	<version>${dubbo.springboot.version}</version>
	<exclusions>
		<exclusion>
			<artifactId>spring-boot-starter-logging</artifactId>
			<groupId>org.springframework.boot</groupId>
		</exclusion>
	</exclusions>
</dependency>

<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-context</artifactId>
	<version>${spring.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-log4j2</artifactId>
	<version>${springboot.version}</version>
</dependency>

<dependency>
	<groupId>com.lmax</groupId>
	<artifactId>disruptor</artifactId>
	<version>${disruptor.version}</version>
</dependency>

<dependency>
	<groupId>io.netty</groupId>
	<artifactId>netty-all</artifactId>
	<version>${netty.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<version>${springboot.version}</version>
	<exclusions>
		<exclusion>
			<artifactId>spring-boot-starter-tomcat</artifactId>
			<groupId>org.springframework.boot</groupId>
		</exclusion>
	</exclusions>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-actuator</artifactId>
	<version>${springboot.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<version>${springboot.version}</version>
	<scope>test</scope>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-aop</artifactId>
	<version>${springboot.version}</version>
</dependency>
```
	
	
### 项目可选继承的依赖
| groupId  | artifactId  | version  | scope(默认为compile) | description |
|---|---|---|---|---|---|
| org.mybatis.spring.boot  | mybatis-spring-boot-starter | 1.3.2  |  | springboot集成mybatis |
| org.springframework.boot  | spring-boot-starter-jdbc | 2.0.3.RELEASE  |  | spring-boot jdbc |
| org.springframework.boot  | spring-boot-autoconfigure | 2.0.3.RELEASE  |  | spring-boot autoconfigure |
| mysql  | mysql-connector-java | 5.1.24  |  | mysql |
| com.github.pagehelper  | pagehelper-spring-boot-starter | 1.2.5  |  | 分页 |
| com.alibaba | druid-spring-boot-starter | 1.1.10 |  | springboot集成druid |
| com.esotericsoftware  | kryo | 4.0.1  |  | kryo实现 |
| de.javakaffee  | kryo-serializers | 0.42  |  | kryo序列化 |	

```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>${mybatis.version}</version>
			<exclusions>
				<exclusion>
					<artifactId>spring-boot-starter-logging</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
				<exclusion>
					<artifactId>spring-boot-starter</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
			</exclusions>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
			<version>${springboot.version}</version>
			<exclusions>
				<exclusion>
					<artifactId>spring-boot-starter</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
			</exclusions>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-autoconfigure</artifactId>
			<version>${springboot.version}</version>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>${mysql.version}</version>
		</dependency>

		<!-- 分页插件 -->
		<dependency>
			<groupId>com.github.pagehelper</groupId>
			<artifactId>pagehelper-spring-boot-starter</artifactId>
			<version>${pagehelper.version}</version>
			<exclusions>
				<exclusion>
					<artifactId>spring-boot-starter</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
			</exclusions>
		</dependency>

		<!-- alibaba的druid数据库连接池 -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid-spring-boot-starter</artifactId>
			<version>${druid.version}</version>
		</dependency>

		<!-- kyro序列化依赖 -->
		<dependency>
			<groupId>com.esotericsoftware</groupId>
			<artifactId>kryo</artifactId>
			<version>${kryo.version}</version>
		</dependency>
		<dependency>
			<groupId>de.javakaffee</groupId>
			<artifactId>kryo-serializers</artifactId>
			<version>${kryo-serializers.version}</version>
		</dependency>
	</dependencies>
</dependencyManagement>
```
	

## build构建属性

包括项目的源码目录配置、输出目录配置、插件配置、插件管理配置等

### 项目输出目录配置

```xml
<resources>
	<!--recources文件夹下的所有文件都打进jar包 -->
	<resource>
		<targetPath>${project.build.directory}/classes</targetPath>
		<directory>src/main/resources</directory>
		<filtering>true</filtering>
		<includes>
			<include>**/*.xml</include>
			<include>**/*.properties</include>
			<include>**/*.json</include>
			<include>**/*.yml</include>
			<include>META-INF/dubbo/*</include>
		</includes>
	</resource>

	<!-- 满足docker-maven-plugin的约定:将 dockerfile文件拷贝到docker目录下 -->
	<resource>
		<targetPath>${project.build.directory}/docker</targetPath>
		<directory>src/main/resources/docker</directory>
		<filtering>true</filtering>
		<includes>
			<include>Dockerfile</include>
			<include>.dockerignore</include>
		</includes>
	</resource>
</resources>
```

### 插件管理配置

```xml
<pluginManagement><!-- lock down plugins versions to avoid using Maven 
		defaults (may be moved to parent pom) -->
	<plugins>
		<plugin>
			<artifactId>maven-clean-plugin</artifactId>
			<version>3.0.0</version>
		</plugin>
		<plugin>
			<artifactId>maven-resources-plugin</artifactId>
			<version>3.0.2</version>
		</plugin>
		<plugin>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.7.0</version>
		</plugin>
		<plugin>
			<artifactId>maven-surefire-plugin</artifactId>
			<version>2.20.1</version>
		</plugin>
		<plugin>
			<artifactId>maven-install-plugin</artifactId>
			<version>2.5.2</version>
		</plugin>
		<plugin>
			<artifactId>maven-deploy-plugin</artifactId>
			<version>2.8.2</version>
		</plugin>
	</plugins>
</pluginManagement>
```

### JAVA源码插件

```xml
<plugin>
	<artifactId>maven-source-plugin</artifactId>
	<version>2.3</version>
	<executions>
		<execution>
			<id>attach-source</id>
			<goals>
				<goal>jar</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

### JAVA文档插件

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-javadoc-plugin</artifactId>
	<version>2.9.1</version>
	<configuration>
		<charset>UTF-8</charset>
		<encoding>UTF-8</encoding>
		<docencoding>UTF-8</docencoding>
		<includePom>true</includePom>
		<excludeResources>true</excludeResources>
		<attach>true</attach>
	</configuration>
	<executions>
		<execution>
			<id>attach-javadocs</id>
			<goals>
				<goal>jar</goal>
			</goals>
			<configuration>
				<encoding>UTF-8</encoding>
				<additionalparam>${javadoc.opts}</additionalparam>
			</configuration>
		</execution>
	</executions>
</plugin>
```	
			
### jar生成插件
继承POM后需在`properties`中定义`springboot.starter`变量，值为spring-boot启动类路径

```xml
<!-- 打包jar文件时，配置manifest文件，加入lib包的jar依赖 -->
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<configuration>
		<classesDirectory>target/classes/</classesDirectory>
		<archive>
			<manifest>
				<mainClass>${springboot.starter}</mainClass>
				<!-- 打包时 MANIFEST.MF文件不记录的时间戳版本 -->
				<useUniqueVersions>false</useUniqueVersions>
				<addClasspath>true</addClasspath>
				<classpathPrefix>lib/</classpathPrefix>
			</manifest>
			<manifestEntries>
				<Class-Path>.</Class-Path>
			</manifestEntries>
		</archive>
	</configuration>
</plugin>
```

### 项目依赖拷贝插件

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-dependency-plugin</artifactId>
	<executions>
		<execution>
			<id>copy-dependencies</id>
			<phase>package</phase>
			<goals>
				<goal>copy-dependencies</goal>
			</goals>
			<configuration>
				<type>jar</type>
				<includeTypes>jar</includeTypes>
				<useUniqueVersions>false</useUniqueVersions>
				<outputDirectory> ${project.build.directory}/lib </outputDirectory>
			</configuration>
		</execution>
	</executions>
</plugin>
```

### Docker镜像化插件

使用方法详见[Docker镜像化Dubbo微服务](https://blog.csdn.net/why_still_confused/article/details/81391827)，详细信息可见[官方文档](https://github.com/spotify/docker-maven-plugin)

```xml
<plugin>
	<groupId>com.spotify</groupId>
	<artifactId>docker-maven-plugin</artifactId>
	<version>1.0.0</version>
	<configuration>
		<imageName>${registry.url}dubbo/${project.artifactId}</imageName>
		<dockerDirectory>src/main/resources/docker</dockerDirectory>
		<imageTags>
			<imageTag>${image.tag}</imageTag>
			<imageTag>latest</imageTag>
		</imageTags>
		<resources>
			<resource>
				<targetPath>/</targetPath>
				<directory>${project.build.directory}</directory>
				<includes>
					<include>${project.build.finalName}.jar</include>
					<include>lib/*.jar</include>
				</includes>
			</resource>
		</resources>
	</configuration>
</plugin>
```


---
参考资料：
1. [POM Reference](http://maven.apache.org/pom.html)
2. [Maven POM元素继承](https://www.cnblogs.com/maxiaofang/p/5944362.html)
