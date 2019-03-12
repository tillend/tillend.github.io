---
layout:     post
title:      "Prometheus + Grafana 可视化监控"
subtitle:   "微服务框架（二十二）"
date:       2019-03-05 22:51:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Prometheus
    - Grafana
---

# Prometheus + Grafana

`Prometheus`使用`pull`模式采集应用中暴露的时间序列数据(`push gateway`可使用`push`模式)，将监控数据持久化在磁盘中，最后通过`Grafana`提供的UI界面进行数据的展示、指标统计和错误报警。

![](https://prometheus.io/assets/architecture.svg)


## Prometheus

`Prometheus` 是一套开源的系统监控报警框架。它启发于 Google 的 borgmon 监控系统，由工作在 `SoundCloud` 的 google 前员工在 2012 年创建，作为社区开源项目进行开发，并于 2015 年正式发布。2016 年，Prometheus 正式加入 Cloud Native Computing Foundation，成为受欢迎度仅次于 Kubernetes 的项目。

作为新一代的监控框架，Prometheus 具有以下特点：

- 强大的多维度数据模型：
    1. 时间序列数据通过 metric 名和键值对来区分。
    2. 所有的 metrics 都可以设置任意的多维标签。
    3. 数据模型更随意，不需要刻意设置为以点分隔的字符串。
    4. 可以对数据模型进行聚合，切割和切片操作。
    5. 支持双精度浮点类型，标签可以设为全 unicode。
- 灵活而强大的查询语句（PromQL）：在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。
- 易于管理： Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
- 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
- 使用 pull 模式采集时间序列数据，这样不仅有利于本机测试而且可以避免有问题的服务器推送坏的 metrics。
- 可以采用 push gateway 的方式把时间序列数据推送至 Prometheus server 端。
- 可以通过服务发现或者静态配置去获取监控的 targets。
- 有多种可视化图形界面。
- 易于伸缩。

> 需要指出的是，由于数据采集可能会有丢失，所以 Prometheus 不适用对采集数据要 100% 准确的情形。但如果用于记录时间序列数据，Prometheus 具有很大的查询优势，此外，Prometheus 适用于微服务的体系架构。

![](/img/in-post/post-2019-03/prometheus.png)

## Grafana

`Grafana`使你能够把来自不同数据源比如`Elasticsearch`, `Prometheus`, `Graphite`, `influxDB`等多样的数据以绚丽的图标展示出来。它也能基于你的`metrics`数据发出告警。当一个告警状态改变时，它能通知你通过email，slack或者其他途径。


![](/img/in-post/post-2019-03/grafana.png)
