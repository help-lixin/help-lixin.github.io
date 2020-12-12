---
layout: post
title: 'ElasticSearch Search(URI)(十)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). URI Search
> URI Search就是通过URI进行Query.   
> q       : 指定查询的文本.  
> df      : 指定查询的字段(不指定会查询所有的字段).    
> sort    : 排序       
> from    : 跳过开始的结果数,默认是:0      
> size    : 结果数,默认是:10   
> profile : 查看Query是如何执行的.    

```
GET /movies/_search?q=Lolita&df=title&sort=year:desc&from=0&size=10
{
  "profile": "true"
}
```  
### (2). URI Search案例
```
# URI查询(title:Lolita)
GET /movies/_search?q=title:Lolita


# URI查询(title:Lolita)
GET /movies/_search?q=Lolita&df=title&sort=year:desc&from=0&size=10
{
  "profile": "true"
}

# URI查询(泛查询,查询所有字段)
# (title.keyword:2012 | id.keyword:2012 | year:[2012 TO 2012] | genre:2012 | @version:2012 | @version.keyword:2012 | id:2012 | genre.keyword:2012 | title:2012)
GET /movies/_search?q=2012
{
  "profile": "true"
}
```
### (3). Term vs Phrase

>  "Beautiful Mind"(注意:有双引号),等效于:Beautiful AND Mind.Phrase查询,还要求前后顺序保持一致.   

```
GET /movies/_search?q=title:"Beautiful Mind"
{
  "profile": "true"
}
```


>  Beautiful Mind等效于Beautiful OR Mind.

```
# BooleanQuery
GET /movies/_search?q=title:(Beautiful Mind)
{
  "profile": "true"
}
```
### (4). 布尔操作
> AND / OR / NOT 或者 && / || / !         
> 必须大写    
> title:(matrix NOT reloaded)      

```
# "type" : "BooleanQuery"
# "description" : "+title:beautiful +title:mind"
GET /movies/_search?q=title:(Beautiful AND Mind)
{
  "profile": "true"
}

# "type" : "BooleanQuery"   
# "description" : "title:beautiful -title:mind"
GET /movies/_search?q=title:(Beautiful NOT Mind)
{
  "profile": "true"
}
```
### (5). 分组
> + 表示:must(AND)    
> - 表示:must_not(NOT)    
> title(+matrix -reloaded)    

```
# "type" : "BooleanQuery",
# "description" : "title:beautiful title:mind",
GET /movies/_search?q=title:(Beautiful +Mind)
{
  "profile": "true"
}
```

### (6). 其它查询
```
# 电影的年份大于等于1980年以上的. 
GET /movies/_search?q=year:>=1980
{
  "profile": "true"
}


# 通配符查询:
# 电影的title包含有:b的
# "type" : "MultiTermQueryConstantScoreWrapper"
# "description" : "title:b*"
GET /movies/_search?q=title:b*
{
  "profile": "true"
}
```
