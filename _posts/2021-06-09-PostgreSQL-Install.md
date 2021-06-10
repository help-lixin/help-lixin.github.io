---
layout: post
title: 'PostgreSQL 免安装版(一)'
date: 2021-06-09
author: 李新
tags:  PostgreSQL
---

### (1). 下载PostgreSQL
```
# postgresql 下载地址
# https://www.enterprisedb.com/download-postgresql-binaries
>  wget https://get.enterprisedb.com/postgresql/postgresql-10.17-1-linux-x64-binaries.tar.gz
>  tar -zxvf postgresql-10.17-1-linux-x64-binaries.tar.gz 
```
### (2). 配置PostgreSQL目录
```
# 1. 进入解压后的pgsql
> cd pgsql

# 2. 创建数据目录
> mkdir data

# 3. 编辑系统环境
> vi  ~/.bash_profile
   export PGDATA=/home/lixin/pgsql/data
   export PATH=$PATH:home/lixin/pgsql/bin
```
### (3). 初始化数据库
```
[lixin@postgre-sql pgsql]$ pwd
/home/lixin/pgsql

[lixin@postgre-sql pgsql]$ ./bin/initdb -D ./data/ -E UTF8 --locale=C
The files belonging to this database system will be owned by user "lixin".
This user must also own the server process.

The database cluster will be initialized with locale "C".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory ./data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default timezone ... PRC
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    ./bin/pg_ctl -D ./data/ -l logfile start
```
### (4). 启动数据库
```
[lixin@postgre-sql pgsql]$ ./bin/pg_ctl -D ./data/ -l logfile start
waiting for server to start.... done
server started
```
### (5). 访问测试
```
[lixin@postgre-sql pgsql]$ ./bin/psql -h 127.0.0.1 -d postgres
psql.bin (10.17)
Type "help" for help.
# 1. 创建角色(并设置用户名和密码)
postgres=# CREATE ROLE devuser LOGIN PASSWORD 'lixin@111111';
CREATE ROLE

# 2. 创建数据库,并指定所属角色
postgres=# CREATE DATABASE test2 WITH OWNER = devuser ENCODING = 'UTF8';
CREATE DATABASE

# 3. 切换到数据库(test2)
postgres=# \c test2;
You are now connected to database "test2" as user "lixin".

# 4. 在test2库下,创建表(t_person)
test2=# create table t_person(id int, name varchar(20));
CREATE TABLE

# 5. 查看test2库下有哪些表
test2=# \dt
         List of relations
 Schema |   Name   | Type  | Owner
--------+----------+-------+-------
 public | t_person | table | lixin
 
# 6. 登录 
[lixin@postgre-sql pgsql]$ ./bin/psql -h 127.0.0.1 -d postgres -U lixin -W 

# 7. 查看当前用户
postgres=# select * from current_user;
 current_user
--------------
 lixin
(1 row)
```
### (6). 配置postgresql伴随开机启动
```
# vi /usr/lib/systemd/system/postgresql.service
[Unit]
Description=PostgreSQL database server
After=network.target

[Service]
Type=forking

User=lixin
Group=lixin

# Where to send early-startup messages from the server (before the logging
# options of postgresql.conf take effect)
# This is normally controlled by the global default set by systemd
# StandardOutput=syslog

# Disable OOM kill on the postmaster
OOMScoreAdjust=-1000
# ... but allow it still to be effective for child processes
# (note that these settings are ignored by Postgres releases before 9.5)
Environment=PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
Environment=PG_OOM_ADJUST_VALUE=0

# Maximum number of seconds pg_ctl will wait for postgres to start.  Note that
# PGSTARTTIMEOUT should be less than TimeoutSec value.
Environment=PGSTARTTIMEOUT=270

Environment=PGDATA=/home/lixin/pgsql/data


ExecStart=/home/lixin/pgsql/bin/pg_ctl start -D ${PGDATA} -s -w -t ${PGSTARTTIMEOUT}
ExecStop=/home/lixin/pgsql/bin/pg_ctl stop -D ${PGDATA} -s -m fast
ExecReload=/home/lixin/pgsql/bin/pg_ctl reload -D ${PGDATA} -s

# Give a reasonable amount of time for the server to start up/shut down.
# Ideally, the timeout for starting PostgreSQL server should be handled more
# nicely by pg_ctl in ExecStart, so keep its timeout smaller than this value.
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```
### (7). 启用postgresql
```
> systemctl daemon-reload
> systemctl enable postgresql
> systemctl start postgresql
> systemctl stop postgresql
```
### (8). 配置postgresql允许远程访问
```
[lixin@postgre-sql pgsql]$ pwd
/home/lixin/pgsql

#1. 配置pg_hba.conf
[lixin@postgre-sql pgsql]$ vi data/pg_hba.conf
host    all             all             0.0.0.0/0                 md5

#2. 修改postgresql.conf
[lixin@postgre-sql pgsql]$ vi data/postgresql.conf
listen_addresses = '*'          # what IP address(es) to listen on;

#3. 重启postgresql
```