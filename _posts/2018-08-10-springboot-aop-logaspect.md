---
layout:     post
title:      "Spring Boot AOP 日志切面实现"
subtitle:   "微服务框架（八）"
date:       2018-08-10 19:49:15
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    - AOP
    
---

　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。

　　**本文为使用Spring Boot AOP 实现日志切面、分离INFO和ERROR级别日志**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。


## 通用日志组件

 为了便于记录日志，实现了通用的日志组件，通过使用注解`@Loggable`标记方法即可编制入日志切面中

> 日至切面及日志输出规则均已集成于[Maven Archetype]()

通用日志组件通过以下配置引用


### 日志切面

根据定义日志切点（`@Loggable`）环绕处理逻辑：

 1. 根据注解值获取相应的日志对象
 2. 记录日志信息


```java
@Aspect
@Component
public class LogAspect {

	private static final Map<Class<?>, Logger> loggerHolder = new ConcurrentHashMap<Class<?>, Logger>();

	@Pointcut(value = "@annotation(com.linghit.common.log.annotation.Loggable)")
	public void log() {
	}

	/**
	 * 
	 * around:根据日志注解类型为方法调用记录日志 <br/>
	 */
	@Around("log()")
	public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
		AbstractLogBean logBean = null;

		Method method = MethodSignature.class.cast(joinPoint.getSignature())
				.getMethod();
		Annotation[] annotations = method.getAnnotations();
		for (Annotation annotation : annotations) {
			if (annotation instanceof Loggable) {
				Loggable loggable = Loggable.class.cast(annotation);
				logBean = getLogBean(loggable.value(), joinPoint);
				break;
			}
		}

		Object retVal = null;
		if (null == logBean) {
			retVal = joinPoint.proceed(joinPoint.getArgs());
		} else {
			logBean.setEventName(StringUtils.isBlank(logBean.getEventName()) ? method
					.getName() : logBean.getEventName() + "_"
					+ method.getName());
			logBean.setRequest(joinPoint.getArgs());
			try {
				retVal = joinPoint.proceed(joinPoint.getArgs());
			} catch (Exception e) {
				retVal = Resp.createError(RespCode.BUSINESS_INVALID,
						"service.fail", "服务失败");

				logBean.setE(e);

			} finally {

				logBean.setResponse(retVal);

				Logger logger = getLogger(joinPoint.getTarget().getClass());
				LogUtils.log(logger, logBean);
			}
		}

		return retVal;

	}

	private AbstractLogBean getLogBean(DataAccessType type, JoinPoint joinPoint) {
		AbstractLogBean logBean = null;
		switch (type.getValue()) {
		case "MySQL":
			logBean = DataAccessLogBean.newDataAccessMysqlLogBean();
			break;
		case "Http":
			logBean = DataAccessLogBean.newDataAccessHttpLogBean();
			break;
		case "Redis":
			logBean = DataAccessLogBean.newDataAccessRedisLogBean();
			break;
		default:
			logBean = new ServiceAccessLogBean(joinPoint.getTarget().getClass()
					.getSimpleName(), DataAccessType.DUBBO.getValue(),
					RpcContextUtils.getClientIp(),
					RpcContextUtils.getLocalAddress());
		}
		return logBean;
	}

	private Logger getLogger(Class<?> clazz) {
		if (!loggerHolder.containsKey(clazz)) {
			loggerHolder.put(clazz, LoggerFactory.getLogger(clazz));
		}
		return loggerHolder.get(clazz);
	}
}
```

### 日志注解

```java
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Loggable {

	/**
	 * Name of the logType in which the logging takes place.
	 * <p>
	 * May be used to determine the target cache (or caches), matching the
	 * qualifier value (or the bean name(s)) of (a) specific bean definition.
	 */
	DataAccessType value() default DataAccessType.DUBBO;
}
```

### 切面扫描及使用

Spring Boot启动类扫描日志切面组件，及配置切面代理为`true`

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
@ComponentScan("com.linghit.ocs.zhanxing.service.handler.LogAspect")
```

> 使用时只需在相应方法处加入`@Loggable`注解

```java
@Loggable
public void test{}
```

### 相关依赖

Spring Boot AOP 起步依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-aop</artifactId>
	<version>${springboot.version}</version>
</dependency>
```


## 日志分离术

为了便于查看及采集错误日志，下述配置设置`INFO`与`ERROR`日志输出至不同文件

> filePattern中需含有%i，每次rollover时，计数器将每次加1，若达到max的值，将删除旧的文件

`log4j2.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="info" monitorInterval="30"
	name="Log4j2Config">
	<Properties>
		<Property name="PATTERN">[%p] [%d{yyyy-MM-dd HH:mm:ss}][%c{10}]%m%n
		</Property>
		<Property name="filePatch">./log/${projectName}/</Property>
		<Property name="fileName">${projectName}.log</Property>
		<Property name="errorFileName">${projectName}-error.log</Property>
	</Properties>
	<Appenders>
		<!-- 类型名为Console，名称为必须属性 -->
		<Console name="STDOUT">
			<PatternLayout charset="UTF-8" pattern="${PATTERN}" />
		</Console>

		<RollingFile name="DailyRollingFile" fileName="${filePatch}${fileName}"
			filePattern="${filePatch}${fileName}.%d{yyyy-MM-dd}.%i">
			<PatternLayout charset="UTF-8" pattern="${PATTERN}" />
			<Filters>
				<!--如果是error级别拒绝 -->
				<ThresholdFilter level="error" onMatch="DENY"
					onMismatch="NEUTRAL" />
				<!--如果是debug\info\warn输出 -->
				<ThresholdFilter level="debug" onMatch="ACCEPT"
					onMismatch="DENY" />
			</Filters>
			<Policies>
				<!-- 一般与 filePattern联用 以日志的命名精度来确定单位 这里用yyyy-MM-dd来记录 所以1 表示是以天为周期存储文件 -->
				<TimeBasedTriggeringPolicy interval="1"
					modulate="true" />
				<!-- 日志文件大小 <SizeBasedTriggeringPolicy size="1 MB" /> -->
			</Policies>

			<!-- 最多保留文件数 -->
			<DefaultRolloverStrategy max="7" />
		</RollingFile>

		<RollingFile name="ErrorDailyRollingFile" fileName="${filePatch}${errorFileName}"
			filePattern="${filePatch}${errorFileName}.%d{yyyy-MM-dd}.%i">
			<PatternLayout charset="UTF-8" pattern="${PATTERN}" />
			<Filters>
				<ThresholdFilter level="error" onMatch="ACCEPT"
					onMismatch="DENY" />
			</Filters>
			<Policies>
				<!-- 一般与 filePattern联用 以日志的命名精度来确定单位 这里用yyyy-MM-dd来记录 所以1 表示是以天为周期存储文件 -->
				<TimeBasedTriggeringPolicy interval="1"
					modulate="true" />
				<!-- 日志文件大小 <SizeBasedTriggeringPolicy size="1 MB" /> -->
			</Policies>

			<!-- 最多保留文件数 -->
			<DefaultRolloverStrategy max="7" />
		</RollingFile>

		<!-- Socket Apppender配置，通过TCP协议连接Logstash <Socket name="LOGSTASH" host="127.0.0.1" 
			port="9001" protocol="TCP"> <PatternLayout pattern="${PATTERN}" /> </Socket> -->

	</Appenders>

	<Loggers>
		<!-- root loggerConfig设置 -->
		<AsyncRoot level="info" additivity="false">
			<AppenderRef ref="STDOUT" />
			<AppenderRef ref="DailyRollingFile" />
			<AppenderRef ref="ErrorDailyRollingFile" />
		</AsyncRoot>
	</Loggers>

</Configuration>
```

日志输出效果如下：
```
log
	project.name
		service.log
		service.log.2018-8-10
		service-error.log
		service-error.log.2018-8-10
```
