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
### (3). 配置hbase-site.xml
```
# hbase配置:hbase-site.xml
<!--phoenix -->
<property>
   <name>hbase.regionserver.wal.codec</name>
   <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
<property>
    <name>hbase.rpc.timeout</name>
    <value>60000000</value>
</property>
<property>
    <name>hbase.client.scanner.timeout.period</name>
    <value>60000000</value>
</property>
<property>
    <name>phoenix.query.timeoutMs</name>
    <value>60000000</value>
</property>
```
### (4). 启动Phoneix客户端
> 注意事项:  
> Phoenix不支持直接显示HBase Shell中创建的表格.  
> 原因很简单:当在Phoenix创建一张表时,Phoenix是将表进行了重组装. 
> 而对HBase Shell创建的表Phoenix并未进行加工,所以无法直接显示.  
> 如果需要将HBase Shell中创建的表格关联到Phoenix中查看,就需要在Phoenix中创建一个视图(View)做关联. 


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
### (5). phoenix CRUD操作
```
# 1. 创建表
0: jdbc:phoenix:localhost:2181> create table info (id varchar primary key , name varchar, age integer);
No rows affected (1.552 seconds)

# 2. hbase shell查看表信息
hbase(main):005:0> describe 'INFO'
Table INFO is ENABLED
INFO, {TABLE_ATTRIBUTES => {coprocessor$1 => '|org.apache.phoenix.coprocessor.ScanRegionObserver|805306366|', coprocesso
r$2 => '|org.apache.phoenix.coprocessor.UngroupedAggregateRegionObserver|805306366|', coprocessor$3 => '|org.apache.phoe
nix.coprocessor.GroupedAggregateRegionObserver|805306366|', coprocessor$4 => '|org.apache.phoenix.coprocessor.ServerCach
ingEndpointImpl|805306366|', coprocessor$5 => '|org.apache.phoenix.hbase.index.IndexRegionObserver|805306366|org.apache.
hadoop.hbase.index.codec.class=org.apache.phoenix.index.PhoenixIndexCodec,index.builder=org.apache.phoenix.index.Phoenix
IndexBuilder', coprocessor$6 => '|org.apache.phoenix.coprocessor.PhoenixTTLRegionObserver|805306364|'}
COLUMN FAMILIES DESCRIPTION  # 列簇描述信息
# 列簇为:0,复制状态为:1
{NAME => '0', BLOOMFILTER => 'NONE', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_EN
CODING => 'FAST_DIFF', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE =>
'65536', REPLICATION_SCOPE => '0'}

# 3. 查看所有的表
0: jdbc:phoenix:localhost:2181> !tables
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+
| TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE_NAME  | SELF_REFERENCING_COL_NAME  | REF_G |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+
|            |              | INFO        | TABLE         |          |            |                            |       |
+------------+--------------+-------------+---------------+----------+------------+----------------------------+-------+

# 4. 保存或插入数据
0: jdbc:phoenix:localhost:2181> UPSERT INTO info VALUES('1001','zhgnsan',25);
1 row affected (0.06 seconds)
0: jdbc:phoenix:localhost:2181> UPSERT INTO info VALUES('1002','lishi',27);
1 row affected (0.007 seconds)


# 5. 检索表
0: jdbc:phoenix:localhost:2181> SELECT * FROM info;
+-------+----------+------+
|  ID   |   NAME   | AGE  |
+-------+----------+------+
| 1001  | zhgnsan  | 25   |
| 1002  | lishi    | 27   |
+-------+----------+------+

# 6. 检索表并排序
0: jdbc:phoenix:localhost:2181> SELECT * FROM info ORDER BY age DESC;
+-------+----------+------+
|  ID   |   NAME   | AGE  |
+-------+----------+------+
| 1002  | lishi    | 27   |
| 1001  | zhgnsan  | 25   |
+-------+----------+------+
```

### (6). phoenix索引
> 全局索引:   
> 全局索引适合*读多写少*的场景.如果使用全局索引,读数据基本不损耗性能,所有的性能损耗都来源于写数据.数据表的添加、删除和修改都会更新相关的索引表(数据删除了,索引表中的数据也会删除,数据增加了,索引表的数据也会增加).   
> 注意:对于全局索引在默认情况下,在查询语句中检索的列如果不在索引表中,Phoenix不会使用索引表将,除非强制使用hint.   

```
# 1. 创建全局索引
0: jdbc:phoenix:localhost:2181> create index name_index on info(name);

# 2. 底层原理是什么?
# 创建全局索引时,会多增加一张表(表名称就是索引名称)
hbase(main):005:0> list
TABLE
INFO
NAME_INDEX

# 3. 查看索引内容(噢!原来是以空间换时间,创建全局索引时,会把表里的数据提取出来,另建一张表.)
#    自然而然,如果表比较大,建立索引是比较耗时,而且,占据的空间也比较大.
#    读取时可以直接走索引表,获得rowkey和数据,而写的时候要写info表name_index表
hbase(main):004:0> scan 'NAME_INDEX'
ROW                             COLUMN+CELL
 lishi\x001002                  column=0:\x00\x00\x00\x00, timestamp=1617881825598, value=\x01
 zhgnsan\x001001                column=0:\x00\x00\x00\x00, timestamp=1617881815888, value=\x01
 

# 4. 查询返回的列和条件,在索引里能的情况下,会走索引覆盖,否则,全表扫描 
#    这个例子,就是典型的全表扫描.
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT name FROM info WHERE age > 25;
+----------------------------------------------------------------+-----------------+----------------+--------------+
|                              PLAN                              | EST_BYTES_READ  | EST_ROWS_READ  | EST_INFO_TS  |
+----------------------------------------------------------------+-----------------+----------------+--------------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN FULL SCAN OVER INFO  | null            | null           | null         |
|     SERVER FILTER BY AGE > 25                                  | null            | null           | null         |
+----------------------------------------------------------------+-----------------+----------------+--------------+

# 5. 这个例子,就是典型的全表扫描(虽然条件name建了索引,但是,SELECT *)
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT *  FROM info WHERE name='zhgnsa';
+----------------------------------------------------------------+-----------------+----------------+--------------+
|                              PLAN                              | EST_BYTES_READ  | EST_ROWS_READ  | EST_INFO_TS  |
+----------------------------------------------------------------+-----------------+----------------+--------------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN FULL SCAN OVER INFO  | null            | null           | null         |
|     SERVER FILTER BY NAME = 'zhgnsa'                           | null            | null           | null         |
+----------------------------------------------------------------+-----------------+----------------+--------------+

# 6. 这个例子走了索引(NAME_INDEX)
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT name FROM info WHERE name='zhgnsa';
+----------------------------------------------------------------------------------+-----------------+----------------+------+
|                                       PLAN                                       | EST_BYTES_READ  | EST_ROWS_READ  | EST_ |
+----------------------------------------------------------------------------------+-----------------+----------------+------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN RANGE SCAN OVER NAME_INDEX ['zhgnsa']  | null            | null           | null |
|     SERVER FILTER BY FIRST KEY ONLY                                              | null            | null           | null |
+----------------------------------------------------------------------------------+-----------------+----------------+------+

# 7. 删除全局索引
0: jdbc:phoenix:localhost:2181> DROP INDEX name_index ON info;

# 8. 查看所有的表
hbase(main):006:0> list
TABLE
INFO
```


> 本地索引:  
> 本地索引适合*写多读少*的场景,或者存储空间有限的场景.和全局索引一样,Phoenix也会在查询的时候自动选择是否使用本地索引.本地索引因为索引数据和原数据存储在同一台机器上,避免网络数据传输的开销,所以更适合写多的场景.由于无法提前确定数据在哪个Region上,所以在读数据的时候,需要检查每个Region上的数据从而带来一些性能损耗.   

```
# 1. 创建本地索引
0: jdbc:phoenix:localhost:2181> CREATE LOCAL INDEX name_local_index ON info(name);

# 2. 创建本地索引在hbase不存有索引表
hbase(main):008:0> list
TABLE
INFO

# 3. 本地索引,虽然不存在一张表,但是,却在表上创建了一个新列簇.通过这个列簇来保存索引信息.
#    明显:比较适合写多,读少.
hbase(main):010:0> scan 'INFO'
ROW                             COLUMN+CELL
 \x00\x00lishi\x001002          column=L#0:\x00\x00\x00\x00, timestamp=1617881825598, value=x
 \x00\x00zhgnsan\x001001        column=L#0:\x00\x00\x00\x00, timestamp=1617881815888, value=x
 1001                           column=0:\x00\x00\x00\x00, timestamp=1617881815888, value=x
 1001                           column=0:\x80\x0B, timestamp=1617881815888, value=zhgnsan
 1001                           column=0:\x80\x0C, timestamp=1617881815888, value=\x80\x00\x00\x19
 1002                           column=0:\x00\x00\x00\x00, timestamp=1617881825598, value=x
 1002                           column=0:\x80\x0B, timestamp=1617881825598, value=lishi
 1002                           column=0:\x80\x0C, timestamp=1617881825598, value=\x80\x00\x00\x1B
4 row(s) in 0.0320 seconds


# 4. 使用本地索引案例
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT * FROM info WHERE name = 'zhgnsan';
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
|                                     PLAN                                      | EST_BYTES_READ  | EST_ROWS_READ  | EST_INF |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN RANGE SCAN OVER INFO [1,'zhgnsan']  | null            | null           | null    |
|     SERVER FILTER BY FIRST KEY ONLY                                           | null            | null           | null    |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+

# 5. 使用本地索引案例
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT * FROM info WHERE name = 'zhgnsa%';
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
|                                     PLAN                                      | EST_BYTES_READ  | EST_ROWS_READ  | EST_INF |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN RANGE SCAN OVER INFO [1,'zhgnsa%']  | null            | null           | null    |
|     SERVER FILTER BY FIRST KEY ONLY                                           | null            | null           | null    |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+

# 6. 使用本地索引案例
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT name  FROM info WHERE name = 'zhgnsan';
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
|                                     PLAN                                      | EST_BYTES_READ  | EST_ROWS_READ  | EST_INF |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN RANGE SCAN OVER INFO [1,'zhgnsan']  | null            | null           | null    |
|     SERVER FILTER BY FIRST KEY ONLY                                           | null            | null           | null    |
+-------------------------------------------------------------------------------+-----------------+----------------+---------+

# 7. 全表扫描
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT *  FROM info WHERE age > 25;
+----------------------------------------------------------------+-----------------+----------------+--------------+
|                              PLAN                              | EST_BYTES_READ  | EST_ROWS_READ  | EST_INFO_TS  |
+----------------------------------------------------------------+-----------------+----------------+--------------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN FULL SCAN OVER INFO  | null            | null           | null         |
|     SERVER FILTER BY AGE > 25                                  | null            | null           | null         |
+----------------------------------------------------------------+-----------------+----------------+--------------+

# 8. 全表扫描
0: jdbc:phoenix:localhost:2181> EXPLAIN SELECT *  FROM info;
+----------------------------------------------------------------+-----------------+----------------+--------------+
|                              PLAN                              | EST_BYTES_READ  | EST_ROWS_READ  | EST_INFO_TS  |
+----------------------------------------------------------------+-----------------+----------------+--------------+
| CLIENT 1-CHUNK PARALLEL 1-WAY ROUND ROBIN FULL SCAN OVER INFO  | null            | null           | null         |
+----------------------------------------------------------------+-----------------+----------------+--------------+
```

> 不论是全局索引还是本地索引,从Phoneix查询,都会存在一张表.

```
0: jdbc:phoenix:localhost:2181> !tables
+------------+--------------+-------------------+---------------+----------+------------+----------------------------+-------+
| TABLE_CAT  | TABLE_SCHEM  |    TABLE_NAME     |  TABLE_TYPE   | REMARKS  | TYPE_NAME  | SELF_REFERENCING_COL_NAME  | REF_G |
+------------+--------------+-------------------+---------------+----------+------------+----------------------------+-------+
|            |              | NAME_INDEX        | INDEX         |          |            |                            |       |
|            |              | NAME_LOCAL_INDEX  | INDEX         |          |            |                            |       |
|            |              | INFO              | TABLE         |          |            |                            |       |
+------------+--------------+-------------------+---------------+----------+------------+----------------------------+-------+
```