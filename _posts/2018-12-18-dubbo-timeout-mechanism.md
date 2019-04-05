---
layout:     post
title:      "Dubbo 超时机制及服务降级"
subtitle:   "微服务框架（十五）"
date:       2018-12-18 21:31:11
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Dubbo
    - Filter
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文为Dubbo超时机制及服务降级**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## 生产者超时机制

#### 创建超时

当服务出现创建超时的时候，`TimeoutFilter`会打印该创建记录的详细信息，日志级别为`WARN`，即为<font color="red">可恢复异常，或瞬时的状态不一致
</font>

```java
/**
 * Log any invocation timeout, but don't stop server from running
 */
@Activate(group = Constants.PROVIDER)
public class TimeoutFilter implements Filter {

    private static final Logger logger = LoggerFactory.getLogger(TimeoutFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        long start = System.currentTimeMillis();
        Result result = invoker.invoke(invocation);
        long elapsed = System.currentTimeMillis() - start;
        if (invoker.getUrl() != null
                && elapsed > invoker.getUrl().getMethodParameter(invocation.getMethodName(),
                "timeout", Integer.MAX_VALUE)) {
            if (logger.isWarnEnabled()) {
                logger.warn("invoke time out. method: " + invocation.getMethodName()
                        + " arguments: " + Arrays.toString(invocation.getArguments()) + " , url is "
                        + invoker.getUrl() + ", invoke elapsed " + elapsed + " ms.");
            }
        }
        return result;
    }

}
```

#### 线程池满载

当业务线程池满载，且没设置线程池队列时，`AbortPolicyWithReport`会打印线程池的详细信息，日志级别为`WARN`，并dump出十分钟内的堆栈信息

```
[WARN] [2018-12-16 22:06:00][com.alibaba.dubbo.common.threadpool.support.AbortPolicyWithReport] [DUBBO] Thread pool is EXHAUSTED! Thread Name: DubboServerHandler-127.0.0.1:9000, Pool Size: 10 (active: 10, core: 10, max: 10, largest: 10), Task: 3350 (completed: 3340), Executor status:(isShutdown:false, isTerminated:false, isTerminating:false), in dubbo://127.0.0.1:9000!, dubbo version: 2.6.2, current host: 127.0.0.1
```

> 建议不要设置[线程池队列](http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-protocol.html)，当线程程池时应立即失败


## 消费者超时机制

>  `Dubbo`默认采用了`netty`作为网络组件，它属于`NIO`的模式。消费者发起远程请求后，线程不会阻塞等待服务端的返回，而是马上得到一个`ResponseFuture`，消费端通过不断的轮询机制判断结果是否有返回。

`DefaultFuture`实现了`ResponseFuture`接口，在`NIO(Non-blocking IO)`的`Channel`中轮循获取结果。当请求超时或抛出中断异常(`InterruptedException`)时，对外抛出`TimeoutException`


```java
//DefaultFuture#get()
@Override
public Object get(int timeout) throws RemotingException {
    if (timeout <= 0) {
        timeout = Constants.DEFAULT_TIMEOUT;
    }
    if (!isDone()) {
        long start = System.currentTimeMillis();
        lock.lock();
        try {
            while (!isDone()) {
                done.await(timeout, TimeUnit.MILLISECONDS);
                if (isDone() || System.currentTimeMillis() - start > timeout) {
                    break;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        if (!isDone()) {
            throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
        }
    }
    return returnFromResponse();
}
```

## 超时配置及优先级

#### 超时配置

> 建议Provider 上尽量多配置 Consumer 端属性

- 作服务的提供者，比服务使用方更清楚服务性能参数，如调用的超时时间、合理的重试次数等
- 在 Provider 配置后，Consumer 不配置则会使用 Provider 的配置值，即 Provider 配置可以作为 Consumer 的缺省值。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider 是不可控的，并且往往是不合理的
- Provider 上尽量多配置 Consumer 端的属性，让 Provider 实现者一开始就思考 Provider 服务特点、服务质量等问题。

### 配置覆盖关系
以 timeout 为例，显示了配置的查找顺序，其它 retries, loadbalance, actives 等类似：(详见[官方文档](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html))

- 方法级优先，接口级次之，全局配置再次之。
- 如果级别一样，则消费方优先，提供方次之。

其中，服务提供方配置，通过 URL 经由注册中心传递给消费方。

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-config-override.jpg)

## 服务降级

> - 可以通过服务降级功能，**临时屏蔽**某个出错的非关键服务，并定义降级后的返回策略。
> - 当整个微服务架构整体的负载超出了预设的上限阈值或即将到来的流量预计将会超过预设的阈值时，为了保证重要或基本的服务能正常运行，我们可以将一些**不重要**或 **不紧急**的服务或任务进行服务的**延迟使用**或**暂停使用**。

在Dubbo服务中配置`dubbo.service.mock = fail:return+null `，当消费方对该服务的方法调用在失败后，再返回null值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。


---
参考资料：    
1. [dubbo源码分析（二）：超时原理以及应用场景](http://www.cnblogs.com/ASPNET2008/p/7292472.html)
2. [Dubbo推荐用法](http://dubbo.apache.org/zh-cn/docs/user/recommend.html)
3. [Dubbo线程模型](http://dubbo.apache.org/zh-cn/docs/user/demos/thread-model.html)
4. [Dubbo服务降级](http://dubbo.apache.org/zh-cn/docs/user/demos/service-downgrade.html)
5. [微服务架构—服务降级](https://baijiahao.baidu.com/s?id=1607017969929896292&wfr=spider&for=pc)
