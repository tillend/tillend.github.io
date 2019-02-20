---
layout:     post
title:      "Docker镜像化Dubbo微服务"
subtitle:   "微服务框架（五）"
date:       2018-08-03 17:52:07
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
	- Docker
    - Dubbo
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为将Dubbo微服务实现Docker镜像化**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Docker镜像化

> 需要打包为docker镜像的项目，artifactId请设置为**无特殊字符的小写字母字串**，否则docker-maven-plugin会出错

### 配置docker-maven-plugin插件

`maven` `pom.xml`加入`com.spotify:docker-maven-plugin`插件配置(配置对应properties)

> [通用Dubbo POM]()中已集成，作为parent引入即可，详见文档

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

**properties配置示例**

| 参数  | 描述    | 可否为空      |
|---|---|---|
| registry.url |Docker私有仓库url |可为空，此时镜像名为`dubbo/${project.artifactId}` |
| image.tag |镜像标签 | 可为空，此时只生成`latest`镜像标签   |

```xml
<properties>
	<registry.url>0.0.0.1/</registry.url>
	<image.tag>0.0.1</image.tag>
</properties>
```

### 编写Dockerfile

Dockerfile文件路径需在docker-maven-plugin插件配置

> 注：文件存放路径需对应上述插件配置中``<dockerDirectory>``

```Dockerfile
 # Dockerfile
 # 基础镜像
FROM openjdk:8-jdk-alpine

 # 设置工作目录
WORKDIR /

 # 将jar文件拷贝到镜像中。注：docker-maven-plugin 会将jar文件拷贝到镜像构建目录中
ADD *.jar app.jar
ADD lib lib/

 # 运行时设置JAVA环境参数
ENV JAVA_OPTS="-Duser.timezone=Asia/Shanghai"

 # 端口暴露（推荐不暴露端口，在运行容器时才对端口进行绑定）
EXPOSE 9702

 #ENTRYPOINT不支持环境变量展开
ENTRYPOINT ["java", "-Duser.timezone=Asia/Shanghai", "-jar", "/app.jar"]
```

### 编写CI文件

编写CI文件.gitlab-ci.yaml，将docker镜像打包设置为手动触发

> 注：gitlab项目需配置环境变量``DOCKER_HOST=tcp://<ip>:<port>``

```yaml
deploy_docker:
  stage: deploy
  script:
    - echo "Deploy as a docker container"
    - 'mvn clean package docker:build -DskipTests'
  when: manual
  only:
  - master
```

### 查看并运行镜像

**查看镜像**

```bash
docker images
```

**运行镜像**

```bash
docker run -d \
	--name <containerName> \
	--net dubbo \
	-e DUBBO_IP_TO_REGISTRY=<ip> \
	-e DUBBO_PORT_TO_REGISTRY=<port> \
	-p <ip>:<port>:<port> \
	-v dubbo:/log \
	<imageName> \
	--spring.profiles.active=<env>
```

---
参考资料：
 1. [Maven Docker插件](https://github.com/spotify/docker-maven-plugin)
