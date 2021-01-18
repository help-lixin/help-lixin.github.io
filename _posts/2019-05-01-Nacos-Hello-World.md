---
layout: post
title: 'Nacos 源码编译及入门(二)'
date: 2019-05-01
author: 李新
tags: Nacos
---

### (1). Nacos源码下载
```
# 工作目录
lixin-macbook:GitRepository lixin$ pwd
/Users/lixin/GitRepository

# 下载源码
lixin-macbook:GitRepository lixin$ git clone -b 1.3.0 https://github.com/alibaba/nacos.git nacos-1.3.0

# 进入源码目录
lixin-macbook:GitRepository lixin$ cd nacos-1.3.0/
```
### (2). 编译并打包
```
# 编译,并打包
lixin-macbook:nacos-1.3.0 lixin$ mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U  

# 二进制文件目录
lixin-macbook:nacos-1.3.0 lixin$ ll distribution/target/
-rw-r--r--   1 lixin  staff  73432485  1 17 16:55 nacos-server-1.3.0.tar.gz
-rw-r--r--   1 lixin  staff  73435270  1 17 16:55 nacos-server-1.3.0.zip
```
### (3). 环境搭建
```
# 解压二进制包(~/Developer目录下)
lixin-macbook:nacos-1.3.0 lixin$ tar  zxvf ./distribution/target/nacos-server-1.3.0.tar.gz  -C  ~/Developer/

# 进入工作目录
lixin-macbook:nacos-1.3.0 lixin$ cd ~/Developer/nacos/

# nacos目录结构如下
lixin-macbook:nacos lixin$ tree
.
├── LICENSE
├── NOTICE
├── bin
│   ├── shutdown.cmd
│   ├── shutdown.sh
│   ├── startup.cmd
│   └── startup.sh
├── conf
│   ├── application.properties
│   ├── application.properties.example
│   ├── cluster.conf.example
│   ├── nacos-logback.xml
│   ├── nacos-mysql.sql
│   └── schema.sql
└── target
    └── nacos-server.jar
```
### (4). 配置外部数据库
> 修改配置文件(application.properties),支持MySQL.   

```
# /application.properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=123456
```

> 导入并执行:nacos-mysql.sql文件   

```
# 登录mysql
lixin-macbook:conf lixin$ mysql -u root -p

# 创建schema
mysql> create database nacos;
Query OK, 1 row affected (0.01 sec)

# 进入schema
mysql> use nacos;
Database changed


# 执行外部脚本文件
mysql> source /Users/lixin/Developer/nacos/conf/nacos-mysql.sql
Query OK, 0 rows affected (0.03 sec)
... ...
```

### (5). 单机模式启动

```
# 进入bin目录
lixin-macbook:nacos lixin$ cd bin/

# 启动nacos(需要指定为:standalone)
lixin-macbook:bin lixin$ ./startup.sh -m standalone
/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/bin/java  -server -Xms2g -Xmx2g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/lixin/Developer/nacos/logs/java_heapdump.hprof -XX:-UseLargePages -Dnacos.member.list= -Djava.ext.dirs=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/jre/lib/ext:/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/lib/ext -Xloggc:/Users/lixin/Developer/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/Users/lixin/Developer/nacos/plugins/health,/Users/lixin/Developer/nacos/plugins/cmdb,/Users/lixin/Developer/nacos/plugins/mysql -Dnacos.home=/Users/lixin/Developer/nacos -jar /Users/lixin/Developer/nacos/target/nacos-server.jar  --spring.config.location=classpath:/,classpath:/config/,file:./,file:./config/,file:/Users/lixin/Developer/nacos/conf/ --logging.config=/Users/lixin/Developer/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with cluster
nacos is starting，you can check the /Users/lixin/Developer/nacos/logs/start.out

# 查看日志是否启动成功(能看到监听的端口为:8848)
lixin-macbook:bin lixin$ cat /Users/lixin/Developer/nacos/logs/start.out

# 查看口也绑定成功
lixin-macbook:bin lixin$ netstat -AaLlnW|grep 8848
a1859c6198ee5943        0 0/0/100        *.8848
```

### (6). 通过Open API,测试服务发现和服务注册
> ["http://localhost:8848/nacos"](http://localhost:8848/nacos/)   
> 账号和密码(nacos/nacos)   
> 通过API测试(服务注册和服务发现)  

```
# 服务注册(微服务名称=user-auth-service,端口=8080)
lixin-macbook:bin lixin$ curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=user-auth-service&ip=127.0.0.1&port=8080'
ok

# 服务发现(获取微服务名称为:user-auth-service的信息)
lixin-macbook:bin lixin$ curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=user-auth-service'
{
	"hosts": [{
		"ip": "127.0.0.1",
		"port": 8080,
		"valid": true,
		"healthy": true,
		"marked": false,
		"instanceId": "127.0.0.1#8080#DEFAULT#DEFAULT_GROUP@@user-auth-service",
		"metadata": {},
		"enabled": true,
		"weight": 1.0,
		"clusterName": "DEFAULT",
		"serviceName": "user-auth-service",
		"ephemeral": true
	}],
	"dom": "user-auth-service",
	"name": "DEFAULT_GROUP@@user-auth-service",
	"cacheMillis": 3000,
	"lastRefTime": 1610876446790,
	"checksum": "9cb360c307976228113cdaea97c78885",
	"useSpecifiedURL": false,
	"clusters": "",
	"env": "",
	"metadata": {}
}
```
### (7). 通过Open API,测试发布(获取)配置

```
# 发布配置
# dataId   : 可以理解为:配置文件名称(application.yml),它包含配置集
# group    : 对dataId进行分组,主要用于区分不同项目名称,应用名称.比如:erp-user-auth-service
# content  : 配置文件内容
lixin-macbook:bin lixin$ curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=HelloWorld"

# 获取配置
lixin-macbook:bin lixin$ curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test'
HelloWorld
```