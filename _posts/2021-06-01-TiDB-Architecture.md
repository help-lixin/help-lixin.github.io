---
layout: post
title: 'TiDB 架构详解(二)'
date: 2021-06-01
author: 李新
tags:  TiDB
---

### 1. 前言
> 在安装TiDB之前,首先要了解下TiDB,这是[TiDB官网](https://docs.pingcap.com/zh/tidb/stable/tidb-architecture)文档还是比较全面的.  

### 2. TiDB整体架构
!["TiDB架构图"](/assets/tidb/imgs/tidb-architecture-v3.1.png)

1. TiDB(TiDB Cluster)
   - 对外暴露MySQL协议.
   - <font color='red'>负责接受客户端的连接,执行SQL解析和优化,最终生成分布式执行计划.</font>  
   - 只是解析SQL，将实际的数据读取请求转发给底层的存储节点TiKV.  
   - 有没有感觉,有点像MyCat(Proxy).  
2. PD(Placement Driver)
   - <font color='red'>整个TiDB集群的元信息管理模块.</font>  
   - 存储每个TiKV节点实时的数据分布情况和集群的整体拓扑结构.     
   - 下发数据调度命令给具体的TiKV节点.  
   - 分配分布式事务分配事务ID.     
   - 建议部署奇数个PD节点.   
   - 有没有感觉像ZK(HBase利用ZK来存储Region数据).  
3. TiKV
   -  <font color='red'>存储数据的基本单位是Region,每个Region负责存储一个Key Range(从StartKey到EndKey的左闭右开区间)的数据,每个TiKV节点会负责多个Region(还记得HBase(LSM)的架构吗?).</font>  
   -  在KV键值对层面提供对分布式事务的原生支持.  
   -  TiKV中的数据都会自动维护多副本(默认为三副本)天然支持高可用和自动故障转移.   
4. TiFlash
   - 主要的功能是为分析型的场景加速.
   - <font color='red'>数据是以列式的形式进行存储.</font>  

### 3. TiKV(存储)架构
!["TiKV存储"](/assets/tidb/imgs/tikv-storage-architecture.png)

> TiKV存储特性
   -  可以理解为:一个巨大的Map,也就是存储的是:Key-Value Pairs(键值对)  
   -  TiKV将整个Key-Value空间分成很多段,每一段是一系列连续(顺序)的Key,将每一段叫做一个Region(还记得:HBase吗?)
   -  在写入数据时,会路由到某个Region,然后,会以Region为单位进行副本复制(Raft),达到高可性(可以理解为:一个TiKV会包含N个Region).    
   -  TiKV没有选择直接向磁盘上写数据,而是把数据保存在RocksDB中,具体的数据落地由RocksDB负责.  
   -  <font color='red'>通过单机的RocksDB,TiKV可以将数据快速地存储在磁盘上.</font>  
   -  通过Raft协议,将数据复制到多台机器上,以防单机失效(数据的写入是通过Raft这一层的接口写入,而不是直接写RocksDB),即使,少数几台机器宕机也能通过原生的Raft协议自动把副本补全,可以做到对业务无感知.   

### 4. TiDB(计算)架构
> TiDB如何将库表中的数据映射到TiKV中的(Key, Value)键值对的呢?   
> 以TiDB官网的一张图为例,来剖析:

!["TiDB计算架构"](/assets/tidb/imgs/tidb-computing.png)

```
# 1. 创建表:t_test
CREATE TABLE t_test (
  id      INT,
  name    VARCHAR(20),
  age     INT,
  score   DECIMAL(10,2),
  PRIMARY KEY (id),
  UNIQUE  KEY (name),
  INDEX   idx_age (age)
);

# 2. 插入数据
INSERT INTO t_test(id,name,age,score) VALUES(1,"Bob",12,99);

# 3. 查看一下索引信息
mysql> show index from t_test;
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+-----------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression | Clustered |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+-----------+
| t_test |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               | YES     | NULL       | YES       |
| t_test |          0 | name     |            1 | name        | A         |           0 |     NULL | NULL   | YES  | BTREE      |         |               | YES     | NULL       | NO        |
| t_test |          1 | idx_age  |            1 | age         | A         |           0 |     NULL | NULL   | YES  | BTREE      |         |               | YES     | NULL       | NO        |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+-----------+

# 4. TiDB中是如何把表中的数据与Key映射的呢?
#  4.1 TiDB在创建表时,会为表(t_test)分配一个唯一的表ID(TableID)
#  4.2 TiDB在插入数据时,会为每一行数据生成(表没有主键的情况下)一个唯一的行ID(RowID),当,表有主键的情况下,RowID就是主键ID.  
#  4.3 TiDB为了支持索引,会为表中每个索引分配了一个索引ID,用 IndexID 表示.  
#  4.4 tablePrefix(t)/recordPrefixSep(r)/indexPrefixSep(i)都是字符串常量.  
#  4.5 如果,按官网这种方式存储数据,对于LIKE肯定是支持不到位的(因为,要支持LIKE,意味着:要存储大量的索引数据).  

# 5. 主键索引与KV的映射
#  Key:   tablePrefix{TableID}_recordPrefixSep{RowID}
#  Value: [col1, col2, col3, col4]

在主键索引的情况下,上图的数据在TiDB存储方式为:  
KEY :    t{TableID}_r1   
VALUE :  [Bob,12,99]


# 6. 唯一索引与KV的映射
#  Key:   tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue
#  Value: RowID

在唯一索引情况下,上图的数据在TiDB存储方式为: 
  KEY:   t_{TableID}_i_Bob   
  VALUE: 1

# 7. 非唯一索引与KV映射
# Key:   tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue_{RowID}
# Value: null

在非唯一索引情况下,上图的数据在TiDB存储方式为(直接解析KEY,就可以找到对应的数据):
  KEY:      t_{TableID}_i_12_1
  VALUE :  NULL
```
### 5. 总结
> 1. 感觉:TiDB是运用Raft协议,协调所有的RocksDB进行数据存储.    
> 2. 由于LSM的特性(数据有序性),在逻辑上划分多个Region(对Map中的Key进行分区,不至于让所有数据都在一个Map里),以此:实现数据的负载均衡.     