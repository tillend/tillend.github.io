---
layout:     post
title:      "Docker容器部署（Dubbo、Zookeeper、Dubbo-admin）"
subtitle:   "微服务框架（七）"
date:       2018-08-09 22:36:00
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - Dubbo
    - Zookeeper
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Docker容器部署，包括Dubbo微服务、Zookeeper、Dubbo-admin的部署**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Docker容器启动参数

运行参数说明：

- `-d`: 后台运行容器，并返回容器ID
- `-i`: 以交互模式运行容器，通常与-t同时使用
- `-t`: 为容器重新分配一个伪输入终端，通常与 -i 同时使用
- `--name`: 为容器指定一个名称
- `-e`: 设置环境变量
- `--env-file`: 从指定文件读取环境变量
- `-p`: 端口映射，如果不做端口映射，容器外部无法访问容器内部
- `-v`: 文件挂载
- `--link`: 添加链接到容器，在default网络下，默认不会将容器名称解析到容器IP地址，必须要添加link选项才可以。而在自定义网络下，则不需要添加此选项

更多选项可以查看docker文档

## Docker容器部署

部署时绑定容器内外IP，可使用获取内网IP命令
```bash
ip=$(ip addr | grep inet | grep -v inet6 | grep eth0 | awk '{print $2}' |awk -F "/" '{print $1}')
```

#### 网络接口及数据卷

```bash
 # 创建网络接口
sudo docker network create zookeeper
sudo docker network create dubbo

 # volumn创建，用来持久化数据
sudo docker volume create background
sudo docker volume create dubbo
```

#### Zookeeper

```bash
 # zookeeper注册中心
sudo docker run -d \
	--name zookeeper \
	--net zookeeper \
	--net dubbo \
	-v background:/var/lib/zookeeper/data \
	-p 2181:2181 \
	-p 2888:2888 \
	-p 3888:3888 \
	--restart=always \
	jplock/zookeeper:3.4.11
```

#### dubbo-admin

```bash
 # dubbo-admin管理中心
docker run -d \
	--name dubbo-admin \
	--net zookeeper \
	-v background:/data \
	-p 9600:8080 \
	-e DUBBO_REGISTRY="zookeeper:\/\/ip:port" \
	-e DUBBO_ROOT_PASSWORD=root \
	-e DUBBO_GUEST_PASSWORD=guest \
	--restart=always \
	riveryang/dubbo-admin
```

#### dubbo-monitor

```bash
 # dubbo-monitor监控中心
docker run -d \
	--name dubbo-monitor \
	--net zookeeper \
	-v background:/dubbo-monitor/data \
	-e DUBBO_IP_TO_REGISTRY=ip \
	-e DUBBO_PORT_TO_REGISTRY=9700 \
	-p <ip>:9700:9700 \
	-p 9601:9601 \
	--restart=always \
	jeromefromcn/dubbo-monitor
```

#### Dubbo微服务
```bash
 # Dubbo微服务
docker run -d \
	--name <containerName> \
	--net dubbo \
	-e DUBBO_IP_TO_REGISTRY=<ip> \
	-e DUBBO_PORT_TO_REGISTRY=<port> \
	-p <ip>:<port>:<port> \
	-v dubbo:/log \
	--restart=always \
	<imageName> \
	--spring.profiles.active=<env>
```

## Docker常用命令

![这里写图片描述](https://img-blog.csdn.net/20180816143528351?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3doeV9zdGlsbF9jb25mdXNlZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

查看容器状态
```bash
docker ps | grep ${CONTAINER_ID}
```

查看容器日志
```bash
docker logs ${CONTAINER_ID}
```

交互式进入容器中
```bash
docker exec -i -t ${IMAGE_NAME} sh
```

镜像打包
```bash
docker commit -m "message" -a  "author" ${CONTAINER_ID}  ${NEW_IMAGE_NAME}
```

标签
```bash
docker tag ${IMAGE_NAME}  ${NEW_IMAGE_NAME}
```

推送至对应仓库
```bash
docker push ${REGISTRY_URL}/${IMAGE_NAME}
```

删除所有退出的容器
```bash
docker rm $(docker ps -a | grep Exit | awk '{ print $1 }')
```

删除所有名称为none的镜像
```bash
docker images | grep none | awk '{print $3} ' | xargs docker rmi
```


---
参考资料：
1. 《容器与容器云》
