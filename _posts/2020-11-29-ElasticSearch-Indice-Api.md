---
layout: post
title: 'ElasticSearch Indice和Type管理(四)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). ElasticSearch索引库管理官网文档

> https://www.elastic.co/guide/cn/elasticsearch/guide/current/index-management.html

### (2). 添加索引库(blog),并配置Type

```
# 添加一个博客(blog)索引库,并指定Type(article)
curl -H "Content-Type: application/json" -X PUT -d @create-index.json  "http://localhost:9200/blog"

# 返回结果
{"acknowledged":true,"shards_acknowledged":true,"index":"blog"}
```

> create-index.json 

```
{
    "mappings": {
        "article": {
            "properties":{
                "id":{
                    "type":"long",
                    "store":true
                },
                "title":{
                    "type":"text",
                    "store":true,
                    "index":true,
                    "analyzer":"standard"
                },
                "content":{
                    "type":"text",
                    "store":true,
                    "index":true,
                    "analyzer":"standard"
                }
            }
        }
    }
}
```


### (3). 更新索引库(blog),添加Type

```
# 给指定的索引库(blog),添加Type(article2),并设置mappings
curl -H "Content-Type: application/json" -X POST -d @update-index.json  "http://localhost:9200/blog/article2/_mappings"

# 返回结果
{"acknowledged":true}

```

> update-index.json 

```
{
    "article2": {
        "properties": {
            "content": {
                "type": "text",
                "store": true,
                "analyzer": "standard"
            },
            "id": {
                "type": "long",
                "store": true
            },
            "title": {
                "type": "text",
                "store": true,
                "analyzer": "standard"
            }
        }
    }
}

```

### (4). 获取索引库(blog)信息
```
curl -H "Content-Type: application/json" -X GET "http://localhost:9200/blog"
```

```
{
    "blog": {
        "aliases": {},
        "mappings": {
            "article": {
                "properties": {
                    "content": {
                        "type": "text",
                        "store": true,
                        "analyzer": "standard"
                    },
                    "id": {
                        "type": "long",
                        "store": true
                    },
                    "title": {
                        "type": "text",
                        "store": true,
                        "analyzer": "standard"
                    }
                }
            }
        },
        "settings": {
            "index": {
                "creation_date": "1606738660470",
                "number_of_shards": "5",
                "number_of_replicas": "1",
                "uuid": "IZ5t_e8yQx-7BE1zShIQkA",
                "version": {
                    "created": "5061799"
                },
                "provided_name": "blog"
            }
        }
    }
}
```

### (5). 删除索引库
```
# 删除索引库
curl -H "Content-Type: application/json" -X DELETE "http://localhost:9200/blog"

# 返回结果
{"acknowledged":true}
```
