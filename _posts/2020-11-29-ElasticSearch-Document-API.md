---
layout: post
title: 'ElasticSearch Document API(八)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 文档的CRUD

|  操作   | 案例  | 说明  |
|  ----  | ----  |
| Index  | <font color='red'>PUT movies/_doc/99999</font> { "id":99999,"year":1991} | ID存在先删除,后添加,ID不存在,创建新的文档. |
| Create  | <font color='red'>POST movies/_doc</font> { "id":99998,"year":1990,"title":"Hello World-999998""genre":["Lixin","XinLi"]}  <br/> <font color='red'>PUT movies/_doc/99999?op_type=create</font> {"id":99999,"year":1990,"title":"Hello World","genre":["Lixin","XinLi"]}  | ID存在创建失败 |
| Read  | GET movies/_doc/99999 |  获取文档  |
| Update  | <font color='red'>POST movies/_update/99999</font> { "doc": { "title":"Hello World","genre":["Lixin""XinLi"] }} |  文档ID必须存在,仅只会对相应的字段进行更新  |
| Delete  | DELETE movies/_doc/99999 |  删除文档  |


### (2). Create创建文档.
> 添加文档不指定ID

```
POST movies/_doc
{
  "id":99998,
  "year":1990,
  "title":"Hello World-999998",
  "genre":[
     "Lixin",
     "XinLi"
   ]
}

# 执行结果(ID自动生成了) 
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "tgNwRnYB_nVREyEiPetQ",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 9768,
  "_primary_term" : 1
}

****************************************************************************************

# 添加文档,并指定ID
POST movies/_doc/99999
{
  "id":99999,
  "year":1992,
  "title":"Hello World",
  "genre":[
     "Lixin",
     "XinLi"
   ]
}

```


> 使用Create方式添加文档:<font color='red'>如果文档ID不存在,则添加,如果文档ID存在,则抛出异常.<font>    

```
# (create)添加文档,如果文档id不存在,则添加,如果文档id存在,则抛出异常.
PUT movies/_doc/99999?op_type=create
{
  "id":99999,
  "year":1990,
  "title":"Hello World",
  "genre":[
     "Lixin",
     "XinLi"
   ]
}

# 执行后结果:
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 9767,
  "_primary_term" : 1
}
```

### (3). 查看文档ID的信息
```
# 查看文档
GET movies/_doc/99999

# 执行结果
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 12,
  "_seq_no" : 9767,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "id" : 99999,
    "year" : 1990,
    "title" : "Hello World",
    "genre" : [
      "Lixin",
      "XinLi"
    ]
  }
}
```

### (4). Index创建文档(先删,后添加)
> <font color='red'>如果文档存在,则先删除,后再添加.</font>

```
# Index(如果文档存在,则先删除,后再添加)
PUT movies/_doc/99999
{
  "id":99999,
  "year":1991
}

# 执行结果
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 13,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 9769,
  "_primary_term" : 1
}

****************************************************************************************

# 查看文档
GET movies/_doc/99999

# 查看文档结果(Document Field只有两个[id/year])
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 14,
  "_seq_no" : 9770,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "id" : 99999,
    "year" : 1991
  }
}

```
### (5). 更新文档
```
# 局部更新文档,为文档添加:[title/genre]字段
POST movies/_update/99999
{ "doc":
  {
    "title":"Hello World",
    "genre":[
       "Lixin",
       "XinLi"
     ]
  }
}

# 局部更新文档结果
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 15,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 9771,
  "_primary_term" : 1
}

****************************************************************************************

# 查看文档
GET movies/_doc/99999

# 查看文档结果(文档增加了:title/genre字段)
{
  "_index" : "movies",
  "_type" : "_doc",
  "_id" : "99999",
  "_version" : 15,
  "_seq_no" : 9771,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "id" : 99999,
    "year" : 1991,
    "genre" : [
      "Lixin",
      "XinLi"
    ],
    "title" : "Hello World"
  }
}
```
### (6). 删除文档
```
# 删除文档
DELETE movies/_doc/99999
```