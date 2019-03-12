---
layout:     post
title:      "Prometheus 监控埋点"
subtitle:   "微服务框架（二十四）"
date:       2019-03-05 22:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Prometheus
    - Micrometer
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Prometheus 监控埋点**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Prometheus + Grafana

SpringBoot2.x上已引入第三方实现的`metrics Facade`，默认与`micrometer`集成，而`micrometer`具有`Prometheus`的`MeterRegistry`规范的实现。<br/>
`Prometheus`通过`Micrometer`拉取及处理`SpringBoot`应用中的监控数据，最后通过`Grafana`提供的UI界面进行数据的实时展示。

![](https://prometheus.io/assets/architecture.svg)

## 快速集成

### Parent POM

版本`1.1.2+`后支持
```xml
<parent>
    <groupId>com.linghit</groupId>
    <artifactId>dubbo.common.pom</artifactId>
    <version>1.1.2</version>
</parent>
```

### Maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
    <version>${springboot.version}</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>${micrometer.version}</version>
</dependency>
```

### application配置

> 需开启`server.port`及`management.port`；若未定义`management.port`则会默认使用`server.port`

```properties
# dubbo endpoints
management.endpoint.dubbo.enabled = true
management.endpoint.dubbo-configs.enabled = true
management.endpoint.dubbo-properties.enabled = true
management.health.dubbo.status.defaults = memory
management.health.dubbo.status.extras = load,threadpool

# metrics
management.metrics.export.prometheus.enabled = true
management.metrics.export.prometheus.step = 1m
management.metrics.export.prometheus.descriptions = true
management.web.server.auto-time-requests = true
management.endpoints.web.exposure.include = env,health,info,metrics,threaddump,prometheus,dubbo,dubbo-configs,dubbo-properties
```

## Web埋点

Spring boot支持Web及JMX(Java管理拓展)的健康指标监控，JMX的方式不适合于可视化监控。现有使用Prometheus拉取监控数据，并以Grafana进行数据可视化展示的方案更优<br/>

此处`Spring Boot Actuator`通过`REST API`提供对应用系统的自省和监控的集成功能

### 依赖

[通用Dubbo POM](http://docs.mmclick.com/algorithm/#/src/common/dubbo-common)已集成`spring boot`及`dubbo`的`actuator`起步依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
    <version>${springboot.version}</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>${micrometer.version}</version>
</dependency>
```

### application配置

```properties
management.metrics.export.prometheus.enabled=true
management.metrics.export.prometheus.step=1m
management.metrics.export.prometheus.descriptions=true
management.web.server.auto-time-requests=true
management.endpoints.web.exposure.include=health,info,env,prometheus,metrics,httptrace,threaddump,heapdump
```

### web容器

undertow的servlet容器
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

### Log4j2 Metrics

`Spring Boot 2.1.0.RELEASE`才开始支持`Log4j2 metrics`的自动装配，低版本只支持`Logback metrics`，欲使用需升级版本或自实现

> 详见[Spring Boot 2.1.0](https://spring.io/blog/2018/10/30/spring-boot-2-1-0#metrics)

## Dubbo埋点

`dubbo-spring-boot-actuator`提供生产就绪(`Production-Ready`)特性 (比如健康检查,OPS端点,及外部化配置)

> 详见[Dubbo Spring Boot Production-Ready](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/dubbo-spring-boot-actuator/README_CN.md)

### 依赖

[通用Dubbo POM](http://docs.mmclick.com/algorithm/#/src/common/dubbo-common)已集成`dubbo-actuator`起步依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-actuator</artifactId>
    <version>${dubbo.springboot.version}</version>
</dependency>
```

### application配置

```properties
# dubbo endpoints
management.endpoint.dubbo.enabled = true
management.endpoint.dubbo-configs.enabled = true
management.endpoint.dubbo-properties.enabled = true
management.health.dubbo.status.defaults = memory
management.health.dubbo.status.extras = load,threadpool
```

> 注：需在`management.endpoints.web.exposure.include`添加对应的`endpoints`，如：`dubbo,dubbo-configs,dubbo-properties`

## Docker埋点

要将Docker守护程序配置为Prometheus目标，您需要指定 `metrics-address`。最好的方法是通过`daemon.json`，默认情况下位于以下位置之一。(如果该文件不存在，请创建它)

- Linux：/etc/docker/daemon.json
- Windows Server：C:\ProgramData\docker\config\daemon.json
- Docker Desktop for Mac / Docker Desktop for Windows：单击工具栏中的Docker->Preferences->Daemon->高级

如果文件当前为空，请粘贴以下内容：

```json
{
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
}
```
