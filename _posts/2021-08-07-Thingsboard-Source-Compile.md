---
layout: post
title: 'ThingsBoard源码编译(一)' 
date: 2021-08-07
author: 李新
tags:  ThingsBoard
---

### (1). ThingsBoard源码编译官方指导
```
https://thingsboard.io/docs/user-guide/install/building-from-source/
```
### (2). ThingsBoard编译
```
# 注意,要求安装:jdk11

# 进入工作目录
lixin-macbook:~ lixin$ cd ~/GitRepository/

# clone源码
lixin-macbook:GitRepository lixin$ git clone git@github.com:help-lixin/thingsboard.git

# 切换分支
lixin-macbook:thingsboard lixin$ git checkout release-3.2

# 编译(过程比较漫长)
lixin-macbook:thingsboard lixin$ mvn clean install -DskipTests

# 应用程序生成后的目录
lixin-macbook:thingsboard lixin$ ll application/target/
drwxr-xr-x   2 lixin  staff         64  8  7 13:05 archive-tmp/
drwxr-xr-x   3 lixin  staff         96  8  7 13:05 bin/
drwxr-xr-x   8 lixin  staff        256  8  7 13:05 classes/
drwxr-xr-x   8 lixin  staff        256  8  7 13:05 conf/
drwxr-xr-x   5 lixin  staff        160  8  7 13:05 control/
drwxr-xr-x   7 lixin  staff        224  8  7 13:05 data/
drwxr-xr-x   8 lixin  staff        256  8  7 13:05 debian/
drwxr-xr-x   3 lixin  staff         96  8  7 13:05 generated-sources/
drwxr-xr-x   3 lixin  staff         96  8  7 13:05 generated-test-sources/
drwxr-xr-x   3 lixin  staff         96  8  7 13:05 maven-archiver/
drwxr-xr-x   3 lixin  staff         96  8  7 13:05 maven-status/
-rw-r--r--   1 lixin  staff   11857728  8  7 13:05 protoc-3.11.4-osx-x86_64.exe
-rw-r--r--   1 lixin  staff  136096896  8  7 13:05 thingsboard-3.2.2-1.noarch.rpm
-rwxr--r--   1 lixin  staff  148502358  8  7 13:05 thingsboard-3.2.2-boot.jar*
-rw-r--r--   1 lixin  staff    1074627  8  7 13:05 thingsboard-3.2.2.jar
-rw-r--r--   1 lixin  staff  136284994  8  7 13:05 thingsboard-windows.zip
-rw-r--r--   1 lixin  staff  136088534  8  7 13:05 thingsboard.deb
-rw-r--r--   1 lixin  staff  136096896  8  7 13:05 thingsboard.rpm
-rw-r--r--   1 lixin  staff        570  8  7 13:05 thingsboard_3.2.2-1_all.changes
-rw-r--r--   1 lixin  staff  136088534  8  7 13:05 thingsboard_3.2.2-1_all.deb

```
### (3). 总结
ThingsBoard还是做得挺不错的,基本上是一把编译通过.只是时间长一点而已,后面会深入使用,并研究源码.  