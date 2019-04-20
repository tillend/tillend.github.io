---
layout:     post
title:      "Logstash 使用文档"
subtitle:   "微服务框架（二十八）"
date:       2019-04-13 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Logstash
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Logstash 使用文档**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Logstash

Logstash 是开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到存储库中。(ELK套件中存储库为ES)

![](https://images.contentstack.io/v3/assets/bltefdd0b53724fa2ce/blta9ac878ee2870b41/5bbde50f6c9763b95d07aae8/logstash-img1.png)

## Logstash 启动

`pipelines`目录下存放了`logstash`所有的数据处理管道配置文件，故会启动所有流水线

> `--config.reload.automatic`参数，自动重载配置

```bash
nohup ./bin/logstash --config.reload.automatic -f ./pipelines/ &
```

!> 目前为`nohup`实现后台启动，后续可考虑统一至`supervisor`管理

## Logstash input

为了接收到日志信息，需要配置`input`插件

### tcp

> 若日志信息为`json`格式，则`codec`为`json`，文本为`plain`

```conf
input {
  tcp {
    host => "172.16.7.7"
    port => "5045"
    codec => "json"
  }
}
```

### beat

```conf
input{
  beats {
    host => "172.16.7.7"
    port => "5044"
    add_field => ["log_channel", "nginx"]
  }
}
```

## Logstash filter


`Logstash filter`插件在事件执行的中间过程进行相关的数据处理，通常根据事件的特征选择性地使用相应的过滤器。

> Filter插件列表详见[Logstash Filter](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)


### grok

#### 简述

Grok可以将非结构化日志数据解析为结构化和可查询的内容。

此工具非常适用于syslog日志，apache和其他Web服务器日志，mysql日志，以及通常为人类而非计算机使用而编写的任何日志格式。

Logstash默认提供约120种模式。详见[grok原生正则](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)，亦可在`patterns_dir`参数使用自定义的正则模式串


#### 工作原理

Grok的工作原理是将文本模式组合成与日志匹配的内容。grok模式的语法是 %{SYNTAX:SEMANTIC}


#### 示例
```
55.3.244.1 GET /index.html 15824 0.043
```

上述HTTP日志可使用以下模式匹配
```
%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}
```

对应Logstash文件为
```conf
input {
  file {
    path => "/var/log/http.log"
  }
}
filter {
  grok {
    match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
  }
}
```

经过grok过滤后，日志事件解析为以下字段

- client: 55.3.244.1
- method: GET
- request: /index.html
- bytes: 15824
- duration: 0.043

### json

- `skip_on_invalid_json`:允许对无效`json`不进行过滤操作
- `source`:进行`json`转换的字段
- `remove_field`:删除字段(可用于剔除转换前的字段)

```conf
 json {
    skip_on_invalid_json => true
    source => "message"
    remove_field => ["message"]
 }
```

> 详细配置见[Json filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-json.html)

#### geoip

`geoip`过滤器根据Maxmind geolite2数据库的数据，转换相应IP地址地理位置的信息。

```
geoip {
    source => "client_ip"
}
```

> 详细配置见[Geoip filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-geoip.html)


## FAQ

1. 数据处理管道配置文件会在启动时自动merge，注意各pipeline中的操作会应用在所有流水线中


