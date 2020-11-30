---
layout: post
title: 'ElasticSearch术语(一)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 传统数据库与ES比较
> ES在后续版本,已经不再使用Type,因为:Indice本身就只支持同一类型的Document.  

|  MySQL    | ES       |
|  ----     | ----     |
| DataBase  | Indice  |
| Table     | *Type*  |
| Row      | Document |
| Column   | Field   |

### (2). 映射Mapping(类似于表结构的定义)
> mapping是对:Indice里Document中一个Field进行描述(相当于元数据).
> 如某个字段的数据类型/默认值/分词器/是否被索引/是否存储等等.

### (3). 分片
> 1. 分片,ES是分布式搜索引擎,每个索引有一个或多个分片,索引的数据被分配到各个分片上,可以理解为:一块饼切成N块,N块加起来就是一个完整的Indice库.
> 2. 分片有助于横向扩展,N个分片会被尽可能平均地(rebalance)分配在不同的节点上(例如你有2个节点,4个主分片(不考虑备份),那么每个节点会分到2个分片,后来你增加了2个节点,那么你这4个节点上都会有1个分片,这个过程叫relocation,ES感知后自动完成). 
> 3. 分片是独立的,对于一个Search Request的行为,每个分片都会执行这个Request.   
> 4. 每个分片都是一个Lucene Index,所以一个分片只能存放 Integer.MAX_VALUE - 128 = 2,147,483,519个docs.

### (3). 复制
> 1. 复制,可以理解为备份分片.主要解决负载均衡和高可用问题.   
> 2. 主分片和备分片不会出现在同一个节点上(防止单点故障),默认情况下一个索引库创建5个分片,每个分片指定:1个副本数.总共是:10分片（5primary+5replica=10个分片）
> 3. 如果你只有一个节点(Node),那么5个replica都无法分配(unassigned),此时cluster status会变成Yellow. 

