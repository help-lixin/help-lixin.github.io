---
layout: post
title: 'ElasticSearch 集群相关API(七)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 查看集群状态
```
# 查看集群状态
GET _cluster/health

# 查看集群状态结果
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 2,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 2,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 50.0
}
```
### (2). 查看集群所有节点
```
# 查看集群所有节点
GET _cat/nodes

# 查看集群所有节点结果
127.0.0.1 24 100 15 1.71   mdi * lixin-macbook.local
```
### (3). 查看集群所有的分片信息
```
# 查看集群所有的分片信息
GET _cat/shards

# 查看集群所有的分片信息结果                
movies  0 p STARTED    9743 1.4mb 127.0.0.1 lixin-macbook.local
movies  0 r UNASSIGNED                      
```

### (4). 集群状态说明
> 1. green  ==> 代表主分片和副本分片都是正常的.   
> 2. yellow ==> 代表主分片全部正常分配,有副本分片未正常分配.  
> 3. red    ==> 有主分片未能分配.