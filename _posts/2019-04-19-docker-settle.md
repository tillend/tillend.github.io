---
layout:     post
title:      "Docker项目发布流程"
subtitle:   "微服务框架（三十一）"
date:       2019-04-19 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - Piplin
    - Gitlab
---

> piplin的详细部署流程见[piplin部署](https://tillend.github.io/2019/02/27/gitlab-piplin-docker/)

目前所有的算法微服务暴露两个端口，9700段为服务调用端口，22800段为服务QOS端口，
所有的服务均会通过zookeeper注册中心获取服务地址

## 发布流程

> docker镜像标签、git标签、构建产物版本 应严格保持一致
> 部署环境的环境名称需与`spring.profiles.active`的项目配置名称保持一致

 1. 项目设置docker镜像标签`image.tag`(如1.0.1)
 2. git tag的标签与docker镜像标签保持一致(如1.0.1)
 3. gitlab-CI标签自动触发，打包对应版本的docker镜像并上传至`docker`私有仓库
 4. 由标签事件触发`piplin Webhook`，自动构建对应版本产物
 5. 创建构建产物版本与docker镜像标签保持一致(如1.0.1)
 6. 机柜直接发布 或 选择灰度服务器灰度发布
 

> 容器启动时可 动态选择配置文件 或 配置线程策略、线程数等参数<br/>
(e.g. --spring.profiles.active=dev --dubbo.provider.timeout=1500)

## 灰度部署流程

可在piplin设置灰度服务器，并手动发布

> 若为REST接口，可依照每个服务至少需要两台服务器，按照目前的调用情况，维持一台权重为 100，另一台权重为 50，并实时根据调用情况做调整


## 管理中心

若希望人工管理服务提供者的上线和下线，此时需将注册中心标识为非动态管理模式。
```
dubbo.registry.dynamic = false
```
服务提供者初次注册时为禁用状态，需人工启用。断线时，将不会被自动删除，需人工禁用。

> 于`dubbo-admin`后台启动禁用操作

