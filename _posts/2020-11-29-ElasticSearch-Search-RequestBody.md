---
layout: post
title: 'ElasticSearch Search(Request Body)(十一)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). Reqeust Body Search(DSL)
> Reqeust Body Search是ES提供的DSL语言.

### (2). Request Body 

```
GET /movies/_search
{
  "sort":[{"year":"asc"}],
  "_source": ["id","title","year"], 
  "from":0,
  "size": 20, 
  "profile": "true",
  "query":{
    "match_all": {}
  }
}
```
### (3). 布尔查询
```
# OR
GET /movies/_search
{
  "profile": "true",
  "query":{
    "match": {
      // Beautiful OR Mind
      "title": "Beautiful Mind"
    }
  }
}


# AND
GET /movies/_search
{
  "profile": "true",
  "query":{
    "match": {
      "title": {
        // Beautiful AND Mind
        "query": "Beautiful Mind",
        "operator": "AND"
      }
    }
  }
}


```

### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 
