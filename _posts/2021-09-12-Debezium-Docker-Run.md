---
layout: post
title: '在Docker运行Debezium(二)' 
date: 2021-09-12
author: 李新
tags:  Debezium
---

### (1). 概述
前面对debezium进行了一个大概的了解,在这一里,通过docker来运行一个简单的案例.

### (2). Docker运行Debezium后架构图如下
!["在Docker运行Debezium"](/assets/debezium/imgs/docker-run-debezium-example.jpg)
### (3). 提前拉取镜像
```
lixin-macbook:~ lixin$ docker pull debezium/zookeeper:1.2
lixin-macbook:~ lixin$ docker pull debezium/kafka:1.2
lixin-macbook:~ lixin$ docker pull debezium/example-mysql:1.2

lixin-macbook:~ lixin$ docker images|grep debezium
debezium/example-mysql                   1.2                                              8eca9c821ad6   7 weeks ago     448MB
debezium/connect                         1.2                                              db770c03ced5   7 months ago    692MB
debezium/zookeeper                       1.2                                              94dc5bf38ce0   7 months ago    488MB
debezium/kafka                           1.2                                              c2829a011b30   7 months ago    657MB
```
### (4). 启动zk
```
lixin-macbook:~ lixin$ docker run -d -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.2
```
### (5). 启动kakfa
```
lixin-macbook:~ lixin$ docker run -d -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.2
```
### (6). 启动mysql
```
lixin-macbook:~ lixin$ docker run -d -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.2
```
### (7). 启动kafka-connect
```
lixin-macbook:~ lixin$ docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.2
```
### (8). 检查启动的容器
```
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS         PORTS                                                                                                                                                 NAMES
8a218a19dcbc   debezium/connect:1.2         "/docker-entrypoint.…"   3 minutes ago   Up 3 minutes   8778/tcp, 9092/tcp, 0.0.0.0:8083->8083/tcp, :::8083->8083/tcp, 9779/tcp                                                                               connect
8cecb93b3df6   debezium/example-mysql:1.2   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp                                                                                                  mysql
0a7816254c1f   debezium/kafka:1.2           "/docker-entrypoint.…"   3 minutes ago   Up 3 minutes   8778/tcp, 9779/tcp, 0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                                                                         kafka
8216c51e9051   debezium/zookeeper:1.2       "/docker-entrypoint.…"   3 minutes ago   Up 3 minutes   0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 0.0.0.0:2888->2888/tcp, :::2888->2888/tcp, 8778/tcp, 0.0.0.0:3888->3888/tcp, :::3888->3888/tcp, 9779/tcp   zookeeper
```
### (9). 检查connect
```
# 查看connect信息
lixin-macbook:~ lixin$ curl -H "Accept:application/json" localhost:8083/
{
	"version":"2.5.0",
	"commit":"66563e712b0b9f84",
	"kafka_cluster_id":"Y7k-bwb6T6y_RkiZkphQjg"
}

# 查看所有的connectors
lixin-macbook:~ lixin$ curl -H "Accept:application/json" localhost:8083/connectors/
[]
```
### (10). 注册一个MySQL connector
```
# *************************************************************************
# 1. 注册一个:MySQL connector,注意:此时:kafka-connect会有大量的日志输出
# *************************************************************************
lixin-macbook:~ lixin$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
HTTP/1.1 201 Created
Date: Sun, 12 Sep 2021 02:27:19 GMT
Location: http://localhost:8083/connectors/inventory-connector
Content-Type: application/json
Content-Length: 487
Server: Jetty(9.4.24.v20191120)
{
	"name": "inventory-connector",
	"config": {
		"connector.class": "io.debezium.connector.mysql.MySqlConnector",
		"tasks.max": "1",
		"database.hostname": "mysql",
		"database.port": "3306",
		"database.user": "debezium",
		"database.password": "dbz",
		"database.server.id": "184054",
		"database.server.name": "dbserver1",
		"database.whitelist": "inventory",
		"database.history.kafka.bootstrap.servers": "kafka:9092",
		"database.history.kafka.topic": "dbhistory.inventory",
		"name": "inventory-connector"
	},
	"tasks": [],
	"type": "source"
}


# *************************************************************************
# 2. 再次查看:inventory-connector
# *************************************************************************
lixin-macbook:~ lixin$ curl -H "Accept:application/json" localhost:8083/connectors/inventory-connector
{
	"name": "inventory-connector",
	"config": {
		"connector.class": "io.debezium.connector.mysql.MySqlConnector",
		"database.user": "debezium",
		"database.server.id": "184054",
		"tasks.max": "1",
		"database.hostname": "mysql",
		"database.password": "dbz",
		"database.history.kafka.bootstrap.servers": "kafka:9092",
		"database.history.kafka.topic": "dbhistory.inventory",
		"name": "inventory-connector",
		"database.server.name": "dbserver1",
		"database.whitelist": "inventory",
		"database.port": "3306"
	},
	"tasks": [{
		"connector": "inventory-connector",
		"task": 0
	}],
	"type": "source"
}

```
### (11). 查看下采集的binlog
```
lixin-macbook:~ lixin$ docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:1.2 watch-topic -a -k dbserver1.inventory.customers
WARNING: Using default BROKER_ID=1, which is valid only for non-clustered installations.
Using ZOOKEEPER_CONNECT=172.17.0.2:2181
Using KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.0.6:9092
Using KAFKA_BROKER=172.17.0.3:9092
Contents of topic dbserver1.inventory.customers:
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"1.2.5.Final","connector":"mysql","name":"dbserver1","ts_ms":0,"snapshot":"true","db":"inventory","table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1631413644935,"transaction":null}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1002}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1002,"first_name":"George","last_name":"Bailey","email":"gbailey@foobar.com"},"source":{"version":"1.2.5.Final","connector":"mysql","name":"dbserver1","ts_ms":0,"snapshot":"true","db":"inventory","table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1631413644935,"transaction":null}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1003}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"},"source":{"version":"1.2.5.Final","connector":"mysql","name":"dbserver1","ts_ms":0,"snapshot":"true","db":"inventory","table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1631413644935,"transaction":null}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope"},"payload":{"before":null,"after":{"id":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"},"source":{"version":"1.2.5.Final","connector":"mysql","name":"dbserver1","ts_ms":0,"snapshot":"true","db":"inventory","table":"customers","server_id":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1631413644935,"transaction":null}}
```
### (12). 查看某一项具体信息
```
{
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "int32",
			"optional": false,
			"field": "id"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Key"
	},
	"payload": {
		"id": 1001
	}
} {
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "before"
		}, {
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "after"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "version"
			}, {
				"type": "string",
				"optional": false,
				"field": "connector"
			}, {
				"type": "string",
				"optional": false,
				"field": "name"
			}, {
				"type": "int64",
				"optional": false,
				"field": "ts_ms"
			}, {
				"type": "string",
				"optional": true,
				"name": "io.debezium.data.Enum",
				"version": 1,
				"parameters": {
					"allowed": "true,last,false"
				},
				"default": "false",
				"field": "snapshot"
			}, {
				"type": "string",
				"optional": false,
				"field": "db"
			}, {
				"type": "string",
				"optional": true,
				"field": "table"
			}, {
				"type": "int64",
				"optional": false,
				"field": "server_id"
			}, {
				"type": "string",
				"optional": true,
				"field": "gtid"
			}, {
				"type": "string",
				"optional": false,
				"field": "file"
			}, {
				"type": "int64",
				"optional": false,
				"field": "pos"
			}, {
				"type": "int32",
				"optional": false,
				"field": "row"
			}, {
				"type": "int64",
				"optional": true,
				"field": "thread"
			}, {
				"type": "string",
				"optional": true,
				"field": "query"
			}],
			"optional": false,
			"name": "io.debezium.connector.mysql.Source",
			"field": "source"
		}, {
			"type": "string",
			"optional": false,
			"field": "op"
		}, {
			"type": "int64",
			"optional": true,
			"field": "ts_ms"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "id"
			}, {
				"type": "int64",
				"optional": false,
				"field": "total_order"
			}, {
				"type": "int64",
				"optional": false,
				"field": "data_collection_order"
			}],
			"optional": true,
			"field": "transaction"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Envelope"
	},
	"payload": {
		"before": null,                     # 因为是insert,所以before为空
		"after": {                          # 插入的数据
			"id": 1001,
			"first_name": "Sally",
			"last_name": "Thomas",
			"email": "sally.thomas@acme.com"
		},
		"source": {
			"version": "1.2.5.Final",
			"connector": "mysql",
			"name": "dbserver1",
			"ts_ms": 0,
			"snapshot": "true",
			"db": "inventory",
			"table": "customers",
			"server_id": 0,                                # 每条记录会对应server_id
			"gtid": null,                                  # 每条记录会寺应gtid(解决跨区域回环问题)
			"file": "mysql-bin.000003",
			"pos": 154,
			"row": 0,
			"thread": null,
			"query": null
		},
		"op": "c",
		"ts_ms": 1631413644935,
		"transaction": null
	}
}
```
### (13). 再次查看docker容器运行
```
# 其中有一个kafka是消费者监听来着的.
lixin-macbook:~ lixin$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                 NAMES
75cbd9048d54   debezium/kafka:1.2           "/docker-entrypoint.…"   7 minutes ago    Up 7 minutes    8778/tcp, 9092/tcp, 9779/tcp                                                                                                                          watcher
8a218a19dcbc   debezium/connect:1.2         "/docker-entrypoint.…"   21 minutes ago   Up 21 minutes   8778/tcp, 9092/tcp, 0.0.0.0:8083->8083/tcp, :::8083->8083/tcp, 9779/tcp                                                                               connect
8cecb93b3df6   debezium/example-mysql:1.2   "docker-entrypoint.s…"   21 minutes ago   Up 21 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp                                                                                                  mysql
0a7816254c1f   debezium/kafka:1.2           "/docker-entrypoint.…"   21 minutes ago   Up 21 minutes   8778/tcp, 9779/tcp, 0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                                                                         kafka
8216c51e9051   debezium/zookeeper:1.2       "/docker-entrypoint.…"   22 minutes ago   Up 22 minutes   0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 0.0.0.0:2888->2888/tcp, :::2888->2888/tcp, 8778/tcp, 0.0.0.0:3888->3888/tcp, :::3888->3888/tcp, 9779/tcp   zookeeper
```
### (14). 进入mysql,创建数据,查看kafka消费者是否能实时监听
```
# 1. 查看mysql在docker下的容器id
lixin-macbook:~ lixin$ docker ps|grep mysql
8cecb93b3df6   debezium/example-mysql:1.2   "docker-entrypoint.s…"   44 minutes ago   Up 44 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp                                                                                                  mysql

# 2. 进入容器内部
lixin-macbook:~ lixin$ docker exec -it 8cecb93b3df6 /bin/bash

# 3. 运行mysql客户端(mysqluser/mysqlpw)
root@8cecb93b3df6:/# mysql -u mysqluser -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.35-log MySQL Community Server (GPL)
Copyright (c) 2000, 2021, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 4. 查看有哪些库
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| inventory          |
+--------------------+
2 rows in set (0.01 sec)

# 5. 进入:inventory库
mysql> USE inventory;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

# 6. 查看有哪些库
mysql> SHOW TABLES;
+---------------------+
| Tables_in_inventory |
+---------------------+
| addresses           |
| customers           |
| geom                |
| orders              |
| products            |
| products_on_hand    |
+---------------------+
6 rows in set (0.00 sec)

# 7. 查看表结构信息
mysql> DESC customers;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| first_name | varchar(255) | NO   |     | NULL    |                |
| last_name  | varchar(255) | NO   |     | NULL    |                |
| email      | varchar(255) | NO   | UNI | NULL    |                |
+------------+--------------+------+-----+---------+----------------+

# 8. 插入数据,注意:观察kafka consumer
mysql> INSERT INTO customers(first_name,last_name,email) VALUES("xin","lixin","test@126.com");
Query OK, 1 row affected (0.01 sec)

# 9. 更新刚插入的数据
mysql> UPDATE customers SET first_name = "xin--" WHERE id = 1005;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
### (15). 查看kafka-consumer插入的数据
```
# 1. 插入数据的数据结构
{
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "int32",
			"optional": false,
			"field": "id"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Key"
	},
	"payload": {
		"id": 1005
	}
} {
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "before"
		}, {
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "after"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "version"
			}, {
				"type": "string",
				"optional": false,
				"field": "connector"
			}, {
				"type": "string",
				"optional": false,
				"field": "name"
			}, {
				"type": "int64",
				"optional": false,
				"field": "ts_ms"
			}, {
				"type": "string",
				"optional": true,
				"name": "io.debezium.data.Enum",
				"version": 1,
				"parameters": {
					"allowed": "true,last,false"
				},
				"default": "false",
				"field": "snapshot"
			}, {
				"type": "string",
				"optional": false,
				"field": "db"
			}, {
				"type": "string",
				"optional": true,
				"field": "table"
			}, {
				"type": "int64",
				"optional": false,
				"field": "server_id"
			}, {
				"type": "string",
				"optional": true,
				"field": "gtid"
			}, {
				"type": "string",
				"optional": false,
				"field": "file"
			}, {
				"type": "int64",
				"optional": false,
				"field": "pos"
			}, {
				"type": "int32",
				"optional": false,
				"field": "row"
			}, {
				"type": "int64",
				"optional": true,
				"field": "thread"
			}, {
				"type": "string",
				"optional": true,
				"field": "query"
			}],
			"optional": false,
			"name": "io.debezium.connector.mysql.Source",
			"field": "source"
		}, {
			"type": "string",
			"optional": false,
			"field": "op"
		}, {
			"type": "int64",
			"optional": true,
			"field": "ts_ms"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "id"
			}, {
				"type": "int64",
				"optional": false,
				"field": "total_order"
			}, {
				"type": "int64",
				"optional": false,
				"field": "data_collection_order"
			}],
			"optional": true,
			"field": "transaction"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Envelope"
	},
	"payload": {
		"before": null,
		"after": {                                          # 
			"id": 1005,
			"first_name": "xin",
			"last_name": "lixin",
			"email": "test@126.com"
		},
		"source": {
			"version": "1.2.5.Final",
			"connector": "mysql",
			"name": "dbserver1",
			"ts_ms": 1631416116000,
			"snapshot": "false",
			"db": "inventory",
			"table": "customers",
			"server_id": 223344,
			"gtid": null,
			"file": "mysql-bin.000003",
			"pos": 364,
			"row": 0,
			"thread": 6,
			"query": null
		},
		"op": "c",
		"ts_ms": 1631416116417,
		"transaction": null
	}
}
```
### (16). 查看kafka-consumer更新的数据
```
{
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "int32",
			"optional": false,
			"field": "id"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Key"
	},
	"payload": {
		"id": 1005
	}
} {
	"schema": {
		"type": "struct",
		"fields": [{
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "before"
		}, {
			"type": "struct",
			"fields": [{
				"type": "int32",
				"optional": false,
				"field": "id"
			}, {
				"type": "string",
				"optional": false,
				"field": "first_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "last_name"
			}, {
				"type": "string",
				"optional": false,
				"field": "email"
			}],
			"optional": true,
			"name": "dbserver1.inventory.customers.Value",
			"field": "after"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "version"
			}, {
				"type": "string",
				"optional": false,
				"field": "connector"
			}, {
				"type": "string",
				"optional": false,
				"field": "name"
			}, {
				"type": "int64",
				"optional": false,
				"field": "ts_ms"
			}, {
				"type": "string",
				"optional": true,
				"name": "io.debezium.data.Enum",
				"version": 1,
				"parameters": {
					"allowed": "true,last,false"
				},
				"default": "false",
				"field": "snapshot"
			}, {
				"type": "string",
				"optional": false,
				"field": "db"
			}, {
				"type": "string",
				"optional": true,
				"field": "table"
			}, {
				"type": "int64",
				"optional": false,
				"field": "server_id"
			}, {
				"type": "string",
				"optional": true,
				"field": "gtid"
			}, {
				"type": "string",
				"optional": false,
				"field": "file"
			}, {
				"type": "int64",
				"optional": false,
				"field": "pos"
			}, {
				"type": "int32",
				"optional": false,
				"field": "row"
			}, {
				"type": "int64",
				"optional": true,
				"field": "thread"
			}, {
				"type": "string",
				"optional": true,
				"field": "query"
			}],
			"optional": false,
			"name": "io.debezium.connector.mysql.Source",
			"field": "source"
		}, {
			"type": "string",
			"optional": false,
			"field": "op"
		}, {
			"type": "int64",
			"optional": true,
			"field": "ts_ms"
		}, {
			"type": "struct",
			"fields": [{
				"type": "string",
				"optional": false,
				"field": "id"
			}, {
				"type": "int64",
				"optional": false,
				"field": "total_order"
			}, {
				"type": "int64",
				"optional": false,
				"field": "data_collection_order"
			}],
			"optional": true,
			"field": "transaction"
		}],
		"optional": false,
		"name": "dbserver1.inventory.customers.Envelope"
	},
	"payload": {
		"before": {       
			"id": 1005,                            # 修改之前的数据
			"first_name": "xin",                   
			"last_name": "lixin",
			"email": "test@126.com"
		},
		"after": {
			"id": 1005,                            # 修改之后的数据
			"first_name": "xin--",
			"last_name": "lixin",
			"email": "test@126.com"
		},
		"source": {
			"version": "1.2.5.Final",
			"connector": "mysql",
			"name": "dbserver1",
			"ts_ms": 1631416216000,
			"snapshot": "false",
			"db": "inventory",
			"table": "customers",
			"server_id": 223344,
			"gtid": null,
			"file": "mysql-bin.000003",
			"pos": 668,
			"row": 0,
			"thread": 6,
			"query": null
		},
		"op": "u",
		"ts_ms": 1631416216727,
		"transaction": null
	}
}
```
### (17). 总结
