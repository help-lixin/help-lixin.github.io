---
layout: post
title: 'ElasticSearch 父子文档[订单/订单详细](十七)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 创建Mappings
```
DELETE /order


PUT /order
{   
  "mappings" : {
  		"properties" : {
  			"order" : {
  				"properties" : {
  					"order_id"     : { "type" : "integer" },
  					"order_status" : { "type" : "integer" },
  					"merchant_id"  : { "type" : "integer" },
  					"user_id"      : { "type" : "integer" }
  				}
  			},
  			"order_detail" : {
  				"properties" : {
  					"order_detail_id"   : { "type" : "integer" },
  					"order_id"          : { "type" : "integer" },
  					"goods_id"          : { "type" : "integer" },
  					"order_detail_name" : { "type" : "text" }
  				}
  			},
  			"relation" : {
  				"type": "join",
  				"relations" : {
  					"order": [ "order_detail" ]
  				}
  			}	
  		}
	}
}
```
### (2). 添加父子文档
> 注意:父子文档的ID不能相同,否则,会覆盖文档的.

```

# 添加父文档
PUT /order/_doc/1
{
  "order_id": 1,
  "order_status" : 0,
  "merchant_id" : 8888,
  "user_id" : 6666,
  "relation":{
    "name" : "order"
  }
}

PUT /order/_doc/2
{
  "order_id": 2,
  "order_status" : 1,
  "merchant_id" : 9999,
  "user_id" : 6666,
  "relation":{
    "name" : "order"
  }
}

# 添加子文档
PUT /order/_doc/11?routing=1
{
  "order_detail_id" : 11,
  "order_id" : 1,
  "goods_id" : 11111,
  "order_detail_name" : "Hadoop Book",
  "relation":{
    "name" : "order_detail",
    "parent": 1
  }
}


PUT /order/_doc/22?routing=2
{
  "order_detail_id" : 22,
  "order_id" : 2,
  "goods_id" : 22222,
  "order_detail_name" : "HBase Book",
  "relation":{
    "name" : "order_detail",
    "parent": 2
  }
}
```
### (3). 检索文档
```
# 检索所有的父子文档
POST /order/_search
{
}


# 根据id查找文档
# id只要唯一就好了.
GET /order/_doc/1
GET /order/_doc/2

GET /order/_doc/11
GET /order/_doc/22

# 根据parent_id检索,返回子文档信息
GET /order/_search
{
  "query": {
    "parent_id" : {
      "type" : "order_detail",
      "id" : 1
    }
  }
}


# 检索子文档,返回父文档信息
GET /order/_search
{
  "query": {
    "has_child": {
      "type": "order_detail",
      "query": {
        "match": {
          "goods_id": 11111
        }
      }
    }
  }
}


# 根据父文档进行检索,返回子文档信息
GET /order/_search
{
  "query": {
    "has_parent": {
      "parent_type": "order",
      "query": {
        "match": {
          "order_status": 1
        }
      }
    }
  }
}
```
