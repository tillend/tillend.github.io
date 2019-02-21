﻿---
layout:     post
title:      "Dubbo Spring Boot 生产就绪特性"
subtitle:   "dubbo-spring-boot-actuator"
date:       2019-02-21 21:27:58
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - Dubbo
    - 译
---

## Dubbo埋点

`dubbo-spring-boot-actuator`提供生产就绪(`Production-Ready`)特性 (比如健康检查,OPS端点,及外部化配置)

> 详见[Dubbo Spring Boot Production-Ready](https://github.com/apache/incubator-dubbo-spring-boot-project/tree/master/dubbo-spring-boot-actuator)

#### Maven依赖

```xml
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-actuator</artifactId>
    <version>${dubbo.springboot.version}</version>
</dependency>
```

> 最新版本为贡献给apache基金会后的依赖

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-actuator</artifactId>
    <version>2.7.0</version>
</dependency>
```

#### application配置

```properties
# dubbo endpoints
management.endpoint.dubbo.enabled = true
management.endpoint.dubbo-configs.enabled = true
management.endpoint.dubbo-properties.enabled = true
management.health.dubbo.status.defaults = memory
management.health.dubbo.status.extras = load,threadpool
```

> 注：需在`Web`或`JMX`暴露对应的端点，即添加对应的`endpoints`，如：

```properties
management.endpoints.web.exposure.include = dubbo,dubbo-configs,dubbo-properties
```


## 健康检查

`dubbo-spring-boot-actuator`支持标准的Spring Boot健康指示器作为生产就绪特性，它将聚合到Spring Boot的`Health`中，并在运行MVC (Spring Web MVC)和JMX (Java管理扩展)的`HealthEndpoint`上暴露

#### Web Health端点

您可以通过Web Client访问[health endpoints](http://localhost:8080/actuator/health)，亦可独立通过`management.server.port`为访问端点设置端口号。获得JSON格式的响应，如下所示

```json
{
  "status": "UP",
  "dubbo": {
    "status": "UP",
    "memory": {
      "source": "management.health.dubbo.status.defaults",
      "status": {
        "level": "OK",
        "message": "max:3641M,total:383M,used:92M,free:291M",
        "description": null
      }
    },
    "load": {
      "source": "management.health.dubbo.status.extras",
      "status": {
        "level": "OK",
        "message": "load:1.73583984375,cpu:8",
        "description": null
      }
    },
    "threadpool": {
      "source": "management.health.dubbo.status.extras",
      "status": {
        "level": "OK",
        "message": "Pool status:OK, max:200, core:200, largest:0, active:0, task:0, service port: 12345",
        "description": null
      }
    },
    "server": {
      "source": "dubbo@ProtocolConfig.getStatus()",
      "status": {
        "level": "OK",
        "message": "/192.168.1.103:12345(clients:0)",
        "description": null
      }
    }
  }
  // ignore others
}
```

在上面的例子中，`memory`, `load`, `threadpool`,`server`是Dubbo的内置`StatusCheckers`。 Dubbo允许应用程序扩展`StatusChecker`的SPI。

在缺省配置中，`memory` 及 `load`将添加到Dubbo的`HealthIndicator`中，它可以被外部化配置[StatusChecker的默认值](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/dubbo-spring-boot-actuator/README.md#statuschecker-defaults)覆盖。

#### JMX Health端点

`Health`是一个JMX (Java管理扩展)端点，加载于`org.springframework.boot:type=Endpoint,name=Health`，它可以由JMX代理管理。如JDK自带的工具`jconsole`等

![](https://github.com/apache/incubator-dubbo-spring-boot-project/raw/master/dubbo-spring-boot-actuator/JMX_HealthEndpoint.png)

#### 内置StatusCheckers

`META-INF/dubbo/internal/com.alibaba.dubbo.common.status.StatusChecker`声明了内置StatusCheckers

```
registry=org.apache.dubbo.registry.status.RegistryStatusChecker
spring=org.apache.dubbo.config.spring.status.SpringStatusChecker
datasource=org.apache.dubbo.config.spring.status.DataSourceStatusChecker
memory=org.apache.dubbo.common.status.support.MemoryStatusChecker
load=org.apache.dubbo.common.status.support.LoadStatusChecker
server=org.apache.dubbo.rpc.protocol.dubbo.status.ServerStatusChecker
threadpool=org.apache.dubbo.rpc.protocol.dubbo.status.ThreadPoolStatusChecker
```

> 可以使用外部化配置中的`management.health.dubbo.status.*`的有效值，来作为`StatusChecker`名称的属性键

## 端点(Endpoints)

Actuator端点`dubbo`支持Actuator端点：

| ID       | Enabled          | HTTP URI            | HTTP Method | Description                         | Content Type       |
| ------------------- | ----------- | ----------------------------------- | ------------------ | ------------------ | ------------------ |
| `dubbo`    | `true`      | `/actuator/dubbo`            | `GET`       | Exposes Dubbo's meta data           | `application/json` |
| `dubboproperties` | `true` | `/actuator/dubbo/properties` | `GET`       | Exposes all Dubbo's Properties      | `application/json` |
| `dubboservices` | `false`     | `/dubbo/services`            | `GET`       | Exposes all Dubbo's `ServiceBean`   | `application/json` |
| `dubboreferences` | `false` | `/actuator/dubbo/references` | `GET`       | Exposes all Dubbo's `ReferenceBean` | `application/json` |
| `dubboconfigs` | `true` | `/actuator/dubbo/configs`    | `GET`       | Exposes all Dubbo's `*Config`       | `application/json` |
| `dubboshutdown` | `false` | `/actuator/dubbo/shutdown`   | `POST`      | Shutdown Dubbo services             | `application/json` |

> 端点暴露的监控数据详见[web-endpoints](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/dubbo-spring-boot-actuator/README.md#web-endpoints)

## 外部化配置

#### 默认的StatusChecker

   management.health.dubbo.status.defaults是用于设置StatusCheckers名称的属性名称，其值允许为多个值，例如
```properties
management.health.dubbo.status.defaults = registry,memory,load
```


默认值为
```properties
management.health.dubbo.status.defaults = memory,load
```

#### StatusChecker额外功能

`management.health.dubbo.status.extras`用于覆盖[StatusChecker的默认值](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/dubbo-spring-boot-actuator/README.md#statuschecker-defaults)，多个值由逗号分隔：

```properties
management.health.dubbo.status.extras = load,threadpool
```

#### 健康检查启用


`management.health.dubbo.enabled`是一个启用配置来打开或关闭运行状况检查功能，其默认值为`true`。

如果您要禁用运行状况检查，则应将management.health.dubbo.enabled应用于false：

```properties
management.health.dubbo.enabled = false
```

#### 端点启用

Dubbo Spring Boot提供执行器端点，但其中一些是禁用的。如果您要启用它们，请将以下属性添加到外部配置中：

```properties
# Enables Dubbo All Endpoints
management.endpoint.dubbo.enabled = true
management.endpoint.dubbo-shutdown.enabled = true
management.endpoint.dubbo-configs.enabled = true
management.endpoint.dubbo-services.enabled = true
management.endpoint.dubbo-references.enabled = true
management.endpoint.dubbo-properties.enabled = true
```


---
参考资料：
1.[Dubbo Spring Boot Production-Ready](https://github.com/apache/incubator-dubbo-spring-boot-project/blob/master/dubbo-spring-boot-actuator/README_CN.md)
2.[Securing HTTP Endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-endpoints-security)