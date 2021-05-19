---
layout: post
title: 'Docker 安装 ClickHouse(二)'
date: 2021-05-16
author: 李新
tags:  ClickHouse
---

### (1). 拉取镜像
```
# 拉取镜像
$ docker pull yandex/clickhouse-server
$ docker pull yandex/clickhouse-clinet
```

### (2). 启动容器
```
# 基于镜像创建容器运行(指定本地磁盘与容器目录的挂载映射)
#    --volume=/Users/lixin/DockerWorkspace/clickhouse/clickhouse-test-db
$ docker run -d --name clickhouse-test-server --ulimit nofile=262144:262144 --volume=/Users/lixin/DockerWorkspace/clickhouse/clickhouse-test-db:/var/lib/clickhouse yandex/clickhouse-server
```

### (3). 进入容器内部
```
# 1. 进入容器内部
$ docker exec -it clickhouse-test-server /bin/bash

# 2. 在容器内部,运行clickhouse-client
#   root@9e40ca366829:注意:这里已经是进入到了容器内部了.
root@9e40ca366829:/# clickhouse-client -m
ClickHouse client version 21.3.5.42 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.3.5 revision 54447.

9e40ca366829 :)
```

### (4). clickhouse-client常用参数

```
# clickhouse-client命令常用参数
clickhouse-client
    --host, -h     	：服务端host名称，默认 localhost
    --port         	：连接端口，默认9000
    --user, -u     	：用户名，默认 default
    --password     	：密码，默认空
    --query, -q    	：非交互模式下的查询语句
    --database, -d 	：默认当前操作的数据库，默认default
    --multiline, -m ：允许多行语句查询，在clickhouse中默认回车即为sql结束，可使用该参数多行输入
    --format, -f		：使用指定的默认格式输出结果      csv,以逗号分隔
    --time, -t			：非交互模式下会打印查询执行的时间
    --stacktrace		：出现异常会打印堆栈跟踪信息
    --config-file		：配置文件名称
```


### (5). clickhouse测试

```
# 1. 查看有哪些数据库
9e40ca366829 :) show databases;
SHOW DATABASES
Query id: 9df9088d-f36c-4da1-86a5-56e187d160dd

┌─name────┐
│ default │
│ system  │
└─────────┘

# 2. 选择数据库
9e40ca366829 :) use system;
Ok.

# 3. 常用函数
9e40ca366829 :) SELECT * FROM functions;

┌─name────────────────────────────────────────┬─is_aggregate─┬─case_insensitive─┬─alias_to─────────────────┐
│ aes_encrypt_mysql                           │            0 │                0 │                          │
│ decrypt                                     │            0 │                0 │                          │
│ wordShingleMinHashArg                       │            0 │                0 │                          │
│ ngramMinHashArgUTF8                         │            0 │                0 │                          │
│ ngramMinHashArgCaseInsensitive              │            0 │                0 │                          │
│ wordShingleMinHashCaseInsensitiveUTF8       │            0 │                0 │                          │
│ wordShingleMinHashCaseInsensitive           │            0 │                0 │                          │


# 3. 查看当前所在的数据库
9e40ca366829 :) SELECT currentDatabase();

┌─currentDatabase()─┐
│ system            │
└───────────────────┘
```

### (6). clickhouse非交互测试

```
# 1. 以交互式的方式进入clickhouse
root@9e40ca366829:/# clickhouse-client  -m
ClickHouse client version 21.3.5.42 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.3.5 revision 54447.

# 2. 创建库
9e40ca366829 :) CREATE DATABASE test5;
Ok.

9e40ca366829 :) USE test5;
Ok.

# 3. 创建表
9e40ca366829 :) CREATE TABLE t_test(id Int8,name String,age UInt8,gender String)ENGINE=Log;
Ok.


##################################################################
# 4. 在clickhouse机器上,创建csv文件.
root@9e40ca366829:~# cat test.csv
1,张三,20,M
2,李四,21,F
3,赵六,22,M
4,王五,23,F

# 支持导入的数所格式: https://clickhouse.tech/docs/en/interfaces/formats/#formats
# 5. 以非交互的方式,导入数据
root@9e40ca366829:~# clickhouse-client   --format_csv_delimiter="," -n -q "USE test5;INSERT INTO t_test FORMAT CSV" < /var/lib/clickhouse/test.csv

# 6. 验证数据
# -n : 允许多Statement执行
# -q : 指定SQL语句
root@9e40ca366829:~# clickhouse-client -n -q "USE test5;SELECT * FROM t_test;"
1	张三	  20	M
2	李四	  21	F
3	赵六	  22	M
4	王五	  23	F
```