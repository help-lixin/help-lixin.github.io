---
layout: post
title: 'ElasticSearch Mapping(十二)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). Mapping
> ES在创建索引时,如果没有指定Mapping,那么在我们插入数据时,会自动进行类型的断判,但是,有时候我们需要进行明确的字段类型.自动判断类型的规则如下:

|  JSON Type                      | Field Type  |
|  ----                           | ----        |
| Boolean:true or false           | boolean     |
| Number:123                      | long        |
| Float:123.45                    | double      |
| String,valid date:"2020-12-12"  | date        |
| String:"foo bar"                | String      |

### (2). ES支持的类型

|  类型                            | 表示的数据类型              |
|  ----                            | ----                     | 
| String                           | string,text,keyworld     |
| Number                           | byte,short,integer,long  |
| Float                            | float,double             |
| Boolean                          | boolean                  |
| Date                             | date                     |

> string类型在ES旧版本中使用较多,从ES5.X开始不再支持,由text和keyword类型替代.   
> <font color='red'>text类型:当一个字段要被全文搜索的</font>,比如:商品描述,应该使用text类型,当设置为:text类型以后,字段内容会被分词器进行分析,并解析成一个一个的词项,并生成倒排索引.该类型的字段不用于排序,很少用于聚合.    
> <font color='red'>keyword类型:该类型适用于索引结构化的字段</font>,比如:emial地址,商品状态(status),商品类别ID(category_id),标签.如果字段只需要进行过滤,排序,聚合,则应该定义为:keyword,该类型的字段只能通过精确值搜索到.

### (3). 创建索引并指定Mapping
```
PUT /blog
{
  "settings": {
    "index":{
      "number_of_shards":"1",
      "number_of_replicas":"0"
    }
  },
  "mappings": {
    "properties":{
      "name": { "type": "text"  },
      "age" : { "type": "integer" },
      "email" : { "type" : "keyword"  },
      "hobby" : { "type" : "text" }
    }
  }
}

# 创建索引返回结果: 
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "blog"
}
```

### (4). 修改索引mappings信息
```
PUT /blog/_mappings
{
  "properties":{
    "sex":{ "type" : "keyword" }    
  }
}
```
### (5). 查看索引信息
```
# 查看索引信息
GET /blog

# 查看索引信息结果
{
  "blog" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "integer"
        },
        "email" : {
          "type" : "keyword"
        },
        "hobby" : {
          "type" : "text"
        },
        "name" : {
          "type" : "text"
        },
        "sex" : {
          "type" : "keyword"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1607754571299",
        "number_of_shards" : "1",
        "number_of_replicas" : "0",
        "uuid" : "nvOsX3qQSlm5OUeGNsj9Cw",
        "version" : {
          "created" : "7010099"
        },
        "provided_name" : "blog"
      }
    }
  }
}
```
