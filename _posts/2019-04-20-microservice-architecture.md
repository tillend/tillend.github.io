---
layout:     post
title:      "微服务系统架构"
subtitle:   "微服务框架（三十二）"
date:       2019-04-20 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - Piplin
    - Gitlab
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为微服务系统架构**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## 系统架构
![](/img/in-post/post-2019-04/architecture.png)

- 开发语言`Java 8`
- 框架使用`Spring boot`
- 服务治理框架`Dubbo`
- 容器部署`Docker`
- 持续集成`Gitlab CI`
- 持续部署`Piplin`
- 注册中心`Zookeeper`
- 服务管理`Dubbo-admin`
- 日志采集及分析`ELK`
- 链路追踪`Zipkin`/`Tracing Analysis`(阿里云)
- 可视化监控`Prometheus + Grafana`
- API网关`Kong`


## 系统流程
![](/img/in-post/post-2019-04/system.png)

### CI流水线

1. 项目代码提交后，生成镜像在测试服务器进行功能测试
2. 功能测试完成，`git tag`触发gitlab-CI镜像部署流程(镜像推送至Docker私有仓库)
3. 发布服务器运行该镜像，进行最后的功能测试
4. 确认功能无误后，`piplin`手动发布

### Dubbo架构

![](/img/in-post/post-2019-04/dubbo-architecture.png)
