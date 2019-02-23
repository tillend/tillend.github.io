---
layout:     post
title:      "Spring Boot @ControllerAdvice异常处理"
subtitle:   "微服务框架（十四）"
date:       2018-12-04 22:13:22
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Spring Boot
    
---

&#32;　　此系列文章将会描述Java框架**Spring Boot**、服务治理框架**Dubbo**、应用容器引擎**Docker**，及使用Spring Boot集成Dubbo、Mybatis等开源框架，其中穿插着Spring Boot中日志切面等技术的实现，然后通过gitlab-CI以持续集成为Docker镜像。 

　　**本文为Spring Boot使用`@ControllerAdvice`进行自定义异常捕捉及处理**

> 本系列文章中所使用的框架版本为Spring Boot 2.0.3-RELEASE，Spring 5.0.7-RELEASE，Dubbo 2.6.2。

## 注解

#### @ControllerAdvice

**控制器增强**，所有在Controller中被抛出的异常将会被使用`@ControllerAdvice`注解的类捕捉。默认情况下，	`@ControllerAdvice`全局应用于所有控制器的方法。

>若所有异常处理类返回的数据格式为`json`，则可以使用 `@RestControllerAdvice` 代替 `@ControllerAdvice` 
`@RestControllerAdvice = @ControllerAdvice  +  @ResponseBody`

#### @ExceptionHandler
**异常处理器**，在特定异常处理程序的类或方法中以处理异常的注解。



## 业务异常

常规的Controller层，会抛出业务异常、校检异常等

```java
@RestController
@RequestMapping(value = "/")
public class TestController {

    @Autowired
    private BussinessService bussinessService ;

    @RequestMapping(value = "/basic", method = RequestMethod.POST)
    public Resp<RespVO> getBasic(@RequestBody @Valid ReqVO reqVO) {

		RespVO respVO = bussinessService.getResult(reqVO);
		if（respVO  == null）
			throws BussinessExceptionMapper.build("调用失败");

        return Resp.createSuccess(respVO);
    }
}
```

#### 业务异常定义

```java
@Setter
@Getter
public class BussinessException extends RuntimeException {

    public static final String UNKNOWN_EXCEPTION = "unknown.exception";
    public static final String SERVICE_FAIL = "service.fail";
    public static final String NETWORK_EXCEPTION = "network.exception";
    public static final String TIMEOUT_EXCEPTION = "timeout.exception";

    private String code;

    public BussinessException() {
        super();
    }

    public BussinessException(String message, Throwable cause) {
        super(message, cause);
    }

    public BussinessException(String message) {
        super(message);
    }

    public BussinessException(Throwable cause) {
        super(cause);
    }

    public BussinessException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public BussinessException(String code, String message) {
        super(message);
        this.code = code;
    }
}
```

#### 业务异常构造类
```java
public class BussinessExceptionMapper {

    public static BussinessException build(String message) {
        return build(BussinessException.SERVICE_FAIL, message);
    }

    public static BussinessException build(String code, String message) {
        return new BussinessException(code, message);
    }

    public static BussinessException build(String message, Throwable cause) {
        return build(BussinessException.SERVICE_FAIL, message, cause);
    }

    public static BussinessException build(String code, String message, Throwable cause) {
        return new BussinessException(code, message, cause);
    }

}
```
## 异常处理
`@Controller` 及 `@RestController`中的异常捕捉及处理

#### 参数校检异常
使用`hibernate validator`校检参数，`@Valid`注解抛出的校检异常为`MethodArgumentNotValidException`，在`@ExceptionHandler`注释的方法中捕捉第一个错误信息

```java
@ResponseBody
@ExceptionHandler(value = MethodArgumentNotValidException.class)
public Resp validHandler(MethodArgumentNotValidException ex) {

    logger.error("ValidException:", ex);

    List<ObjectError> list = ex
            .getBindingResult().getAllErrors();

    String errorMsg = StringUtils.defaultString(list.get(0).getDefaultMessage(), "参数错误");

    return Resp.createError(RespCode.PARAM_ERR, "input.param.error", errorMsg);
}
```

#### 业务异常
```java
@ResponseBody
@ExceptionHandler(value = BussinessException.class)
public Resp bussinessHandler(BussinessException ex) {

    logger.error("BussinessException:", ex);

    return Resp.createError(RespCode.BUSINESS_INVALID, ex.getCode(), ex.getMessage());
}
```

#### Dubbo Rpc异常及运行时异常
```java
@ControllerAdvice
public class GlobalControllerAdvice {

    private Logger logger = LoggerFactory.getLogger(GlobalControllerAdvice.class);

    @ResponseBody
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public Resp validHandler(MethodArgumentNotValidException ex) {

        logger.error("ValidException:", ex);

        List<ObjectError> list = ex
                .getBindingResult().getAllErrors();

        String errorMsg = StringUtils.defaultString(list.get(0).getDefaultMessage(), "参数错误");

        return Resp.createError(RespCode.PARAM_ERR, "input.param.error", errorMsg);
    }

    @ResponseBody
    @ExceptionHandler(value = BussinessException.class)
    public Resp bussinessHandler(BussinessException ex) {

        logger.error("BussinessException:", ex);

        return Resp.createError(RespCode.BUSINESS_INVALID, ex.getCode(), ex.getMessage());
    }

    @ResponseBody
    @ExceptionHandler(value = Exception.class)
    public Resp errorHandler(Exception e) {

        logger.error("uncaught Exception:", e);

        Resp resp;
        if (e instanceof RpcException) {
            resp = Resp.createError(RespCode.ERROR, "rpc.exception",
                    e.getMessage());
        } else if (e instanceof RuntimeException) {
            resp = Resp.createError(RespCode.ERROR, "runtime.exception",
                    "运行时异常,详见堆栈日志");
        } else {
            resp = Resp.createError(RespCode.BUSINESS_INVALID,
                    "service.fail", "服务失败");
        }

        return resp;
    }
}
```

---
参考资料：    
1.[ControllerAdvice](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html)    
2.[RestControllerAdvice](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestControllerAdvice.html)
