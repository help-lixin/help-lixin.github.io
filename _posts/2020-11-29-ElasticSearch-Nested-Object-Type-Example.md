---
layout: post
title: 'ElasticSearch 嵌套对象(十五)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 概述
> 范式设计:范式设计的最主要目标是:减少不必要的更新,相比反范式化设计还节省了磁盘空间(现在的存储空间越来越便宜了),自然,也存在缺陷:造成Join查询.  
> 反范式设计的:它把数据打平(Flattening),不使用关联关系,而是在文档中保存冗余的数据拷贝(磁盘空间消耗比较大).最主要的目的是:无需处理Join操作,数据读取性能好.缺点:不适合在数据频繁修改的场景(一条数据的改动,可能会引起很多数据的更新.)  
> 关系型数据库,一般会考虑范式化设计,而ES往往考虑的是反范式化设计数据.反范式化的好处在于:读的速度变快,无需Join,无需行锁.  
> ES并不擅长处理关联关系,一般采用四种方法处理关联:  
> 1. 对象类型   
> 2. 嵌套对象(Nested Object)
> 3. 父子关联关系(Parent/Child)   
> 4. 应用程序处理   

### (2). 对象类型(博客与作者关联关系)
```
# 删除博客
DELETE /blog

# 博客与作者信息
PUT /blog
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "time" : {
        "type" :"date"
      },
      "user": {
        "properties": {
          "city" : {
            "type" : "text"
          },
          "userid" : {
            "type" : "long"
          },
          "username" : {
            "type" : "keyword"
          }
        }
      }
    }
  }
}

# 添加文档
PUT /blog/_doc/1
{
  "content":"I like ElasticSearch",
  "time" : "2019-01-01T00:00:00",
  "user" : {
    "userid" : 1,
    "username" : "Jack",
    "city" : "Shanghai"
  }
}

# 检索文档
POST /blog/_search
{
  "query": {
    "bool": {
      "must": [
        { 
          "match": { "content": "ElasticSearch" } 
        },
        {
          "match": { "user.username": "Jack" }
        }
      ]
    }
  }
}
```
### (3). 对象类型(电影名称与演员关系)
> 对象类型存储时,JSON格式处理成扁平式键值对的结构,当对多个这段进行查询时,导致意外的搜索结果.  
> 可以使用Nested Data Type解决这个问题.   

```

DELETE my_movies

PUT /my_movies 
{
 "mappings": {
   "properties": {
     "actors" : {
       "properties": {
         "first_name" : { "type" : "keyword" },
         "last_name" : { "type" : "keyword"  }
       }
     },
     "title" : {
         "type" : "text",
         "fields": {
           "keyword" : {
               "type" : "keyword",
               "ignore_above" : 256
           }
         }
     }
   }
 } 
}

POST /my_movies/_doc/1
{
  "title" : "Speed",
  "actors" :[
      {
        "first_name" : "Keanu",
        "last_name"  : "Reeves"
      },
      {
        "first_name" : "Dennis",
        "last_name"  : "Hopper"
      }
  ]
}

#  SELECT * FROM my_movies WHERE actors.first_name = 'Keanu' AND actors.last_name = 'Hopper';
# 实际来说:按关系型的理解,是查询不到该数据的
POST /my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "actors.first_name": "Keanu"
        } },
        {
          "match": {
            "actors.last_name": "Hopper"
          }
        }
      ]
    }
  }
}

##########################检索结果##############################
# 按理来说:是不应该被检索到的,可通过(Nested解决这个问题)
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
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.723315,
    "hits" : [
      {
        "_index" : "my_movies",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.723315,
        "_source" : {
          "title" : "Speed",
          "actors" : [
            {
              "first_name" : "Keanu",
              "last_name" : "Reeves"
            },
            {
              "first_name" : "Dennis",
              "last_name" : "Hopper"
            }
          ]
        }
      }
    ]
  }
}

```
### (4). Nested Data Type
```
DELETE my_movies

# 注意,增加了:"type" : "nested", 
PUT /my_movies 
{
 "mappings": {
   "properties": {
     "actors" : {
      "type" : "nested", 
       "properties": {
         "first_name" : { "type" : "keyword" },
         "last_name" : { "type" : "keyword"  }
       }
     },
     "title" : {
         "type" : "text",
         "fields": {
           "keyword" : {
               "type" : "keyword",
               "ignore_above" : 256
           }
         }
     }
   }
 } 
}

POST /my_movies/_doc/1
{
  "title" : "Speed",
  "actors" :[
      {
        "first_name" : "Keanu",
        "last_name"  : "Reeves"
      },
      {
        "first_name" : "Dennis",
        "last_name"  : "Hopper"
      }
  ]
}




POST /my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "title": "Speed"
        }},
        {
          "nested": {
            "path": "actors",
            "query": {
              "bool": {
                "must": [
                  { "match": {
                    "actors.first_name": "Keanu"
                  } },
                  {
                    "match": {
                      "actors.last_name": "Hopper"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}

##########################检索结果##############################
# SELECT * FROM my_movies WHERE actors.first_name='Keanu' AND actors.last_name = 'Hopper';
# 这时,确实是检索不到数据了(因为:Keanu对应的lastName为:Reeves)
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}


# 测试正确检索
# SELECT * FROM my_movies WHERE actors.first_name='Keanu' AND actors.last_name = 'Reeves';
# 这时,确实是检索到数据了(因为:Keanu对应的lastName为:Reeves)
POST /my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "title": "Speed"
        }},
        {
          "nested": {
            "path": "actors",
            "query": {
              "bool": {
                "must": [
                  { "match": {
                    "actors.first_name": "Keanu"
                  } },
                  {
                    "match": {
                      "actors.last_name": "Reeves"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}

##########################检索结果##############################
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
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.6739764,
    "hits" : [
      {
        "_index" : "my_movies",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.6739764,
        "_source" : {
          "title" : "Speed",
          "actors" : [
            {
              "first_name" : "Keanu",
              "last_name" : "Reeves"
            },
            {
              "first_name" : "Dennis",
              "last_name" : "Hopper"
            }
          ]
        }
      }
    ]
  }
}
```
### (5). 总结
> 在ES中Nested数据类型是允许对象数组中的对象被独立索引.  
> 使用nested和properties关键字,将所有的actors索引到多个分隔的文档.  
> Nested文档会被保存在两个在Lucene内部,在查询时做join处理.  
> Nested文档优点:文档存储在一起,因此读取性能高,缺点:更新父或者子文档时,需要对整个文档进行更新.适合,读多写少情况.  
