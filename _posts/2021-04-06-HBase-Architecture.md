---
layout: post
title: 'HBase架构详解(六)'
date: 2021-04-06
author: 李新
tags:  HBase
---

### (1). HBase架构图
!["HBase架构图"](/assets/hbase/imgs/hbase-architecture.png)

### (2). Master(是所有Region Server的管理者,其实现类为:HMaster)
> 1. 监控RegionServer.   
> 2. 处理RegionServer故障转移.   
> 3. 处理元数据的变更.  
> 4. 处理region的分配或转移.  
> 5. 在空闲时间进行数据的负载均衡.  
> 6. 通过Zookeeper发布自己的位置给客户.   
### (3). Region Server(是Region的管理者,其实现类为:HRegionServer)
> 1. 负责存储HBase的实际数据.   
> 2. 处理分配给它的Region.  
> 3. 刷新缓存到HDFS.  
> 4. 维护Hlog.  
> 5. 执行压缩.  
> 6. 负责处理Region分片.   

!["region server架构图"](/assets/hbase/imgs/region-server.jpg)

### (4). Region
> Hbase表的分片,HBase表会根据RowKey值被切分成不同的region存储在RegionServer中,在一个RegionServer中可以有N个Region.   
!["Region 架构图"](/assets/hbase/imgs/region.jpg)

### (5). HLog(WAL)
> Hlog存储的是:HBase的修改记录(增/删/改).当对HBase读写数据的时候,数据不是直接写进磁盘,它会在内存中保留一段时间.由于数据要经MemStore排序后才能刷写到:StoreFile,但把数据保存在内存中可能会引起数据丢失,为了解决这个问题,数据会先写在一个叫做Write-Ahead logfile的文件中,然后再写入内存中.所以在系统出现故障的时候,数据可以通过这个日志文件重建.  
> HLog和MySQL binlog很相似,每个HRegion Server仅只包含有一个:HLog.  
### (6). Store
> Store是一个逻辑概念,StoreFile存储在Store中,一个Store对应HBase表中的一个列族(Column Family).  
### (7). MemStore
> 由于 StoreFile中的数据要求是有序的,所以数据是先存储在MemStore中,排好序后,等到达刷写时机才会刷写到StoreFile,每次刷写都会形成一个新的StoreFile.  
### (8). StoreFile
> 这是在磁盘上保存原始数据的实际的物理文件,是实际的存储文件.StoreFile是以HFile的形式存储在HDFS的.每个Store会有一个或多个StoreFile,数据在每个StoreFile中都是有序的. 
### (9). HFile
> 可以理解成一种文件格式(比如:txt),StoreFile是以HFile格式存储的.  
### (10). Region Server/HLog/Region/Store/MemStore/HFile之间关系.
!["HBase Region Server"](/assets/hbase/imgs/region-server-2.png)
