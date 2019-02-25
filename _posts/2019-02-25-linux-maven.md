---
layout:     post
title:      "linux maven 安装"
subtitle:   ""
date:       2019-02-25 21:19:58
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Linux
    - Maven
---

## linux maven 安装

#### 下载

```bash
wget https://mirrors.cnnic.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
```

#### 解压

```bash
tar -zxvf apache-maven-3.3.9-bin.tar.gz
```

#### 移动

```bash
mkdir /usr/local/maven
mv apache-maven-3.3.9 /usr/local
```

#### 环境变量

```bash
vim /etc/profile.d/maven
```

配置Maven HOME及PATH

```bash
MAVEN_HOME=/usr/local/apache-maven-3.3.9
export MAVEN_HOME
PATH=$MAVEN_HOME/bin:$PATH
```

#### 验证

```bash
mvn -v
```