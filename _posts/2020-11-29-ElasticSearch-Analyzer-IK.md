---
layout: post
title: 'ElasticSearch 安装IK(九)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载IK与ES对应的版本
> https://github.com/medcl/elasticsearch-analysis-ik   
> https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.0/elasticsearch-analysis-ik-7.1.0.zip   
> <font coloe='red'>自己不要通过maven去编译,git clone下来编译几次都有问题.</font> 

### (2). 解压到ES插件目录下
> 解压并重命令(analysis-ik)

```
lixin-macbook:plugins lixin$ pwd
/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/plugins

lixin-macbook:plugins lixin$ ll
total 0
drwxr-xr-x@  3 lixin  staff   96 12 10 21:29 ./
drwxr-xr-x@ 13 lixin  staff  416 12 10 21:25 ../
drwxr-xr-x@ 10 lixin  staff  320 12 10 21:29 analysis-ik/

lixin-macbook:plugins lixin$ tree
.
└── analysis-ik
    ├── commons-codec-1.9.jar
    ├── commons-logging-1.2.jar
    ├── config
    │   ├── IKAnalyzer.cfg.xml
    │   ├── extra_main.dic
    │   ├── extra_single_word.dic
    │   ├── extra_single_word_full.dic
    │   ├── extra_single_word_low_freq.dic
    │   ├── extra_stopword.dic
    │   ├── main.dic
    │   ├── preposition.dic
    │   ├── quantifier.dic
    │   ├── stopword.dic
    │   ├── suffix.dic
    │   └── surname.dic
    ├── elasticsearch-analysis-ik-7.1.0.jar
    ├── httpclient-4.5.2.jar
    ├── httpcore-4.4.4.jar
    ├── plugin-descriptor.properties
    └── plugin-security.policy
```
### (3). 启动ES
```
lixin-macbook:bin lixin$ ./elasticsearch
# 加载插件
[2020-12-10T21:30:58,420][INFO ][o.e.p.PluginsService     ] [lixin-macbook] loaded plugin [analysis-ik]
```
### (4). 测试分词器
```
# 测试IK分词器
GET _analyze
{
  "analyzer": "ik_max_word",
  "text":"我是中国人"
}

# 测试IK分词器,结果
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "中国人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "中国",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "国人",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 4
    }
  ]
}
```
