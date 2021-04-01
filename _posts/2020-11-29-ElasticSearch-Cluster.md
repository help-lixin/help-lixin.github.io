---
layout: post
title: 'ElasticSearch 集群搭建(十八)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 集群机器

|  机器名称   | IP            |
|  ----      | ----          |
| es-1       | 10.211.55.100 |
| es-2       | 10.211.55.101 |
| es-3       | 10.211.55.102 |


### (2). 所有机器,准备工作

> 1. 配置ip与host映射(/etc/hosts).  

```
# 配置机器名称与IP地址映射
# vi /etc/hosts
10.211.55.100 es-1
10.211.55.101 es-2
10.211.55.102 es-3
```

> 2. 配置软硬连接数(/etc/security/limits.conf)

```
# vi /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
```

> 3. 配置max_map_count(/etc/sysctl.conf)

```
# vi /etc/sysctl.conf
vm.max_map_count=262144
```

### (3). 配置elasticsearch.yml
> 注意:TODO部份的配置.

```
# 集群名称
cluster.name: es-cluster

# // TODO 
# 注意:每台机器,配置不一样
# 节点名称(es-1/es-2/es-3)
node.name: es-1

# 是否可以成为master
node.master: true

# 是否为数据节点
node.data: true

# 绑定的主机名称
network.host: 0.0.0.0

# 绑定的端口
http.port: 9200

# 服务发现的列表
discovery.seed_hosts: ["10.211.55.100", "10.211.55.101","10.211.55.102"]

# master选举节点
cluster.initial_master_nodes: ["es-1", "es-2","es-3"]

# 是否支持跨域
http.cors.enabled: true

# *表示支持所有域名
http.cors.allow-origin: "*"
```

### (4). 启动es

```
# 在后台启动es
/home/es/elasticsearch-7.1.0/bin/elasticsearch -d
```

### (5). 查看集群状态
> 查看集群状态[ curl  http://localhost:9200/_cluster/health ]

|  颜色         | 含义                                  |
|  ----        | ----                                  |
| green        | 所有主要分片和复制分的都可用               |
| yellow       | 所有主要分片可用,但不是所有的复制分片都可用  |
| red          | 不是所有的主要分片都可用                  |

