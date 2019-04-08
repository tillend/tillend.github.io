---
layout:     post
title:      "gitlab-runner docker安装"
subtitle:   ""
date:       2019-04-08 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
    - gitlab-runner
---

## 镜像拉取

```bash
sudo docker pull gitlab/gitlab-runner:v1.10.7
```

## 容器启动

添加 gitlab-runner container
```bash
sudo docker run -d
--net host \
 --name gitlab-runner \
--restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:v1.10.7
```

## Runner注册

```bash
sudo docker exec -it gitlab-runner gitlab-ci-multi-runner register
```


注册Runner步骤
```bash
Please enter the gitlab-ci coordinator URL:
    http://git.linghit.com:666/ci
Please enter the gitlab-ci token for this runner:
    38sUHxxStGXytDTyfxDg
Please enter the gitlab-ci description for this runner:
    dubbo-runner(IP)
Please enter the gitlab-ci tags for this runner (comma separated):
    dubbo
Whether to run untagged builds [true/false]:
    true
Please enter the executor: docker, parallels, shell, kubernetes, docker-ssh, ssh, virtualbox, docker+machine, docker-ssh+machine:
    docker
Please enter the default Docker image (e.g. ruby:2.1):
    maven:3.3.9-jdk-8
```

## Runner启用

gitlab -> project -> Runners

![](/img/in-post/post-2019-04/gitlab-runner.png)


## Maven依赖

docker image每次构建都是在独立的container里， maven的 .m2文件并不会被多次构建公用，这里我们可以通过修改gitlab-runner的配置，将maven .m2目录加到volumes中，并增加镜像拉取规则（默认是从远程拉取镜像，这里修改为优先获取本地镜像，不存在时才去远程拉取镜像）。

> `config.toml`为runner配置文件，路径见容器启动的挂载目录

```toml
[runners.docker]
    tls_verify = false
    image = "maven:3.3.9-jdk-8"
    privileged = false
    disable_cache = false
    volumes = ["/cache", "/media/raid10/maven/.m2:/root/.m2"]
    pull_policy = "if-not-present"
```



