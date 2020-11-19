---
layout: post
title: 'Canal Server HA搭建'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). Canal Server机器信息
> 我这里是**伪集群配置**  

> 可参考(https://github.com/alibaba/canal/wiki/AdminGuide)  

Canal Server | Canal Server Port
- | :-: 
127.0.0.1 | 11111
127.0.0.1 | 22222

### (2). ZK集群信息

Zookeeper Server  | Zookeeper Port
- | :-: 
127.0.0.1 | 2181
127.0.0.1 | 2182
127.0.0.1 | 2183

### (3). MySQL服务器信息

MySQL Server  | MySQL Port
- | :-: 
127.0.0.1 | 3306


### (4). 当前工作目录
```
# 当前工作目录
lixin-macbook:canal-server-cluster lixin$ pwd
/Users/lixin/Developer/canal-server-cluster

# 两台Canal Server
lixin-macbook:canal-server-cluster lixin$ ll
drwxr-xr-x   7 lixin  staff  224 11 19 14:55 canal-server-1/
drwxr-xr-x   7 lixin  staff  224 11 19 14:55 canal-server-2/
```
### (5). 修改全局canal.properties配置文件

> <font color='red'>canal-server-1/conf/canal.properties</font> 

```
# 修改如下配置
# 指定端口
canal.port = 11111
canal.metrics.pull.port = 11112
canal.admin.port = 11110

# 配置ZK地址
canal.zkServers = 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183

# 注释掉:canal.instance.global.spring.xml = classpath:spring/file-instance.xml
# HA模式:必须都选择default-instance.xml配置.
canal.instance.global.spring.xml = classpath:spring/default-instance.xml
```

> <font color='red'>canal-server-2/conf/canal.properties</font>  

```
# 指定端口
canal.port = 22222
canal.metrics.pull.port = 22223
canal.admin.port = 22220

# 配置ZK地址
canal.zkServers = 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183


# 注释掉:canal.instance.global.spring.xml = classpath:spring/file-instance.xml
# HA模式:必须都选择default-instance.xml配置.
canal.instance.global.spring.xml = classpath:spring/default-instance.xml
```

### (6). 修改实例(example)instance.properties配置文件

> canal-server-1/conf/<font color='red'>example</font>/instance.properties  

```
# slaveId要保证不可重复
canal.instance.mysql.slaveId=12345

# 指定MySQL服务器/账号/密码 
# 参考Canal Server的单机配置(创建的复制账号) 
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset = UTF-8
```

> canal-server-2/conf/<font color='red'>example</font>/instance.properties   

```
# slaveId要保证不可重复
canal.instance.mysql.slaveId=54321

# 指定MySQL服务器/账号/密码 
# 参考Canal Server的单机配置(创建的复制账号) 
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset = UTF-8
```

### (7). 启动ZK(略)

### (8). 启动Canal Server

```
# 启动第一台
/Users/lixin/Developer/canal-server-cluster/canal-server-1/bin/startup.sh
# 启动第二台
/Users/lixin/Developer/canal-server-cluster/canal-server-2/bin/startup.sh
```
### (9). 查看(canal-server-1)日志

```
lixin-macbook:canal lixin$ ll /Users/lixin/Developer/canal-server-cluster/canal-server-1/logs/
# canal启动日志
drwxr-xr-x  4 lixin  staff  128 11 19 15:31 canal/
# 实例example日志
drwxr-xr-x  3 lixin  staff   96 11 19 15:31 example/

# 查看实例日志
lixin-macbook:canal lixin$ cat /Users/lixin/Developer/canal-server-cluster/canal-server-1/logs/example/example.log 

2020-11-19 15:31:54.053 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2020-11-19 15:31:54.094 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2020-11-19 15:31:54.412 [main] WARN  o.s.beans.GenericTypeAwarePropertyDescriptor - Invalid JavaBean property 'connectionCharset' being accessed! Ambiguous write methods found next to actually used [public void com.alibaba.otter.canal.parse.inbound.mysql.AbstractMysqlEventParser.setConnectionCharset(java.nio.charset.Charset)]: [public void com.alibaba.otter.canal.parse.inbound.mysql.AbstractMysqlEventParser.setConnectionCharset(java.lang.String)]
2020-11-19 15:31:54.517 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2020-11-19 15:31:54.518 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2020-11-19 15:31:55.130 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
2020-11-19 15:31:55.140 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table filter : ^.*\..*$
2020-11-19 15:31:55.140 [main] WARN  c.a.o.canal.parse.inbound.mysql.dbsync.LogEventConvert - --> init table black filter : 

#############################提示启动成功#############################
2020-11-19 15:31:55.154 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
2020-11-19 15:31:55.277 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> begin to find start position, it will be long time for reset or first position
2020-11-19 15:31:55.278 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - prepare to find start position just show master status
2020-11-19 15:31:55.679 [destination = example , address = /127.0.0.1:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=mysql-bin.000018,position=4,serverId=1,gtid=<null>,timestamp=1605768072000] cost : 385ms , the next step is binlog dump
```

### (10). 查看(canal-server-2)日志

```
# canal-server-2只有canal日志信息,并无实例信息
lixin-macbook:canal-server-cluster lixin$  ll ./canal-server-2/logs/
drwxr-xr-x  4 lixin  staff  128 11 19 15:31 canal/

# 查看canal日志信息
lixin-macbook:canal-server-cluster lixin$ cat  ./canal-server-2/logs/canal/canal.log 
2020-11-19 16:35:23.042 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## set default uncaught exception handler
2020-11-19 16:35:23.222 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## load canal configurations
2020-11-19 16:35:23.269 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## start the canal server.
2020-11-19 16:35:23.650 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[172.17.0.125(172.17.0.125):22222]
2020-11-19 16:35:24.220 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## the canal server is running now ......

```

### (11). ZK客户端查看实例信息 

```
# example的实现,在运行状态的是:172.17.0.125:11111
[zk: localhost:2181(CONNECTED) 9] get /otter/canal/destinations/example/running
	{"active":true,"address":"172.17.0.125:11111"}
```

### (12). Canal Client连接Canal Server进行消费
> 请参考:com.alibaba.otter.canal.example.ClusterCanalClientTest  

```
String destination = "example";
CanalConnector connector = CanalConnectors
   .newClusterConnector(
	   // ZK集群列表
	   "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183", 
	   // 实例
	   destination, 
	   null,
	   null);
```
### (14). 查看ZK信息

```
# 在实例(example)节点下,增加了:1001节点,并包含有两个子节点:cursor/running
# /otter/canal/destinations/example/1001/cursor
# /otter/canal/destinations/example/1001/running
[zk: localhost:2181(CONNECTED) 62] ls /otter/canal/destinations/example/1001
[cursor, running]

# 查看活动的canal client,意味着:canal client也可以集群.
# 那么1001是怎么来的?是固定的吗?
[zk: localhost:2181(CONNECTED) 64] get /otter/canal/destinations/example/1001/running 
{"active":true,"address":"172.17.0.125:63912","clientId":1001}

# 查看position信息(ZK持久化记录:上次消费的position)
[zk: localhost:2181(CONNECTED) 63] get /otter/canal/destinations/example/1001/cursor
{"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"localhost","port":3306}},"postion":{"gtid":"","included":false,"journalName":"mysql-bin.000018","position":384,"serverId":1,"timestamp":1605773474000}}
```

### (15). Canal Client日志
```
2020-11-19 16:38:13.102 [ZkClient-EventThread-10-127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183] INFO  org.I0Itec.zkclient.ZkEventThread - Starting ZkClient event thread.
2020-11-19 16:38:13.104 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:zookeeper.version=3.4.5-1392090, built on 09/30/2012 17:52 GMT
2020-11-19 16:38:13.106 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:host.name=localhost
2020-11-19 16:38:13.106 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.version=1.8.0_251
2020-11-19 16:38:13.106 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.vendor=Oracle Corporation
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/jre
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.library.path=/Users/lixin/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.io.tmpdir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:java.compiler=<NA>
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:os.name=Mac OS X
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:os.arch=x86_64
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:os.version=10.15.7
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:user.name=lixin
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:user.home=/Users/lixin
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Client environment:user.dir=/Users/lixin/GitRepository/canal-1.1.4/example

###########################初始化连接到ZK##########################################
2020-11-19 16:38:13.107 [main] INFO  org.apache.zookeeper.ZooKeeper - Initiating client connection, connectString=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 sessionTimeout=90000 watcher=com.alibaba.otter.canal.common.zookeeper.ZkClientx@6a4f787b
2020-11-19 16:38:13.131 [main] INFO  org.I0Itec.zkclient.ZkClient - Waiting for keeper state SyncConnected
2020-11-19 16:38:13.133 [main-SendThread(localhost:2183)] INFO  org.apache.zookeeper.ClientCnxn - Opening socket connection to server localhost/127.0.0.1:2183. Will not attempt to authenticate using SASL (unknown error)
2020-11-19 16:38:13.225 [main-SendThread(localhost:2183)] INFO  org.apache.zookeeper.ClientCnxn - Socket connection established to localhost/127.0.0.1:2183, initiating session
2020-11-19 16:38:13.235 [main-SendThread(localhost:2183)] INFO  org.apache.zookeeper.ClientCnxn - Session establishment complete on server localhost/127.0.0.1:2183, sessionid = 0x3000034a4be0004, negotiated timeout = 40000
2020-11-19 16:38:13.236 [main-EventThread] INFO  org.I0Itec.zkclient.ZkClient - zookeeper state changed (SyncConnected)

###########################MySQL插入一条数据##########################################

****************************************************
* Batch Id: [3] ,count : [3] , memsize : [150] , Time : 2020-11-19 16:38:22
* Start : [mysql-bin.000018:1263:1605775102000(2020-11-19 16:38:22)] 
* End : [mysql-bin.000018:1428:1605775102000(2020-11-19 16:38:22)] 
****************************************************

================> binlog[mysql-bin.000018:1263] , executeTime : 1605775102000(2020-11-19 16:38:22) , gtid : () , delay : 995ms
 BEGIN ----> Thread id: 14
----------------> binlog[mysql-bin.000018:1379] , name[db,t1] , eventType : INSERT , executeTime : 1605775102000(2020-11-19 16:38:22) , gtid : () , delay : 998 ms
id : 10004    type=int(11)    update=true
name : tom10004    type=varchar(20)    update=true
----------------
 END ----> transaction id: 170
================> binlog[mysql-bin.000018:1428] , executeTime : 1605775102000(2020-11-19 16:38:22) , gtid : () , delay : 1007ms


###########################停止canal-server-1(11111)##########################################

2020-11-19 16:38:57.135 [Thread-2] WARN  c.alibaba.otter.canal.client.impl.ClusterCanalConnector - something goes wrong when getWithoutAck data from server:null
com.alibaba.otter.canal.protocol.exception.CanalClientException: java.io.IOException: Connection reset by peer
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.getWithoutAck(SimpleCanalConnector.java:325) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.getWithoutAck(SimpleCanalConnector.java:295) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.ClusterCanalConnector.getWithoutAck(ClusterCanalConnector.java:183) ~[classes/:na]
	at com.alibaba.otter.canal.example.AbstractCanalClientTest.process(AbstractCanalClientTest.java:64) [classes/:na]
	at com.alibaba.otter.canal.example.AbstractCanalClientTest$1.run(AbstractCanalClientTest.java:31) [classes/:na]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_251]
Caused by: java.io.IOException: Connection reset by peer
	at sun.nio.ch.FileDispatcherImpl.read0(Native Method) ~[na:1.8.0_251]
	at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39) ~[na:1.8.0_251]
	at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223) ~[na:1.8.0_251]
	at sun.nio.ch.IOUtil.read(IOUtil.java:197) ~[na:1.8.0_251]
	at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:380) ~[na:1.8.0_251]
	at sun.nio.ch.SocketAdaptor$SocketInputStream.read(SocketAdaptor.java:206) ~[na:1.8.0_251]
	at sun.nio.ch.ChannelInputStream.read(ChannelInputStream.java:103) ~[na:1.8.0_251]
	at java.nio.channels.Channels$ReadableByteChannelImpl.read(Channels.java:385) ~[na:1.8.0_251]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.read(SimpleCanalConnector.java:411) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.readNextPacket(SimpleCanalConnector.java:401) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.readNextPacket(SimpleCanalConnector.java:385) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.receiveMessages(SimpleCanalConnector.java:330) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.getWithoutAck(SimpleCanalConnector.java:323) ~[classes/:na]
	... 5 common frames omitted

# 连接到:canal-server2(22222)
2020-11-19 16:39:02.266 [Thread-2] ERROR c.a.otter.canal.client.impl.running.ClientRunningMonitor - There is an error when execute initRunning method, with destination [example].
com.alibaba.otter.canal.protocol.exception.CanalClientException: failed to subscribe with reason: something goes wrong with channel:[id: 0x291b1bbf, /172.17.0.125:52599 => /172.17.0.125:22222], exception=com.alibaba.otter.canal.server.exception.CanalServerException: destination:example should start first

	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.subscribe(SimpleCanalConnector.java:249) [classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector$1.processActiveEnter(SimpleCanalConnector.java:434) ~[classes/:na]
	at com.alibaba.otter.canal.client.impl.running.ClientRunningMonitor.processActiveEnter(ClientRunningMonitor.java:221) [classes/:na]
	at com.alibaba.otter.canal.client.impl.running.ClientRunningMonitor.initRunning(ClientRunningMonitor.java:123) [classes/:na]
	at com.alibaba.otter.canal.client.impl.running.ClientRunningMonitor.start(ClientRunningMonitor.java:93) [classes/:na]
	at com.alibaba.otter.canal.client.impl.SimpleCanalConnector.connect(SimpleCanalConnector.java:108) [classes/:na]
	at com.alibaba.otter.canal.client.impl.ClusterCanalConnector.connect(ClusterCanalConnector.java:64) [classes/:na]
	at com.alibaba.otter.canal.client.impl.ClusterCanalConnector.restart(ClusterCanalConnector.java:273) [classes/:na]
	at com.alibaba.otter.canal.client.impl.ClusterCanalConnector.getWithoutAck(ClusterCanalConnector.java:189) [classes/:na]
	at com.alibaba.otter.canal.example.AbstractCanalClientTest.process(AbstractCanalClientTest.java:64) [classes/:na]
	at com.alibaba.otter.canal.example.AbstractCanalClientTest$1.run(AbstractCanalClientTest.java:31) [classes/:na]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_251]

# 连接失败,进行重试
2020-11-19 16:39:02.283 [Thread-2] WARN  c.alibaba.otter.canal.client.impl.ClusterCanalConnector - failed to connect to:/172.17.0.125:22222 after retry 0 times
2020-11-19 16:39:02.288 [Thread-2] WARN  c.a.otter.canal.client.impl.running.ClientRunningMonitor - canal is not run any in node
2020-11-19 16:39:07.307 [Thread-2] INFO  c.alibaba.otter.canal.client.impl.ClusterCanalConnector - restart the connector for next round retry.

###########################MySQL中重新插入一条数据##########################################

****************************************************
* Batch Id: [1] ,count : [1] , memsize : [31] , Time : 2020-11-19 16:39:07
* Start : [mysql-bin.000018:1428:1605775102000(2020-11-19 16:38:22)] 
* End : [mysql-bin.000018:1428:1605775102000(2020-11-19 16:38:22)] 
****************************************************
----------------
 END ----> transaction id: 170
================> binlog[mysql-bin.000018:1428] , executeTime : 1605775102000(2020-11-19 16:38:22) , gtid : () , delay : 45313ms

****************************************************
* Batch Id: [2] ,count : [3] , memsize : [150] , Time : 2020-11-19 16:39:28
* Start : [mysql-bin.000018:1524:1605775168000(2020-11-19 16:39:28)] 
* End : [mysql-bin.000018:1689:1605775168000(2020-11-19 16:39:28)] 
****************************************************

================> binlog[mysql-bin.000018:1524] , executeTime : 1605775168000(2020-11-19 16:39:28) , gtid : () , delay : 228ms
 BEGIN ----> Thread id: 14
----------------> binlog[mysql-bin.000018:1640] , name[db,t1] , eventType : INSERT , executeTime : 1605775168000(2020-11-19 16:39:28) , gtid : () , delay : 228 ms
id : 10005    type=int(11)    update=true
name : tom10005    type=varchar(20)    update=true
----------------
 END ----> transaction id: 191
================> binlog[mysql-bin.000018:1689] , executeTime : 1605775168000(2020-11-19 16:39:28) , gtid : () , delay : 229ms
```

### (16). ZK节点信息介绍

> /otter/canal/destinations/example/running : <font color='red'>(EPHEMERAL节点)记录集群中活动的CanalServer</font>   

```
[zk: localhost:2181(CONNECTED) 129] get /otter/canal/destinations/example/running
{"active":true,"address":"172.17.0.125:22222"}
```

> /otter/canal/destinations/example/1001/running   : <font color='red'>(EPHEMERAL节点)记录集群中活动的CanalClient</font>    

```
[zk: localhost:2181(CONNECTED) 130] get /otter/canal/destinations/example/1001/running

{"active":true,"address":"172.17.0.125:52603","clientId":1001}
```

> /otter/canal/destinations/example/1001/cursor    : <font color='red'>(Persistent节点)记录MySQL position信息</font>      

```
[zk: localhost:2181(CONNECTED) 131] get /otter/canal/destinations/example/1001/cursor

{"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"localhost","port":3306}},"postion":{"gtid":"","included":false,"journalName":"mysql-bin.000018","position":1689,"serverId":1,"timestamp":1605775168000}}
```

> /otter/canal/destinations/example/cluster        : <font color='red'>(EPHEMERAL节点)记录CanalServer集群的机器列表(ip:port)</font>  

```
[zk: localhost:2181(CONNECTED) 134] ls /otter/canal/destinations/example/cluster
[172.17.0.125:11111, 172.17.0.125:22222]
```

### (17). 总结
> Canal Server利用了ZK进行了选举.<font color='red'>启动时,多台机器抢占创建ZK(EPHEMERAL/瞬时)节点(/otter/canal/destinations/example/running),同一时间只有一台能抢占成功,抢占失败的监听该节点变化,一旦发现该节点不存在,继续进行抢占.</font> 

