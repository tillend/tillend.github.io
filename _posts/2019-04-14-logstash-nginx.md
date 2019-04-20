---
layout:     post
title:      "Logstash Nginx 日志上报"
subtitle:   "微服务框架（二十九）"
date:       2019-04-12 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Logstash
    - Nginx
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Logstash Nginx 日志上报**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Nginx 日志上报

Nginx 日志上报采用`filebeat`，需在服务器安装相关版本的`filebeat`并配置日志路径即可

> [filebeat安装文档](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-getting-started.html)

## Nginx 日志格式

Nginx access log日志格式
```
log_format  access  '$proxy_add_x_forwarded_for - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" [ $request_body ] - "$request_time" - "$host"';
```

## Logstash 日志处理

> 相关插件详见[微服务框架（二十八）Logstash 使用文档](https://blog.csdn.net/why_still_confused/article/details/89243570)

Nginx 日志索引为`nginx-*`，该索引将自动适配es中自定义的索引模板

```conf
input{
  beats {
    host => "172.16.7.7"
    port => "5044"
    add_field => ["log_channel", "nginx"]
  }
}

filter {
  if [log_channel] == "nginx" {
    grok {
      match => {
         "message" => "%{DATA:forwarded_for} - %{DATA:remote_user} \[%{DATA:request_time}\] \"%{WORD:method} %{URIPATHPARAM:uri} %{DATA:http_version}\" %{INT:http_status} %{INT:body_byte} \"%{PROG:http_referer}\" \"%{DATA:user_agent}\" \[ %{DATA:request_body}\ ] - \"%{BASE10NUM:spend_time}\" - \"%{DATA:request_host}\""
      }
    }

    mutate {
      convert => {
        "http_status" => "integer"
        "spend_time" => "float"
        "body_byte" => "integer"
      }
    }

    mutate {
      split => ["forwarded_for",","]
      add_field => ["request_ip", "%{[forwarded_for][0]}"]
      remove_field => ["tags", "forwarded_for"]
    }

    geoip {
      source => "request_ip"
      remove_field => ["tags", "[geoip][latitude]", "[geoip][longitude]", "[geoip][continent_code]", "[geoip][country_code3]", "[geoip][country_code2]"]
    }

    mutate {
        add_field => {
            "[@metadata][index_prefix]" => "nginx"
        }
    }

  }
}

output {
  elasticsearch {
      hosts => ["localhost:9200"]
      index => "%{[@metadata][index_prefix]}-%{+YYYY.MM.dd}"
  }
}
```
