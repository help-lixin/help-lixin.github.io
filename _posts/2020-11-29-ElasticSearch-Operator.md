---
layout: post
title: 'ElasticSearch 常用操作'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 记录学习ES常用操作
```
# 批量插入
POST _bulk
{"index" : { "_index" : "books" , "_type" : "IT" , "_id" : "1"}}
{ "id" : "1" , "title" : "Java编程思想" , "language" : "java" , "author" :"Bruce Eckel" , "price": 70.20 , "publish_time" : "2007-10-01" , "description" : "Java学习必读经典,殿堂级著作,赢得了全球程序员的广泛赞誉" }
{"index" : { "_index" : "books" , "_type" : "IT" , "_id" : "2"}}
{ "id" : "2" , "title" : "Java程序性能优化" , "language" : "java" , "author" :"葛一鸣" , "price": 46.50 , "publish_time" : "2012-08-01" , "description" : "让你的Java程序更快,更稳定.深入剖析软件层面,代码层面,JVM虚拟机层面的优化方法" }
{"index" : { "_index" : "books" , "_type" : "IT" , "_id" : "3"}}
{ "id" : "3" , "title" : "Python科学计算" , "language" : "python" , "author" :"张惹愚" , "price": 81.40 , "publish_time" : "2016-05-01" , "description" : "零基础学Python,光盘中作者独家整合开发winPython环境,涵盖了Python各个扩展库" }
{"index" : { "_index" : "books" , "_type" : "IT" , "_id" : "4"}}
{ "id" : "4" , "title" : "Python基础教程" , "language" : "python" , "author" :"Helant" , "price": 54.50 , "publish_time" : "2014-03-01" , "description" : "经典Python入门教程,层次鲜明,结构严谨,内容翔实" }
{"index" : { "_index" : "books" , "_type" : "IT" , "_id" : "5"}}
{ "id" : "5" , "title" : "JavaScript高级程序设计" , "language" : "javascript" , "author" :"Nicholas C. Zakas" , "price": 66.40 , "publish_time" : "2012-10-01" , "description" : "JavaScript经典名著" }


GET /books/_mapping

# 检索所有数据
GET /books/_search
{
  "query": {
    "match_all": {}
  }
}

# 精确查询 
# SELECT * FROM books WHERE title = 'Java编程思想';
GET /books/_search
{
  "query": {
    "term": {
      "title.keyword": {
        "value": "Java编程思想"
      }
    }
  }
}



# 分页查询,并只显示指定的列
# SELECT title,author FROM books LIMIT 0,2;
GET /books/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["title","author"], 
  "from": 0,
  "size": 2
}


# 查询结果带上version
# SELECT title,author,_version FROM books LIMIT 0,2;
GET /books/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["title","author"], 
  "version": true, 
  "from": 0,
  "size": 2
}

# 最不评份过滤器(>0.9)
# SELECT *,_version FROM books WHERE min_score > 0.9;
GET /books/_search
{
  "query": {
    "match": {
      "title": "java"
    }
  },
  "min_score":0.9,
  "version": true
}


# 高亮显示
GET /books/_search
{
  "query": {
    "match": {
      "title": "java"
    }
  },
  "min_score":0.9,
  "highlight": {
    "fields": {
      "title":{}
    }
  }, 
  "version": true
}


GET _analyze
{
  "text":"java编程思想"
}


# 由于是标准分词器,所以中文分拆分成每一个字,英文分拆分成单词.
# 1. 在保存文档时,就按照标准分词器的方式进行索引了
# 2. query的内容(java编程思想),也会按照标准分词器进行拆份
# 3. 交给ES进行检索
# SQL语句如下:
# SELECT * FROM books 
# WHERE title LIKE 'java' 
# OR title LIKE '%编%' 
# OR title LIKE '%程%' 
# OR title LIKE '%思%' 
# OR title LIKE '%想%';

GET /books/_search
{
  "query": {
    "match": {
      "title": {
        "query": "java编程思想",
        "operator": "or"
      }
    }
  }
}



PUT /test/test/1
{"foo" : "I just said hello world"}


PUT /test/test/2
{"foo" : "Hello world"}

PUT /test/test/3
{"foo" : "World Hello "}


# 查看mapping
GET /test/_mapping
{
  
}


# 查看所有文档
GET /test/_search
{
  "query": {
    "match_all": {}
  }
}


# SELECT * FROM test WHERE foo LIKE 'Hello World%';
# 1. "Hello World"分词后,都要出列在字段中
# 2. 字段中的顺序也要和分词后的内容保存一致.
GET /test/_search
{
  "query": {
    "match_phrase": {
      "foo": "Hello World"
    }
  }
}


# SELECT * FROM test WHERE foo LIKE 'World h*%';
GET /test/_search
{
  "query": {
    "match_phrase_prefix": {
      "foo": "World h"
    }
  }
}

# 多域检索
# SELECT * FROM books WHERE title LIKE '%java编程思想%' OR description LIKE '%java编程思想%';
GET /books/_search
{
  "query": {
    "multi_match": {
      "query": "java编程思想",
      "fields": ["title","description"]
    }
  }
}


# 查看所有文档
GET /books/_search
{
}

# 范围查询
# gt   : 大于
# gte  : 大于等于
# lt   : 小于
# lte  : 小于等于
# SELECT * FROM books WHERE price >= 50 AND price <=70;
GET /books/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 50,
        "lte": 70
      }
    }
  }
}


# 根据时间范围查询
# SELECT * FROM books WHERE publish_time >= "2014-01-01" AND publish_time <= "2021-12-31";
GET /books/_search
{
  "query": {
    "range": {
      "publish_time": {
        "gte": "2014-01-01",
        "lte": "2021-12-31",
        "format": "yyyy-MM-dd"
      }
    }
  }
}


# 前缀查询
# SELECT * FROM books WHERE title LIKE '%java';
GET /books/_search
{
  "query": {
    "prefix": {
      "title": {
        "value": "java"
      }
    }
  }
}


# 通配符查询
# ? : 用来match一个字符
# * : 用来match零到多个字符
# 有点像MySQL里的正则写法
GET /books/_search
{
  "query": {
    "wildcard": {
      "title": {
        "value": "*java*"
      }
    }
  }
}


# id查询
# SELECT * FROM books WHERE id IN(1,3);
GET /books/_search
{
  "query": {
    "ids": {
      "type": "IT",
      "values": [1,3]
    }
  }  
}




# SELECT * FROM books WHERE title LIKE '%java%' AND _score = 1.2;
GET /books/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "title": "java"
        }
      },
      "boost": 1.2
    }
  }
}


# SELECT * FROM books 
# WHERE title LIKE '%%java' AND price < 70 OR description LIKE '%虚拟机%';
GET /books/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "java"
          }
        },
        {
          "range": {
            "price": {
              "lt": 70
            }
          }
        }
      ],
      "should": [
        {
          "match": {
            "description": "虚拟机"
          }
        }
      ]
    }
  }
}


# 以添加文档的顺序进持排序显示
GET /books/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_doc": {
        "order": "asc"
      }
    }
  ]
}

# 查询并排序
# SELECT * FROM books WHERE title LIKE '%java%' ORDER BY price DESC
GET /books/_search
{
  "query": {
    "match": {
      "title": "java"
    }
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}

# 自定义高亮片段
# require_field_match : 允许多个字段高亮显示
# 多个字段高亮显示
GET /books/_search
{
  "query": {
    "match": {
      "title": "java"
    }
  },
  "highlight": {
    "require_field_match": "false", 
    "fields": {
      "title" : {
        "pre_tags": "<em color='red'>",
        "post_tags": "</em>"
      },
      "description": {
        "pre_tags": "<em color='red'>",
        "post_tags": "</em>"
      }
    }
  }
}


# SELECT MAX(price) AS max_price FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "max_price": {
      "max": { "field": "price"  }
    }
  }
}


# SELECT MIN(publish_time) AS min_year FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "min_year": {
      "min": { "field": "publish_time"  }
    }
  }
}



# SELECT AVG(price) AS avg_price FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "avg_price": {
      "avg": { "field": "price"  }
    }
  }
}


# SELECT SUM(price) AS total_price FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "total_price": {
      "sum": { "field": "price"  }
    }
  }
}


# SELECT COUNT(language) AS  all_language FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "all_language": {
      "cardinality": { "field": "language.keyword" }
    }
  }
}


# SELECT
# COUNT(price) AS count, 
# MIN(price) AS min,
# MAX(price) AS max,
# AVG(price) AS avg,
# SUM(price) AS sum
# FROM books;
GET /books/_search
{
  "size": 0,
  "aggs": {
    "grades_stats": {
      "stats": {
        "field": "price"
      }
    }
  }
}


## 分桶

# 按照language进行分类统计
# SELECT 
# language 
# FROM books 
# GROUP BY language
GET /books/_search
{
  "size": 0,
  "aggs": {
    "language_count": {
      "terms": {
        "field": "language.keyword",
        "size": 4
      }
    }
  }
}


# 在分组的基础上,再进行聚合统计
# SELECT 
# language ,
# AVG(price) AS avg_price
# FROM books 
# GROUP BY language
GET /books/_search
{
  "size": 0,
  "aggs": {
    "language_count": {
      "terms": {
        "field": "language.keyword",
        "size": 4
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}


# 把符合条件的文档分到一个桶中.
# SELECT 
# COUNT(*) AS doc_count,
# AVG(price) AS avg_price
# FROM books
# WHERE title LIKE '%java%'
# GROUP BY title
GET /books/_search
{
  "size": 0,
  "aggs": {
    "java_avg_price": {
      "filter": {
        "match": {
          "title": "java"
        }
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}



# SELECT 
# COUNT(*) AS doc_count,
# AVG(price) AS avg_price
# FROM books
# WHERE title LIKE '%java%' OR title LIKE '%python%'
# GROUP BY price
GET /books/_search
{
  "aggs": {
    "per_avg_price": {
      "filters": {
        "filters": [
          { "match": { "title" : "java"  } },
          { "match": { "title" : "python"} }
        ]
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}

# 区间聚合
# SELECT COUNT(*) AS doc_count WHERE price < 50
# UNION
# SELECT COUNT(*) AS doc_count WHERE price >= 50 < 80;
# UNION
# SELECT COUNT(*) AS doc_count WHERE price >= 80;
GET /books/_search
{
  "aggs": {
    "price_rang": {
      "range": {
        "field": "price",
        "ranges": [
          { 
            "to" : 50 
          },
          {
            "from": 50,
            "to": 80
          },
          {
            "from" : 80
          }
        ]
      }
    }
  }
}


# 区间聚合,还支持日期类型
GET /books/_search
{
  "aggs": {
    "publish_time_rang": {
      "range": {
        "field": "publish_time",
        "ranges": [
          { 
            "to" : "2013-09-01" 
          },
          {
            "from": "2013-09-01" ,
            "to": "2014-09-01" 
          },
          {
            "from" : "2013-09-01" 
          }
        ]
      }
    }
  }
}


# 1. 当别名与索引是1:1时,别名与索引没有任何区别.
# 2. 当别名与索引是1:N时,只能对别名进行查询,不能有其它操作(因为增/删/改),除了要指定_id,还得要指定:_index,别名并非真正的_index.



# 创建两个索引
POST _bulk
{ "index" :{ "_index" : "test_07" , "_type" : "_doc" , "_id" :  1 } }
{ "id" : 1 , "name" : "zhagnsan" , "age" : 25 }
{ "index" :{ "_index" : "test_08" , "_type" : "_doc" , "_id" :  2 } }
{ "id" : 2 , "name" : "lixin" , "age" : 26 }

# 查看索引映射信息
GET /test_07/_mapping
# 查看索引映射信息
GET /test_08/_mapping

# 给索引:test_07增加索引别名:tests
# test_07 --> tests
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "test_07",
        "alias": "tests"
      }
    }
  ]
}

# 对别名进行检索
GET /tests/_search
{
  "query": {
    "match_all": {}
  }
}

# 索引与别名1对一时,添加数据成功
PUT /tests/_doc/3
{ "id" : 3 , "name" : "xin-li" , "age" : 28 }


# 给索引:test_08增加索引别名:tests
# test_08 --> tests
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "test_08",
        "alias": "tests"
      }
    }
  ]
}


# 别名与索引1对多时,增加数据失败,虽然有指定了id,但是,无法确保要将数据保存到哪个索引里
PUT /tests/_doc/4
{ "id" : 4 , "name" : "hello world" , "age" : 33 }

# 别名与索引1对多时,删除数据了是失败了,原理同上.
DELETE /tests/_doc/3


# 别名与索引1:N时,检索不会受影响
# 对别名进行检索
GET /tests/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "name": "lixin"
          }
        },
        {
          "match": {
            "name": "zhagnsan"
          }
        }
      ]
    }
  }
}

# 查看别名下有哪些索引
GET /_alias/tests

# 查看索引对应的别名
GET /test_07/_alias


# 解除test_07 与 别名(tests)的关联.
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "test_07",
        "alias": "tests"
      }
    }
  ]
}


# 创建新的索引库
PUT /test_09
{
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "long"
        },
        "id" : {
          "type" : "long"
        },
        "name" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
} 


# 查看test_07现在的数据
GET /test_07/_search
{
  "query": {
    "match_all": {}
  }
}


# 拷贝test_07的数据到:test_09
POST /_reindex?wait_for_completion=true
{
  "source": { "index": "test_07" },
  "dest": { "index": "test_09" }
}



# 查看test_09现在的数据
GET /test_09/_search
{
  "query": {
    "match_all": {}
  }
}

# 绑定test_09 与 别名(tests)的关联.
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "test_09",
        "alias": "tests"
      }
    }
  ]
}

# 查看别名:tests现在的数据
GET /tests/_search
{
  "query": {
    "match_all": {}
  }
}
```