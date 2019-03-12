---
layout:     post
title:      "Kibana 可视化图表及 Timelion 插件"
subtitle:   "微服务框架（二十五）"
date:       2019-03-15 22:21:28
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Kibana
    - Timelion
---

## Kibana 图表

Kibana 是一款开源的数据分析和可视化平台，它是 Elastic Stack 成员之一，设计用于和 Elasticsearch 协作。您可以使用 Kibana 对 Elasticsearch 索引中的数据进行搜索、查看、交互操作。您可以很方便的利用图表、表格及地图对数据进行多元化的分析和呈现。

可视化大盘
![](/img/in-post/post-2019-03/dashboard.png)

## 创建索引

Management -> Index Pattern -> Create Index Pattern

> 推荐索引为`%{[@metadata][index_prefix]}-*`，详见[微服务框架（十三）Spring Boot Logstash日志采集](https://blog.csdn.net/why_still_confused/article/details/84346951#t6)

#### 请求总量

Metric -> Count

![](/img/in-post/post-2019-03/total-request.png)

#### 日志级别情况

Metric -> Count
Buckets -> Spilt Slices{Aggregation: Terms, Field: level.keyword, Order By:metric:Count, Order: Descending, Size:10}


![](/img/in-post/post-2019-03/log-level.png)

#### 访问情况

Metric -> Y-Axis{Aggregation: Count}
Buckets -> X-Axis{Aggregation: Date Histogram, Field: @timestamp, Interval: Auto}

![](/img/in-post/post-2019-03/access-situation.png)

#### 响应时间情况

Metric -> Y-Axis{Aggregation: Average, Field: resp_time_ms}
Buckets -> X-Axis{Aggregation: Date Histogram, Field: @timestamp, Interval: Auto}

![](/img/in-post/post-2019-03/response-time.png)


## Timelion

Timelion是一个时间序列数据可视化工具，使您能够在单个可视化中组合完全独立的数据源。它由一种简单的表达式语言驱动，用于检索时间序列数据，执行计算以梳理复杂问题的答案，并可视化结果。

> [Timelion基本表达式](https://blog.csdn.net/qq_16077957/article/details/80023060)

#### 环比折线图

索引的请求数量环比折线图(今日、昨天、一个月前)
```es
.es(index=prod-*,timefield=@timestamp,metric=count,fit=average).label("current day").lines(fill=1,width=2),
.es(index=prod-*,timefield=@timestamp,metric=count,offset=-1d,fit=average).label("last day").lines(width=2).legend(columns=3, position=nw),
.es(index=prod-*,timefield=@timestamp,metric=count,offset=-1M,fit=average).label("last month").lines(width=2).color(gray).legend(columns=3, position=nw)
```

![](/img/in-post/post-2019-03/access-situation-chain-ratio.png)


---
参考资料：    
1.[Timelion](https://www.elastic.co/guide/en/kibana/5.0/timelion.html)    
2.[Timelion入门](https://segmentfault.com/a/1190000016679290)
