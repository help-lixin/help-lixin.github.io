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


# 添加文档
PUT user/_doc/1006
{"id" : 1006, "name":"小何" , "age":33 ,"sex":"男" }
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
> <font color='red'>Match会对待查文本进行分词,然后进行Query.</font>  

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
### (6). 高亮显示
> 指定高亮的字段为哪些.

```
# SELECT * FROM user WHERE name = '张三' OR name = "李四";
GET /user/_search
{
  "query":{
    "match": {
      "name": "张三 李四"
    }
  },
  "highlight": {
    "fields": {
      "name":{}
    }
  }
}


# 查询返回结果.
{
  "took" : 54,
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
    "max_score" : 2.7725887,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1001",
        "_score" : 2.7725887,
        "_source" : {
          "id" : 1001,
          "name" : "张三",
          "age" : 20,
          "sex" : "男"
        },
        "highlight" : {
          "name" : [
            "<em>张</em><em>三</em>"
          ]
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1002",
        "_score" : 2.7725887,
        "_source" : {
          "id" : 1002,
          "name" : "李四",
          "age" : 21,
          "sex" : "女"
        },
        "highlight" : {
          "name" : [
            "<em>李</em><em>四</em>"
          ]
        }
      }
    ]
  }
}
```
### (7). 聚合(group by )
```
# 聚合
# SELECT age key,COUNT(age) document_count FROM user GROUP BY age;
GET /user/_search
{
  "aggs": {
    "all_interests":{
      "terms": {
        "field": "age"
      }
    }
  }
}

# 聚合结果
{
  "took" : 265,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 6,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1001",
        "_score" : 1.0,
        "_source" : {
          "id" : 1001,
          "name" : "张三",
          "age" : 20,
          "sex" : "男"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1002",
        "_score" : 1.0,
        "_source" : {
          "id" : 1002,
          "name" : "李四",
          "age" : 21,
          "sex" : "女"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 1.0,
        "_source" : {
          "id" : 1003,
          "name" : "王五",
          "age" : 31,
          "sex" : "男"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1004",
        "_score" : 1.0,
        "_source" : {
          "id" : 1004,
          "name" : "赵六",
          "age" : 32,
          "sex" : "女"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1005",
        "_score" : 1.0,
        "_source" : {
          "id" : 1005,
          "name" : "孙七",
          "age" : 33,
          "sex" : "男"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1006",
        "_score" : 1.0,
        "_source" : {
          "id" : 1006,
          "name" : "小何",
          "age" : 33,
          "sex" : "男"
        }
      }
    ]
  },
  "aggregations" : {
    "all_interests" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 33,
          "doc_count" : 2
        },
        {
          "key" : 20,
          "doc_count" : 1
        },
        {
          "key" : 21,
          "doc_count" : 1
        },
        {
          "key" : 31,
          "doc_count" : 1
        },
        {
          "key" : 32,
          "doc_count" : 1
        }
      ]
    }
  }
}
```
### (8). pretty美化结果
> 通过pretty参数美化返回结果.

```
http://localhost:9200/user/_search?pretty
```
### (9). _source指定返回的字段
> 通过_source参数,指定返回的结果

```
POST /user/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["id","name"]
}
```
### (10). 判断Document是否存在
```
HEAD /user/_doc/1001

# HEAD结果:
200 - OK
```
