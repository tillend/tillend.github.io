---
layout:     post
title:      "Maven Archetype制作Dubbo项目原型"
subtitle:   "微服务框架（十）"
date:       2018-08-15 22:30:15
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Maven
    - Dubbo
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Maven Archetype的制作及使用，使用archetype插件制作Dubbo项目原型**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Maven Archetype

原型(Archetypes)打包在JAR中，它们包含描述原型内容的原型元数据，以及构成原型项目的一组敏捷开发模板。

> Archetypes are packaged up in a JAR and they consist of the archetype metadata which describes the contents of archetype, and a set of Velocity templates which make up the prototype project. 

原型项目的开发分为以下步骤：

 1. **配置Maven archetype插件**
 2. **配置原型描述文件**
 3. **配置相关原型中的文件**
 4. **构建至本地或推送至私有仓库**


## POM配置

配置Maven archetype插件

```xml
<pluginManagement>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-archetype-plugin</artifactId>
			<version>3.0.0</version>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.6.1</version>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
			</configuration>
		</plugin>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-resources-plugin</artifactId>
			<version>3.0.2</version>
			<configuration>
				<encoding>UTF-8</encoding>
				<includeEmptyDirs>true</includeEmptyDirs>
			</configuration>
		</plugin>
	</plugins>
</pluginManagement>

```

若需推送到私有仓库，还需配置对应的私有仓库部署配置

```xml
	<properties>
		<nexus.url>ip:port</nexus.url>
	</properties>


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

## archetype-descriptor

原型描述文件需命名为[archetype-metadata.xml](http://maven.apache.org/archetype/archetype-models/archetype-descriptor/archetype-descriptor.html),必须设在`src/main/resources/META-INF/maven/`

```xml
<?xml version="1.0" encoding="UTF-8"?>

<archetype-descriptor name="dubbo-common-archetype">
	<fileSets>
		<fileSet filtered="true" encoding="UTF-8">
			<directory>src/main/java</directory>
			<includes>
				<include>**/*.**</include>
			</includes>
		</fileSet>
		<fileSet filtered="true" encoding="UTF-8">
			<directory>src/main/resources</directory>
			<includes>
				<include>**/*.xml</include>
				<include>**/**</include>
			</includes>
		</fileSet>
		<fileSet filtered="true" encoding="UTF-8">
			<directory>src/test/java</directory>
			<includes>
				<include>**/*.**</include>
			</includes>
		</fileSet>
		<fileSet filtered="true" encoding="UTF-8">
			<directory></directory>
			<includes>
				<include>.**</include>
			</includes>
		</fileSet>
	</fileSets>
</archetype-descriptor>

```

原型项目路径结构：

> `__packageInPathFormat__`会自动适配生成Maven项目时设置的path

```
archetype
|-- pom.xml
`-- src
    `-- main
        `-- resources
            |-- META-INF
            |   `-- maven
            |       `--archetype.xml
            `-- archetype-resources
                |-- pom.xml
                `-- src
                    |-- main
                    |   |-- java
                    |   |   `-- __packageInPathFormat__
                    |   |   	`-- handler
                    |   |   	`-- model
                    |   |   	`-- provider
                    |   |   	`-- resources
                    |   |   	`-- starter
                    |   |   	`-- utils
                    |   |-- resources
                    |       `-- config
                    |       `-- mapper
                    |       `-- docker
                    |       `-- application.properties
                    |       `-- log4j2.xml
                    `-- test
                        `-- java

```

在Java文件头中配置以下参数,则会在生成Maven项目时自动适配相应的包路径

```java
#set( $symbol_pound = '#' )
#set( $symbol_dollar = '$' )
#set( $symbol_escape = '\' )
package ${package}.handler;
```


![项目结构图](/img/in-post/post-2018-08/archetype.png)
## 本地使用

本地使用archetype只需`install`后使用相应archetype

```bash
mvn install
```


```bash
mvn archetype:generate                                  \
  -DarchetypeGroupId=<archetype-groupId>                \
  -DarchetypeArtifactId=<archetype-artifactId>          \
  -DarchetypeVersion=<archetype-version>                \
  -DgroupId=<my.groupid>                                \
  -DartifactId=<my-artifactId>
```

在eclipse或idea中则需在`install`后刷新目录`update-local-catalog`

```bash
mvn rchetype:update-local-catalog
```

## 私有仓库使用

推送至私有仓库后，只需在eclipse或idea中创建Maven项目，添加对应archetype即可

![这里写图片描述]((/img/in-post/post-2018-08/archetype-add.png)


## archetype命令

帮助命令
```bash
mvn archetype:help
```

爬取仓库以创建目录
```bash
mvn archetype:crawl
```

创建archetype
```bash
mvn archetype:create-from-project
```

根据archetype创建工程
```bash
mvn archetype:generate
```

根据当前的archetype工程创建jar
```bash
mvn archetype:jar
```

更新本地的maven目录
```bash
mvn archetype:update-local-catalog
```


---
参考资料：
1. [官方文档](http://maven.apache.org/guides/mini/guide-creating-archetypes.html)
2. [Maven自定义archetype生成项目骨架](https://blog.csdn.net/jeikerxiao/article/details/60324029)
