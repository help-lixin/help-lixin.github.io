---
layout: post
title: 'Apache BookKeeper集群搭建(四)' 
date: 2021-10-05
author: 李新
tags:  BookKeeper
---

### (1). 概述
在这一小节,搭建一个BookKeeper的集群
### (2). 机器准备

|  节点名称       | Port    |
|  ----         | ----    |
| bookkeeper-1  | 3181    |
| bookkeeper-2  | 3182    |
| bookkeeper-3  | 3183    |

### (3). 集群搭建步骤
+ Zookeeper(略)
+ BookKeeper(3台/伪集群)  

### (4). BookKeeper骨架搭建
```
# 1. 这个目录是我从git拉取下来的源码:bookkeeper
lixin-macbook:~ lixin$ cd /Users/lixin/GitRepository/bookkeeper

# 2. bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz 是源码编译后的结果
lixin-macbook:bookkeeper lixin$ cp bookkeeper-dist/server/target/bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz ~/Downloads/

# 3. 创建应用程序目录
lixin-macbook:bookkeeper lixin$ mkdir -p  ~/Developer/bookkeeper-servers/{bookkeeper-1,bookkeeper-2,bookkeeper-3}

# 4. 解压程序到:bookkeeper-1目录下
# --strip-components 1  : 去除解压的第一层目录
lixin-macbook:~ lixin$ tar -zxvf ~/Downloads/bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz --strip-components 1  -C ~/Developer/bookkeeper-servers/bookkeeper-1/
lixin-macbook:~ lixin$ tar -zxvf ~/Downloads/bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz --strip-components 1  -C ~/Developer/bookkeeper-servers/bookkeeper-2/
lixin-macbook:~ lixin$ tar -zxvf ~/Downloads/bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz --strip-components 1  -C ~/Developer/bookkeeper-servers/bookkeeper-3/

# 5. 创建数据目录
lixin-macbook:~ lixin$ mkdir ~/Developer/bookkeeper-servers/bookkeeper-1/{bk-data,bk-txn}
lixin-macbook:~ lixin$ mkdir ~/Developer/bookkeeper-servers/bookkeeper-2/{bk-data,bk-txn}
lixin-macbook:~ lixin$ mkdir ~/Developer/bookkeeper-servers/bookkeeper-3/{bk-data,bk-txn}


# 6. 查看程序目录
lixin-macbook:~ lixin$ tree ~/Developer/bookkeeper-servers/  -L 2
├── bookkeeper-1
│   ├── bin
│   ├── bk-data
│   ├── bk-txn
│   ├── conf
│   ├── deps
│   ├── lib
│   └── logs
├── bookkeeper-2
│   ├── bin
│   ├── bk-data
│   ├── bk-txn
│   ├── conf
│   ├── deps
│   └── lib
└── bookkeeper-3
    ├── bin
    ├── bk-data
    ├── bk-txn
    ├── conf
    ├── deps
    └── lib
```
### (4). bookkeeper-1配置
```
lixin-macbook:~ lixin$ sed -i \
-e 's/bookiePort=3181/bookiePort=3181/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf/bk_server.conf

lixin-macbook:~ lixin$ sed -i \
-e 's/httpServerPort=8080/httpServerPort=8081/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's/storageserver.grpc.port=4181/storageserver.grpc.port=4181/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#ledgerDirectories=/tmp/bk-data#ledgerDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/bk-data#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#journalDirectories=/tmp/bk-txn#journalDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/bk-txn#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf/bk_server.conf
```
### (5). bookkeeper-2配置
```
lixin-macbook:~ lixin$ sed -i \
-e 's/bookiePort=3181/bookiePort=3182/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/conf/bk_server.conf

lixin-macbook:~ lixin$ sed -i \
-e 's/httpServerPort=8080/httpServerPort=8082/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's/storageserver.grpc.port=4181/storageserver.grpc.port=4182/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#ledgerDirectories=/tmp/bk-data#ledgerDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/bk-data#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#journalDirectories=/tmp/bk-txn#journalDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/bk-txn#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/conf/bk_server.conf
```
### (6). bookkeeper-3配置
```
lixin-macbook:~ lixin$ sed -i \
-e 's/bookiePort=3181/bookiePort=3183/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/conf/bk_server.conf

lixin-macbook:~ lixin$ sed -i \
-e 's/httpServerPort=8080/httpServerPort=8083/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's/storageserver.grpc.port=4181/storageserver.grpc.port=4183/g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#ledgerDirectories=/tmp/bk-data#ledgerDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/bk-data#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/conf/bk_server.conf


lixin-macbook:~ lixin$ sed -i \
-e 's#journalDirectories=/tmp/bk-txn#journalDirectories=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/bk-txn#g' \
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/conf/bk_server.conf
```
### (7). 初始化元数据
```
# 1. 在3个bookkeeper中,挑一个即可.
lixin-macbook:bookkeeper-1 lixin$ cd ~/Developer/bookkeeper-servers/bookkeeper-1/

# 2. 挑一台,初始化集群meta信息.
lixin-macbook:bookkeeper-1 lixin$ ./bin/bookkeeper shell metaformat
14:44:18,852 INFO  BookKeeper metadata driver manager initialized
14:44:18,867 INFO  Initialize zookeeper metadata driver at metadata service uri zk+hierarchical://localhost:2181/ledgers : zkServers = localhost:2181, ledgersRootPath = /ledgers.
14:44:18,908 INFO  Client environment:zookeeper.version=3.6.2--803c7f1a12f85978cb049af5e4ef23bd8b688715, built on 09/04/2020 12:44 GMT
14:44:18,908 INFO  Client environment:host.name=172.17.5.238
14:44:18,908 INFO  Client environment:java.version=1.8.0_251
14:44:18,908 INFO  Client environment:java.vendor=Oracle Corporation
14:44:18,908 INFO  Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/jre
14:44:18,908 INFO  Client environment:java.class.path=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/conf:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-server-4.15.0-SNAPSHOT.jar::/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.beust-jcommander-1.78.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.fasterxml.jackson.core-jackson-annotations-2.11.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.fasterxml.jackson.core-jackson-core-2.11.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.fasterxml.jackson.core-jackson-databind-2.11.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.android-annotations-4.1.1.4.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.api.grpc-proto-google-common-protos-1.17.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.auth-google-auth-library-credentials-0.20.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.auth-google-auth-library-oauth2-http-0.20.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.auto.value-auto-value-annotations-1.7.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.code.gson-gson-2.8.6.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.errorprone-error_prone_annotations-2.4.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.guava-failureaccess-1.0.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.guava-guava-30.0-jre.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.guava-listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.http-client-google-http-client-1.34.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.http-client-google-http-client-jackson2-1.34.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.j2objc-j2objc-annotations-1.3.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.protobuf-protobuf-java-3.14.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.protobuf-protobuf-java-util-3.12.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.google.re2j-re2j-1.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.yahoo.datasketches-memory-0.8.3.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/com.yahoo.datasketches-sketches-core-0.8.3.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-cli-commons-cli-1.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-codec-commons-codec-1.6.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-configuration-commons-configuration-1.10.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-io-commons-io-2.4.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-lang-commons-lang-2.6.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/commons-logging-commons-logging-1.1.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.dropwizard.metrics-metrics-core-3.2.5.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-all-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-alts-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-api-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-auth-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-context-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-core-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-grpclb-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-netty-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-protobuf-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-protobuf-lite-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-services-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-stub-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-testing-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.grpc-grpc-xds-1.33.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-buffer-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-codec-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-codec-dns-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-codec-http-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-codec-http2-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-codec-socks-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-common-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-handler-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-handler-proxy-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-resolver-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-resolver-dns-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-tcnative-boringssl-static-2.0.42.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-transport-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-transport-native-epoll-4.1.68.Final-linux-x86_64.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.netty-netty-transport-native-unix-common-4.1.68.Final.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.opencensus-opencensus-api-0.24.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.opencensus-opencensus-contrib-http-util-0.24.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.opencensus-opencensus-proto-0.2.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.perfmark-perfmark-api-0.19.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.prometheus-simpleclient-0.8.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.prometheus-simpleclient_common-0.8.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.prometheus-simpleclient_hotspot-0.8.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.prometheus-simpleclient_servlet-0.8.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.vertx-vertx-auth-common-3.9.8.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.vertx-vertx-bridge-common-3.9.8.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.vertx-vertx-core-3.9.8.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.vertx-vertx-web-3.9.8.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/io.vertx-vertx-web-common-3.9.8.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/javax.servlet-javax.servlet-api-4.0.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/log4j-log4j-1.2.17.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/net.java.dev.jna-jna-3.2.7.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/net.jpountz.lz4-lz4-1.3.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-common-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-common-allocator-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-proto-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-server-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-tools-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-tools-framework-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-bookkeeper-tools-ledger-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-circe-checksum-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-cpu-affinity-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-statelib-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-stream-storage-cli-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-stream-storage-java-client-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-stream-storage-server-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-stream-storage-service-api-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper-stream-storage-service-impl-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper.http-http-server-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper.http-vertx-http-server-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper.stats-bookkeeper-stats-api-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper.stats-prometheus-metrics-provider-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.bookkeeper.tests-stream-storage-tests-common-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.commons-commons-collections4-4.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.commons-commons-lang3-3.6.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.curator-curator-client-5.1.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.curator-curator-framework-5.1.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.curator-curator-recipes-5.1.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.distributedlog-distributedlog-common-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.distributedlog-distributedlog-core-4.15.0-SNAPSHOT-tests.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.distributedlog-distributedlog-core-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.distributedlog-distributedlog-protocol-4.15.0-SNAPSHOT.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.httpcomponents-httpclient-4.5.5.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.httpcomponents-httpcore-4.4.9.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.thrift-libthrift-0.14.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.yetus-audience-annotations-0.5.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.zookeeper-zookeeper-3.6.2-tests.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.zookeeper-zookeeper-3.6.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.apache.zookeeper-zookeeper-jute-3.6.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.bouncycastle-bc-fips-1.0.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.checkerframework-checker-qual-3.5.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.conscrypt-conscrypt-openjdk-uber-2.5.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-http-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-io-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-security-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-server-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-servlet-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.eclipse.jetty-jetty-util-9.4.33.v20201020.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.inferred-freebuilder-2.7.0.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.jctools-jctools-core-2.1.2.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.rocksdb-rocksdbjni-6.22.1.1.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.slf4j-slf4j-api-1.7.25.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.slf4j-slf4j-log4j12-1.7.25.jar:/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/lib/org.xerial.snappy-snappy-java-1.1.7.jar:
14:44:18,910 INFO  Client environment:java.library.path=/Users/lixin/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
14:44:18,910 INFO  Client environment:java.io.tmpdir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/
14:44:18,910 INFO  Client environment:java.compiler=<NA>
14:44:18,910 INFO  Client environment:os.name=Mac OS X
14:44:18,910 INFO  Client environment:os.arch=x86_64
14:44:18,910 INFO  Client environment:os.version=10.16
14:44:18,910 INFO  Client environment:user.name=lixin
14:44:18,910 INFO  Client environment:user.home=/Users/lixin
14:44:18,910 INFO  Client environment:user.dir=/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1
14:44:18,910 INFO  Client environment:os.memory.free=999MB
14:44:18,910 INFO  Client environment:os.memory.max=1024MB
14:44:18,910 INFO  Client environment:os.memory.total=1024MB
14:44:18,918 INFO  Initiating client connection, connectString=localhost:2181 sessionTimeout=10000 watcher=org.apache.bookkeeper.zookeeper.ZooKeeperWatcherBase@9660f4e
14:44:18,935 INFO  Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
14:44:18,949 INFO  jute.maxbuffer value is 1048575 Bytes
14:44:18,965 INFO  zookeeper.request.timeout value is 0. feature enabled=false
14:44:18,980 INFO  Opening socket connection to server localhost/127.0.0.1:2181.
14:44:18,980 INFO  SASL config status: Will not attempt to authenticate using SASL (unknown error)
14:44:18,999 INFO  Socket connection established, initiating session, client: /127.0.0.1:62089, server: localhost/127.0.0.1:2181
14:44:19,014 INFO  Session establishment complete on server localhost/127.0.0.1:2181, session id = 0x10000d978ac0001, negotiated timeout = 10000
14:44:19,019 INFO  ZooKeeper client is connected now.
14:44:19,296 INFO  Successfully formatted BookKeeper metadata
14:44:19,409 INFO  Session: 0x10000d978ac0001 closed
14:44:19,409 INFO  EventThread shut down for session: 0x10000d978ac0001
```
### (8). 启动所有bookie
```
# bookkeeper-1 控制台启动
lixin-macbook:bookkeeper-1 lixin$ ./bin/bookkeeper  bookie
2021-10-16 15:08:31,846 - INFO  - [main:ComponentStarter@86] - Started component bookie-server.
2021-10-16 15:08:32,344 - INFO  - [BookieJournal-3181:NativeIO@48] - Unable to link C library. Native methods will be disabled.

# bookkeeper-2 控制台启动
lixin-macbook:bookkeeper-2 lixin$ ./bin/bookkeeper  bookie
2021-10-16 15:08:38,923 - INFO  - [main:ComponentStarter@84] - Starting component bookie-server.
2021-10-16 15:08:38,939 - INFO  - [main:BookieImpl@920] - Finished replaying journal in 2 ms.

# bookkeeper-3 控制台启动
lixin-macbook:bookkeeper-3 lixin$ ./bin/bookkeeper  bookie
2021-10-16 15:08:41,319 - INFO  - [main:ComponentStarter@84] - Starting component bookie-server.
2021-10-16 15:08:41,332 - INFO  - [main:BookieImpl@920] - Finished replaying journal in 3 ms.
```
### (9). 创建批量启动脚本
```
# 1. 当前工作目录
lixin-macbook:bookkeeper-servers lixin$ pwd
/Users/lixin/Developer/bookkeeper-servers

# 2. 创建批量启动脚本
lixin-macbook:bookkeeper-servers lixin$ cat start-bookkeeper.sh
#!/bin/bash
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-1/bin/bookkeeper-daemon.sh start  bookie
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-2/bin/bookkeeper-daemon.sh start  bookie
/Users/lixin/Developer/bookkeeper-servers/bookkeeper-3/bin/bookkeeper-daemon.sh start  bookie

# 3. 设置执行权限
lixin-macbook:bookkeeper-servers lixin$ chmod 755 start-bookkeeper.sh
```
### (10). 查看ZK数据
```
# 在ZK中这些数据是持久化的
[zk: localhost:2181(CONNECTED) 9] ls /ledgers/cookies
[127.0.0.1:3181, 127.0.0.1:3182, 127.0.0.1:3183]
```
### (11). 注意事项
我的机器是动态获得IP,所以,当机器重启后IP就发生了变化,造成BK启动失败(启动时,是会向ZK进行校验的).建议配置:advertisedAddress   

### (12). 总结
BookKeeper集群,相对来说还是有一些麻烦来着的,后面,会结合一些Java Api进行入门.