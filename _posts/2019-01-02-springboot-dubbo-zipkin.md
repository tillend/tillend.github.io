---
layout:     post
title:      "Spring Boot及Dubbo zipkin 链路追踪组件埋点"
subtitle:   "微服务框架（十六）"
date:       2019-01-02 21:12:23
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - Dubbo
    - Zipkin
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文第一部分为调用链、OpenTracing、Zipkin和Jeager的简述；第二部分为Spring Boot及Dubbo zipkin 链路追踪组件埋点**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。



## 调用链

在广义上，一个调用链代表一个事务或者流程在（分布式）系统中的执行过程。在 `OpenTracing` 标准中，调用链是多个 `Span` 组成的一个**有向无环图**（`Directed Acyclic Graph`，简称 `DAG`），每一个 `Span`代表调用链中被命名并计时的连续性执行片段。

> 下图是一个分布式调用的例子：客户端发起请求，请求首先到达负载均衡器，接着经过认证服务、计费服务，然后请求资源，最后返回结果。

![](http://aliware-images.oss-cn-hangzhou.aliyuncs.com/arms/xtrace_dg_distributed_call.png)


数据被采集存储后，分布式追踪系统一般会选择使用包含时间轴的时序图来呈现这个调用链。

![](http://aliware-images.oss-cn-hangzhou.aliyuncs.com/arms/xtrace_dg_trace_graph.png)

### OpenTracing

为了解决不同的分布式追踪系统 API 不兼容的问题，诞生了 [OpenTracing](http://opentracing.io/?spm=a2c4e.11153940.blogcont514488.21.11b711f43Rqqkk) 规范。
OpenTracing 是一个轻量级的标准化层，它位于**应用程序/类库**和**追踪或日志分析程序**之间。

```
+-------------+  +---------+  +----------+  +------------+
| Application |  | Library |  |   OSS    |  |  RPC/IPC   |
|    Code     |  |  Code   |  | Services |  | Frameworks |
+-------------+  +---------+  +----------+  +------------+
       |              |             |             |
       |              |             |             |
       v              v             v             v
  +------------------------------------------------------+
  |                     OpenTracing                      |
  +------------------------------------------------------+
     |                |                |               |
     |                |                |               |
     v                v                v               v
+-----------+  +-------------+  +-------------+  +-----------+
|  Tracing  |  |   Logging   |  |   Metrics   |  |  Tracing  |
| System A  |  | Framework B |  | Framework C |  | System D  |
+-----------+  +-------------+  +-------------+  +-----------+
```

#### OpenTracing 的优势

- OpenTracing 已进入 CNCF，正在为全球的分布式追踪，提供统一的概念和数据标准。
- OpenTracing 通过提供平台无关、厂商无关的 API，使得开发人员能够方便的添加（或更换）追踪系统的实现。


### Zipkin

Zipkin是一种分布式跟踪系统。它有助于收集解决微服务架构中的延迟问题所需的时序数据。它管理这些数据的收集和查找。Zipkin的设计基于 [Google Dapper](http://research.google.com/pubs/pub36356.html)论文。

![](https://logz.io/wp-content/uploads/2018/07/Zipkin-system-architecture.png)

### Jeager

Jaeger受[Dapper](https://research.google.com/pubs/pub36356.html)和[OpenZipkin](http://zipkin.io/)的启发，是Uber Technologies公开发布的分布式跟踪系统。它用于监视和排除基于微服务的分布式系统，包括：

- 分布式上下文传播
- 分布式事务监控
- 根本原因分析
- 服务依赖分析
- 性能/延迟优化

Uber发表了一篇博客文章[Evolving Distributed Tracing at Uber](https://eng.uber.com/distributed-tracing/)，在那里他们解释了Jaeger所做的架构性选择的历史和原因

![](https://logz.io/wp-content/uploads/2018/07/Jaeger-system-architecture.png)

## 依赖

### brave依赖
```xml
<properties>
     <brave.version>5.4.2</brave.version>
     <zipkin-reporter.version>2.7.9</zipkin-reporter.version>
 </properties>

 <dependencyManagement>
     <dependencies>
         <!-- 引入 zipkin brave 的 BOM 文件 -->
         <dependency>
             <groupId>io.zipkin.brave</groupId>
             <artifactId>brave-bom</artifactId>
             <version>${brave.version}</version>
             <type>pom</type>
             <scope>import</scope>
         </dependency>

         <!-- 引入 zipkin repoter 的 BOM 文件 -->
         <dependency>
             <groupId>io.zipkin.reporter2</groupId>
             <artifactId>zipkin-reporter-bom</artifactId>
             <version>${zipkin-reporter.version}</version>
             <type>pom</type>
             <scope>import</scope>
         </dependency>
     </dependencies>
 </dependencyManagement>

 <dependencies>
     <!-- 1. brave 的 spring bean 支持 -->
     <dependency>
         <groupId>io.zipkin.brave</groupId>
         <artifactId>brave-spring-beans</artifactId>
         <version>${brave.version}</version>
     </dependency>

     <!-- 2. 在 SLF4J 的 MDC (Mapped Diagnostic Context) 中支持 traceId 和 spanId -->
     <dependency>
         <groupId>io.zipkin.brave</groupId>
         <artifactId>brave-context-slf4j</artifactId>
         <version>${brave.version}</version>
     </dependency>

     <!-- 3. 使用 okhttp3 作为 reporter -->
     <dependency>
         <groupId>io.zipkin.reporter2</groupId>
         <artifactId>zipkin-sender-okhttp3</artifactId>
         <version>${zipkin-reporter.version}</version>
     </dependency>
 </dependencies>
```

### Dubbo & brave

```xml
<!-- 1. brave 对 dubbo 的集成 -->
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-instrumentation-dubbo-rpc</artifactId>
    <version>${brave.version}</version>
</dependency>
```

### Spring web  & brave
```xml
<!-- brave 对 spring web 的集成 -->
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-instrumentation-spring-web</artifactId>
    <version>${brave.version}</version>
</dependency>
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-instrumentation-spring-webmvc</artifactId>
    <version>${brave.version}</version>
</dependency>
```


## 使用方法

### zipkin

```properties
zipkin.url = http://<domain>/api/v2/spans?userName=<username>&userKey=<userkey>
```

### Spring bean 注入

- `@ComponentScan`
- 配置 `autoconfigure`（`spring.factories`)。
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=brave.webmvc.TracingConfiguration
```

### 服务端
```properties
dubbo.provider.filter=tracing
```

### 客户端
```properties
dubbo.consumer.filter=tracing
```

> `tracing`为链路追踪filter组件启用


## 插件埋点

### RPC插件埋点

通过`brave.dubbo.rpc`的`TracingFilter`实现`Dubbo Filter`接口，注入对应的`spring bean`

```java
@Configuration
public class TracingConfiguration {

    @Value("${zipkin.url}")
    private String url;

    /**
     * Configuration for how to send spans to Zipkin
     */
    @Bean
    Sender sender() {
        return OkHttpSender.create(url);
    }

    /**
     * Configuration for how to buffer spans into messages for Zipkin
     */
    @Bean
    AsyncReporter<Span> spanReporter() {
        return AsyncReporter.create(sender());
    }

    /**
     * Controls aspects of tracing such as the name that shows up in the UI
     */
    @Bean
    Tracing tracing(@Value("${spring.application.name}") String serviceName) {
        return Tracing.newBuilder()
                .localServiceName(serviceName)
                .propagationFactory(ExtraFieldPropagation.newFactory(B3Propagation.FACTORY, "user-name"))
                .currentTraceContext(ThreadLocalCurrentTraceContext.newBuilder()
                        .addScopeDecorator(MDCScopeDecorator.create())
                        .build()
                )
                .spanReporter(spanReporter()).build();
    }
}
```

### HTTP插件埋点

通过`brave.servlet`的`TracingFilter`实现`Servlet Filter`接口，注入对应的`spring bean`

```java
@Configuration
@Import(SpanCustomizingAsyncHandlerInterceptor.class)
public class HttpTracingConfiguration {


    @Value("${zipkin.url}")
    private String url;

    /**
     * Configuration for how to send spans to Zipkin
     */
    @Bean
    Sender sender() {
        return OkHttpSender.create(url);
    }

    /**
     * Configuration for how to buffer spans into messages for Zipkin
     */
    @Bean
    AsyncReporter<Span> spanReporter() {
        return AsyncReporter.create(sender());
    }

    /**
     * Controls aspects of tracing such as the name that shows up in the UI
     */
    @Bean
    Tracing tracing(@Value("${spring.application.name}") String serviceName) {
        return Tracing.newBuilder()
                .localServiceName(serviceName)
                .propagationFactory(ExtraFieldPropagation.newFactory(B3Propagation.FACTORY, "user-name"))
                .currentTraceContext(ThreadLocalCurrentTraceContext.newBuilder()
                        .addScopeDecorator(MDCScopeDecorator.create())
                        .build()
                )
                .spanReporter(spanReporter()).build();
    }

    /**
     * decides how to name and tag spans. By default they are named the same as the http method.
     */
    @Bean
    HttpTracing httpTracing(Tracing tracing) {
        return HttpTracing.create(tracing);
    }

    /**
     * Creates client spans for http requests
     * <p>
     * We are using a BPP as the Frontend supplies a RestTemplate bean prior to this configuration
     */
    @Bean
    BeanPostProcessor connectionFactoryDecorator(final BeanFactory beanFactory) {
        return new BeanPostProcessor() {
            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName) {
                return bean;
            }

            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) {
                if (!(bean instanceof RestTemplate)) {
                    return bean;
                }
                RestTemplate restTemplate = (RestTemplate) bean;
                List<ClientHttpRequestInterceptor> interceptors =
                        new ArrayList<>(restTemplate.getInterceptors());
                interceptors.add(0, getTracingInterceptor());
                restTemplate.setInterceptors(interceptors);
                return bean;
            }

            // Lazy lookup so that the BPP doesn't end up needing to proxy anything.
            ClientHttpRequestInterceptor getTracingInterceptor() {
                return TracingClientHttpRequestInterceptor.create(beanFactory.getBean(HttpTracing.class));
            }
        };
    }

    /**
     * Creates server spans for http requests
     */
    @Bean
    Filter tracingFilter(HttpTracing httpTracing) {
        return TracingFilter.create(httpTracing);
    }
}
```




---
参考资料：    
1. [链路追踪 Tracing Analysis](https://help.aliyun.com/document_detail/90277.html?spm=a2c4g.11186623.6.544.56d949ffk9xot6)
2. [在 Dubbo 中使用 Zipkin](http://dubbo.apache.org/zh-cn/blog/use-zipkin-in-dubbo.html)
3. [分布式系统调用跟踪实践](https://t.hao0.me/devops/2016/10/15/distributed-invoke-trace.html)
4. [Zipkin vs Jaeger: Getting Started With Tracing](https://logz.io/blog/zipkin-vs-jaeger/)
5. [Zipkin官方文档](https://zipkin.io/)
6. [Jeager官方文档](https://www.jaegertracing.io/docs/1.8/)
7. [开放分布式追踪（OpenTracing）入门与 Jaeger 实现](https://yq.aliyun.com/articles/514488)
8. [全链路监控（一）：方案概述与比较](https://juejin.im/post/5a7a9e0af265da4e914b46f1)
