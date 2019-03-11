---
layout:     post
title:      "Docker安装及私有仓库配置"
subtitle:   ""
date:       2019-03-04 22:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
---

## docker安装

```bash
yum install docker
```

## 增加私有仓库配置

#### 启动参数方式


在`/usr/lib/systemd/system/docker.service`增加以下配置

```bash
--insecure-registry=registry.example.com:5000
```


#### 配置文件方式

在`/etc/docker/daemon.json`中写入如下内容（如果文件不存在请新建该文件）
```json
{
  "registry-mirror": [
    "https://registry.docker-cn.com"
  ],
  "insecure-registries": [
    "registry.example.com:5000"
  ]
}
```


## 重启docker

守护进程重新加载配置，并重启docker进程
```bash
systemctl daemon-reload
systemctl restart docker
```


## 权限分配

编辑权限
```bash
visudo
```

分配用户docker权限(`,`分隔)
```bash
# 分配www用户docker权限
www localhost=NOPASSWD: /usr/bin/docker
www ALL=NOPASSWD: /usr/bin/docker
```bash

---
相关资料：
1.[私有仓库高级配置](https://docker_practice.gitee.io/repository/registry_auth.html)