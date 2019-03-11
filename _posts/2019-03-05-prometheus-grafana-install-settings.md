---
layout:     post
title:      "Prometheus + Grafana 安装、配置及使用"
subtitle:   "可视化监控"
date:       2019-03-05 22:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Docker
---

## Prometheus

#### prometheus.yml

先创建`prometheus.yml`文件，初始化配置

```yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['127.0.0.1:9090']
```

#### docker容器启动

采集数据持久化在磁盘中，docker启动时挂载的data目录需授权

> `prometheus` 运行过程中使用的用户是`nobody`，`data`目录权限需设置为`757`

- `prometheus.yml`为promethus的配置文件，修改后需重启生效
- `/prometheus/data`为promethus的磁盘持久化数据
- `--config.file=/etc/prometheus/prometheus.yml`为启动容器时加载的配置文件

```bash
docker run -d \
--name prometheus \
--net monitor \
--hostname prom \
-p 9090:9090 \
-v /media/raid10/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
-v /media/raid10/prometheus/data:/prometheus/data \
prom/prometheus \
--config.file=/etc/prometheus/prometheus.yml
```

> 增加`promtheus`拉取数据的项目，需在挂载的配置文件`prometheus.yml`中增加对应的`Endpoint`设置并重启服务

![](../../assets/prometheus.png)

#### 增加数据采集点

修改`prometheus.yml`配置文件
```yaml
scrape_configs:
  - job_name: 'controller'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:9700']
        labels: 
          application: dubbo
```

重启Prometheus
```bash
docker restart prometheus
```

#### 最佳做法

1. `--storage.tsdb.retention.time`定义旧数据的删除策略，默认为`15d`


## Grafana

#### docker容器启动

- 环境变量`GF_SERVER_ROOT_URL`为服务根URL
- 环境变量`GF_SECURITY_ADMIN_PASSWORD`为`admin`用户的初始密码

```bash
docker run -d \
--name grafana \
--net monitor \
-p 3000:3000 \
-e "GF_SERVER_ROOT_URL=http://grafana.linghit.io" \
-e "GF_SECURITY_ADMIN_PASSWORD=secret" \
grafana/grafana
```

![](../../assets/grafana.png)

### 监控报警

1. 添加监控通知渠道`Alerting->Notifications channels`
2. 为图表设置报警配置`Alert Config`
3. 为图表设置添加报警渠道及信息

> 常用的报警渠道有`Email`、`DingDing`、`Webhook`等


#### 模板变量

!> 报警中不能使用带模板变量的查询语句

[issue-查询语句的模板变量支持](https://github.com/grafana/grafana/issues/6557)


#### dashboard

常用模板
- [JVM (Micrometer)](https://grafana.com/dashboards/4701)
- [Spring Boot Statistics](https://grafana.com/dashboards/6756)
- [Kong dashboard](https://grafana.com/dashboards/7424)
- [Elasticsearch Nginx Logs](https://grafana.com/dashboards/2292)