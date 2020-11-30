---
layout: post
title: 'ElasticSearch Document管理(五)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 添加document

```
# 添加一个文档
curl -H "Content-Type: application/json" -X POST -d @create-document-1.json "http://localhost:9200/blog/article/1"

# 返回结果
{
	"_index": "blog",
	"_type": "article",
	"_id": "1",
	"_version": 1,
	"result": "created",
	"_shards": {
		"total": 2,
		"successful": 1,
		"failed": 0
	},
	"created": true
}
```

> create-document-1.json 

```
{
    "id":1,
    "title":"ElasticSearch简介",
    "content":"ElasticSearch是基于Lucene基础上研发的一套全文搜索引擎!"
}
```
### (2). 

### (3). 

### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
