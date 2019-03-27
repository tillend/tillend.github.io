---
layout:     post
title:      "Dubbo协议及编码过程源码解析"
subtitle:   "微服务框架（十七）"
date:       2019-01-24 22:41:16
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Dubbo
    - Source Code
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文为Dubbo协议、线程模型及协议编码过程源码**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## Dubbo传输协议

Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。

反之，Dubbo 缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-protocol.jpg)

> 红色为dubbo协议缺省值

- Transporter(协议的服务端和客户端实现类型): mina, <font color="red">netty</font>, grizzy
- Serialization(协议序列化方式): dubbo, <font color="red">hessian2</font>, java, json
- Dispatcher(协议的消息派发方式，用于指定线程模型): <font color="red">all</font>, direct, message, execution, connection
- ThreadPool(线程池类型): <font color="red">fixed</font>, cached

#### 报文格式

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019012422393653.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3doeV9zdGlsbF9jb25mdXNlZA==,size_16,color_FFFFFF,t_70)
协议头是16字节的定长数据

- **魔法数**(2 byte)。用于识别dubbo协议数据包，魔法数为`short`类型的常量`0xdabb`
- **消息标志位**(1 byte)
    - **数据包类型**(1 bit)。(0-Response,1-Request)
    - **调用方式**(1 bit)。仅在第16位被设为`Request`的情况下有效(0-单向调用，1-双向调用)
    - **事件标识**(1 bit)。(0-当前数据包是请求或响应包，1-当前数据包是心跳包)
    - **序列化器编号**(5 bit)
- **状态**(1 byte)。当消息类型为响应时，设置请求响应状态，dubbo定义了一些响应的类型。具体类型见`com.alibaba.dubbo.remoting.exchange.Response`
- **消息ID**(8 byte)。`long`类型，请求的唯一识别id（由于采用异步通讯的方式，用以将请求和响应绑定）
- **消息体长度**(4 byte)。`int`类型，即记录Body Content有多少个字节

#### 特性

缺省协议，使用基于 mina 1.1.7 和 hessian 3.2.1 的 tbremoting 交互。

- 连接个数：单连接
- 连接方式：长连接
- 传输协议：TCP
- 传输方式：NIO 异步传输
- 序列化：Hessian 二进制序列化
- 适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用dubbo协议传输大文件或超大字符串。
- 适用场景：常规远程服务方法调用

## Dubbo线程模型

如果事件处理的逻辑能迅速完成，并且不会发起新的 IO 请求，比如只是在内存中记个标识，则直接在 IO 线程上处理更快，因为减少了线程池调度。

但如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。

如果用 IO 线程处理事件，又在事件处理过程中发起新的 IO 请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。

![](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-protocol.jpg)

因此，需要通过不同的派发策略和不同的线程池配置的组合来应对不同的场景:

```xml
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

**Dispatcher**

- **all** 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。(缺省)
- **direct** 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
- **message** 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- **execution** 只请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- **connection** 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。

**ThreadPool**

- **fixed** 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
- **cached** 缓存线程池，空闲一分钟自动删除，需要时重建。
- **limited** 可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
- **eager** 优先创建`Worker`线程池。在任务数量大于`corePoolSize`但是小于`maximumPoolSize`时，优先创建`Worker`来处理任务。当任务数量大于`maximumPoolSize`时，将任务放入阻塞队列中。阻塞队列充满时抛出`RejectedExecutionException`。(相比于`cached`:`cached`在任务数量超过`maximumPoolSize`时直接抛出异常而不是将任务放入阻塞队列)




## Dubbo编码过程源码解析

- 构造消息头，并将除消息体长度外的信息写入`header`
	1. 设置魔法数
	2. 设置数据包类型和序列化器编号
	3. 设置通信方式(单向/双向)
	4. 设置事件标识
	5. 设置请求编号
- 创建序列化器，并将序列化后的数据写入`ChannelBuffer`
- 将消息体长度(`data length`)写入消息头`header`
- 将消息头`header`写入`ChannelBuffer`


```java
public class ExchangeCodec extends TelnetCodec {

    // header length.
    protected static final int HEADER_LENGTH = 16;
    // magic header.
    protected static final short MAGIC = (short) 0xdabb;
    protected static final byte MAGIC_HIGH = Bytes.short2bytes(MAGIC)[0];
    protected static final byte MAGIC_LOW = Bytes.short2bytes(MAGIC)[1];
    // message flag.
    protected static final byte FLAG_REQUEST = (byte) 0x80;
    protected static final byte FLAG_TWOWAY = (byte) 0x40;
    protected static final byte FLAG_EVENT = (byte) 0x20;
    protected static final int SERIALIZATION_MASK = 0x1f;
    private static final Logger logger = LoggerFactory.getLogger(ExchangeCodec.class);

    @Override
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        if (msg instanceof Request) {
            encodeRequest(channel, buffer, (Request) msg);
        } else if (msg instanceof Response) {
            encodeResponse(channel, buffer, (Response) msg);
        } else {
            super.encode(channel, buffer, msg);
        }
    }
    
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        Serialization serialization = getSerialization(channel);

        // 创建消息头字节数组，长度为 16
        byte[] header = new byte[HEADER_LENGTH];

        // 设置魔法数
        Bytes.short2bytes(MAGIC, header);

        // 设置数据包类型（Request/Response）和序列化器编号
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());

        // 设置通信方式(单向/双向)
        if (req.isTwoWay()) {
            header[2] |= FLAG_TWOWAY;
        }
        
        // 设置事件标识
        if (req.isEvent()) {
            header[2] |= FLAG_EVENT;
        }

        // 设置请求编号，8个字节，从第4个字节开始设置
        Bytes.long2bytes(req.getId(), header, 4);

        // 获取 buffer 当前的写位置
        int savedWriteIndex = buffer.writerIndex();
        // 更新 writerIndex，为消息头预留 16 个字节的空间
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        // 创建序列化器，比如 Hessian2ObjectOutput
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (req.isEvent()) {
            // 对事件数据进行序列化操作
            encodeEventData(channel, out, req.getData());
        } else {
            // 对请求数据进行序列化操作
            encodeRequestData(channel, out, req.getData(), req.getVersion());
        }
        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }
        bos.flush();
        bos.close();
        
        // 获取写入的字节数，也就是消息体长度
        int len = bos.writtenBytes();
        checkPayload(channel, len);

        // 将消息体长度写入到消息头中
        Bytes.int2bytes(len, header, 12);

        // 将 buffer 指针移动到 savedWriteIndex，为写消息头做准备
        buffer.writerIndex(savedWriteIndex);
        // 从 savedWriteIndex 下标处写入消息头
        buffer.writeBytes(header);
        // 设置新的 writerIndex，writerIndex = 原写下标 + 消息头长度 + 消息体长度
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
}
```

**请求数据的序列化操作：**
```java
public class DubboCodec extends ExchangeCodec implements Codec2 {
    
	protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
        RpcInvocation inv = (RpcInvocation) data;

        // 依次序列化 dubbo version、path、version
        out.writeUTF(version);
        out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
        out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));

        // 序列化调用方法名
        out.writeUTF(inv.getMethodName());
        // 将参数类型转换为字符串，并进行序列化
        out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
        Object[] args = inv.getArguments();
        if (args != null)
            for (int i = 0; i < args.length; i++) {
                // 对运行时参数进行序列化
                out.writeObject(encodeInvocationArgument(channel, inv, i));
            }
        
        // 序列化 attachments
        out.writeObject(inv.getAttachments());
    }
}
```

---
参考资料：    
1. [线程模型](http://dubbo.apache.org/zh-cn/docs/user/demos/thread-model.html)
2. [Dubbo 关于同步/异步调用的几种方式](http://dubbo.apache.org/zh-cn/blog/dubbo-invoke.html)
3. [Netty在dubbo中的应用浅析](https://www.jianshu.com/p/23589c019de5#fnref1)
4. [Dubbo学习笔记](http://blog.51cto.com/13933361/2164498)
5. [服务调用过程](http://dubbo.apache.org/zh-cn/docs/source_code_guide/service-invoking-process.html)



























