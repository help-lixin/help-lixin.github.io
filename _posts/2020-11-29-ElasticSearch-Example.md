---
layout: post
title: 'ElasticSearch IK中文分词结合案例(十四)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 添加索引(指定IK分词器)
```
PUT /hobby
{
  "settings": {
    "index":{
      "number_of_shards":"1",
      "number_of_replicas":"0"
    }
  },
  "mappings": {
    "properties":{
      "name": { "type": "text"  },
      "age" : { "type": "integer" },
      "email" : { "type" : "keyword"  },
      "hobby" : { "type" : "text" , "analyzer" : "ik_max_word" }
    }
  }
}
```
### (2). 动态修改索引
```
PUT /hobby/_mappings
{
  "properties":{
    "sex":{ "type" : "keyword" }    
  }
}
```
### (3). 查看索引信息
```
GET /hobby

{
  "hobby" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "integer"
        },
        "email" : {
          "type" : "keyword"
        },
        "hobby" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        },
        "name" : {
          "type" : "text"
        },
        "sex" : {
          "type" : "keyword"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1607834235715",
        "number_of_shards" : "1",
        "number_of_replicas" : "0",
        "uuid" : "N4_W0P5cSK6QcbZlAHCTOA",
        "version" : {
          "created" : "7010099"
        },
        "provided_name" : "hobby"
      }
    }
  }
}
```
### (4). 批量添加数据
```
POST _bulk
{"index":{"_index":"hobby","_id":1001}}
{"id":"1001","name":"张三","age":20,"email":"1111@qq.com","sex":"男","hobby":"羽毛球,乒乓球,足球"}
{"index":{"_index":"hobby","_id":1002}}
{"id":"1002","name":"李四","age":21,"email":"2222@qq.com","sex":"女","hobby":"羽毛球,乒乓球,足球,篮球"}
{"index":{"_index":"hobby","_id":1003}}
{"id":"1003","name":"王五","age":22,"email":"3333@qq.com","sex":"男","hobby":"羽毛球,篮球,游泳,听音乐"}
{"index":{"_index":"hobby","_id":1004}}
{"id":"1004","name":"赵六","age":23,"email":"4444@qq.com","sex":"女","hobby":"跑步,游泳"}
{"index":{"_index":"hobby","_id":1005}}
{"id":"1005","name":"孙七","age":24,"email":"5555@qq.com","sex":"男","hobby":"听音乐,看电影"}

# 添加数据
PUT /hobby/_doc/1006
{"id":"1006","name":"小玲","age":25,"email":"6666@qq.com","sex":"女","hobby":"乐器,看电影"}

```
### (5). math检索
> 从math检索结果能看出,"音乐"被IK分词器,分成了一个词汇.

```
# 检索兴趣爱好包含有:音乐
# 因为分词器不同,所以检索不同
# SELECT * FROM hobby WHERE hobby LIKE '%音乐%';
POST hobby/_search
{
  "query": {
    "match": {
      "hobby": "音乐"
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}


# 检索兴趣爱好包含有:音乐,返回结果.
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0693902,
    "hits" : [
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1005",
        "_score" : 1.0693902,
        "_source" : {
          "id" : "1005",
          "name" : "孙七",
          "age" : 24,
          "email" : "5555@qq.com",
          "sex" : "男",
          "hobby" : "听音乐,看电影"
        },
        "highlight" : {
          "hobby" : [
            "听<em>音乐</em>,看电影"
          ]
        }
      },
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 0.8681809,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "3333@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,篮球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,篮球,游泳,听<em>音乐</em>"
          ]
        }
      }
    ]
  }
}

```
### (6). 查看蓝球对应的分词结果
```
POST _analyze
{
  "analyzer": "ik_max_word",
  "text":"篮球"
}

# 查看蓝球分词后结果(篮球)
{
  "tokens" : [
    {
      "token" : "篮球",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    }
  ]
}

```
### (7). 多词搜索(OR)
```
# 查询出爱好包含有:音乐 或 蓝球的
# SELECT * FROM hobby WHERE hobby LIKE '%音乐%' OR hobby LIKE '%篮球%';
POST hobby/_search
{
  "query": {
    "match": {
      "hobby": "音乐 篮球"
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}

# 查询出爱好包含有:音乐 或 篮球的,返回结果
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.7363617,
    "hits" : [
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 1.7363617,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "3333@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,篮球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
          ]
        }
      },
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1005",
        "_score" : 1.0693902,
        "_source" : {
          "id" : "1005",
          "name" : "孙七",
          "age" : 24,
          "email" : "5555@qq.com",
          "sex" : "男",
          "hobby" : "听音乐,看电影"
        },
        "highlight" : {
          "hobby" : [
            "听<em>音乐</em>,看电影"
          ]
        }
      },
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1002",
        "_score" : 0.8681809,
        "_source" : {
          "id" : "1002",
          "name" : "李四",
          "age" : 21,
          "email" : "2222@qq.com",
          "sex" : "女",
          "hobby" : "羽毛球,乒乓球,足球,篮球"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,乒乓球,足球,<em>篮球</em>"
          ]
        }
      }
    ]
  }
}
```
### (8). 多词搜索(operator)
> <font color='red'>空格划分的词汇之间,默认是:OR,可通过增加:operator.</font>    

```
# 查询出爱好包含有:音乐 并且 还要满足包含有:篮球的.
# SELECT * FROM hobby WHERE hobby LIKE '%音乐%' AND hobby LIKE '%篮球%'
POST hobby/_search
{
  "query": {
    "match": {
      "hobby": {
        "operator": "and",
        "query": "音乐 篮球"
      }
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}

# 查询出爱好包含有:音乐 并且 还要满足包含有:篮球的.返回结果
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
    "max_score" : 1.7363617,
    "hits" : [
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 1.7363617,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "3333@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,篮球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
          ]
        }
      }
    ]
  }
}
```
### (9). 按相似度查询
> 相似度(minimum_should_match)降低,查询返回的数据就越多.   
> 相似度(minimum_should_match)增加,查询返回的数据就越少越精准.   

```
POST hobby/_search
{
  "query": {
    "match": {
      "hobby": {
        "query": "音乐 篮球",
        "minimum_should_match": "80%"
      }
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}
```
### (10). 组合搜索
> minimum_should_match:多个条件中,最少要满足多少个? 

```
# 查询爱好包含有:篮球,但是,不能包含有:音乐,如果包含了游泳,那么它的相似度更高.
# 此处:should相当于一个得分的计算.
# SELECT * FROM hobby WHERE (hobby LIKE '%篮球%' OR hobby LIKE '%游泳%') AND hobby NOT LIKE '%音乐%';

POST hobby/_search
{
  "query": {
    "bool": {
      
      "must": [
        {
          "match": {
            "hobby": "篮球"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "hobby": "音乐"
          }
        }
      ],
      "should": [
        {
          "match": {
            "hobby": "游泳"
          }
        }
      ],
      "minimum_should_match": 0
      
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}


# 查询爱好包含有:篮球,但是,不能包含有:音乐,如果包含了游泳,那么它的相似度更高,返回结果:
{
  "took" : 3,
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
    "max_score" : 0.8681809,
    "hits" : [
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1002",
        "_score" : 0.8681809,
        "_source" : {
          "id" : "1002",
          "name" : "李四",
          "age" : 21,
          "email" : "2222@qq.com",
          "sex" : "女",
          "hobby" : "羽毛球,乒乓球,足球,篮球"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,乒乓球,足球,<em>篮球</em>"
          ]
        }
      }
    ]
  }
}
```
### (11). 权重
> 通过权重来控制评分.最终影响数据返回的顺序. 

```
POST hobby/_search
{
  "query": {
    "bool": {
      
      "must": [
        {
          "match": {
            "hobby": {
              "query": "游泳篮球",
              "operator": "OR"
            }
          }
        }
      ],
      "should": [
        {
          "match": {
            "hobby": {
              "query": "音乐",
              "boost": 10
            }
          }
        },
        {
          "match": {
            "hobby": {
              "query": "跑步",
              "boost": 2
            }
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "hobby":{}
    }
  }  
}

# 通过权重,控制数据的顺序.
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 10.41817,
    "hits" : [
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 10.41817,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "3333@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,篮球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,<em>篮球</em>,<em>游泳</em>,听<em>音乐</em>"
          ]
        }
      },
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1004",
        "_score" : 5.5572257,
        "_source" : {
          "id" : "1004",
          "name" : "赵六",
          "age" : 23,
          "email" : "4444@qq.com",
          "sex" : "女",
          "hobby" : "跑步,游泳"
        },
        "highlight" : {
          "hobby" : [
            "<em>跑步</em>,<em>游泳</em>"
          ]
        }
      },
      {
        "_index" : "hobby",
        "_type" : "_doc",
        "_id" : "1002",
        "_score" : 0.8681809,
        "_source" : {
          "id" : "1002",
          "name" : "李四",
          "age" : 21,
          "email" : "2222@qq.com",
          "sex" : "女",
          "hobby" : "羽毛球,乒乓球,足球,篮球"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,乒乓球,足球,<em>篮球</em>"
          ]
        }
      }
    ]
  }
}
```