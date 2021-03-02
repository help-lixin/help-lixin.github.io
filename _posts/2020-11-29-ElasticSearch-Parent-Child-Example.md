---
layout: post
title: 'ElasticSearch 父子文档(十六)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 概述
> 当使用嵌套对象时,每次更新数据时,都会触发整个文档重新建立索引,所以,在ES中提供了父子对象.   
> 父文档和子文档是两个独立的文档.  
> 更新父文档无需重新索引子文档,子文档被添加,更新,删除也不会影响父文档和其它子文档.   

### (2). 设置父子关系的mapping
> relations用来声明Parent/Child关系.     
> type:"join",指明join类型.   
> "blog" : "comment",Parent为blog,Child为comment.   

```
# 创建Mappings
PUT /my_blogs
{
  "mappings": {
    "properties": {
      "blog_comments_relation" : {
        "type": "join",
        "relations" :{
          "blog" : "comment"
        }
      },
      "conent" : {
        "type" : "text"
      },
      "title" : {
        "type" : "keyword"
      }
    }
  }
}
```
### (3). 创建父子文档
```
# 创建父文档
PUT /my_blogs/_doc/blog1
{
  "title" : "Learning ElasticSearch",
  "conent" : "learning ELK@geektime",
  "blog_comments_relation" : {
    "name" : "blog"
  }
}


# 创建父文档
PUT /my_blogs/_doc/blog2
{
  "title" : "Learning Hadoop",
  "conent" : "learning Hadoop",
  "blog_comments_relation" : {
    "name" : "blog"
  }
}


# 创建子文档
# 通过routing指定父文档ID,使得子文档和父文档保存在同一分片.
# 创建子文档时,必须指定它的:父文档ID
PUT /my_blogs/_doc/comment1?routing=blog1
{
  "comment" : "I am learning ELK",
  "username" : "Jack",
  "blog_comments_relation" : {
    "name" : "comment",
    "parent" : "blog1"
  }
}


# 创建子文档
PUT /my_blogs/_doc/comment2?routing=blog2
{
  "comment" : "I like Hadoop...",
  "username" : "Jack",
  "blog_comments_relation" : {
    "name" : "comment",
    "parent" : "blog2"
  }
}
```
### (4). 检索
```


# 检索所有父子文档
POST /my_blogs/_search
{
}


# 检索父文档
GET /my_blogs/_doc/blog2


# 根据父文档id,返回文档信息
GET /my_blogs/_search
{
  "query": {
    "parent_id" : {
      "type" : "comment",
      "id" : "blog2"
    }
  }
}

# Hash Child查询,返回父文档信息
GET /my_blogs/_search
{
  "query": {
    "has_child": {
      "type": "comment",
      "query": {
        "match": {
          "username": "Jack"
        }
      }
    }
  }
}

# 根据父文档进行检索,返回子文档信息
GET /my_blogs/_search
{
  "query": {
    "has_parent": {
      "parent_type": "blog",
      "query": {
        "match": {
          "title": "Learning Hadoop"
        }
      }
    }
  }
}



# 检索子文档(需要提父ID的路由)
GET /my_blogs/_doc/comment1?routing=blog1
```
### (5). 更新子文档
```
# 更新子文档
PUT /my_blogs/_doc/comment1?routing=blog1
{
  "comment" : "I am learning ELK--1",
  "username" : "Jack",
  "blog_comments_relation" : {
    "name" : "comment",
    "parent" : "blog1"
  }
}
```
### (6). 总结
> 嵌套对象优缺点:   
> 优点:嵌套对象文档存储在一起,读取性能高,缺点:更新嵌套的子文档时,需要更新整个文档.比较适合:子文档偶尔更新,并以以查询为主.   
> 父子文档优缺点:   
> 优点:父子文档可以独立更新,缺点:需要额外的内存维护关系,读取性能相对差一点,适用场景:子文档更新频繁.   