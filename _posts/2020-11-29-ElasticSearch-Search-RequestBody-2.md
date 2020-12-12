---
layout: post
title: 'ElasticSearch Search(Request Body)(十三)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (0). 创建索引和Mapping信息请参考:
> https://www.lixin.help/2020/11/29/ElasticSearch-Mapping.html

### (1). 批量插入测试数据(_bulk)
```
# 批量插入数据

POST _bulk
{"index":{"_index":"hobbys","_id":1001}}
{"id":"1001","name":"张三","age":20,"email":"1111@qq.com","sex":"男","hobby":"羽毛球,乒乓球,足球"}
{"index":{"_index":"hobbys","_id":1002}}
{"id":"1002","name":"李四","age":21,"email":"2222@qq.com","sex":"女","hobby":"羽毛球,乒乓球,足球,蓝球"}
{"index":{"_index":"hobbys","_id":1003}}
{"id":"1003","name":"王五","age":22,"email":"3333@qq.com","sex":"男","hobby":"羽毛球,蓝球,游泳,听音乐"}
{"index":{"_index":"hobbys","_id":1004}}
{"id":"1004","name":"赵六","age":23,"email":"4444@qq.com","sex":"女","hobby":"跑步,游泳"}
{"index":{"_index":"hobbys","_id":1005}}
{"id":"1005","name":"孙七","age":24,"email":"5555@qq.com","sex":"男","hobby":"听音乐,看电影"}

# 添加数据
PUT /hobbys/_doc/1006
{"id":"1006","name":"小玲","age":25,"email":"6666@qq.com","sex":"女","hobby":"乐器,看电影"}

```

### (2). 检索(match)
ES在保存数据时有如下几步操作:   
> 1. 获取文档Field(hobby)配置的分词器(Standard).   
> 2. 对Document Field进行分词(Standard分词器,英文按空格划分,中文按单字划分).    
> 3. 把分词后的内容,建立倒排索引,同时,把原始Document进行保存.   

ES match数据时有如下几步操作:   
> 1. match会获取待检索Field(hobby)的分词器(Standard分词器,英文按空格划分,中文按单字划分.)     
> 2. 根据分词器(Standard),对检索内容("音乐")进行分词.      
> 3. 根据分词后的内容("音","乐"),让ES到各分片的:"倒排搜索中进行检索".    
> 4. 在倒排索引中,有相应的documentID,获取:documentID,对应的:document

总结:   
<font color='red'>math会对内容进行分词.</font>

```
# 搜索爱好:包含音乐的
# SELECT * FROM hobbys WHERE hobby LIKE '%音%' OR hobby LIKE '%乐%';
POST /hobbys/_search
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

# 搜索爱好包含音乐的,结果集
{
  "took" : 802,
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
    "max_score" : 1.8456821,
    "hits" : [
      {
        "_index" : "hobbys",
        "_type" : "_doc",
        "_id" : "1005",
        "_score" : 1.8456821,
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
            "听<em>音</em><em>乐</em>,看电影"
          ]
        }
      },
      {
        "_index" : "hobbys",
        "_type" : "_doc",
        "_id" : "1003",
        "_score" : 1.4829273,
        "_source" : {
          "id" : "1003",
          "name" : "王五",
          "age" : 22,
          "email" : "3333@qq.com",
          "sex" : "男",
          "hobby" : "羽毛球,蓝球,游泳,听音乐"
        },
        "highlight" : {
          "hobby" : [
            "羽毛球,蓝球,游泳,听<em>音</em><em>乐</em>"
          ]
        }
      },
      {
        "_index" : "hobbys",
        "_type" : "_doc",
        "_id" : "1006",
        "_score" : 0.7909737,
        "_source" : {
          "id" : "1006",
          "name" : "小玲",
          "age" : 25,
          "email" : "6666@qq.com",
          "sex" : "女",
          "hobby" : "乐器,看电影"
        },
        "highlight" : {
          "hobby" : [
            "<em>乐</em>器,看电影"
          ]
        }
      }
    ]
  }
}
```
### (3). 精确匹配(term/terms) 
> term主要用于精确匹配哪些值,比如:数字,日期,布尔值.    
> <font color='red'>或者not_analyzed的字符串(未经分词器处理的文本数据类型[keyword]).</font>    

```
# 精确查找:性别为男性的
# SELECT * FROM hobbys WHERE sex = '男';
GET /hobbys/_search
{
  "query": {
    "terms": {
      "sex": [
        "男"
      ]
    }
  }
}

# 精确查找年龄是20/21/22岁
# SELECT * FROM hobbys WHERE age IN (20,21,22);
GET /hobbys/_search
{
  "query": {
    "terms": {
      "age": [ 20,21,22 ]
    }
  }
}


# 精确查找
# SELECT * FROM hobbys WHERE email = '5555@qq.com';
GET /hobbys/_search
{
  "query": {
    "terms": {
      "email": [
        "5555@qq.com"
      ]
    }
  }
}
```
### (4). 范围查询(rang)
> gt  : 大于            
> gte : 大于等于        
> lt  : 小于           
> lte : 小于等于       

```
# 范围查询
# SELECT * FROM hobbys WHERE age >= 20 AND age <=24;
GET /hobbys/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 20,
        "lte": 24
      }
    }
  }
}
```
### (5). Exits
> 判断某个Field不为空(IS_NOT_NUL).   

```
# 添加一条数据,hobby为空的数据.
PUT /hobbys/_doc/1007
{"id":"1007","name":"小何","age":26,"email":"7777@qq.com","sex":"男"}


# 查询出字段(hobby)不为空的数据(会发现:id为1007的数据没有在结果集展示)
# SELECT * FROM hobbys WHERE hobby IS NOT NULL;
GET /hobbys/_search
{
  "query": {
    "exists": {
      "field": "hobby"
    }
  }
}
```
### (6). Match查询
> match查询是一个标准查询,不管你是否需要全文查询还是精确查询基本上都要用到它.   
> <font color='red'>如果你使用match查询一个全文本字段(text),它会在真正查询之前先用分词器分析match一下查询字符.</font>    
> <font color='red'>如果用match查询,指定的是一个确切值,在遇到:数字,日期,布尔值或者not_analyzed的这符串时,它将为你搜索你给定的值.</font>   

```
# text分词字段.
# SELECT * FROM hobbys WHERE name LIKE '%小%' OR name LIKE '%玲%';
GET /hobbys/_search
{
  "query": {
    "match": {
      "name": "小玲"
    }
  },
  "highlight": {
    "fields": {
      "name":{}
    }
  }
}

# not_analyzed字段
# SELECT * FROM hobbys WHERE age = 20;
GET /hobbys/_search
{
  "query": {
    "match": {
      "age": 20
    }
  }
}
```

### (7). boolean查询
> 组合查询   
> must        :  多个查询条件完全匹配(and).   
> must_not    :  多个查夜条件相反匹配(not).   
> should      :  至少有一个查询条件匹配(or).    


```
# 组合查询
# SELECT * FROM hobbys WHERE sex = "男" AND age >= 20 AND age <=24;
POST /hobbys/_search
{
  "query": {
    "bool": {
      
      "filter": { "match":{ "sex" : "男" } },
      "must": { "range":{ "age":{ "gte" : 20, "lte" : 24 }} }
        
    }
  }
}


# boolean组合查询
# 查询年龄在20-24岁之间的,性别为:男
# SELECT * FROM hobbys 
# WHERE hobby IS NOT NULL
# AND age >= 20 AND age <=24 
# AND sex = "男" ;
POST /hobbys/_search
{
  "query": {
    "bool": {
      
      "must": [  
        {
         "exists" :{ "field": "hobby" } 
        },
        {
          "range":{ 
             "age":{
               "gte":20,
               "lte":24
             }
          }
        },
        {
          "match": {
            "sex": "男"
          }
        }
      ]
      
    }
  }
}

# boolean 组合查询
# 查询年龄在20到24岁之间的,并且,性别不能为:"男"
# SELECT * FROM hobbys WHERE hobby IS NOT NULL AND age <= 20 AND age >= 24 AND sex != "男";
POST /hobbys/_search
{
  "query": {
    "bool": {
      
      "must": [  
        {
         "exists" :{ "field": "hobby" } 
        },
        {
          "range":{ 
             "age":{
               "gte":20,
               "lte":24
             }
          }
        }
      ],
     "must_not": [
       {"match":{"sex":"男"}}
     ]
      
    }
  }
}

# exists找出不为NULL,must_not取反.
# SELECT * FROM hobbys WHERE hobby IS  NULL;
POST /hobbys/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "exists": {
             "field":"hobby"
          }
        }
      ]
    }
  }
}
```

### (8). 过滤(filter)查询

```
POST /hobbys/_search
{
  "query": {
    "bool": {
      "filter": {
        "terms": {
          "age": [ 20,21 ]
        }
      }
    }
  }
}
```
### (9). filter和match对比
> 一条过滤(filter)语句会询问每个文档的字段是否包含着特定的值.   
> 查询语句(match)会询问每个文档的字段值与特定值的匹配程度如何.该查询语句会计算每个文档与查询语句的相关性,会给出一些相关性评分(_score),并且,按照相关性对匹配到的文档进行排序,这种评分方式非常适用于一个没有完全配置结果(没人工干扰)的全文本搜索.    
> 查询语句(match)不仅需要查找匹配的文档,还需要计算每个文档的相关性,所以一般来说:查询语句(match)要比过滤语句(filter)更耗时,并且查询结果也不可缓存.   
> <font color='red'>建议: 做精确匹配搜索时,最好用过滤语句(filter),因为:过滤语句可以缓存数据.</font>

### (10). 

### (11). 

### (12). 

### (13). 
