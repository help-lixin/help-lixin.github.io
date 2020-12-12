---
layout: post
title: 'ElasticSearch Search(Request Body)(十三)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (0). Mapping信息请参考:


### (1). 批量插入测试数据(_bulk)
```
# 批量插入数据

POST _bulk
{"index":{"_index":"blog","_id":1001}}
{"id":"1001","name":"张三","age":20,"email":"1234@qq.com","sex":"男","hobby":"羽毛球,乒乓球,足球"}
{"index":{"_index":"blog","_id":1002}}
{"id":"1002","name":"李四","age":21,"email":"12345@qq.com","sex":"女","hobby":"羽毛球,乒乓球,足球,蓝球"}
{"index":{"_index":"blog","_id":1003}}
{"id":"1003","name":"王五","age":22,"email":"123456@qq.com","sex":"男","hobby":"羽毛球,蓝球,游泳,听音乐"}
{"index":{"_index":"blog","_id":1004}}
{"id":"1004","name":"赵六","age":23,"email":"1234567@qq.com","sex":"女","hobby":"跑步,游泳"}
{"index":{"_index":"blog","_id":1005}}
{"id":"1005","name":"孙七","age":24,"email":"12345678@qq.com","sex":"男","hobby":"听音乐,看电影"}

```

### (2). 检索(match)
ES在保存数据时有如下几步操作:   
> 1. 获取文档Field(hobby)配置的分词器(Standard).   
> 2. 对Document Field进行分词(Standard分词器,英文按空格划分,中文按单字划分).    
> 3. 把分词后的内容,建立倒排索引,同时,把原始Document进行保存.   

ES match数据时有如下几步操作:   
> 1. match会获取待检索Field(hobby)的分词器(Standard分词器,英文按空格划分,中文按单字划分.)     
> 2. 根据分词器(Standard),对检索内容("音乐")进行分词.      
> 3. 根据分词后的内容("音","乐"),让ES到各分片的:"倒排搜索中进行检索".    
> 4. 在倒排索引中,有相应的documentID,获取:documentID,对应的:document

总结:   
<font color='red'>math会对内容进行分词.</font>

```
# 搜索爱好:包含音乐的
POST /blog/_search
{
  "query": {
    "match": {
      "hobby": "音乐"
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }
}

# 搜索爱好包含音乐的,结果集
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.9159472,
    "hits" : [
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "1005",
        "_score" : 1.9159472,
        "_source" : {
          "id" : "1005",
          "name" : "孙七",
          "age" : 24,
          "email" : "12345678@qq.com",
          "sex" : "男",
          "hobby" : "听音乐,看电影"
        },
        "highlight" : {
          "hobby" : [
            "听<em>音</em><em>乐</em>,看电影"
          ]
        }
      },
      {
        "_index" : "blog",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 1.5506183,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "123456@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,蓝球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,蓝球,游泳,听<em>音</em><em>乐</em>"
          ]
        }
      }
    ]
  }
}
```
### (3). 

### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 

### (11). 

### (12). 

### (13). 
