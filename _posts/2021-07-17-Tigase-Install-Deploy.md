---
layout: post
title: 'Tigase安装与部署(二)' 
date: 2021-07-17
author: 李新
tags:  Tigase 
---

### (0). 准备工作
>  配置hosts

```
127.0.0.1 tigase.lixin.help
```

> 修改mysql的字符集
> SHOW VARIABLES WHERE Variable_name LIKE 'character_set_%';  

```
[mysql]
default-character-set=utf8mb4


[mysqld]
character-set-server=utf8mb4
log_bin_trust_function_creators=1

[client]
default-character-set=utf8mb4
```
### (1). 下载tigbase
```
# 1. 进入下载目录
lixin-macbook:~ lixin$ cd ~/Downloads/
# 2. 下载
lixin-macbook:Downloads lixin$ wget https://github.com/tigase/tigase-server/releases/download/tigase-server-8.0.0/tigase-server-8.0.0-b10083-dist-max.tar.gz
# 3. 解压
lixin-macbook:Downloads lixin$ tar -zxf tigase-server-8.0.0-b10083-dist-max.tar.gz
# 4. 安装到程序目录下
lixin-macbook:Downloads lixin$ mv tigase-server-8.0.0-b10083 ~/Developer/tigase-server-8.0.0
# 5. 进入到程序目录下
lixin-macbook:Downloads lixin$ cd ~/Developer/tigase-server-8.0.0/
```
### (2). 查看项目目录
```
lixin-macbook:tigase-server-8.0.0 lixin$ ll
drwxr-xr-x    5 lixin  staff    160  7 17 10:42 certs/                    # 证书目录
drwxr-xr-x  121 lixin  staff   3872  7 17 10:42 database/                 # 数据库脚本
drwxr-xr-x    5 lixin  staff    160  7 17 10:42 docs/
drwxr-xr-x   12 lixin  staff    384  7 17 10:42 etc/                      # 配置目录
drwxr-xr-x   81 lixin  staff   2592  7 17 10:42 jars/                     # jar包目录
drwxr-xr--@   2 lixin  staff     64  3  1  2019 logs/                     # 日志目录
-rw-r--r--@   1 lixin  staff   3614  3  1  2019 package.html 
drwxr-xr-x   24 lixin  staff    768  7 17 10:42 scripts/                  # 启动脚本
drwxr-xr-x    5 lixin  staff    160  7 17 10:42 tigase/                   # 文档
drwxr-xr-x    9 lixin  staff    288  7 17 10:42 win-stuff/
```
### (3). 配置tigase.conf
```
# 只需要指定:JAVA_HOME
lixin-macbook:tigase-server-8.0.0 lixin$ vi etc/tigase.conf
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```

### (4). config.tdsl
```
lixin-macbook:tigase-server-8.0.0 lixin$ cat etc/config.tdsl
'config-type' = 'setup'

http () {
    setup () {  // 这是登录tigase的后台账号和密码
        'admin-user' = 'admin'
        'admin-password' = 'tigase'
    }
}
```
### (5). 启动
```
lixin-macbook:tigase-server-8.0.0 lixin$ ./scripts/tigase.sh start etc/tigase.conf
Starting Tigase:
Tigase running pid=1558

# 检查
lixin-macbook:tigase-server-8.0.0 lixin$ lsof -i tcp:8080
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    1558 lixin  102u  IPv6 0x336f92458e0b8089      0t0  TCP *:http-alt (LISTEN)
```
### (6). 登录tigase
> 账号和密码在:etc/config.tdsl
!["tigase关于界面"](/assets/tigase/imgs/tigase-about.jpg)
!["填写公司名称"](/assets/tigase/imgs/tigase-input-company-name.png)
!["填写配置信息"](/assets/tigase/imgs/tigase-server-configuration.png)
!["高级配置选项"](/assets/tigase/imgs/tigase-advancde-configuration-options.jpg)
!["插件选择"](/assets/tigase/imgs/tigase-plugin-select.jpg)
!["数据库配置"](/assets/tigase/imgs/tigase-database-config.jpg)
!["创数据库并检查"](/assets/tigase/imgs/tigase-dtabase-check.png)
!["创建登录的账号和密码"](/assets/tigase/imgs/tigase-security.png)
!["确认配置"](/assets/tigase/imgs/tigase-configuration.png)
!["安装完成"](/assets/tigase/imgs/tigase-install-finished.png)

### (7). etc/config.tdsl
> 在我这里admin的账号和密码是:  admin@tigase.lixin.help/admin 

```
admins = [
    'admin@tigase.lixin.help'
]
'config-type' = 'default'
debug = [ 'server' ]
'default-virtual-host' = 'tigase.lixin.help'
dataSource () {
    default () {
        uri = 'jdbc:mysql://127.0.0.1:3306/tigasedb?user=tigase_user&password=123456&useSSL=false&useLegacyDatetimeCode=false&allowPublicKeyRetrieval=true'
    }
}
http () {
    setup () {
        'admin-password' = '123456'
        'admin-user' = 'lixin'
    }
}
pubsub () {
    trusted = [ 'http@{clusterNode}' ]
}
```
### (8). 重启服务
```
# 1. 停止服务
lixin-macbook:tigase-server-8.0.0 lixin$ ./scripts/tigase.sh stop
No params-file.conf given. Using: ''
Shutting down Tigase: 1558
1. Waiting for the server to terminate...
2. Tigase terminated.

# 2. 启动服务
lixin-macbook:tigase-server-8.0.0 lixin$ ./scripts/tigase.sh start etc/tigase.conf
Starting Tigase:
Tigase running pid=3000

# 3. 检查是否启动成功
lixin-macbook:tigase-server-8.0.0 lixin$ jps -l
3000 tigase.server.XMPPServer
```
### (9). 进入后台
!["tigase admin"](/assets/tigase/imgs/tigase-admin.png)

### (10). Tigase Spark 连接测试
["Spark下载"](http://www.igniterealtime.org/downloads/index.jsp)

### (11). Spark连接Tigase
> 在连接过程中,有任何问题看日志(logs/tigase.log.0)

!["Spark配置主机"](/assets/tigase/imgs/tigase-spark-setting-1.png)
!["Spark禁用Security"](/assets/tigase/imgs/tigase-spark-setting-2.png )
!["Spark创建用户"](/assets/tigase/imgs/tigase-spark-create-user.png)

### (12). 监听Spark连接日志内容
> 创建用户成功的日志.

```
# 创建用户成功的日志
lixin-macbook:tigase-server-8.0.0 lixin$ tail -500f logs/tigase.log.0
2021-07-17 13:39:18.798 [ConnectionOpenThread]  ConnectionManager$ConnectionListenerImpl.accept()  FINEST: Accept called for service: null@null, port_props: {remote-host=localhost, port-no=5222, new-connections-throttling=200, ifc=[Ljava.lang.String;@2e2cd42c, socket=plain, type=accept}
2021-07-17 13:39:18.798 [ConnectionOpenThread]  ConnectionManager.serviceStarted()  FINER:   [[c2s]] Connection started: null, type: accept, Socket: nullSocket[addr=/172.17.20.211,port=51661,localport=5222], jid: null
2021-07-17 13:39:18.800 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: Stream opened: {xmlns:stream=http://etherx.jabber.org/streams, xmlns=jabber:client, xml:lang=en, from=username@tigase.lixin.help, to=tigase.lixin.help, version=1.0}
2021-07-17 13:39:18.800 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: No Session ID, generating a new one: 81057674-6699-443a-a0a0-c49bedea5d51
2021-07-17 13:39:18.801 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: Writing raw data to the socket: <?xml version='1.0'?><stream:stream xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' from='tigase.lixin.help' id='81057674-6699-443a-a0a0-c49bedea5d51' version='1.0' xml:lang='en'>
2021-07-17 13:39:18.801 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: DONE
2021-07-17 13:39:18.802 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: Sending a system command to SM: from=null, to=null, DATA=<iq to="sess-man@tigase.lixin.help" type="set" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" id="c2s--c2s51"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=455, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 13:39:18.802 [pool-31-thread-11]  ClientConnectionManager.xmppStreamOpened()  FINER: DONE 2
2021-07-17 13:39:18.803 [in_5-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=null, to=null, DATA=<iq retryCount="15" id="c2s--c2s51" to="sess-man@tigase.lixin.help" type="set" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" delay="45"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=482, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 13:39:18.804 [in_5-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@tigase.lixin.help
2021-07-17 13:39:18.804 [in_5-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@tigase.lixin.help
2021-07-17 13:39:18.804 [in_5-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: sess-man@tigase.lixin.help, from=null, to=null, DATA=<iq retryCount="15" id="c2s--c2s51" to="sess-man@tigase.lixin.help" type="set" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" delay="45"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=482, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 13:39:18.804 [in_37-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=null, to=null, DATA=<iq retryCount="15" id="c2s--c2s51" to="sess-man@tigase.lixin.help" type="set" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" delay="45"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=482, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 13:39:18.804 [in_37-sess-man]   SessionManager.processCommand()         FINER:    STREAM_OPENED command from: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.804 [session-open Queue Worker 5]  SessionManager$SessionOpenProc.process()  FINER: Adding resource connection for: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.804 [session-open Queue Worker 5]  SessionManager.createUserSession()  FINEST: Setting hostname tigase.lixin.help for connection: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, VHostItem: Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 13:39:18.804 [session-open Queue Worker 5]  SessionManager.createUserSession()  FINEST: Domain set for connectionId c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.805 [session-open Queue Worker 5]  SessionManager$SessionOpenProc.process()  FINEST: Setting session-id 81057674-6699-443a-a0a0-c49bedea5d51 for connection: XMPPResourceConnection=[user_jid=null, packets=0, connectioId=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq to="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" type="result" from="sess-man@tigase.lixin.help" id="c2s--c2s51"/>, SIZE=135, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.805 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@tigase.lixin.help, from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq to="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" type="result" from="sess-man@tigase.lixin.help" id="c2s--c2s51"/>, SIZE=135, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 13:39:18.805 [in_5-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=null, to=null, DATA=<iq to="sess-man@tigase.lixin.help" type="get" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=235, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.806 [in_5-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@tigase.lixin.help
2021-07-17 13:39:18.806 [in_5-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@tigase.lixin.help
2021-07-17 13:39:18.806 [in_5-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: sess-man@tigase.lixin.help, from=null, to=null, DATA=<iq to="sess-man@tigase.lixin.help" type="get" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=235, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.806 [in_37-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=null, to=null, DATA=<iq to="sess-man@tigase.lixin.help" type="get" from="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=235, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.806 [in_37-sess-man]   SessionManager.processCommand()         FINER:    GETFEATURES command from: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq to="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" type="result" from="sess-man@tigase.lixin.help" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=781, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.807 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@tigase.lixin.help, from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq to="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" type="result" from="sess-man@tigase.lixin.help" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=781, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 13:39:18.807 [in_4-c2s]         ClientConnectionManager.processPacket()  FINEST:  Processing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq to="c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661" type="result" from="sess-man@tigase.lixin.help" id="d3649ff3-ec17-414e-8910-14a4418ea7a4"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=781, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 13:39:18.807 [in_4-c2s]         ConnectionManager.writePacketToSocket()  FINEST:  c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, type: accept, Socket: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661 Socket[addr=/172.17.20.211,port=51661,localport=5222], jid: null, Writing packet: from=null, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<stream:features><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></stream:features>, SIZE=569, XMLNS=null, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null
2021-07-17 13:39:18.810 [pool-31-thread-12]  ClientConnectionManager.processSocketData()  FINEST: Processing socket data: from=null, to=null, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get"><query xmlns="jabber:iq:register"/></iq>, SIZE=92, XMLNS=null, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get from connection: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.810 [pool-31-thread-12]  ClientConnectionManager.processSocketData()  FINEST: XMLNS set for packet: from=null, to=null, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get from connection: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.810 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, to=sess-man@tigase.lixin.help, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.811 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : tigase.lixin.help
2021-07-17 13:39:18.811 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): tigase.lixin.help, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.811 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No component name matches (VHost lookup against component name): tigase.lixin.help, for map: [vhost-man, c2s, amp, monitor, ws2s, bosh, eventbus, s2s, stats, http, muc, sess-man, message-archive, message-router, pubsub], for all VHosts: [tigase.lixin.help, tigase.lixin.help,172.17.20.211]; trying other forms of addressing
2021-07-17 13:39:18.811 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Component match failed: tigase.lixin.help, for comp: [vhost-man, c2s, amp, monitor, ws2s, bosh, eventbus, s2s, stats, http, muc, sess-man, message-archive, message-router, pubsub], basename: lixin.help
2021-07-17 13:39:18.811 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@tigase.lixin.help
2021-07-17 13:39:18.812 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: sess-man@tigase.lixin.help, from=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, to=sess-man@tigase.lixin.help, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.812 [in_20-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, to=sess-man@tigase.lixin.help, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get
2021-07-17 13:39:18.812 [in_20-sess-man]   SessionManager.processPacket()          FINEST:   processing packet: from=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, to=sess-man@tigase.lixin.help, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=get, connection: XMPPResourceConnection=[user_jid=null, packets=0, connectioId=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 13:39:18.813 [in_20-sess-man]   SessionManager.walk()                   FINEST:   XMPPProcessorIfc: JabberIqRegister (jabber:iq:register)Request: from=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, to=sess-man@tigase.lixin.help, DATA=<iq id="ylJhd-58" to="tigase.lixin.help" type="get" xmlns="jabber:client"><query xmlns="jabber:iq:register"/></iq>, SIZE=114, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=get, conn: XMPPResourceConnection=[user_jid=null, packets=1, connectioId=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 13:39:18.813 [in_20-sess-man]   SessionManager.processPacket()          FINEST:   Packet processed by: [jabber:iq:register]
2021-07-17 13:39:18.813 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq xmlns="jabber:client" type="result" from="tigase.lixin.help" id="ylJhd-58"><query xmlns="jabber:iq:register">CData size: 157</query></iq>, SIZE=283, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=result
2021-07-17 13:39:18.814 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661
2021-07-17 13:39:18.814 [in_4-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, for map: [sess-man@tigase.lixin.help, monitor@tigase.lixin.help, message-archive@tigase.lixin.help, eventbus@tigase.lixin.help, s2s@tigase.lixin.help, amp@tigase.lixin.help, ws2s@tigase.lixin.help, stats@tigase.lixin.help, http@tigase.lixin.help, message-router@tigase.lixin.help, muc@tigase.lixin.help, pubsub@tigase.lixin.help, vhost-man@tigase.lixin.help, c2s@tigase.lixin.help, bosh@tigase.lixin.help]; trying VHost lookup
2021-07-17 13:39:18.814 [in_4-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@tigase.lixin.help, from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq xmlns="jabber:client" type="result" from="tigase.lixin.help" id="ylJhd-58"><query xmlns="jabber:iq:register">CData size: 157</query></iq>, SIZE=283, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=result
2021-07-17 13:39:18.814 [in_4-c2s]         ClientConnectionManager.processPacket()  FINEST:  Processing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq xmlns="jabber:client" type="result" from="tigase.lixin.help" id="ylJhd-58"><query xmlns="jabber:iq:register">CData size: 157</query></iq>, SIZE=283, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=result
2021-07-17 13:39:18.814 [in_4-c2s]         ConnectionManager.writePacketToSocket()  FINEST:  c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, type: accept, Socket: c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661 Socket[addr=/172.17.20.211,port=51661,localport=5222], jid: null, Writing packet: from=sess-man@tigase.lixin.help, to=c2s@tigase.lixin.help/172.17.20.211_5222_172.17.20.211_51661, DATA=<iq xmlns="jabber:client" type="result" from="tigase.lixin.help" id="ylJhd-58"><query xmlns="jabber:iq:register">CData size: 157</query></iq>, SIZE=283, XMLNS=jabber:client, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=result
```

> 创建用户失败的日志.

```
lixin-macbook:tigase-server-8.0.0 lixin$ tail -500f logs/tigase.log.0
2021-07-17 12:33:25.283 [ConnectionOpenThread]  ConnectionManager$ConnectionListenerImpl.accept()  FINEST: Accept called for service: null@null, port_props: {remote-host=localhost, port-no=5222, new-connections-throttling=200, ifc=[Ljava.lang.String;@2e2cd42c, socket=plain, type=accept}
2021-07-17 12:33:25.284 [ConnectionOpenThread]  ConnectionManager.serviceStarted()  FINER:   [[c2s]] Connection started: null, type: accept, Socket: nullSocket[addr=/127.0.0.1,port=50377,localport=5222], jid: null
2021-07-17 12:33:25.286 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: Stream opened: {xmlns:stream=http://etherx.jabber.org/streams, xmlns=jabber:client, xml:lang=en, from=username@tigase.lixin.help, to=tigase.lixin.help, version=1.0}
2021-07-17 12:33:25.287 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: No Session ID, generating a new one: 7eeab729-d142-4ccf-adf3-5d7d438b86de
2021-07-17 12:33:25.287 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: Writing raw data to the socket: <?xml version='1.0'?><stream:stream xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' from='tigase.lixin.help' id='7eeab729-d142-4ccf-adf3-5d7d438b86de' version='1.0' xml:lang='en'>
2021-07-17 12:33:25.288 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: DONE
2021-07-17 12:33:25.288 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: Sending a system command to SM: from=null, to=null, DATA=<iq to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="c2s--c2s13"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=431, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.288 [pool-31-thread-10]  ClientConnectionManager.xmppStreamOpened()  FINER: DONE 2
2021-07-17 12:33:25.289 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   Processing packet: from=null, to=null, DATA=<iq retryCount="15" delay="45" id="c2s--c2s13" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=458, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.289 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.289 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.289 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   1. Packet will be processed by: sess-man@localhost, from=null, to=null, DATA=<iq retryCount="15" delay="45" id="c2s--c2s13" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=458, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.290 [in_46-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=null, to=null, DATA=<iq retryCount="15" delay="45" id="c2s--c2s13" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_OPENED"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="hostname"><value>CData size: 17</value></field><field var="xml:lang"><value>en</value></field></x></command></iq>, SIZE=458, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.290 [in_46-sess-man]   SessionManager.processCommand()         FINER:    STREAM_OPENED command from: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.291 [session-open Queue Worker 6]  SessionManager$SessionOpenProc.process()  FINER: Adding resource connection for: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.291 [session-open Queue Worker 6]  SessionManager.createUserSession()  FINEST: Setting hostname tigase.lixin.help for connection: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, VHostItem: Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 12:33:25.291 [session-open Queue Worker 6]  SessionManager.createUserSession()  FINEST: Domain set for connectionId c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.291 [session-open Queue Worker 6]  SessionManager$SessionOpenProc.process()  FINEST: Setting session-id 7eeab729-d142-4ccf-adf3-5d7d438b86de for connection: XMPPResourceConnection=[user_jid=null, packets=0, connectioId=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 12:33:25.295 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="c2s--c2s13"/>, SIZE=111, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.296 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.296 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.296 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.297 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.297 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@localhost, from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="c2s--c2s13"/>, SIZE=111, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.300 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   Processing packet: from=null, to=null, DATA=<iq to="sess-man@localhost" type="get" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=211, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 12:33:25.301 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.301 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.301 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   1. Packet will be processed by: sess-man@localhost, from=null, to=null, DATA=<iq to="sess-man@localhost" type="get" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=211, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 12:33:25.302 [in_46-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=null, to=null, DATA=<iq to="sess-man@localhost" type="get" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"/></iq>, SIZE=211, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=get
2021-07-17 12:33:25.303 [in_46-sess-man]   SessionManager.processCommand()         FINER:    GETFEATURES command from: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.305 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=757, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.305 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.305 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.305 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.305 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.306 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@localhost, from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=757, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.306 [in_0-c2s]         ClientConnectionManager.processPacket()  FINEST:  Processing packet: from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="fbf1fa20-3d09-464f-b24c-6efbc465e082"><command xmlns="http://jabber.org/protocol/commands" node="GETFEATURES"><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></command></iq>, SIZE=757, XMLNS=null, PRIORITY=HIGH, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.306 [in_0-c2s]         ConnectionManager.writePacketToSocket()  FINEST:  c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, type: accept, Socket: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377 Socket[addr=/127.0.0.1,port=50377,localport=5222], jid: null, Writing packet: from=null, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<stream:features><auth xmlns="http://jabber.org/features/iq-auth"/><register xmlns="http://jabber.org/features/iq-register"/><mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl"><mechanism>CData size: 13</mechanism><mechanism>CData size: 11</mechanism><mechanism>CData size: 5</mechanism><mechanism>CData size: 9</mechanism></mechanisms><ver xmlns="urn:xmpp:features:rosterver"/><sub xmlns="urn:xmpp:features:pre-approval"/><starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/><compression xmlns="http://jabber.org/features/compress"><method>CData size: 4</method></compression></stream:features>, SIZE=569, XMLNS=null, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null
2021-07-17 12:33:25.308 [pool-31-thread-11]  ClientConnectionManager.processSocketData()  FINEST: Processing socket data: from=null, to=null, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null from connection: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.308 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null
2021-07-17 12:33:25.308 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.308 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: sess-man@localhost, from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null
2021-07-17 12:33:25.309 [in_32-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null
2021-07-17 12:33:25.309 [in_32-sess-man]   SessionManager.processPacket()          FINEST:   processing packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=NONE, TYPE=null, connection: XMPPResourceConnection=[user_jid=null, packets=0, connectioId=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 12:33:25.309 [in_32-sess-man]   SessionManager.walk()                   FINEST:   XMPPProcessorIfc: StartTLS (starttls)Request: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>, SIZE=51, XMLNS=urn:ietf:params:xml:ns:xmpp-tls, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=null, conn: XMPPResourceConnection=[user_jid=null, packets=1, connectioId=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 12:33:25.309 [in_32-sess-man]   SessionManager.processPacket()          FINEST:   Packet processed by: [starttls]
2021-07-17 12:33:25.309 [in_7-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@localhost, to=null, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="set" from="sess-man@localhost" id="tig1"><command xmlns="http://jabber.org/protocol/commands" node="STARTTLS"><x xmlns="jabber:x:data" type="submit"/><proceed xmlns="urn:ietf:params:xml:ns:xmpp-tls"/></command></iq>, SIZE=275, XMLNS=null, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=set
2021-07-17 12:33:25.310 [in_7-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.310 [in_7-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.310 [in_7-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.310 [in_7-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.310 [in_7-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@localhost, from=sess-man@localhost, to=null, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="set" from="sess-man@localhost" id="tig1"><command xmlns="http://jabber.org/protocol/commands" node="STARTTLS"><x xmlns="jabber:x:data" type="submit"/><proceed xmlns="urn:ietf:params:xml:ns:xmpp-tls"/></command></iq>, SIZE=275, XMLNS=null, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=set
2021-07-17 12:33:25.310 [in_0-c2s]         ClientConnectionManager.processPacket()  FINEST:  Processing packet: from=sess-man@localhost, to=null, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="set" from="sess-man@localhost" id="tig1"><command xmlns="http://jabber.org/protocol/commands" node="STARTTLS"><x xmlns="jabber:x:data" type="submit"/><proceed xmlns="urn:ietf:params:xml:ns:xmpp-tls"/></command></iq>, SIZE=275, XMLNS=null, PRIORITY=NORMAL, PERMISSION=LOCAL, TYPE=set
2021-07-17 12:33:25.310 [in_0-c2s]         ClientConnectionManager.processCommand()  FINER:  Starting TLS for connection: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, type: accept, Socket: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377 Socket[addr=/127.0.0.1,port=50377,localport=5222], jid: null
2021-07-17 12:33:25.312 [in_0-c2s]         ClientTrustManagerFactory.getManager()  FINEST:   Creating new TrustManager for VHost Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 12:33:25.312 [in_0-c2s]         ClientTrustManagerFactory.getManager()  FINEST:   CA cert path=null for VHost Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 12:33:25.312 [in_0-c2s]         ClientTrustManagerFactory.getManager()  FINEST:   Creating new TrustManager for VHost Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 12:33:25.313 [in_0-c2s]         ClientTrustManagerFactory.getManager()  FINEST:   CA cert path=null for VHost Domain: tigase.lixin.help, enabled: true, anonym: true, register: true, maxusers: 0, tls: false, s2sSecret: ef04d5e1-5e6c-420d-b2dd-33a7b1c8e5b1, domainFilter: ALL, domainFilterDomains: null, c2sPortsAllowed: null, saslAllowedMechanisms: null
2021-07-17 12:33:25.313 [in_0-c2s]         ClientConnectionManager.processCommand()  FINEST: TLS: wantClientAuth=false; needClientAuth=false for connection c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, type: accept, Socket: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377 Socket[addr=/127.0.0.1,port=50377,localport=5222], jid: null
2021-07-17 12:33:25.352 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   Processing packet: from=null, to=null, DATA=<iq to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="c2s--c2s15"><command xmlns="http://jabber.org/protocol/commands" node="TLS_HANDSHAKE_COMPLETE"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="local-certificate"><value>CData size: 864</value></field></x></command></iq>, SIZE=1249, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.353 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.353 [in_14-message-router]  MessageRouter.getLocalComponent()  FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.353 [in_14-message-router]  MessageRouter.processPacket()      FINEST:   1. Packet will be processed by: sess-man@localhost, from=null, to=null, DATA=<iq to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="c2s--c2s15"><command xmlns="http://jabber.org/protocol/commands" node="TLS_HANDSHAKE_COMPLETE"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="local-certificate"><value>CData size: 864</value></field></x></command></iq>, SIZE=1249, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.354 [in_46-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=null, to=null, DATA=<iq to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" id="c2s--c2s15"><command xmlns="http://jabber.org/protocol/commands" node="TLS_HANDSHAKE_COMPLETE"><x xmlns="jabber:x:data" type="submit"><field var="session-id"><value>CData size: 36</value></field><field var="local-certificate"><value>CData size: 864</value></field></x></command></iq>, SIZE=1249, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.354 [in_46-sess-man]   SessionManager.processCommand()         FINER:    TLS_HANDSHAKE_COMPLETE command from: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.354 [in_46-sess-man]   SessionManager.processCommand()         FINEST:   Handshake details received. connection: XMPPResourceConnection=[user_jid=null, packets=1, connectioId=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 12:33:25.354 [in_46-sess-man]   SessionManager.processCommand()         FINEST:   local-certificate [
[
  Version: V1
  Subject: CN=tigase.lixin.help, CN=*.tigase.lixin.help, EMAILADDRESS=admin@tigase.org, OU=XMPP Service, O=Tigase.org
  Signature Algorithm: SHA1withRSA, OID = 1.2.840.113549.1.1.5

  Key:  Sun RSA public key, 1024 bits
  params: null
  modulus: 111130633715874722190190517850417020768858687853313031603404272865371872352928560995846703516370972846346143713237548127717821804234554776821124467916650898486191495759795559475629309555743969889689581088190958591143856712260374989767511557763188918235138541073528007651262766245921368804963654205971978180047
  public exponent: 65537
  Validity: [From: Sat Jul 17 12:13:22 CST 2021,
               To: Sun Jul 17 12:13:22 CST 2022]
  Issuer: CN=tigase.lixin.help, CN=*.tigase.lixin.help, EMAILADDRESS=admin@tigase.org, OU=XMPP Service, O=Tigase.org
  SerialNumber: [    60f258e2]

]
  Algorithm: [SHA1withRSA]
  Signature:
0000: 13 6D 58 43 DD 76 71 AE   B0 22 86 17 D1 13 AD E7  .mXC.vq.."......
0010: 2D 90 D8 27 30 C6 F2 5D   2C A3 CE 0A 0A 83 D6 FB  -..'0..],.......
0020: 5D 81 DE 2F EE 72 B1 58   35 36 44 29 F6 77 69 14  ]../.r.X56D).wi.
0030: C9 14 04 DC A7 53 07 7E   D6 CF 09 B6 E8 A8 62 72  .....S........br
0040: E8 EC 1D 01 CD 6B E8 2E   76 DF 66 DC 09 16 CF E9  .....k..v.f.....
0050: 32 15 6D AC D4 E7 99 5B   76 5C D1 2E F1 A5 63 73  2.m....[v\....cs
0060: CC 16 11 0F 9F D2 CA AC   ED 55 D6 18 B7 51 C2 1E  .........U...Q..
0070: 06 5D 9A E3 0C 18 37 DB   BF 70 95 51 D0 06 0C 00  .]....7..p.Q....

] stored in session-data. connection: XMPPResourceConnection=[user_jid=null, packets=1, connectioId=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, domain=tigase.lixin.help, authState=NOT_AUTHORIZED, isAnon=false, isTmp=false, parentSession hash=0, parentSession liveTime=]
2021-07-17 12:33:25.365 [pool-31-thread-16]  ConnectionManager.serviceStopped()    FINER:    [[c2s]] Connection stopped: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, type: accept, Socket: TLS: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377 Socket[unconnected], jid: null
2021-07-17 12:33:25.365 [pool-31-thread-16]  ClientConnectionManager.xmppStreamClosed()  FINER: Stream closed: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.368 [pool-31-thread-16]  ClientConnectionManager.xmppStreamClosed()  FINE: Service stopped, sending packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<iq retryCount="15" delay="120" id="a976d860-46be-4da2-b086-445ad0b1cec4" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_CLOSED"/></iq>, SIZE=241, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.369 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<iq retryCount="15" delay="120" id="a976d860-46be-4da2-b086-445ad0b1cec4" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_CLOSED"/></iq>, SIZE=241, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.369 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.369 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : sess-man@localhost
2021-07-17 12:33:25.369 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: sess-man@localhost, from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<iq retryCount="15" delay="120" id="a976d860-46be-4da2-b086-445ad0b1cec4" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_CLOSED"/></iq>, SIZE=241, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.370 [in_32-sess-man]   SessionManager.processPacket()          FINEST:   Received packet: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<iq retryCount="15" delay="120" id="a976d860-46be-4da2-b086-445ad0b1cec4" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_CLOSED"/></iq>, SIZE=241, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.370 [in_32-sess-man]   SessionManager.processCommand()         FINER:    STREAM_CLOSED command from: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.370 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   Processing packet: from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="a976d860-46be-4da2-b086-445ad0b1cec4"/>, SIZE=137, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.371 [session-close Queue Worker 0]  SessionManager$SessionCloseProc.process()  FINEST: Executing connection close for: from=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, to=sess-man@localhost, DATA=<iq retryCount="15" delay="120" id="a976d860-46be-4da2-b086-445ad0b1cec4" to="sess-man@localhost" type="set" from="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377"><command xmlns="http://jabber.org/protocol/commands" node="STREAM_CLOSED"/></iq>, SIZE=241, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=set
2021-07-17 12:33:25.371 [session-close Queue Worker 0]  SessionManager.closeConnection()  FINER: Stream closed from: c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.371 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.371 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.371 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   Called for : c2s@localhost/127.0.0.1_5222_127.0.0.1_50377
2021-07-17 12:33:25.371 [in_0-message-router]  MessageRouter.getLocalComponent()   FINEST:   No componentID matches (fast lookup against exact address): c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, for map: [amp@localhost, eventbus@localhost, bosh@localhost, c2s@localhost, sess-man@localhost, stats@localhost, monitor@localhost, message-archive@localhost, message-router@localhost, vhost-man@localhost, muc@localhost, ws2s@localhost, pubsub@localhost, http@localhost, s2s@localhost]; trying VHost lookup
2021-07-17 12:33:25.372 [in_0-message-router]  MessageRouter.processPacket()       FINEST:   1. Packet will be processed by: c2s@localhost, from=sess-man@localhost, to=c2s@localhost/127.0.0.1_5222_127.0.0.1_50377, DATA=<iq to="c2s@localhost/127.0.0.1_5222_127.0.0.1_50377" type="result" from="sess-man@localhost" id="a976d860-46be-4da2-b086-445ad0b1cec4"/>, SIZE=137, XMLNS=null, PRIORITY=SYSTEM, PERMISSION=NONE, TYPE=result
2021-07-17 12:33:25.372 [in_0-c2s]         ClientConnectionManager$StoppedHandler.responseReceived()  FINEST: Response for stop received...
```