---
layout: post
title: 'HBase + Phoenix搭配(四)'
date: 2021-04-06
author: 李新
tags:  HBase Phoenix
---

### (1). 下载Phoenix对应的版本
> 我的Hbase版本是1.4.13,所以下载:4.16.0  

!["Phoenix下载介绍"](/assets/hbase/imgs/phoneix.png)
### (2). 拷贝phoneix到hbase/lib目录下
```
lixin-macbook:phoenix-hbase-1.4-4.16.0-bin lixin$ cp phoenix-server-hbase-1.4-4.16.0.jar ~/Developer/hbase-1.4.13/lib/
```
### (3). 启动Phoneix客户端
```
# 查看当前所在的目录
lixin-macbook:bin lixin$ pwd
/Users/lixin/Downloads/phoenix-hbase-1.4-4.16.0-bin/bin

# sql客户端
lixin-macbook:bin lixin$ ./sqlline.py 127.0.0.1:2181
Setting property: [incremental, false]
Setting property: [isolation, TRANSACTION_READ_COMMITTED]
issuing: !connect jdbc:phoenix:127.0.0.1:2181 none none org.apache.phoenix.jdbc.PhoenixDriver
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/Users/lixin/Downloads/phoenix-hbase-1.4-4.16.0-bin/phoenix-client-hbase-1.4-4.16.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/Users/lixin/Developer/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
Connecting to jdbc:phoenix:127.0.0.1:2181
21/04/08 13:23:24 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Connected to: Phoenix (version 4.16)
Driver: PhoenixEmbeddedDriver (version 4.16)
Autocommit status: true
Transaction isolation: TRANSACTION_READ_COMMITTED
Building list of tables and columns for tab-completion (set fastconnect to true to skip)...
158/158 (100%) Done
Done
sqlline version 1.5.0
 
# 查看所有的表 
0: jdbc:phoenix:127.0.0.1:2181> !tables
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+
| TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE_NAME  | SELF_REFERENCING_COL_NAME  | REF_G |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+
|            | SYSTEM       | CATALOG     | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | CHILD_LINK  | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | FUNCTION    | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | LOG         | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | MUTEX       | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | SEQUENCE    | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | STATS       | SYSTEM TABLE  |          |            |                            |       |
|            | SYSTEM       | TASK        | SYSTEM TABLE  |          |            |                            |       |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+
0: jdbc:phoenix:127.0.0.1:2181>
```
### (4). 注意事项
> Phoenix不支持直接显示HBase Shell中创建的表格.  
> 原因很简单:当在Phoenix创建一张表时,Phoenix是将表进行了重组装. 
> 而对HBase Shell创建的表Phoenix并未进行加工,所以无法直接显示.  
> 如果需要将HBase Shell中创建的表格关联到Phoenix中查看,就需要在Phoenix中创建一个视图(View)做关联. 
