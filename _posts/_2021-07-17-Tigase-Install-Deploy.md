---
layout: post
title: 'Tigase安装与部署(二)' 
date: 2021-07-17
author: 李新
tags:  Tigase 
---

### (1). Tigase源码下载与编译
```
# 1. 创建源码目录
lixin-macbook:~ lixin$ mkdir -p  ~/GitRepository/tigase
lixin-macbook:~ lixin$ cd ~/GitRepository/tigase/
# 2. 下载源码
lixin-macbook:tigase lixin$ git clone git@github.com:help-lixin/tigase-server.git

# 3. 查看jdk版本(我的tigbase是8,需要jdk12)
lixin-macbook:tigase-server lixin$ java -version
java version "12.0.2" 2019-07-16
Java(TM) SE Runtime Environment (build 12.0.2+10)
Java HotSpot(TM) 64-Bit Server VM (build 12.0.2+10, mixed mode, sharing)

# 4. 编译并生产dist
lixin-macbook:tigase-server lixin$ mvn clean install -Pdist -DskipTests -X

# 5. 查看dist
lixin-macbook:tigase-server lixin$ ll target/_dist/
-rw-r--r--   1 lixin  staff  8725297  7 17 00:00 tigase-server-8.2.0-SNAPSHOT-b5837-javadoc.jar
-rw-r--r--   1 lixin  staff  5671364  7 17 00:00 tigase-server-resources-8.2.0-SNAPSHOT-b5837-resources.tar.gz
-rw-r--r--   1 lixin  staff  5931282  7 17 00:00 tigase-server-resources-8.2.0-SNAPSHOT-b5837-resources.zip
```

### (2). 部署
```
# 1. 创建部署目录
lixin-macbook:~ lixin$ mkdir -p ~/Developer/tigase

# 2. 拷贝编译后的dist到部署目录下
lixin-macbook:~ lixin$ cp ~/GitRepository/tigase/tigase-server/target/_dist/tigase-server-resources-8.2.0-SNAPSHOT-b5837-resources.tar.gz  ~/Developer/tigase/

# 3. 切换目录
lixin-macbook:~ lixin$ cd ~/Developer/tigase

# 4. 解压
lixin-macbook:tigase lixin$ tar -zxvf tigase-server-resources-8.2.0-SNAPSHOT-b5837-resources.tar.gz

# 5. 查看目录结构
lixin-macbook:tigase lixin$ ll
-rw-r--r--   1 lixin  staff    34519  7  5 15:58 COPYING
-rw-r--r--   1 lixin  staff    34203  7  5 15:58 License.html
drwxr-xr-x   4 lixin  staff      128  7 17 00:19 certs/
drwxr-xr-x  67 lixin  staff     2144  7 17 00:19 database/
drwxr-xr-x   5 lixin  staff      160  7 17 00:19 documentation/
drwxr-xr-x  11 lixin  staff      352  7 17 00:19 etc/
-rw-r--r--   1 lixin  staff     3592  7  5 15:58 package.html
drwxr-xr-x  22 lixin  staff      704  7 17 00:19 scripts/
drwxr-xr-x   9 lixin  staff      288  7 17 00:19 win-stuff/
```
### (3). 配置tigase.conf
```
# 只需要指定:JAVA_HOME
lixin-macbook:tigase lixin$ vi etc/tigase.conf
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-12.0.2.jdk/Contents/Home/
```
### (4). 

### (5).

### (6).

### (7). 

### (8). 

### (9). 

### (10). 
