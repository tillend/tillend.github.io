---
layout:     post
title:      "Logstash Kong 日志上报"
subtitle:   "微服务框架（三十）"
date:       2019-04-15 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Logstash
    - Kong
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Logstash Kong 日志上报**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Kong 日志上报

Kong 日志上报采用Kong [tcp-log](https://docs.konghq.com/hub/kong-inc/tcp-log/)插件

## tcp-log插件

可通过API或Konga后台安装Kong `tcp-log`插件

API
```bash
curl -X POST http://localhost:8003/plugins \
    --data "name=tcp-log" \
    --data "config.host=172.16.7.7" \
    --data "config.port=5045"
```

Konga
![在这里插入图片描述](/img/in-post/post-2019-04/tcp-log.png)


### 日志字段

- `request` 包含有关客户端发送的请求的属性
- `response` 包含有关发送到客户端的响应的属性
- `tries` 包含负载均衡器为此请求进行的重试（成功和失败）列表
- `route` 包含有关所请求的特定路由的Kong属性
- `service` 包含与所请求的路线相关联的服务的Kong属性
- `authenticated_entity` 包含有关经过身份验证的凭据的Kong属性（如果已启用身份验证插件）
- `workspaces` 包含与请求的路由关联的工作空间的Kong属性。仅限Kong Enterprise版本> = 0.34。
- `consumer` 包含经过身份验证的使用者（如果已启用身份验证插件）
- `latencies` 包含一些有关延迟的数据：
    - `proxy` 是最终服务处理请求所花费的时间
    - `kong` 是运行所有插件所需的内部Kong延迟
    - `request` 是从客户端读取的第一个字节之间以及最后一个字节发送到客户端之间经过的时间。用于检测慢速客户端。
- `client_ip` 包含原始客户端IP地址
- `started_at` 包含开始处理请求的UTC时间戳。


## Logstash 日志处理


> 相关插件详见[微服务框架（二十八）Logstash 使用文档](https://blog.csdn.net/why_still_confused/article/details/89243570)


kong.conf
```conf
input {
   tcp {
     host => "172.16.7.7"
     port => "5045"
     codec => "json"
     add_field => ["log_channel", "kong"]
  }
}

filter {

  if [log_channel] == "kong" {
    mutate {
       rename =>{"[host]" => "[host][name]"}
    }

    if [host][name] == "172.16.7.243" {
        mutate {
            add_field => {
                "[@metadata][index_prefix]" => "kong-all-dev"
            }
        }
    }

    geoip {
      source => "[request][headers][remoteip]"
      remove_field => ["tags", "[geoip][latitude]", "[geoip][longitude]", "[geoip][continent_code]", "[geoip][country_code3]", "[geoip][country_code2]"]
    }

  }
}

output {
#  stdout { codec => rubydebug }

  elasticsearch {
      hosts => ["localhost:9200"]
      index => "%{[@metadata][index_prefix]}-%{+YYYY.MM.dd}"
  }
}
```


