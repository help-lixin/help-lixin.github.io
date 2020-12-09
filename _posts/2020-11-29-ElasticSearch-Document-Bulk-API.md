---
layout: post
title: 'ElasticSearch Document Bulk API(八)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). Bulk产生原因
> 由于ES是通过http来提供请求的,而http连接的创建和销毁都是损耗性能的,所以,ES Bulk API支持批处理. 

### (2). _bulk语法要求
> bulk对JSON串有很严格的要求,每个JSON串不能换行,只能放在同一行,同时,相邻的JSON串之间必须要有换行.

```
{ action: { metadata }}
{ request body        }
```
> action支持的类型:create/index/update/delete. 

### (3). _bulk批量添加
```
# 批量添加
POST _bulk
{"index" : { "_index":"movies","_id":99998 }}
{"id" : "99998", "year" : 1995, "title" : "Toy Story-1995", "genre" : [ "Adventure" ] }
{"index" : { "_index":"movies","_id":99997 }}
{"id" : "99997", "year" : 1995, "title" : "Toy Story-1997", "genre" : [ "Adventure" ] }
{"index" : { "_index":"movies","_id":99996 }}
{"id" : "99996", "year" : 1995, "title" : "Toy Story-1996", "genre" : [ "Adventure" ] }

# 批量添加返回结果集
{
  "took" : 30,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99998",
        "_version" : 3,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9791,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99997",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9792,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99996",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9793,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}

```

### (4). _bulk批量更新
```
# 批量更新(注意:update需要带上doc:{...})
POST _bulk
{"update" : { "_index":"movies","_id":99998 }}
{ "doc": {"id" : "99998", "year" : 1995, "title" : "Toy Story", "genre" : [ "Adventure" ] }}
{"update" : { "_index":"movies","_id":99997 }}
{ "doc": { "id" : "99997", "year" : 1995, "title" : "Toy Story", "genre" : [ "Adventure" ] }}
{"update" : { "_index":"movies","_id":99996 }}
{ "doc": { "id" : "99996", "year" : 1995, "title" : "Toy Story", "genre" : [ "Adventure" ] }}

# 批量更新结果
{
  "took" : 0,
  "errors" : false,
  "items" : [
    {
      "update" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99998",
        "_version" : 4,
        "result" : "noop",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "status" : 200
      }
    },
    {
      "update" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99997",
        "_version" : 2,
        "result" : "noop",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "status" : 200
      }
    },
    {
      "update" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99996",
        "_version" : 2,
        "result" : "noop",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "status" : 200
      }
    }
  ]
}
```

### (5). _bulk批量删除
```
# 批量删除
POST _bulk
{"delete" : { "_index":"movies","_id":99999 }}
{"delete" : { "_index":"movies","_id":99998 }}
{"delete" : { "_index":"movies","_id":99997 }}
{"delete" : { "_index":"movies","_id":99996 }}

# 批量删除结果
{
  "took" : 38,
  "errors" : false,
  "items" : [
    {
      "delete" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99999",
        "_version" : 1,
        "result" : "not_found",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9797,
        "_primary_term" : 1,
        "status" : 404
      }
    },
    {
      "delete" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99998",
        "_version" : 5,
        "result" : "deleted",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9798,
        "_primary_term" : 1,
        "status" : 200
      }
    },
    {
      "delete" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99997",
        "_version" : 3,
        "result" : "deleted",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9799,
        "_primary_term" : 1,
        "status" : 200
      }
    },
    {
      "delete" : {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "99996",
        "_version" : 3,
        "result" : "deleted",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 9800,
        "_primary_term" : 1,
        "status" : 200
      }
    }
  ]
}
```
### (6). _mget批量获取
```
# 批量获取
GET _mget
{
  "docs":[
    { "_index":"movies" , "_id": 99999},
    { "_index":"movies" , "_id": 1}
  ]
}

# 批量获取结果
{
  "docs" : [
    {
      "_index" : "movies",
      "_type" : "_doc",
      "_id" : "99999",
      "_version" : 16,
      "_seq_no" : 9772,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "id" : 99999,
        "year" : 1992,
        "title" : "Hello World",
        "genre" : [
          "Lixin",
          "XinLi"
        ]
      }
    },
    {
      "_index" : "movies",
      "_type" : "_doc",
      "_id" : "1",
      "_version" : 1,
      "_seq_no" : 16,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "id" : "1",
        "year" : 1995,
        "@version" : "1",
        "title" : "Toy Story",
        "genre" : [
          "Adventure",
          "Animation",
          "Children",
          "Comedy",
          "Fantasy"
        ]
      }
    }
  ]
}
```
### (6). _msearch批量搜索
```
# 批量搜索
GET _msearch
{"index":"movies"}
{"query":{"match_all":{}},"size":1}

# 批量搜索结果
{
  "took" : 1,
  "responses" : [
    {
      "took" : 1,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 9744,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "movies",
            "_type" : "_doc",
            "_id" : "2024",
            "_score" : 1.0,
            "_source" : {
              "id" : "2024",
              "year" : 1991,
              "@version" : "1",
              "title" : "Rapture, The",
              "genre" : [
                "Drama",
                "Mystery"
              ]
            }
          }
        ]
      },
      "status" : 200
    }
  ]
}
```