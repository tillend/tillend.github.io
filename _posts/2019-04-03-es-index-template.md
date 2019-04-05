---
layout:     post
title:      "ElasticSearch 索引模板"
subtitle:   "微服务框架（十五）"
date:       2019-04-03 21:31:11
author:     "Tillend"
catalog:      true
header-img: "img/post-bg-alitrip.jpg"
tags:
    - ElasticSearch
---

# 索引模板

索引模块是按索引创建的模块，用于控制所有与索引相关的方面。

## 索引配置

索引级别设置可以根据具体索引进行对应设置。相关设置为：
- `static`:它们只能在索引创建时或[闭合索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-open-close.html)上设置。
- `dynamic`:可以使用[update-index-settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-update-settings.html) API在实时索引上更改它们。

!> 更改闭合索引上的静态或动态索引设置可能会导致不正确的设置，如果不删除并重新创建索引，则无法纠正这些设置。

> 相关设置详见[Index Modules](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#index-modules)

## 模板操作

ES的模板CRUD可由`ES REST API`或`Kibana Dev Tools`进行操作

![](../../assets/kibana_devtools.png)

#### 创建模板

REST API
```bash
curl -XPUT localhost:9200/_template/template_name -d
{  
    "template" : "example-*",
    "order":1,
    "settings" : {  
        "number_of_shards" : 1  
    },  
    "mappings" : {  
        "type1" : {  
            "_source" : {"enabled" : false }  
        }  
    }  
}
```

Dev Tools
```es
PUT _template/template_name
{  
    "template" : "example-*",
    "order":1,
    "settings" : {  
        "number_of_shards" : 1  
    },  
    "mappings" : {  
        "type1" : {  
            "_source" : {"enabled" : false }  
        }  
    }  
}
```

#### 查看模板

REST API
```bash
curl -XGET localhost:9200/_template/template_name
```

Dev Tools
```es
GET _template/template_name
```

#### 删除模板

REST API
```bash
curl -XDELETE localhost:9200/_template/template_name
```

Dev Tools
```es
DELETE _template/template_name
```

## 索引模板示例

- `template`为模板匹配规则
- `order`为模板优先级(当索引同时匹配多个模板，先应用数值小的模板，再以数值大的模板配置进行覆盖)
- `properties`中指定特定类型，否则则会自动匹配为`dynamic_templates`中的`text`或`keyword`类型

```conf
{
  "kong_template" : {
    "order" : 1,
    "version" : 60001,
    "index_patterns" : [
      "kong-*"
    ],
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [
          {
            "message_field" : {
              "path_match" : "message",
              "match_mapping_type" : "string",
              "mapping" : {
                "type" : "text",
                "norms" : false
              }
            }
          },
          {
            "string_fields" : {
              "match" : "*",
              "match_mapping_type" : "string",
              "mapping" : {
                "type" : "text",
                "norms" : false,
                "fields" : {
                  "keyword" : {
                    "type" : "keyword",
                    "ignore_above" : 256
                  }
                }
              }
            }
          }
        ],
        "properties" : {
          "@timestamp" : {
            "type" : "date"
          },
          "@version" : {
            "type" : "keyword"
          },
          "request.headers.remoteip" : {
            "type" : "ip"
          },
          "geoip" : {
            "dynamic" : true,
            "properties" : {
              "ip" : {
                "type" : "ip"
              },
              "location" : {
                "type" : "geo_point"
              },
              "latitude" : {
                "type" : "half_float"
              },
              "longitude" : {
                "type" : "half_float"
              }
            }
          }
        }
      }
    },
    "aliases" : { }
  }
}
```
