---
layout:     post
title:      "Grafana 数据源及报警设置"
subtitle:   "微服务框架（二十七）"
date:       2019-04-12 21:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Grafana
    - Prometheus
    - ElasticSearch
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为使用grafana数据源及报警规则设置**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## 数据源

#### Prometheus

> 相关配置详见[官方文档](https://grafana.com/docs/features/datasources/prometheus/)

![在这里插入图片描述](/img/in-post/post-2019-04/prometheus-datasource.png)

#### ElasticSearch

es数据源根据索引名称设置，一一对应，故kong和nginx的日志使用两个索引

> 相关配置详见[官方文档](https://grafana.com/docs/features/datasources/elasticsearch/)

![在这里插入图片描述](/img/in-post/post-2019-04/es-datasource.png)

## 报警

当警报更改状态时，它会发出通知。每个警报规则都可以有多个通知。要向警报规则添加通知，首先需要添加和配置`notification`通道

> 配置详见[Alert Notifications](https://grafana.com/docs/alerting/notifications/)

#### 钉钉

选择类型为DingDing，填写钉钉机器人的webhook即可

![在这里插入图片描述](/img/in-post/post-2019-04/dingding-alerting.png)

#### Webhook

webhook通知是将有关HTTP状态更改的信息发送到自定义端点的简单方法。使用此通知，您可以将Grafana集成到您选择的系统中。

![在这里插入图片描述](/img/in-post/post-2019-04/webhook-alerting.png)

#### 邮件

修改`grafana.ini`配置文件，使用SMTP服务器发送邮件

> 相关配置可参考`Grafana dashboard 定时报表#SMTP配置`

```ini
# 邮件服务器配置，自行修改配置
[smtp]
enabled = true
host = smtp.exmail.qq.com:465
user = grafana@qq.com
password = <auth_token>
;cert_file =
;key_file =
;skip_verify = false
from_address = grafana@qq.com
from_name = Grafana
```

#### 短信

grafana webhook目前不支持携带请求头信息，若需接入短信通知，可以jenkins job等其他方式实现

> [Granfan短信和电话报警](https://www.jianshu.com/p/82b8e1e00e9a)

## 报警规则

grafana根据对应的面板中的Query进行报警规则设置

> 报警的Query不能使用模板变量

![在这里插入图片描述](/img/in-post/post-2019-04/grafana-alert.png)



 









