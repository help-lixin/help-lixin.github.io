---
layout: post
title: 'Apache BookKeeper源码编译(一)' 
date: 2021-10-05
author: 李新
tags:  BookKeeper
---

### (1). 概述
前面对BookKeeper的概念有了一个了解,在这里,需要弄个Hello World,以便增加对BookKeeper的入门.   

### (2). 源码下载并编译
```
lixin-macbook:~ lixin$ cd ~/GitRepository/
# 1. 下载bookkeeper
lixin-macbook:GitRepository lixin$ git clone https://github.com/help-lixin/bookkeeper.git

# 2. 进入bookeeper工程目录
lixin-macbook:GitRepository lixin$ cd bookkeeper/

# 3. 查看bookkeeper工程目录结果
lixin-macbook:bookkeeper lixin$ ll
-rw-r--r--    1 lixin  staff   2690 10  5 12:46 Jenkinsfile
drwxr-xr-x   12 lixin  staff    384 10  5 14:28 bin/                      # 脚本程序目录
drwxr-xr-x    7 lixin  staff    224 10  5 12:46 bookkeeper-benchmark/
drwxr-xr-x    6 lixin  staff    192 10  5 13:49 bookkeeper-common/
drwxr-xr-x    6 lixin  staff    192 10  5 13:49 bookkeeper-common-allocator/
drwxr-xr-x    9 lixin  staff    288 10  5 13:56 bookkeeper-dist/           #  二进制包目上录
drwxr-xr-x    6 lixin  staff    192 10  5 12:46 bookkeeper-http/
drwxr-xr-x    6 lixin  staff    192 10  5 13:49 bookkeeper-proto/
drwxr-xr-x    6 lixin  staff    192 10  5 13:50 bookkeeper-server/         # bookkeeper服务端程序
drwxr-xr-x    6 lixin  staff    192 10  5 13:49 bookkeeper-stats/
drwxr-xr-x    5 lixin  staff    160 10  5 12:46 bookkeeper-stats-providers/
-rw-r--r--    1 lixin  staff   7533 10  5 12:46 build.gradle
drwxr-xr-x    6 lixin  staff    192 10  5 13:48 buildtools/
drwxr-xr-x    6 lixin  staff    192 10  5 13:48 circe-checksum/
drwxr-xr-x   13 lixin  staff    416 10  5 14:30 conf/                      # 配置目录
drwxr-xr-x    6 lixin  staff    192 10  5 13:49 cpu-affinity/
-rw-r--r--    1 lixin  staff   8698 10  5 12:46 dependencies.gradle
drwxr-xr-x    4 lixin  staff    128 10  5 12:46 deploy/
drwxr-xr-x   13 lixin  staff    416 10  5 12:46 dev/
drwxr-xr-x    8 lixin  staff    256 10  5 12:46 docker/
drwxr-xr-x    3 lixin  staff     96 10  5 12:46 gradle/
-rw-r--r--    1 lixin  staff    988 10  5 12:46 gradle.properties
-rwxr-xr-x    1 lixin  staff   5896 10  5 12:46 gradlew*
-rw-r--r--    1 lixin  staff   2936 10  5 12:46 gradlew.bat
drwxr-xr-x    3 lixin  staff     96 10  5 14:29 logs/
drwxr-xr-x    4 lixin  staff    128 10  5 12:46 metadata-drivers/
drwxr-xr-x    8 lixin  staff    256 10  5 12:46 microbenchmarks/
-rw-r--r--    1 lixin  staff  42791 10  5 12:46 pom.xml
-rw-r--r--    1 lixin  staff   3886 10  5 12:46 settings.gradle
drwxr-xr-x    6 lixin  staff    192 10  5 12:46 shaded/
drwxr-xr-x   28 lixin  staff    896 10  5 12:46 site/
drwxr-xr-x    8 lixin  staff    256 10  5 12:46 site2/
drwxr-xr-x    4 lixin  staff    128 10  5 12:46 stats/
drwxr-xr-x   21 lixin  staff    672 10  5 13:52 stream/
drwxr-xr-x    4 lixin  staff    128 10  5 13:48 target/
drwxr-xr-x   12 lixin  staff    384 10  5 12:46 tests/
drwxr-xr-x    9 lixin  staff    288 10  5 13:49 tools/

# 4. 编译并产生dist
lixin-macbook:bookkeeper lixin$ mvn clean install package -Dstream -am -pl bookkeeper-dist/server -DskipTests

# 5. 查看产生的bookkeeper二进制文件.
lixin-macbook:bookkeeper lixin$ ll  bookkeeper-dist/server/target/ |grep gz
-rw-r--r--  1 lixin  staff  101102104 10  5 13:56 bookkeeper-server-4.15.0-SNAPSHOT-bin.tar.gz
```
### (3). 总结
一般一个优盘的框架有一个好的表现就是基本能一把编译通过,后面会用一个简单的Hello World,来入门.   