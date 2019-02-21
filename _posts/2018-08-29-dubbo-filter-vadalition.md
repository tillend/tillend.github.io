---
layout:     post
title:      "Dubbo调用拦截及参数校检扩展"
subtitle:   "微服务框架（十一）"
date:       2018-08-29 20:53:54
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Dubbo
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　使用Dubbo框架时，面对自身的业务场景，需根据定制的需求编写SPI拓展实现，再根据配置来加载拓展点。**本文为Dubbo调用拦截及参数校检扩展**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## Dubbo SPI拓展点

Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。
Dubbo改进了 JDK 标准的 SPI的一些问题，详见[拓展点加载](http://dubbo.apache.org/zh-cn/docs/dev/SPI.html)

#### 约定

在扩展类的 jar包内 ，放置扩展点配置文件`META-INF/dubbo/`接口全限定名，内容为：配置名=扩展实现类全限定名，多个实现类用换行符分隔。

#### 示例

以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：`META-INF/dubbo/com.alibaba.dubbo.rpc.Filter`，内容为：
```
global=com.test.filter.GlobalExceptionFilter
```

#### 使用配置

Dubbo 配置模块中，扩展点均有对应配置属性或标签，通过配置指定使用哪个扩展实现。比如：
```xml
<dubbo:provider filter="global" />
```

或

```properties
dubbo.provider.filter = global
```

#### 拓展项目路径

```
project
|-- pom.xml
`-- src
    `-- main
        |-- resources
        |    `-- META-INF
        |        `-- dubbo
        |           `-- com.alibaba.dubbo.rpc.Filter
        |-- java
        	 `-- com.test.filter
        	 	 `-- GlobalExceptionFilter.java

```

## 调用拦截拓展

#### Filter SPI

> // before filter	
> Result result = invoker.invoke(invocation);	
> // after filter

```java
@SPI
public interface Filter {

    /**
     * do invoke filter.
     * <p>
     * <code>
     * // before filter
     * Result result = invoker.invoke(invocation);
     * // after filter
     * return result;
     * </code>
     *
     * @param invoker    service
     * @param invocation invocation.
     * @return invoke result.
     * @throws RpcException
     * @see com.alibaba.dubbo.rpc.Invoker#invoke(Invocation)
     */
    Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException;
}
```

#### 实现Filter接口

```java
public class GlobalExceptionFilter<T> implements Filter {}
```

#### GlobalExceptionFilter

> 逻辑为依层级捕捉消化异常，转换为相应的响应码

```java
try {
	Result result = invoker.invoke(invocation);
	
	try{
		Throwable exception = result.getException();
	} catch (Throwable e) {
		// ExceptionFilter Warn
	}

} catch(RuntimeException e){
	// ConstraintViolationException Error
	
	// RpcException Error
	
	// Other RuntimeException Error
} catch (Exception e) {
	// Exception Error
}
```

#### Validation Error
```java
if (e.getCause() != null
		&& e.getCause() instanceof ConstraintViolationException) {
	ConstraintViolationException exs = (ConstraintViolationException) e
			.getCause();

	Set<ConstraintViolation<?>> violations = exs
			.getConstraintViolations();
	for (ConstraintViolation<?> item : violations) {
		/** 获取校检首个失败原因 */
		errorMsg = item.getMessage();
		break;
	}

	resp = Resp.createError(RespCode.PARAM_ERR,
			"input.param.error", errorMsg);
}
```

## 参数校检拓展

#### 校检器

> 根据校检器工厂创建校检器

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public GlobalValidator(URL url) {
	this.clazz = ReflectUtils.forName(url.getServiceInterface());
	String globalValidator = url.getParameter("globalValidator");
	ValidatorFactory factory;
	if (globalValidator != null && globalValidator.length() > 0) {
		factory = Validation
				.byProvider((Class) ReflectUtils.forName(globalValidator))
				.configure().buildValidatorFactory();
	} else {
		factory = Validation.byProvider(HibernateValidator.class)
				.configure()
				.addProperty("hibernate.validator.fail_fast", "true")
				.buildValidatorFactory();
	}
	this.validator = factory.getValidator();
}
```

#### 异常封装

> 参数校检错误时自定义错误信息，封装为RPCException

```java
if (!violations.isEmpty()) {
	logger.error("Failed to validate service: " + clazz.getName()
			+ ", method: " + methodName + ", cause: " + violations);

	String errorMsg = (violations.size() > 0) ? violations.iterator()
			.next().getMessage() : "参数校检错误";

	throw new RpcException(errorMsg, new ConstraintViolationException(
			"Failed to validate service: " + clazz.getName()
					+ ", method: " + methodName + ", cause: "
					+ violations, violations));
}
```



---
参考资料：
1. [Dubbo SPI](http://dubbo.apache.org/zh-cn/docs/dev/SPI.html)
2. [调用拦截扩展](http://dubbo.apache.org/zh-cn/docs/dev/impls/filter.html)
3. [验证拓展](http://dubbo.apache.org/zh-cn/docs/dev/impls/validation.html)
