---
layout:     post
title:      "Dubbo泛化调用实现"
subtitle:   "微服务框架（四）"
date:       2018-08-01 23:07:47
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Dubbo
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为服务治理框架Dubbo泛化调用的实现**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## 泛化调用组件

Dubbo一般在内部系统间，通过RPC调用，正常情况下服务消费方需要导入服务提供skeleton的jar包，根据此接口编写服务消费者

为了便于开发及测试，实现泛化调用功能，即使用Controller封装请求，获取对应的服务，转接为RPC调用

> 泛化调用组件依赖需添加对应服务接口的jar包

#### 控制器

 1. **获取泛化服务接口**
 2. **组装服务调用参数**
 3. **调用接口、封装数据**

```java
@RestController
@RequestMapping(value = "/")
public class GenericController {

    @RequestMapping(value = "/generic", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE,
            produces = MediaType.APPLICATION_JSON_UTF8_VALUE,
            method = {RequestMethod.POST, RequestMethod.GET})
	@ResponseBody
	public Object call(@ModelAttribute("reqModel") GenericReqModel reqModel) throws ClassNotFoundException {
		Object result = null;

		// 1.获取泛化服务接口
		GenericService service = DubboUtils.fetchGenericService(reqModel);

		// 2.组装调用参数
		String method = reqModel.getMethod();

		String[] parameterTypes = DubboUtils.getMethodParamType(
				reqModel.getService(), reqModel.getMethod());
		
		Object[] args = JSON.parseArray(reqModel.getParamValues()).toArray(
				new Object[] {});

		// 3.调用接口
		return JSON.toJSONString(service
				.$invoke(method, parameterTypes, args));
	}

}
```

#### 泛化引用接口

Dubbo支持的泛化引用接口，可通过在 Spring Boot 配置申明 `generic="true"`使用，详见[泛化引用](http://dubbo.apache.org/#!/docs/user/demos/generic-reference.md?lang=zh-cn)

> 在此组件中无需配置即可使用

```xml
<dubbo:reference id="barService" interface="com.foo.BarService" generic="true" />
```
**Properties**
```properties
dubbo.reference.generic = true
```

**接口定义：**
```java
package com.alibaba.dubbo.rpc.service;

/**
 * Generic service interface
 *
 * @export
 */
public interface GenericService {

    /**
     * Generic invocation
     *
     * @param method         Method name, e.g. findPerson. If there are overridden methods, parameter info is
     *                       required, e.g. findPerson(java.lang.String)
     * @param parameterTypes Parameter types
     * @param args           Arguments
     * @return invocation return value
     * @throws Throwable potential exception thrown from the invocation
     */
    Object $invoke(String method, String[] parameterTypes, Object[] args) throws GenericException;

}
```


#### 工具类

**DubboUtils:**

 1. `fetchGenericService`：获取泛化引用接口(如有缓存, 则从缓存取)
 2. `getMethodParamType`：根据接口类名及方法名通过反射获取参数类型

```java
public class DubboUtils {

	private static Logger logger = LoggerFactory.getLogger(DubboUtils.class);

	private static ApplicationConfig application = new ApplicationConfig(
			"generic-reference");
	
	private static Map<String, GenericService> serviceCache = Maps.newConcurrentMap();

	/**
	 * 获取泛化服务接口(如有缓存, 则从缓存取)
	 * @param reqModel
	 * @return
	 */
	public static GenericService fetchGenericService(GenericReqModel reqModel) {
		// 参数设置
		String serviceInterface = reqModel.getService();
		String serviceGroup = reqModel.getGroup();
		String serviceVersion = reqModel.getVersion();

		// 从缓存中获取服务
		String serviceCacheKey = serviceInterface + "-" + serviceGroup + "-"
				+ serviceVersion;
		GenericService service = serviceCache.get(serviceCacheKey);
		if (service != null) {
			logger.info("fetched generic service from cache");
			return service;
		}

		// 配置调用信息
		ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>();
		reference.setApplication(application);
		reference.setInterface(serviceInterface);
		reference.setGroup(serviceGroup);
		reference.setVersion(serviceVersion);
		reference.setGeneric(true); // 声明为泛化接口

		// 获取对应服务
		service = reference.get();

		// 缓存
		serviceCache.put(serviceCacheKey, service);

		return service;
	}

	/**
	 * 根据接口类名及方法名通过反射获取参数类型
	 * 
	 * @param interfaceName 接口名
	 * @param methodName	方法名
	 * @return
	 * @throws ClassNotFoundException
	 */
	public static String[] getMethodParamType(String interfaceName,
			String methodName) throws ClassNotFoundException {
		String[] paramTypeList = null;
		
		// 创建类
		Class<?> clazz = Class.forName(interfaceName);
		// 获取所有的公共的方法
		Method[] methods = clazz.getMethods();
		for (Method method : methods) {
			if (method.getName().equals(methodName)) {
				
				Class<?>[] paramClassList = method.getParameterTypes();
				paramTypeList = new String[paramClassList.length];
				
				int i = 0;
				for (Class<?> className : paramClassList) {
					paramTypeList[i] = className.getTypeName();
					i++;
				}
				break;
			}
		}

		return paramTypeList;
	}
}
```

#### 启动器

若要使用不同的Web容器，需引入对应的Spring Boot启动依赖，e.g

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jetty</artifactId>
	<version>2.0.3.RELEASE</version>
</dependency>
```

**项目使用Web容器启动**

```java
@SpringBootApplication(scanBasePackages = "com.spring.boot.generic.controller")
public class Starter {

	public static void main(String[] args) {

    	new SpringApplicationBuilder(Starter.class)
        	.web(WebApplicationType.SERVLET)
        	.run(args);

	}

}
```


#### 请求模型

```java
public class GenericReqModel implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = -7389007812411509827L;

	private String registry; // 注册中心地址
	private String service; // 服务包下类名
	private String version; // 版本
	private String group; // 服务组
	private String method; // 方法名
	private String paramTypes; // 参数类型
	private String paramValues; // 参数值
	
	setter/getter
}

```


## POSTMAN使用方法

#### 1.Headers配置

| Key | Value |
|---|---|
|Content-Type | application/json|

#### 2.参数配置(Params)

| Key | Value |
|--|--|
|registry| zookeeper://localhost |
|service|  |
|version| 1.0.0 |
|group|  |
|method| getWord |
|paramValues| [{}]|

#### 3.调用示例
![这里写图片描述](/img/in-post/post-2018-08/postman.png)


---
源码：[GitHub](https://github.com/tillend/spring-boot-dubbo)
