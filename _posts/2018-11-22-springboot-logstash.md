---
layout:     post
title:      "Spring Boot Logstash日志采集"
subtitle:   "微服务框架（十三）"
date:       2018-11-22 22:19:40
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - Logstash
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文为Spring Boot中`Log4j2`对接`Logstash`，进行日志采集**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Log4j2配置
Log4j2.xml：（Logstash作为Appender）

```xml
<!-- Socket Apppender配置，通过TCP协议连接Logstash -->
<Socket name="LOGSTASH" host="${logstashHost}" port="${logstashPort}"
    protocol="TCP">
    <PatternLayout charset="UTF-8" pattern="${PATTERN}" />
</Socket>
```

##  Logstash

Logstah只支持log4j，使用log4j2时需要通过TCP插件调用

> Logstash启动时推荐配置`--config.reload.automatic`参数，自动重载配置

### Logstash input

为了接受到上述日志信息，需要配置`input`插件

> 若日志信息为`json`格式，则`codec`为`json`，文本为`plain`

```conf
input {
  tcp {
    host => "ip"
    port => "port"
    codec => "plain"
  }
}
```

### Logstash filter

`Logstash filter`插件在事件执行的中间过程进行相关的数据处理，通常根据事件的特征选择性地使用相应的过滤器。

> Filter插件列表详见[Logstash Filter](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)


#### grok

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


#### json

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

### Logstash 完整配置

Logstach.conf：
```conf
input {
  tcp {
    host => "ip"
    port => "port"
    codec => "plain"
  }
}

filter {

    grok {
        patterns_dir => ["./patterns"]
        match => {
           "message" => "\[%{LOGLEVEL:level}\] \[%{DATE_FORMAT:time}\]\[%{JAVACLASS:class}\]%{GREEDYDATA:message}"
        }
        overwrite => ["message"]
     }

     json {
        skip_on_invalid_json => true
        source => "message"
        remove_field => ["message"]
     }

    if [host] == "ip" {
        mutate {
            add_field => {
                "[@metadata][index_prefix]" => "service-dev"
            }
        }
    }

    if [host] == "ip" {
        mutate {
            add_field => {
                "[@metadata][index_prefix]" => "service-prod"
            }
        }
    }
}

output {
    stdout {
       codec => rubydebug
    }

    elasticsearch {
        hosts => ["devcdhsrv1:9200","devcdhsrv2:9200","devcdhsrv3:9200"]
        user => ""
        password => ""
        index => "%{[@metadata][index_prefix]}-%{+YYYY.MM.dd}"
        codec => json
    }
}
```

---
参考资料：    
1.[TCP input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-tcp.html)      
2.[logstash grok插件语法介绍](https://blog.csdn.net/qq_34021712/article/details/79746413)     
3.[Grok Debugger](http://grokdebug.herokuapp.com/)    
4.[Grok filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
