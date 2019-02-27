---
layout:     post
title:      "Piplin 持续部署 Docker 容器"
subtitle:   "自动化部署"
date:       2019-02-27 22:19:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - Piplin
    - Gitlab
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为使用Piplin 持续部署 Docker 容器**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

> docker镜像标签、git标签、构建产物版本 应严格保持一致

## 部署流程

1. 创建项目路径（构建服务器和部署服务器）
2. 配置项目详情及代码仓库
3. 部署服务器配置部署公钥
4. gitlab设置钩子通过标签事件触发构建计划

> 详见[piplin部署流程](http://piplin.com/docs/projects)

## 构建计划

![](/img/in-post/post-2019-02/structure.png)

> docker镜像部署不依赖构建产物，构建计划目的为在部署时使用构建产物的标签   
> 目前产物定义为tar包，目的为方便确认该构建计划的源代码

1. 打包
```bash
tar zcvf generic.tar.gz ./* --exclude=generic.tar.gz --exclude=.git
```

2. 导出tar包(勾选出品定义)
```bash
echo "导出tar包"
```

## 部署计划

![](/img/in-post/post-2019-02/deploy.png)

> 构建产物版本必须与docker镜像标签一致

1. 从镜像仓库拉取镜像
```bash
sudo docker pull dubbo/generic:{{ build_release }}
```

2. 删除旧容器
```bash
if sudo docker ps -a | grep generic-reference; then
    sudo docker rm -f generic-reference
fi
```

3. 获取服务器内网IP，并启动新容器
```bash
ip=$(/sbin/ip addr | grep inet | grep -v inet6 | grep eth0 | awk -F ' ' '{print \$2}' | awk -F '/' '{print \$1}')
sudo docker run -d --name generic-reference --net dubbo -e DUBBO_IP_TO_REGISTRY=$ip -e DUBBO_PORT_TO_REGISTRY=9799 -e JVM="-Xms128m -Xmx128m -Xmn48m" -p 9799:9799 -p 9800:9800 -p 22899:22899 --cpuset-cpus="1" --entrypoint="/setup/deploy.sh" dubbo/generic:{{ build_release }} --spring.profiles.active=dev
```

> 容器启动时，程序服务注册需要使用IP。获取服务器内网网卡IP可使用 带绝对路径及转义的命令 或 共享脚本文件

#### 带绝对路径及转义的命令

> piplin bash脚本中无法直接使用`ip`命令，否则会找不到`ip`命令，需使用带绝对路径的命令或配置环境变量(定时脚本亦可能出现此情况)

```bash
bash: line 5: ip: command not found
```

带绝对路径的命令
```bash
/sbin/ip addr
```

> `awk`命令使用变量需转义(猜测piplin处理bash脚本时涉及到字符串拼接的操作)

```bash
awk -F ' ' '{print \$2}'
```

#### 共享脚本文件

> 可通过共享文件，共享获取IP的脚本文件`getIp.sh`以获取内网网卡IP

getIp.sh
```bash
/sbin/ip addr | grep inet | grep -v inet6 | grep eth0 | awk -F ' ' '{print $2}' | awk -F '/' '{print $1}'
```



