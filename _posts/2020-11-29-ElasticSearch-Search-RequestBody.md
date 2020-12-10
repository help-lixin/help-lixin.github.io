---
layout: post
title: 'ElasticSearch Search(Request Body)(十一)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). Reqeust Body Search(DSL)
> Reqeust Body Search是ES提供的DSL语言.

### (2). 准备工作
```
# 创建索引库
PUT /user

# 查看有多少索引库
GET _cat/indices

# 批量添加文档
POST _bulk
{"index" : { "_index":"user","_id":1001 }}
{"id" : 1001, "name":"张三" , "age":20 ,"sex":"男" }
{"index" : { "_index":"user","_id":1002 }}
{"id" : 1002, "name":"李四" , "age":21 ,"sex":"女" }
{"index" : { "_index":"user","_id":1003 }}
{"id" : 1003, "name":"王五" , "age":31 ,"sex":"男" }
{"index" : { "_index":"user","_id":1004 }}
{"id" : 1004, "name":"赵六" , "age":32 ,"sex":"女" }
{"index" : { "_index":"user","_id":1005 }}
{"id" : 1005, "name":"孙七" , "age":33 ,"sex":"男" }
```

### (3). DSL(ALL)

```
# SELECT * FROM user WHERE 1=1;
# 检索user库所有的数据,仅显示10条
GET /user/_search
{
  "query":{
    "match_all": {}
  }
}
```
### (4). DSL(BooleanQuery)
> 组合查询.   

```
# 检索出年龄大于30岁,并且性别为男
# SELECT * FROM user WHERE age > 30 AND sex = "男";
GET /user/_search
{
  "query":{
    "bool": {
      
      "filter": {
        "range": {
          "age": {
            "gt": 30
          }
        }
      },
      "must":{ 
          "match":{
            "sex":"男"
          }  
      }
      
    }
  }
}
```
### (5). DSL(MATCH)
> Match会对待查文本进行分词,然后进行Query. 

```
# SELECT * FROM user WHERE name = '张三' OR name = "李四";
GET /user/_search
{
  "query":{
    "match": {
      "name": "张三  李四"
    }
  }
}
```
### (6). 
  
### (7). 

### (8). 



