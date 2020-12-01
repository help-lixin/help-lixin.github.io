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
### (2). 根据ID查看文档
```
curl -H "Content-Type: application/json" -X GET "http://localhost:9200/blog/article/1"
```

```
# 返回结果
{
	"_index": "blog",
	"_type": "article",
	"_id": "1",
	"_version": 1,
	"found": true,
	"_source": {
		"id": 1,
		"title": "ElasticSearch简介",
		"content": "ElasticSearch是基于Lucene基础上研发的一套全文搜索引擎!"
	}
}
```
### (3). 删除文档
```
curl -H "Content-Type: application/json" -X DELETE  "http://localhost:9200/blog/article/1"
```

```
# 返回结果
{
	"found": true,
	"_index": "blog",
	"_type": "article",
	"_id": "1",
	"_version": 2,
	"result": "deleted",
	"_shards": {
		"total": 2,
		"successful": 1,
		"failed": 0
	}
}
```

### (4). 修改文档
```
# 修改文档(指定ID)
curl -H "Content-Type: application/json" -X PUT -d @modify-document-1.json "http://localhost:9200/blog/article/1"
```

> modify-document-1.json 

```
{
    "id":1,
    "title":"ElasticSearch7.1简介",
    "content":"ElasticSearch7.1是基于Lucene基础上研发的一套全文搜索引擎!"
}
```

```
# 返回结果
{
	"_index": "blog",
	"_type": "article",
	"_id": "1",
	"_version": 2,
	"result": "updated",
	"_shards": {
		"total": 2,
		"successful": 1,
		"failed": 0
	},
	"created": false
}
```


> 只要ID相同,那么就可以对文档进行覆盖.

```
# 修改文档(POST请求,ID相同)
curl -H "Content-Type: application/json" -X POST -d @create-document-2.json "http://localhost:9200/blog/article/1"

```


> create-document-2.json

```
{
    "id":1,
    "title":"ElasticSearch-7.11简介",
    "content":"ElasticSearch-7.11是基于Lucene基础上研发的一套全文搜索引擎!"
}
```


```
{
	"_index": "blog",
	"_type": "article",
	"_id": "1",
	"_version": 3,
	"result": "updated",
	"_shards": {
		"total": 2,
		"successful": 1,
		"failed": 0
	},
	"created": false
}
```


### (5). 根据关键字查询(term)
```
curl -H "Content-Type: application/json" -X POST -d @search.json "http://localhost:9200/blog/article/_search"
```

> search.json

```
{
   "query":{
       "term":{
           "title":"简"
       }
   }
}
```

```
# 查询返回结果
{
	"took": 2,
	"timed_out": false,
	"_shards": {
		"total": 5,
		"successful": 5,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 3,
		"max_score": 0.2876821,
		"hits": [{
			"_index": "blog",
			"_type": "article",
			"_id": "1",
			"_score": 0.2876821,
			"_source": {
				"id": 1,
				"title": "ElasticSearch-7.11简介",
				"content": "ElasticSearch-7.11是基于Lucene基础上研发的一套全文搜索引擎!"
			}
		}, {
			"_index": "blog",
			"_type": "article",
			"_id": "2",
			"_score": 0.25316024,
			"_source": {
				"id": 2,
				"title": "Lucene简介",
				"content": "Lucene一套全文检索引擎!"
			}
		}, {
			"_index": "blog",
			"_type": "article",
			"_id": "3",
			"_score": 0.25316024,
			"_source": {
				"id": 3,
				"title": "Java简介",
				"content": "Java是一门开发语言"
			}
		}]
	}
}
```

### (6). 根据关键字查询(query_string)

```
curl -H "Content-Type: application/json" -X POST -d @search2.json "http://localhost:9200/blog/article/_search"
```

> search2.json

```
{
   "query":{
       "query_string":{
           "default_field":"title",
           "query":"简介"
       }
   }
}
```

```
# 返回结果 
{
	"took": 2,
	"timed_out": false,
	"_shards": {
		"total": 5,
		"successful": 5,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 3,
		"max_score": 0.5753642,
		"hits": [{
			"_index": "blog",
			"_type": "article",
			"_id": "1",
			"_score": 0.5753642,
			"_source": {
				"id": 1,
				"title": "ElasticSearch-7.11简介",
				"content": "ElasticSearch-7.11是基于Lucene基础上研发的一套全文搜索引擎!"
			}
		}, {
			"_index": "blog",
			"_type": "article",
			"_id": "2",
			"_score": 0.5063205,
			"_source": {
				"id": 2,
				"title": "Lucene简介",
				"content": "Lucene一套全文检索引擎!"
			}
		}, {
			"_index": "blog",
			"_type": "article",
			"_id": "3",
			"_score": 0.5063205,
			"_source": {
				"id": 3,
				"title": "Java简介",
				"content": "Java是一门开发语言"
			}
		}]
	}
}
```
### (7). 

### (8). 

### (9). 

### (10). 
