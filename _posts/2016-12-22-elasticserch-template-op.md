---
layout: post
title:  "Elasticsearch的分词与模板的操作"
date:   2016-12-22 19:19:00
categories: elastic
tags: elasticsearch
---

* content
{:toc}

日志系统基于elasticsearch实现日志搜索，实际应用中大部分开发人员只关注日志中异常，例如搜索关键词"exception"，
默认分词系统或者ik分词器都是直接先做小写处理，新版本es5.*不再支持配置文件中配置分词，建议动态配置,以下驼峰效果分词器配置
及模板操作相关API





## 模板

### 添加一个模板

> 模板中只对message字段分词，其他string类型字段都不分词，logid作为long类型，分词规则加入驼峰分词

```
PUT /_template/ule-business
{
  "order": 0,
  "template": "ule-business-*",
  "settings": {
    "index": {
      "refresh_interval": "3s",
      "analysis": {
        "filter": {
          "camel_filter": {
            "type": "word_delimiter",
            "split_on_numerics": "false",
            "stem_english_possessive": "false",
            "generate_number_parts": "false",
            "protected_words": [
              "iPhone",
              "WiFi"
            ]
          }
        },
        "tokenizer": {
          "my_tokenizer": {
            "type": "ik_smart",
            "enable_lowercase": "false"
          }
        },
        "analyzer": {
          "camel_analyzer": {
            "type": "custom",
            "char_filter": [
              "html_strip"
            ],
            "tokenizer": "my_tokenizer",
            "filter": [
              "camel_filter",
              "lowercase"
            ]
          }
        }
      },
      "number_of_shards": "1",
      "number_of_replicas": "0"
    }
  },
  "mappings": {
    "_default_": {
      "dynamic_templates": [
        {
          "message_field": {
            "mapping": {
              "analyzer": "camel_analyzer",
              "index": "analyzed",
              "omit_norms": true,
              "type": "string"
            },
            "match_mapping_type": "string",
            "match": "message"
          }
        },
        {
          "logid_field": {
            "mapping": {
              "type": "long"
            },
            "match_mapping_type": "string",
            "match": "logid"
          }
        },
        {
          "string_fields": {
            "mapping": {
              "index": "not_analyzed",
              "omit_norms": true,
              "type": "string",
              "fields": {
                "raw": {
                  "ignore_above": 256,
                  "index": "not_analyzed",
                  "type": "string"
                }
              }
            },
            "match_mapping_type": "string",
            "match": "*"
          }
        }
      ],
      "_all": {
        "omit_norms": true,
        "enabled": true
      },
      "properties": {
        "geoip": {
          "dynamic": true,
          "type": "object",
          "properties": {
            "location": {
              "type": "geo_point"
            }
          }
        },
        "@version": {
          "index": "not_analyzed",
          "type": "string"
        }
      }
    }
  },
  "aliases": {}
}
```

### 获取模板

#### 获取所有

`GET /_template`

#### 获取指定模板

`GET /_template/ule-business`

### 删除指定模板

`DELETE /_template/ule-business`


## 分词

### 分词测试

```
POST ule-business-test/_analyze
{
  "analyzer": "camel_analyzer",
  "text":     "中华共和国NullPointException"
}
```

### 效果

```
{
  "tokens": [
    {
      "token": "中华",
      "start_offset": 0,
      "end_offset": 2,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "共和国",
      "start_offset": 2,
      "end_offset": 5,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "null",
      "start_offset": 5,
      "end_offset": 9,
      "type": "ENGLISH",
      "position": 2
    },
    {
      "token": "point",
      "start_offset": 9,
      "end_offset": 14,
      "type": "ENGLISH",
      "position": 3
    },
    {
      "token": "exception",
      "start_offset": 14,
      "end_offset": 23,
      "type": "ENGLISH",
      "position": 4
    }
  ]
}
```

## 参考资料

* [Elasticsearch Analyzer 的内部机制](http://mednoter.com/all-about-analyzer-part-one.html)
* [Elasticsearch-analysis-custom-analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/analysis-custom-analyzer.html)
* [Elasticsearch-indices-templates](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/indices-templates.html)