---
layout: post
title: 'HBase 伪集群搭建(二)'
date: 2021-04-06
author: 李新
tags:  HBase
---

### (1). 概述
> Hadoop的安装,请翻看前面的内容,在这里我用的:Hbase-1.2.6(与Hadoop-2.7.5对应).

### (2). 配置conf/hbase-site.xml
```
<!-- 开启集群模式 -->
<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>

<!-- HDFS存储路径 -->
<property>
  <name>hbase.rootdir</name>
  <value>hdfs://lixin-macbook.local:9000/hbase</value>
</property>
```
### (3). 配置conf/hbase-env.sh
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
# 禁用自带的ZK
export HBASE_MANAGES_ZK=false
```
### (4). 检查安装结果
```
# 检查hbase目录是否创建成功
lixin-macbook:bin lixin$ hdfs dfs -ls /
drwxr-xr-x   - lixin supergroup          0 2021-04-06 20:17 /hbase

# 创建库
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/hbase-1.2.6/bin
lixin-macbook:bin lixin$ ./hbase shell
hbase(main):005:0> create 'testTable','testFamily'
hbase(main):007:0> list
TABLE
testTable
1 row(s) in 0.0150 seconds
=> ["testTable"]
```
### (5). HBase Dashboard
!["HBase WebUI界面"](/assets/hbase/imgs/hbase-web-ui.jpg)