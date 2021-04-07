---
layout: post
title: 'HBase 基本操作(三)'
date: 2021-04-06
author: 李新
tags:  HBase
---

### (1). 创建表和列簇
> 需要注意:HBase在创建表时与传统关系型数据不同,传统关系型数据库在创建表时,必须要指定相关的属性,而HBase在创建表时,只需要指定列簇(而且列簇是必须要存在的),元数据信息也是在列簇上配置的.    
> 其实,可以把列簇理解成关系型数据库中的"表".   

```
# 创建表(user),并指定列簇:info
hbase(main):011:0> create 'user','info'
0 row(s) in 1.4630 seconds

=> Hbase::Table - user


```
### (2). 查看所有的表

```
# 查看库中所有的表
hbase(main):012:0> list
TABLE
user
1 row(s) in 0.0140 seconds

=> ["user"]
```
### (3). 查看表属性
```
hbase(main):014:0> describe 'user'
Table user is ENABLED
user

# 列簇描述信息
COLUMN FAMILIES DESCRIPTION
{
	NAME => 'info',                      # 列簇名称
	BLOOMFILTER => 'ROW', 
	VERSIONS => '1',                     # 只存取一个版本的数据
	IN_MEMORY => 'false', 
	KEEP_DELETED_CELLS => 'FALSE', 
	DATA_BLOCK_ENCODING => 'NONE', 
	TTL => 'FOREVER', 
	COMPRESSION => 'NONE', 
	MIN_VERSIONS => '0', 
	BLOCKCACHE => 'true', 
	BLOCKSIZE => '65536', 
	REPLICATION_SCOPE => '0'
}
1 row(s) in 0.0960 seconds

# 修改表结构,支持存储5个版本的数据(建议先关闭表:disable 'user')
hbase(main):050:0> alter 'user',{NAME=>'info',VERSIONS=>5}
Updating all regions with the new schema...
1/1 regions updated.
Done.
0 row(s) in 1.9310 seconds

# 再次查看表结构(VERSIONS => '5')
hbase(main):051:0> desc 'user'
Table user is DISABLED
user
COLUMN FAMILIES DESCRIPTION
{NAME => 'info', BLOOMFILTER => 'ROW', VERSIONS => '5', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_
ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65
536', REPLICATION_SCOPE => '0'}
1 row(s) in 0.0120 seconds

```
### (4). 插入数据
```
# 往表(user)中插入数据,rowkey为1,列簇为:info,单元格的名称为:name,单元格的数据为:张三
put 'user','1','info:name','张三'
put 'user','1','info:sex','男'
put 'user','1','info:pwd','123'
put 'user','2','info:name','李四'
put 'user','2','info:sex','男'
put 'user','2','info:pwd','321'

# 修改单元格信息
put 'user','1','info:name','张三丰'
```
### (5). scan扫描表数据
```
# SELCT * FROM user WHERE id >= 1 AND id < 3;
# STARTROW : 大于等于
# ENDROW   : 小于
# VERSIONS : 显示多少版本以内的数据
# hbase(main):015:0> scan 'user',{VERSIONS=>5}

hbase(main):024:0> scan 'user',{STARTROW=>'1',ENDROW=>'3'}

# ROW           显示的是:rowkey
# COLUMN+CELL   显示的是:列簇/列/时间/单元格

ROW                             COLUMN+CELL
 1                              column=info:name, timestamp=1617781212720, value=\xE5\xBC\xA0\xE4\xB8\x89
 1                              column=info:pwd, timestamp=1617781213784, value=123
 1                              column=info:sex, timestamp=1617781212903, value=\xE7\x94\xB7
 2                              column=info:name, timestamp=1617781220717, value=\xE6\x9D\x8E\xE5\x9B\x9B
 2                              column=info:pwd, timestamp=1617781221447, value=321
 2                              column=info:sex, timestamp=1617781220817, value=\xE7\x94\xB7
2 row(s) in 0.0700 seconds
```

### (6). get获取单元格数据
```
# 获取单元格的数据,并显示最近5个版本的
hbase(main):011:0> get 'user','1',{COLUMN=>'info:name',VERSIONS=>5}
COLUMN                          CELL
 info:name                      timestamp=1617782950726, value=\xE5\xBC\xA0\xE4\xB8\x89\xE4\xB8\xB0
 info:name                      timestamp=1617782922404, value=\xE5\xBC\xA0\xE4\xB8\x89
```
### (7). 删除单元络数据
```
# UPDATE user SET name=NULL WHERE id = 1;
# 删除表(user),rowkey=1下列簇为info:name的数据.
hbase(main):016:0> delete 'user','1','info:name'
0 row(s) in 0.1220 seconds

# 验证,确实是少了:info:name列簇
hbase(main):017:0> get 'user','1'
COLUMN                          CELL
 info:pwd                       timestamp=1617782922531, value=123
 info:sex                       timestamp=1617782922475, value=\xE7\x94\xB7
2 row(s) in 0.0340 seconds

# 在HBase里删除记录,并不是真正的删除了数据,而是打了一个标记(DeleteColumn)
# 然后,HBase再定期的去清理这些被删除的记录.
# 通过scan扫描出:DeleteColumn的列.
hbase(main):020:0> scan 'user',{RAW=>true,VERSIONS=>5}
ROW                             COLUMN+CELL
 1                              column=info:name, timestamp=1617784279842, type=DeleteColumn
 1                              column=info:name, timestamp=1617782950726, value=\xE5\xBC\xA0\xE4\xB8\x89\xE4\xB8\xB0
 1                              column=info:name, timestamp=1617782922404, value=\xE5\xBC\xA0\xE4\xB8\x89
 1                              column=info:pwd, timestamp=1617782922531, value=123
 1                              column=info:sex, timestamp=1617782922475, value=\xE7\x94\xB7
 2                              column=info:name, timestamp=1617782922597, value=\xE6\x9D\x8E\xE5\x9B\x9B
 2                              column=info:pwd, timestamp=1617782923256, value=321
 2                              column=info:sex, timestamp=1617782922666, value=\xE7\x94\xB7
```
### (8). 删除整行数据
> delete只能删除单元格的数据,我们能否把整行给删除呢?

```
# 删除整行前,查看下数据分布(rowkey=[1,2])
hbase(main):021:0> scan 'user'
ROW                             COLUMN+CELL
 1                              column=info:pwd, timestamp=1617782922531, value=123
 1                              column=info:sex, timestamp=1617782922475, value=\xE7\x94\xB7
 2                              column=info:name, timestamp=1617782922597, value=\xE6\x9D\x8E\xE5\x9B\x9B
 2                              column=info:pwd, timestamp=1617782923256, value=321
 2                              column=info:sex, timestamp=1617782922666, value=\xE7\x94\xB7
2 row(s) in 0.0140 seconds

# DELETE user WHERE id = 1;
# 删除表(user)中rowkey=1的数据
hbase(main):022:0> deleteall 'user','1'
0 row(s) in 0.0270 seconds

# 验证结果
hbase(main):023:0> scan 'user'
ROW                             COLUMN+CELL
 2                              column=info:name, timestamp=1617782922597, value=\xE6\x9D\x8E\xE5\x9B\x9B
 2                              column=info:pwd, timestamp=1617782923256, value=321
 2                              column=info:sex, timestamp=1617782922666, value=\xE7\x94\xB7
1 row(s) in 0.0170 seconds
```
### (9). 停(启用)用表
> 在HBase里对表的结构进行调整时,都需要先停用表,然后,更改,再启用.   
> 停用表这个过程视工作情况而定,当停用表时,HBase要通知所有的RegionServer下线这张表. 

```
# 停用表
hbase(main):024:0> disable 'user'
0 row(s) in 2.3600 seconds

# 查看表信息
hbase(main):028:0> describe 'user'
Table user is DISABLED   # 停用状态
user
COLUMN FAMILIES DESCRIPTION
{NAME => 'info', BLOOMFILTER => 'ROW', VERSIONS => '5', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_
ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65
536', REPLICATION_SCOPE => '0'}

# 在停用状态下,进行检索会提示:user is disabled.
hbase(main):032:0> scan 'user'
ROW                             COLUMN+CELL
ERROR: user is disabled.

# 启用表
hbase(main):029:0> enable 'user'
0 row(s) in 1.2670 seconds

# 查看表信处
hbase(main):030:0> describe 'user'
Table user is ENABLED  # 启用状态
user
COLUMN FAMILIES DESCRIPTION
{NAME => 'info', BLOOMFILTER => 'ROW', VERSIONS => '5', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_
ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65
536', REPLICATION_SCOPE => '0'}
```
### (10). 删除表
```
# 查看user表的信息
hbase(main):033:0> describe 'user'
Table user is DISABLED   # 禁用状态
user
COLUMN FAMILIES DESCRIPTION
{NAME => 'info', BLOOMFILTER => 'ROW', VERSIONS => '5', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_
ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65
536', REPLICATION_SCOPE => '0'}
1 row(s) in 0.0320 seconds

# 删除表(user)
hbase(main):034:0> drop 'user'
0 row(s) in 1.2810 seconds

# 查看所有的表
hbase(main):035:0> list
TABLE
0 row(s) in 0.0180 seconds
```
### (11). 总结
> 在这里我只列出开发需要的一些命令,运维方面的命令,暂时不学习,因为,核心是要找到WAL功能的扩展点.  
### (12). 

### (13). 
