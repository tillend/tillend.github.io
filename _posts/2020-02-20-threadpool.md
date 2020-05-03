---
layout:     post
title:      "Java线程池"
subtitle:   "ThreadPoolExecutor"
date:       2020-02-20 17:13:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Java
    - ThreadPool
---

## 线程池

线程池解决了两个不同的问题：
1. 减少线程创建的开销，能提高执行大量异步任务的效率
2. 提供了一种限制和管理资源及线程的方法，并且还维护了一些基本的统计信息(如已完成的任务数)

线程池的使用对`new Thread()`的优势：

 1. 复用存在的线程，减少对象创建、消亡的开销，性能佳。 
 2. 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
 3. 提供定时执行、定期执行、单线程、并发数控制等功能。

## ThreadPoolExecutor

常用默认实现：
1. `Executors#newCachedThreadPool`：无边界线程池，带有自动线程回收
2. `Executors#newFixedThreadPool`：固定大小的线程池
3. `Executors#newSingleThreadExecutor`：单个后台线程，大多数场景用于预初始化配置

有需要执行的任务进入线程池时
-   当前线程数小于核心线程数时，创建线程。
-   当前线程数大于等于核心线程数，且工作队列未满时，将任务放入工作队列。
-   当前线程数大于等于核心线程数，且工作队列已满
	-  若线程数小于最大线程数，创建线程
	-  若线程数等于最大线程数，抛出异常，拒绝任务(具体处理方式取决于`handler`的策略)


### 主要参数

 - corePoolSize
	核心线程数。空闲时仍会保留在池中的线程数，除非设置了`allowCoreThreadTimeOut`参数

 - maximumPoolSize
	最大线程数。允许在池中的最大线程数

 - keepAliveTime 
	存活时间。当前线程数大于核心线程数时，空余线程的最长存活时间

 - unit 
	单位。`keepAliveTime`参数的时间单位

 - workQueue 
	工作队列，接口类为阻塞队列。任务执行前存储的队列，只有通过`submit`方法提交的任务才会进入队列

 - threadFactory 
	线程工厂。创建线程。默认使用`Executors.defaultThreadFactory()`，所有的线程都属于同一个`ThreadGroup`，都有相同的优先级，且均不是守护线程。
	(可用`new NamedThreadFactory("test")`来对线程池中的线程添加前缀标识)

 - handler
	任务丢弃策略。若线程池已经关闭、或线程池已满，那么新的任务会被拒绝。

	 - `ThreadPoolExecutor.AbortPolicy`:丢弃任务并抛出`RejectedExecutionException`异常
   	 - `ThreadPoolExecutor.DiscardPolicy`：丢弃任务，但不抛出异常。
    - 	`ThreadPoolExecutor.DiscardOldestPolicy`：丢弃队列最前面的任务，然后重新尝试执行任务(循环此过程)
    - 	`ThreadPoolExecutor.CallerRunsPolicy`：由调用线程处理该任务

### BlockingQueue
`BlockingQueue`类型
- `ArrayBlockingQueue`：由数组结构组成的有界阻塞队列
- `LinkedBlockingQueue`：由链表结构组成的有界阻塞队列
- `PriorityBlockingQueue`：支持优先级排序的无界阻塞队列
- `DealyQueue`：使用优先级队列实现的无界阻塞队列
- `SynchronousQueue`：不存储元素的阻塞队列
- `LinkedTransferQueue`：由链表结构组成的无界阻塞队列
- `LinkedBlockingDeque`：由链表结构组成的双向阻塞队列

#### ArrayBlockingQueue

`ArrayBlockingQueue`是定长阻塞队列。原理是使用一个可重入锁和`Condition`进行并发控制(classic two-condition algorithm)。

---
参考资料：
1. [精通ThreadPoolExcutor](https://juejin.im/post/5da027a6f265da5ba95c3250)
2. [Java并发编程中四种线程池](https://blog.csdn.net/riemann_/article/details/97617432)