---
layout:     post
title:      "Dubbo领域模型、调用链及调用方式"
subtitle:   "微服务框架（十八）"
date:       2019-01-29 22:10:32
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Dubbo
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为Dubbo领域模型、调用链及调用方式**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。



## Dubbo领域模型

在 Dubbo 的核心领域模型中：

- `Protocol` 是服务域，它是 `Invoker` 暴露和引用的主功能入口，它负责 `Invoker` 的生命周期管理。
- `Invoker`是实体域，它是`Dubbo`的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起`invoke`调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
- `Invocation` 是会话域，它持有调用过程中的变量，比如方法名，参数等。


## Dubbo调用链

![](http://dubbo.apache.org/docs/zh-cn/dev/sources/images/dubbo-extension.jpg)


#### Dubbo协议请求数据发送全调用链

1. 由实体域(`Invoker`)执行会话域(`Invocation`)的集群实现
2. 拦截器(`Filter`)链的拦截过滤
3. 监听器(`Listener`)的监听指向
4. Dubbo调用(`DubboInvoker`)的实现
5. 协议头交互客户端(`HeaderExchangeClient`)
6. Netty通信

Dubbo协议请求数据发送全调用链
```
proxy0#sayHello(String)
  —> InvokerInvocationHandler#invoke(Object, Method, Object[])
    —> MockClusterInvoker#invoke(Invocation)
      —> AbstractClusterInvoker#invoke(Invocation)
        —> FailoverClusterInvoker#doInvoke(Invocation, List<Invoker<T>>, LoadBalance)
          —> Filter#invoke(Invoker, Invocation)  // 包含多个 Filter 调用
            —> ListenerInvokerWrapper#invoke(Invocation) 
              —> AbstractInvoker#invoke(Invocation) 
                —> DubboInvoker#doInvoke(Invocation)    //Dubbo同异步调用
                  —> ReferenceCountExchangeClient#request(Object, int)
                    —> HeaderExchangeClient#request(Object, int)	//协议头交互Client
                      —> HeaderExchangeChannel#request(Object, int)
                        —> AbstractPeer#send(Object)
                          —> AbstractClient#send(Object, boolean)
                            —> NettyChannel#send(Object, boolean)   //Netty通信
                              —> NioClientSocketChannel#write(Object)
```

#### DubboInvoker

> Dubbo 的底层IO操作都是异步的。`Consumer`端发起调用后，得到一个`Future`对象。
对于同步调用，业务线程通过`Future#get(timeout)`，阻塞等待`Provider`端将结果返回；`timeout`则是`Consumer`端定义的超时时间。


- `RpcInvocation`设置相关参数`path`和`version`
- 获取`ExchangeClient`
- 同步/异步调用模式
    1. 同步调用模式:框架获得`DefaultFuture`对象后，会立即调用`get`方法进行等待
    2. 异步调用模式:将该对象封装到`FutureAdapter`实例中，并将`FutureAdapter`实例设置到`RpcContext`中，供用户使用

> `FutureAdapter` 是一个适配器，用于将`Dubbo`中的`ResponseFuture`与`JDK `中的`Future`进行适配。

DubboInvoker:

```java
public class DubboInvoker<T> extends AbstractInvoker<T> {
    
    private final ExchangeClient[] clients;
    
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        // 设置 path 和 version 到 attachment 中
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);

        ExchangeClient currentClient;
        if (clients.length == 1) {
            // 从 clients 数组中获取 ExchangeClient
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            // 获取异步配置
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            // isOneway 为 true，表示“单向”通信
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);

            // 异步无返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                // 发送请求
                currentClient.send(inv, isSent);
                // 设置上下文中的 future 字段为 null
                RpcContext.getContext().setFuture(null);
                // 返回一个空的 RpcResult
                return new RpcResult();
            } 

            // 异步有返回值
            else if (isAsync) {
                // 发送请求，并得到一个 ResponseFuture 实例
                ResponseFuture future = currentClient.request(inv, timeout);
                // 设置 future 到上下文中
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                // 暂时返回一个空结果
                return new RpcResult();
            } 

            // 同步调用
            else {
                RpcContext.getContext().setFuture(null);
                // 发送请求，得到一个 ResponseFuture 实例，并调用该实例的 get 方法进行等待
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(..., "Invoke remote method timeout....");
        } catch (RemotingException e) {
            throw new RpcException(..., "Failed to invoke remote method: ...");
        }
    }
    
    // 省略其他方法
}
```



## Dubbo调用方式

Dubbo 缺省协议采用**单一长连接**，底层实现是 `Netty` 的 `NIO` 异步通讯机制；基于这种机制，Dubbo 实现了以下几种调用方式：

- **同步调用**
- **异步调用**
- **参数回调**
- **事件通知**


> 下述只描述异步调用，其他调用方式详细描述及例子见[Dubbo 关于同步/异步调用的几种方式](http://dubbo.apache.org/zh-cn/blog/dubbo-invoke.html)

#### 异步调用

基于`Dubbo`底层的异步`NIO`实现异步调用，对于`Provider`响应时间较长的场景是必须的，它能有效利用`Consumer`端的资源，相对于`Consumer`端使用多线程来说开销较小。

![](http://dubbo.apache.org/img/blog/dubbo-async.svg)

**Provider 端接口**
```java
public interface AsyncService {
    String goodbye(String name);
}
```

**Consumer 配置**
```xml
<dubbo:reference id="asyncService" interface="com.alibaba.dubbo.samples.async.api.AsyncService">
    <dubbo:method name="goodbye" async="true"/>
</dubbo:reference>
```
需要异步调用的方法，均需要使用 `<dubbo:method/>`标签进行描述。


**Consumer 端发起调用**
```java
AsyncService service = ...;
String result = service.goodbye("samples");// 这里的返回值为空，请不要使用
Future<String> future = RpcContext.getContext().getFuture();
... // 业务线程可以开始做其他事情
result = future.get(); // 阻塞需要获取异步结果时，也可以使用 get(timeout, unit) 设置超时时间
```


---
参考资料：
1. [Dubbo 关于同步/异步调用的几种方式](http://dubbo.apache.org/zh-cn/blog/dubbo-invoke.html)
2. [服务调用过程](http://dubbo.apache.org/zh-cn/docs/source_code_guide/service-invoking-process.html)
3. [框架设计](http://dubbo.apache.org/zh-cn/docs/dev/design.html)
