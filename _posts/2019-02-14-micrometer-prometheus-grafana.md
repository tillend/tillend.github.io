---
layout:     post
title:      "Spring Boot 可视化监控 Prometheus + Grafana"
subtitle:   "微服务框架（十九）"
date:       2019-02-14 22:02:44
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Prometheus
    - Grafana
    - Micrometer
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文为Spring Boot 通过监控门面 Micrometer 集成 Prometheus，再使用Grafana进行数据的实时展示**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

> 监控门面，概念同日志门面`slf4j`，均为基于**外观设计模式**所实现的规范，支持众多监控系统的应用程序`Metrics`外观

## Micrometer

`SpringBoot 2.x`上已引入第三方实现的`metrics Facade`，默认与`Micrometer`集成，而`Micrometer`具有`Prometheus`的`MeterRegistry`规范的实现。
`Prometheus`拉取及处理`SpringBoot`应用中的监控数据，最后通过`Grafana`提供的UI界面进行数据的实时展示。


> 更多关于`Micrometer`功能的信息，请参阅其[参考文档](https://micrometer.io/docs)，特别是[概念部分](https://micrometer.io/docs/concepts)

### metrics tag/label

**关于metrics是否支持tag/label，则代表其metrics是否能够有多维度的支持。** 像statsd不支持tag，如果要区分多host的同一个jvm指标，则通常是通过添加prefix来解决，不过这个给查询统计以及后续扩展带了诸多的不变。

支持tag的好处就是可以进行多维度的统计和查询，以同一微服务但是不同实例的jvm指标来说，可以通过`tag`来添加`host`标识，这样监控系统就可以灵活根据tag查询过滤来查看不同主机粒度的，甚至是不同数据中心的粒度。

## 埋点

### Maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <version>${springboot.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
    <version>${springboot.version}</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.1.2</version>
</dependency>
```

### application配置

```property
management.metrics.export.prometheus.enabled=true
management.metrics.export.prometheus.step=1m
management.metrics.export.prometheus.descriptions=true
management.web.server.auto-time-requests=true
management.endpoints.web.exposure.include=health,info,env,prometheus,metrics,httptrace,threaddump,heapdump
```

### web埋点

servlet容器`undertow`
```java
@SpringBootApplication
@EnableAspectJAutoProxy(proxyTargetClass = true)
@ComponentScan("com.linghit")
public class Starter {

    public static void main(String[] args) {

        new SpringApplicationBuilder(Starter.class)
                .web(WebApplicationType.SERVLET).run(args);

    }

}
```

## Prometheus

`Prometheus`是一个开源的监控系统，起源于`SoundCloud`。它由以下几个核心组件构成：

- **数据爬虫**：根据配置的时间定期的通过HTTP抓去`metrics`数据。
- **time-series 数据库**：存储所有的`metrics`数据。
- **简单的用户交互接口**：可视化、查询和监控所有的`metrics`。

![在这里插入图片描述](https://prometheus.io/assets/architecture.svg)

### Docker安装

```bash
docker run -d \
--name prometheus \
--net dubbo \
--hostname prom \
-p 9090:9090 \
-v /media/raid10/tmp/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
prom/prometheus \
--config.file=/etc/prometheus/prometheus.yml
```

> 增加`promtheus`拉取数据的项目，需在挂载的配置文件`prometheus.yml`中增加对应的`Endpoint`设置并重启服务

![](/img/in-post/post-2019-03/prometheus.png)

## Grafana

`Grafana`使你能够把来自不同数据源比如`Elasticsearch`, `Prometheus`, `Graphite`, `influxDB`等多样的数据以绚丽的图标展示出来。它也能基于你的`metrics`数据发出告警。当一个告警状态改变时，它能通知你通过email，slack或者其他途径。


### Docker安装

```bash
docker run -d \
--name grafana \
--net dubbo \
-p 3000:3000 \
-e "GF_SERVER_ROOT_URL=http://grafana.server.name" \
-e "GF_SECURITY_ADMIN_PASSWORD=" \
grafana/grafana
```

![](/img/in-post/post-2019-03/grafana.png)


---
参考资料：
1. [Spring Boot Actuator: Production-ready features](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html)
2. [Prometheus+Grafana实现SpringCloud服务监控](https://my.oschina.net/werngin/blog/1835239)
3. [Micrometer Prometheus](http://micrometer.io/docs/registry/prometheus)
4. [基于Docker+Prometheus+Grafana监控SpringBoot健康信息](https://yq.aliyun.com/articles/664698)
5. [Micrometer](http://micrometer.io/docs)
6. [Spring Boot Metrics](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics)
7. [聊聊springboot2的micrometer](https://segmentfault.com/a/1190000013804984)
8. [Prometheus 简介](https://blog.csdn.net/luanpeng825485697/article/details/82318204)
