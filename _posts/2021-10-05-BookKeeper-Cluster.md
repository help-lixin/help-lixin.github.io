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

### (3). 集群搭建步骤
+ Zookeeper(略)
+ BookKeeper(3台),因为机器有限,所以,伪集群

### (4). BookKeeper集群搭建
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

# 5. 查看程序目录
lixin-macbook:~ lixin$ tree ~/Developer/bookkeeper-servers/  -L 2
/Users/lixin/Developer/bookkeeper-servers/
├── bookkeeper-1
│   ├── LICENSE
│   ├── NOTICE
│   ├── README.md
│   ├── bin
│   ├── conf
│   ├── deps
│   └── lib
├── bookkeeper-2
│   ├── LICENSE
│   ├── NOTICE
│   ├── README.md
│   ├── bin
│   ├── conf
│   ├── deps
│   └── lib
└── bookkeeper-3
    ├── LICENSE
    ├── NOTICE
    ├── README.md
    ├── bin
    ├── conf
    ├── deps
    └── lib

# 6. 


```
### (4). 

### (5). 

### (6). 

### (7).  

### (8). 
