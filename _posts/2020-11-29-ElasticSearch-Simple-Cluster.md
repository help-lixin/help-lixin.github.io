---
layout: post
title: 'ElasticSearch 单机伪集群运行(四)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 为ES创建数据目录
```
# 查看工作目录
lixin-macbook:elasticsearch-7.1.0 lixin$ pwd
/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0

# 为每个集群创建数据目录
lixin-macbook:elasticsearch-7.1.0 lixin$ mkdir node1_data
lixin-macbook:elasticsearch-7.1.0 lixin$ mkdir node2_data
lixin-macbook:elasticsearch-7.1.0 lixin$ mkdir node3_data
```
### (2). 启动集群
```
# 查看工作目录
lixin-macbook:elasticsearch-7.1.0 lixin$ pwd
/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0

# 单机运行集群(node1)
lixin-macbook:elasticsearch-7.1.0 lixin$ ./bin/elasticsearch -E node.name=node1 -E cluster.name=lixin -E path.data=node1_data

# 单机运行集群(node2)
lixin-macbook:elasticsearch-7.1.0 lixin$ ./bin/elasticsearch -E node.name=node2 -E cluster.name=lixin -E path.data=node2_data

# 单机运行集群(node3)
lixin-macbook:elasticsearch-7.1.0 lixin$ ./bin/elasticsearch -E node.name=node3 -E cluster.name=lixin -E path.data=node3_data
```
### (3). 查看集群信息
```
lixin-macbook:~ lixin$ curl http://localhost:9200/_cat/nodes
127.0.0.1 12 99 17 1.55   mdi - node2
127.0.0.1 24 99 17 1.55   mdi * node1
127.0.0.1 14 99 17 1.55   mdi - node3
```