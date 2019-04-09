---
layout:     post
title:      "Grafana dashboard 定时报表"
subtitle:   "微服务框架（二十六）"
date:       2019-04-05 22:31:11
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Grafana
    - grafana-reporter
---


　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为使用grafana-reporter生成grafana dashboard报表，并使用定时任务邮件发送**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


# 定时报表

Grafana是一套开源的监控图表显示框架，支持监控报警功能，而dashboard定时报表功能可使用开源软件`grafana-reporter`或集成`Grafana API`来实现


## grafana-reporter

[grafana-reporter](https://github.com/IzakMarais/reporter)由Go语言编写，是根据grafana dashboard生成PDF报表的HTTP服务

> `grafana-reporter`生成报表的功能，需设置grafana允许匿名访问

```ini
[auth.anonymous]
# enable anonymous access
enabled = true

# specify organization name that should be used for unauthenticated users
org_name = Org
```

#### docker安装

> 需注意时区问题，默认生成的报表为UTC时区，推荐使用修改时区后镜像或挂载时区文件

```bash
sudo docker run -d \
--name grafana-reporter \
--net monitor \
--net host \
-p 8686:8686 \
IzakMarais/grafana-reporter
```

#### 前台生成

settings —> Links —> New —> Type -> link Url
```bash
link Url：http://ip:8686/api/v5/report/uid
```

#### 后台下载

`wget`命令下载pdf
```bash
wget -O test.pdf http://ip:8686/api/v5/report/uid?from=now-24h&to=now
```

## mail命令

Linux系统发送邮件的命令插件有很多，如`mail`、`mutt`、`sandmail`等

#### SMTP配置

> 阿里云禁止使用25端口，需使用SMTPS的465端口，并配置协议认证文件，详见参考资料

`/etc/mail.rc`文件末增加SMTP配置
```bash
set from=test@qq.com
set smtp=smtps://smtp.exmail.qq.com:465
set smtp-auth-user=test@qq.com
set smtp-auth-password=<auth_token>
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/root/.certs
```

示例
```bash
mail -v -s "title" example@example.com < content.txt
```

## 定时任务

#### contrab
```
59 23 * * * /media/raid10/grafana/report.sh
```


#### 定时报告脚本

> 增加对应的dashboard报告可依照格式添加，并在`mail`命令中使用`-a`参数添加附件

```bash
#/bin/bash
#auuthor:ricardo
#shell for creating grafana dashboard report
filepath=/media/raid10/grafana/report/
date=$(date +%Y-%m-%d)

# dashboard report name
filename_es_general=Elasticsearch-Nginx-generalapi.linghit.com-${date}.pdf
filename_spring=SpringBoot-Statistics-${date}.pdf
filename_es_api=Elasticsearch-Nginx-api.linghit.com-${date}.pdf

# download grafana dashboard report
wget -O ${filepath}${filename_es_general} http://172.16.7.5:8686/api/v5/report/8oPnVDCmz?from=now-24h&to=now&var-host=test.qq.com
wget -O ${filepath}${filename_spring} http://172.16.7.5:8686/api/v5/report/wAu8Swerd?from=now-24h&to=now
wget -O ${filepath}${filename_es_api} http://172.16.7.5:8686/api/v5/report/8oPnVDCmz?from=now-24h&to=now&var-host=test.qq.com

sleep 30s

# send email
mail -v \
-a ${filepath}${filename_es_general} -a ${filepath}${filename_spring} -a ${filepath}${filename_es_api} \
-s "Grafana监控日报"-`date +%Y-%m-%d` \
-c "test@qq.com" test@qq.com < /media/raid10/grafana/content.txt
```


---
参考资料：    
1. [Dashboard API](http://docs.grafana.org/http_api/dashboard/)
2. [六种使用Linux命令发送带附件的邮件](https://www.iteblog.com/archives/2027.html#telnet)
3. [Centos7 配置 sendmail、postfix 端口号25、465](https://my.oschina.net/sunboy2050/blog/2870097)



