---
layout: post
title: 'Kafka单机安装' 
date: 2021-09-13
author: 李新
tags:  Kafka
---

### (1). 概述
最近在研究Debezium,由于它和Kafka有强耦合,所以,需要研究下Kafka.

### (2). 环境准备

+ Jdk1.8(略) 
+ Zookeeper(略) 
+ Kafka_2.13-2.7.0

### (3). 下载并解压kafka
```
# 1. 查看当前目录
lixin@lixin Developer % pwd
/Users/lixin/Developer

# 2. 下载kafka
lixin@lixin Developer % https://archive.apache.org/dist/kafka/2.7.0/kafka_2.13-2.7.0.tgz

# 3. 解压
lixin@lixin Developer % tar -zxvf kafka_2.13-2.7.0.tgz

# 4. 创建logs目录
lixin@lixin kafka_2.13-2.7.0 % mkdir logs
```
### (4). 配置kakfa(config/server.properties)
```
# 每台机器配置唯一ID
broker.id=0
# 每台机器的主机名称
host.name=localhost
# 监听地址
listeners=PLAINTEXT://127.0.0.1:9092
# 自定义kafka数据文件存放目录
log.dirs=/Users/lixin/Developer/kafka_2.13-2.7.0
# 配置zk
zookeeper.connect=127.0.0.1:2181
```
### (5). 启动kafka
```
#  -daemon   : 后台启动
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-server-start.sh -daemon ./config/server.properties
[2021-09-13 09:39:28,715] INFO Registered kafka:type=kafka.Log4jController MBean (kafka.utils.Log4jControllerRegistration$)
[2021-09-13 09:39:28,966] INFO Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation (org.apache.zookeeper.common.X509Util)
[2021-09-13 09:39:29,015] INFO Registered signal handlers for TERM, INT, HUP (org.apache.kafka.common.utils.LoggingSignalHandler)
[2021-09-13 09:39:29,017] INFO starting (kafka.server.KafkaServer)
[2021-09-13 09:39:29,018] INFO Connecting to zookeeper on 127.0.0.1:2181 (kafka.server.KafkaServer)
[2021-09-13 09:39:29,032] INFO [ZooKeeperClient Kafka server] Initializing a new session to 127.0.0.1:2181. (kafka.zookeeper.ZooKeeperClient)
[2021-09-13 09:39:30,124] INFO Client environment:zookeeper.version=3.5.8-f439ca583e70862c3068a1f2a7d4d068eec33315, built on 05/04/2020 15:53 GMT (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,125] INFO Client environment:host.name=172.17.4.31 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,125] INFO Client environment:java.version=11.0.12 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,125] INFO Client environment:java.vendor=Oracle Corporation (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,125] INFO Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk-11.0.12.jdk/Contents/Home (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,125] INFO Client environment:java.class.path=.:/Library/Java/JavaVirtualMachines/jdk-11.0.12.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk-11.0.12.jdk/Contents/Home/lib/tools.jar:/Library/Java/JavaVirtualMachines/jdk-11.0.12.jdk/Contents/Home/lib/dt.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/activation-1.1.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/aopalliance-repackaged-2.6.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/argparse4j-0.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/audience-annotations-0.5.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/commons-cli-1.4.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/commons-lang3-3.8.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-api-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-basic-auth-extension-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-file-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-json-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-mirror-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-mirror-client-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-runtime-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/connect-transforms-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/hk2-api-2.6.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/hk2-locator-2.6.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/hk2-utils-2.6.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-annotations-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-core-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-databind-2.10.5.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-dataformat-csv-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-datatype-jdk8-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-jaxrs-base-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-jaxrs-json-provider-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-module-jaxb-annotations-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-module-paranamer-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jackson-module-scala_2.13-2.10.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.activation-api-1.2.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.annotation-api-1.3.5.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.inject-2.6.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.validation-api-2.0.2.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.ws.rs-api-2.1.6.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jakarta.xml.bind-api-2.3.2.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/javassist-3.25.0-GA.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/javassist-3.26.0-GA.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/javax.servlet-api-3.1.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/javax.ws.rs-api-2.1.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jaxb-api-2.3.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-client-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-common-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-container-servlet-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-container-servlet-core-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-hk2-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-media-jaxb-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jersey-server-2.31.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-client-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-continuation-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-http-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-io-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-security-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-server-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-servlet-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-servlets-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jetty-util-9.4.33.v20201020.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/jopt-simple-5.0.4.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-clients-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-log4j-appender-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-raft-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-streams-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-streams-examples-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-streams-scala_2.13-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-streams-test-utils-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka-tools-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka_2.13-2.7.0-sources.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/kafka_2.13-2.7.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/log4j-1.2.17.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/lz4-java-1.7.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/maven-artifact-3.6.3.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/metrics-core-2.2.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-buffer-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-codec-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-common-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-handler-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-resolver-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-transport-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-transport-native-epoll-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/netty-transport-native-unix-common-4.1.51.Final.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/osgi-resource-locator-1.0.3.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/paranamer-2.8.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/plexus-utils-3.2.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/reflections-0.9.12.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/rocksdbjni-5.18.4.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/scala-collection-compat_2.13-2.2.0.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/scala-java8-compat_2.13-0.9.1.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/scala-library-2.13.3.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/scala-logging_2.13-3.9.2.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/scala-reflect-2.13.3.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/slf4j-api-1.7.30.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/slf4j-log4j12-1.7.30.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/snappy-java-1.1.7.7.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/zookeeper-3.5.8.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/zookeeper-jute-3.5.8.jar:/Users/infinova/Developer/kafka_2.13-2.7.0/bin/../libs/zstd-jni-1.4.5-6.jar (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:java.library.path=/Users/infinova/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:. (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:java.io.tmpdir=/var/folders/6j/p053w95x0_d9_351x71yz_700000gp/T/ (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:java.compiler=<NA> (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.name=Mac OS X (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.arch=x86_64 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.version=11.5 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:user.name=lixin (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:user.home=/Users/lixin (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:user.dir=/Users/lixin/Developer/kafka_2.13-2.7.0 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.memory.free=1012MB (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.memory.max=1024MB (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,128] INFO Client environment:os.memory.total=1024MB (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,131] INFO Initiating client connection, connectString=127.0.0.1:2181 sessionTimeout=18000 watcher=kafka.zookeeper.ZooKeeperClient$ZooKeeperClientWatcher$@2dc995f4 (org.apache.zookeeper.ZooKeeper)
[2021-09-13 09:39:30,139] INFO jute.maxbuffer value is 4194304 Bytes (org.apache.zookeeper.ClientCnxnSocket)
[2021-09-13 09:39:30,144] INFO zookeeper.request.timeout value is 0. feature enabled= (org.apache.zookeeper.ClientCnxn)
[2021-09-13 09:39:30,147] INFO [ZooKeeperClient Kafka server] Waiting until connected. (kafka.zookeeper.ZooKeeperClient)
[2021-09-13 09:39:30,153] INFO Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error) (org.apache.zookeeper.ClientCnxn)
[2021-09-13 09:39:30,167] INFO Socket connection established, initiating session, client: /127.0.0.1:53863, server: localhost/127.0.0.1:2181 (org.apache.zookeeper.ClientCnxn)
[2021-09-13 09:39:30,204] INFO Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x100002a7c400001, negotiated timeout = 18000 (org.apache.zookeeper.ClientCnxn)
[2021-09-13 09:39:30,206] INFO [ZooKeeperClient Kafka server] Connected. (kafka.zookeeper.ZooKeeperClient)
[2021-09-13 09:39:30,693] INFO [feature-zk-node-event-process-thread]: Starting (kafka.server.FinalizedFeatureChangeListener$ChangeNotificationProcessorThread)
[2021-09-13 09:39:30,705] INFO Feature ZK node at path: /feature does not exist (kafka.server.FinalizedFeatureChangeListener)
[2021-09-13 09:39:30,706] INFO Cleared cache (kafka.server.FinalizedFeatureCache)
[2021-09-13 09:39:30,890] INFO Cluster ID = Gt9uC3FEQBSxFvYkdqze_Q (kafka.server.KafkaServer)
[2021-09-13 09:39:30,895] WARN No meta.properties file under dir /Users/lixin/Developer/kafka_2.13-2.7.0/logs/meta.properties (kafka.server.BrokerMetadataCheckpoint)
[2021-09-13 09:39:30,939] INFO KafkaConfig values:
	advertised.host.name = null
	advertised.listeners = null
	advertised.port = null
	alter.config.policy.class.name = null
	alter.log.dirs.replication.quota.window.num = 11
	alter.log.dirs.replication.quota.window.size.seconds = 1
	authorizer.class.name =
	auto.create.topics.enable = true
	auto.leader.rebalance.enable = true
	background.threads = 10
	broker.id = 0
	broker.id.generation.enable = true
	broker.rack = null
	client.quota.callback.class = null
	compression.type = producer
	connection.failed.authentication.delay.ms = 100
	connections.max.idle.ms = 600000
	connections.max.reauth.ms = 0
	control.plane.listener.name = null
	controlled.shutdown.enable = true
	controlled.shutdown.max.retries = 3
	controlled.shutdown.retry.backoff.ms = 5000
	controller.quota.window.num = 11
	controller.quota.window.size.seconds = 1
	controller.socket.timeout.ms = 30000
	create.topic.policy.class.name = null
	default.replication.factor = 1
	delegation.token.expiry.check.interval.ms = 3600000
	delegation.token.expiry.time.ms = 86400000
	delegation.token.master.key = null
	delegation.token.max.lifetime.ms = 604800000
	delete.records.purgatory.purge.interval.requests = 1
	delete.topic.enable = true
	fetch.max.bytes = 57671680
	fetch.purgatory.purge.interval.requests = 1000
	group.initial.rebalance.delay.ms = 0
	group.max.session.timeout.ms = 1800000
	group.max.size = 2147483647
	group.min.session.timeout.ms = 6000
	host.name = localhost
	inter.broker.listener.name = null
	inter.broker.protocol.version = 2.7-IV2
	kafka.metrics.polling.interval.secs = 10
	kafka.metrics.reporters = []
	leader.imbalance.check.interval.seconds = 300
	leader.imbalance.per.broker.percentage = 10
	listener.security.protocol.map = PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
	listeners = PLAINTEXT://127.0.0.1:9092
	log.cleaner.backoff.ms = 15000
	log.cleaner.dedupe.buffer.size = 134217728
	log.cleaner.delete.retention.ms = 86400000
	log.cleaner.enable = true
	log.cleaner.io.buffer.load.factor = 0.9
	log.cleaner.io.buffer.size = 524288
	log.cleaner.io.max.bytes.per.second = 1.7976931348623157E308
	log.cleaner.max.compaction.lag.ms = 9223372036854775807
	log.cleaner.min.cleanable.ratio = 0.5
	log.cleaner.min.compaction.lag.ms = 0
	log.cleaner.threads = 1
	log.cleanup.policy = [delete]
	log.dir = /tmp/kafka-logs
	log.dirs = /Users/lixin/Developer/kafka_2.13-2.7.0/logs
	log.flush.interval.messages = 9223372036854775807
	log.flush.interval.ms = null
	log.flush.offset.checkpoint.interval.ms = 60000
	log.flush.scheduler.interval.ms = 9223372036854775807
	log.flush.start.offset.checkpoint.interval.ms = 60000
	log.index.interval.bytes = 4096
	log.index.size.max.bytes = 10485760
	log.message.downconversion.enable = true
	log.message.format.version = 2.7-IV2
	log.message.timestamp.difference.max.ms = 9223372036854775807
	log.message.timestamp.type = CreateTime
	log.preallocate = false
	log.retention.bytes = -1
	log.retention.check.interval.ms = 300000
	log.retention.hours = 168
	log.retention.minutes = null
	log.retention.ms = null
	log.roll.hours = 168
	log.roll.jitter.hours = 0
	log.roll.jitter.ms = null
	log.roll.ms = null
	log.segment.bytes = 1073741824
	log.segment.delete.delay.ms = 60000
	max.connection.creation.rate = 2147483647
	max.connections = 2147483647
	max.connections.per.ip = 2147483647
	max.connections.per.ip.overrides =
	max.incremental.fetch.session.cache.slots = 1000
	message.max.bytes = 1048588
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	min.insync.replicas = 1
	num.io.threads = 8
	num.network.threads = 3
	num.partitions = 1
	num.recovery.threads.per.data.dir = 1
	num.replica.alter.log.dirs.threads = null
	num.replica.fetchers = 1
	offset.metadata.max.bytes = 4096
	offsets.commit.required.acks = -1
	offsets.commit.timeout.ms = 5000
	offsets.load.buffer.size = 5242880
	offsets.retention.check.interval.ms = 600000
	offsets.retention.minutes = 10080
	offsets.topic.compression.codec = 0
	offsets.topic.num.partitions = 50
	offsets.topic.replication.factor = 1
	offsets.topic.segment.bytes = 104857600
	password.encoder.cipher.algorithm = AES/CBC/PKCS5Padding
	password.encoder.iterations = 4096
	password.encoder.key.length = 128
	password.encoder.keyfactory.algorithm = null
	password.encoder.old.secret = null
	password.encoder.secret = null
	port = 9092
	principal.builder.class = null
	producer.purgatory.purge.interval.requests = 1000
	queued.max.request.bytes = -1
	queued.max.requests = 500
	quota.consumer.default = 9223372036854775807
	quota.producer.default = 9223372036854775807
	quota.window.num = 11
	quota.window.size.seconds = 1
	replica.fetch.backoff.ms = 1000
	replica.fetch.max.bytes = 1048576
	replica.fetch.min.bytes = 1
	replica.fetch.response.max.bytes = 10485760
	replica.fetch.wait.max.ms = 500
	replica.high.watermark.checkpoint.interval.ms = 5000
	replica.lag.time.max.ms = 30000
	replica.selector.class = null
	replica.socket.receive.buffer.bytes = 65536
	replica.socket.timeout.ms = 30000
	replication.quota.window.num = 11
	replication.quota.window.size.seconds = 1
	request.timeout.ms = 30000
	reserved.broker.max.id = 1000
	sasl.client.callback.handler.class = null
	sasl.enabled.mechanisms = [GSSAPI]
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.principal.to.local.rules = [DEFAULT]
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.login.callback.handler.class = null
	sasl.login.class = null
	sasl.login.refresh.buffer.seconds = 300
	sasl.login.refresh.min.period.seconds = 60
	sasl.login.refresh.window.factor = 0.8
	sasl.login.refresh.window.jitter = 0.05
	sasl.mechanism.inter.broker.protocol = GSSAPI
	sasl.server.callback.handler.class = null
	security.inter.broker.protocol = PLAINTEXT
	security.providers = null
	socket.connection.setup.timeout.max.ms = 127000
	socket.connection.setup.timeout.ms = 10000
	socket.receive.buffer.bytes = 102400
	socket.request.max.bytes = 104857600
	socket.send.buffer.bytes = 102400
	ssl.cipher.suites = []
	ssl.client.auth = none
	ssl.enabled.protocols = [TLSv1.2, TLSv1.3]
	ssl.endpoint.identification.algorithm = https
	ssl.engine.factory.class = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.certificate.chain = null
	ssl.keystore.key = null
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.principal.mapping.rules = DEFAULT
	ssl.protocol = TLSv1.3
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.certificates = null
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	transaction.abort.timed.out.transaction.cleanup.interval.ms = 10000
	transaction.max.timeout.ms = 900000
	transaction.remove.expired.transaction.cleanup.interval.ms = 3600000
	transaction.state.log.load.buffer.size = 5242880
	transaction.state.log.min.isr = 1
	transaction.state.log.num.partitions = 50
	transaction.state.log.replication.factor = 1
	transaction.state.log.segment.bytes = 104857600
	transactional.id.expiration.ms = 604800000
	unclean.leader.election.enable = false
	zookeeper.clientCnxnSocket = null
	zookeeper.connect = 127.0.0.1:2181
	zookeeper.connection.timeout.ms = 18000
	zookeeper.max.in.flight.requests = 10
	zookeeper.session.timeout.ms = 18000
	zookeeper.set.acl = false
	zookeeper.ssl.cipher.suites = null
	zookeeper.ssl.client.enable = false
	zookeeper.ssl.crl.enable = false
	zookeeper.ssl.enabled.protocols = null
	zookeeper.ssl.endpoint.identification.algorithm = HTTPS
	zookeeper.ssl.keystore.location = null
	zookeeper.ssl.keystore.password = null
	zookeeper.ssl.keystore.type = null
	zookeeper.ssl.ocsp.enable = false
	zookeeper.ssl.protocol = TLSv1.2
	zookeeper.ssl.truststore.location = null
	zookeeper.ssl.truststore.password = null
	zookeeper.ssl.truststore.type = null
	zookeeper.sync.time.ms = 2000
 (kafka.server.KafkaConfig)
[2021-09-13 09:39:30,946] INFO KafkaConfig values:
	advertised.host.name = null
	advertised.listeners = null
	advertised.port = null
	alter.config.policy.class.name = null
	alter.log.dirs.replication.quota.window.num = 11
	alter.log.dirs.replication.quota.window.size.seconds = 1
	authorizer.class.name =
	auto.create.topics.enable = true
	auto.leader.rebalance.enable = true
	background.threads = 10
	broker.id = 0
	broker.id.generation.enable = true
	broker.rack = null
	client.quota.callback.class = null
	compression.type = producer
	connection.failed.authentication.delay.ms = 100
	connections.max.idle.ms = 600000
	connections.max.reauth.ms = 0
	control.plane.listener.name = null
	controlled.shutdown.enable = true
	controlled.shutdown.max.retries = 3
	controlled.shutdown.retry.backoff.ms = 5000
	controller.quota.window.num = 11
	controller.quota.window.size.seconds = 1
	controller.socket.timeout.ms = 30000
	create.topic.policy.class.name = null
	default.replication.factor = 1
	delegation.token.expiry.check.interval.ms = 3600000
	delegation.token.expiry.time.ms = 86400000
	delegation.token.master.key = null
	delegation.token.max.lifetime.ms = 604800000
	delete.records.purgatory.purge.interval.requests = 1
	delete.topic.enable = true
	fetch.max.bytes = 57671680
	fetch.purgatory.purge.interval.requests = 1000
	group.initial.rebalance.delay.ms = 0
	group.max.session.timeout.ms = 1800000
	group.max.size = 2147483647
	group.min.session.timeout.ms = 6000
	host.name = localhost
	inter.broker.listener.name = null
	inter.broker.protocol.version = 2.7-IV2
	kafka.metrics.polling.interval.secs = 10
	kafka.metrics.reporters = []
	leader.imbalance.check.interval.seconds = 300
	leader.imbalance.per.broker.percentage = 10
	listener.security.protocol.map = PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
	listeners = PLAINTEXT://127.0.0.1:9092
	log.cleaner.backoff.ms = 15000
	log.cleaner.dedupe.buffer.size = 134217728
	log.cleaner.delete.retention.ms = 86400000
	log.cleaner.enable = true
	log.cleaner.io.buffer.load.factor = 0.9
	log.cleaner.io.buffer.size = 524288
	log.cleaner.io.max.bytes.per.second = 1.7976931348623157E308
	log.cleaner.max.compaction.lag.ms = 9223372036854775807
	log.cleaner.min.cleanable.ratio = 0.5
	log.cleaner.min.compaction.lag.ms = 0
	log.cleaner.threads = 1
	log.cleanup.policy = [delete]
	log.dir = /tmp/kafka-logs
	log.dirs = /Users/infinova/Developer/kafka_2.13-2.7.0/logs
	log.flush.interval.messages = 9223372036854775807
	log.flush.interval.ms = null
	log.flush.offset.checkpoint.interval.ms = 60000
	log.flush.scheduler.interval.ms = 9223372036854775807
	log.flush.start.offset.checkpoint.interval.ms = 60000
	log.index.interval.bytes = 4096
	log.index.size.max.bytes = 10485760
	log.message.downconversion.enable = true
	log.message.format.version = 2.7-IV2
	log.message.timestamp.difference.max.ms = 9223372036854775807
	log.message.timestamp.type = CreateTime
	log.preallocate = false
	log.retention.bytes = -1
	log.retention.check.interval.ms = 300000
	log.retention.hours = 168
	log.retention.minutes = null
	log.retention.ms = null
	log.roll.hours = 168
	log.roll.jitter.hours = 0
	log.roll.jitter.ms = null
	log.roll.ms = null
	log.segment.bytes = 1073741824
	log.segment.delete.delay.ms = 60000
	max.connection.creation.rate = 2147483647
	max.connections = 2147483647
	max.connections.per.ip = 2147483647
	max.connections.per.ip.overrides =
	max.incremental.fetch.session.cache.slots = 1000
	message.max.bytes = 1048588
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	min.insync.replicas = 1
	num.io.threads = 8
	num.network.threads = 3
	num.partitions = 1
	num.recovery.threads.per.data.dir = 1
	num.replica.alter.log.dirs.threads = null
	num.replica.fetchers = 1
	offset.metadata.max.bytes = 4096
	offsets.commit.required.acks = -1
	offsets.commit.timeout.ms = 5000
	offsets.load.buffer.size = 5242880
	offsets.retention.check.interval.ms = 600000
	offsets.retention.minutes = 10080
	offsets.topic.compression.codec = 0
	offsets.topic.num.partitions = 50
	offsets.topic.replication.factor = 1
	offsets.topic.segment.bytes = 104857600
	password.encoder.cipher.algorithm = AES/CBC/PKCS5Padding
	password.encoder.iterations = 4096
	password.encoder.key.length = 128
	password.encoder.keyfactory.algorithm = null
	password.encoder.old.secret = null
	password.encoder.secret = null
	port = 9092
	principal.builder.class = null
	producer.purgatory.purge.interval.requests = 1000
	queued.max.request.bytes = -1
	queued.max.requests = 500
	quota.consumer.default = 9223372036854775807
	quota.producer.default = 9223372036854775807
	quota.window.num = 11
	quota.window.size.seconds = 1
	replica.fetch.backoff.ms = 1000
	replica.fetch.max.bytes = 1048576
	replica.fetch.min.bytes = 1
	replica.fetch.response.max.bytes = 10485760
	replica.fetch.wait.max.ms = 500
	replica.high.watermark.checkpoint.interval.ms = 5000
	replica.lag.time.max.ms = 30000
	replica.selector.class = null
	replica.socket.receive.buffer.bytes = 65536
	replica.socket.timeout.ms = 30000
	replication.quota.window.num = 11
	replication.quota.window.size.seconds = 1
	request.timeout.ms = 30000
	reserved.broker.max.id = 1000
	sasl.client.callback.handler.class = null
	sasl.enabled.mechanisms = [GSSAPI]
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.principal.to.local.rules = [DEFAULT]
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.login.callback.handler.class = null
	sasl.login.class = null
	sasl.login.refresh.buffer.seconds = 300
	sasl.login.refresh.min.period.seconds = 60
	sasl.login.refresh.window.factor = 0.8
	sasl.login.refresh.window.jitter = 0.05
	sasl.mechanism.inter.broker.protocol = GSSAPI
	sasl.server.callback.handler.class = null
	security.inter.broker.protocol = PLAINTEXT
	security.providers = null
	socket.connection.setup.timeout.max.ms = 127000
	socket.connection.setup.timeout.ms = 10000
	socket.receive.buffer.bytes = 102400
	socket.request.max.bytes = 104857600
	socket.send.buffer.bytes = 102400
	ssl.cipher.suites = []
	ssl.client.auth = none
	ssl.enabled.protocols = [TLSv1.2, TLSv1.3]
	ssl.endpoint.identification.algorithm = https
	ssl.engine.factory.class = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.certificate.chain = null
	ssl.keystore.key = null
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.principal.mapping.rules = DEFAULT
	ssl.protocol = TLSv1.3
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.certificates = null
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	transaction.abort.timed.out.transaction.cleanup.interval.ms = 10000
	transaction.max.timeout.ms = 900000
	transaction.remove.expired.transaction.cleanup.interval.ms = 3600000
	transaction.state.log.load.buffer.size = 5242880
	transaction.state.log.min.isr = 1
	transaction.state.log.num.partitions = 50
	transaction.state.log.replication.factor = 1
	transaction.state.log.segment.bytes = 104857600
	transactional.id.expiration.ms = 604800000
	unclean.leader.election.enable = false
	zookeeper.clientCnxnSocket = null
	zookeeper.connect = 127.0.0.1:2181
	zookeeper.connection.timeout.ms = 18000
	zookeeper.max.in.flight.requests = 10
	zookeeper.session.timeout.ms = 18000
	zookeeper.set.acl = false
	zookeeper.ssl.cipher.suites = null
	zookeeper.ssl.client.enable = false
	zookeeper.ssl.crl.enable = false
	zookeeper.ssl.enabled.protocols = null
	zookeeper.ssl.endpoint.identification.algorithm = HTTPS
	zookeeper.ssl.keystore.location = null
	zookeeper.ssl.keystore.password = null
	zookeeper.ssl.keystore.type = null
	zookeeper.ssl.ocsp.enable = false
	zookeeper.ssl.protocol = TLSv1.2
	zookeeper.ssl.truststore.location = null
	zookeeper.ssl.truststore.password = null
	zookeeper.ssl.truststore.type = null
	zookeeper.sync.time.ms = 2000
 (kafka.server.KafkaConfig)
[2021-09-13 09:39:30,977] INFO [ThrottledChannelReaper-Fetch]: Starting (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
[2021-09-13 09:39:30,978] INFO [ThrottledChannelReaper-Produce]: Starting (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
[2021-09-13 09:39:30,979] INFO [ThrottledChannelReaper-Request]: Starting (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
[2021-09-13 09:39:30,980] INFO [ThrottledChannelReaper-ControllerMutation]: Starting (kafka.server.ClientQuotaManager$ThrottledChannelReaper)
[2021-09-13 09:39:31,007] INFO Loading logs from log dirs ArraySeq(/Users/lixin/Developer/kafka_2.13-2.7.0/logs) (kafka.log.LogManager)
[2021-09-13 09:39:31,009] INFO Attempting recovery for all logs in /Users/lixin/Developer/kafka_2.13-2.7.0/logs since no clean shutdown file was found (kafka.log.LogManager)
[2021-09-13 09:39:31,014] INFO Loaded 0 logs in 6ms. (kafka.log.LogManager)
[2021-09-13 09:39:31,028] INFO Starting log cleanup with a period of 300000 ms. (kafka.log.LogManager)
[2021-09-13 09:39:31,030] INFO Starting log flusher with a default period of 9223372036854775807 ms. (kafka.log.LogManager)
[2021-09-13 09:39:31,380] INFO Created ConnectionAcceptRate sensor, quotaLimit=2147483647 (kafka.network.ConnectionQuotas)
[2021-09-13 09:39:31,382] INFO Created ConnectionAcceptRate-PLAINTEXT sensor, quotaLimit=2147483647 (kafka.network.ConnectionQuotas)
[2021-09-13 09:39:31,383] INFO Updated PLAINTEXT max connection creation rate to 2147483647 (kafka.network.ConnectionQuotas)
[2021-09-13 09:39:31,387] INFO Awaiting socket connections on 127.0.0.1:9092. (kafka.network.Acceptor)
[2021-09-13 09:39:31,421] INFO [SocketServer brokerId=0] Created data-plane acceptor and processors for endpoint : ListenerName(PLAINTEXT) (kafka.network.SocketServer)
[2021-09-13 09:39:31,447] INFO [ExpirationReaper-0-Produce]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,448] INFO [ExpirationReaper-0-Fetch]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,448] INFO [ExpirationReaper-0-DeleteRecords]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,449] INFO [ExpirationReaper-0-ElectLeader]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,461] INFO [LogDirFailureHandler]: Starting (kafka.server.ReplicaManager$LogDirFailureHandler)
[2021-09-13 09:39:31,461] INFO [broker-0-to-controller-send-thread]: Starting (kafka.server.BrokerToControllerRequestThread)
[2021-09-13 09:39:31,489] INFO Creating /brokers/ids/0 (is it secure? false) (kafka.zk.KafkaZkClient)
[2021-09-13 09:39:31,531] INFO Stat of the created znode at /brokers/ids/0 is: 26,26,1631497171504,1631497171504,1,0,0,72057776511123457,202,0,26
 (kafka.zk.KafkaZkClient)
[2021-09-13 09:39:31,531] INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT://127.0.0.1:9092, czxid (broker epoch): 26 (kafka.zk.KafkaZkClient)
[2021-09-13 09:39:31,584] INFO [ExpirationReaper-0-topic]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,587] INFO [ExpirationReaper-0-Heartbeat]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,588] INFO [ExpirationReaper-0-Rebalance]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,610] INFO Successfully created /controller_epoch with initial epoch 0 (kafka.zk.KafkaZkClient)
[2021-09-13 09:39:31,621] INFO [GroupCoordinator 0]: Starting up. (kafka.coordinator.group.GroupCoordinator)
[2021-09-13 09:39:31,634] INFO [GroupCoordinator 0]: Startup complete. (kafka.coordinator.group.GroupCoordinator)
[2021-09-13 09:39:31,666] INFO [ProducerId Manager 0]: Acquired new producerId block (brokerId:0,blockStartProducerId:0,blockEndProducerId:999) by writing to Zk with path version 1 (kafka.coordinator.transaction.ProducerIdManager)
[2021-09-13 09:39:31,685] INFO Feature ZK node created at path: /feature (kafka.server.FinalizedFeatureChangeListener)
[2021-09-13 09:39:31,697] INFO [TransactionCoordinator id=0] Starting up. (kafka.coordinator.transaction.TransactionCoordinator)
[2021-09-13 09:39:31,699] INFO [Transaction Marker Channel Manager 0]: Starting (kafka.coordinator.transaction.TransactionMarkerChannelManager)
[2021-09-13 09:39:31,699] INFO [TransactionCoordinator id=0] Startup complete. (kafka.coordinator.transaction.TransactionCoordinator)
[2021-09-13 09:39:31,721] INFO [ExpirationReaper-0-AlterAcls]: Starting (kafka.server.DelayedOperationPurgatory$ExpiredOperationReaper)
[2021-09-13 09:39:31,728] INFO Updated cache from existing <empty> to latest FinalizedFeaturesAndEpoch(features=Features{}, epoch=0). (kafka.server.FinalizedFeatureCache)
[2021-09-13 09:39:31,735] INFO [/config/changes-event-process-thread]: Starting (kafka.common.ZkNodeChangeNotificationListener$ChangeEventProcessThread)
[2021-09-13 09:39:31,745] INFO [SocketServer brokerId=0] Starting socket server acceptors and processors (kafka.network.SocketServer)
[2021-09-13 09:39:31,748] INFO [SocketServer brokerId=0] Started data-plane acceptor and processor(s) for endpoint : ListenerName(PLAINTEXT) (kafka.network.SocketServer)
[2021-09-13 09:39:31,749] INFO [SocketServer brokerId=0] Started socket server acceptors and processors (kafka.network.SocketServer)
[2021-09-13 09:39:31,753] INFO Kafka version: 2.7.0 (org.apache.kafka.common.utils.AppInfoParser)
[2021-09-13 09:39:31,753] INFO Kafka commitId: 448719dc99a19793 (org.apache.kafka.common.utils.AppInfoParser)
[2021-09-13 09:39:31,753] INFO Kafka startTimeMs: 1631497171749 (org.apache.kafka.common.utils.AppInfoParser)
[2021-09-13 09:39:31,754] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
[2021-09-13 09:39:31,875] INFO [broker-0-to-controller-send-thread]: Recorded new controller, from now on will use broker 0 (kafka.server.BrokerToControllerRequestThread)
```
### (6). 创建分区
```
# --create                 : 创建topic
# --zookeeper              : 指定zk地址
# --replication-factor     : 指定副本数
# --partitions             : 指定分区数
# --topic                  : 指定topic名称
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 1 --topic hello
Created topic hello.
```
### (7). 查看分区列表
```
#  --list                  : 查看所有的主题
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-topics.sh --list --zookeeper 127.0.0.1:2181
hello
```
### (8). 生产者生产消息
```
# --broker-list            : 指定brokder
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic hello
>hello world
>test world
```
### (9). 消费者消费消息
```
# --from-beginning         : 从开始位置进行消费
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092  --from-beginning --topic hello
hello world
test world
```
### (10). 删除主题
```
# 删除主题
lixin@lixin kafka_2.13-2.7.0 % ./bin/kafka-topics.sh --zookeeper 127.0.0.1:2181 --delete --topic  hello
Topic hello is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```

 