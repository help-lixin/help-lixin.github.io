---
layout: post
title: 'Apache BookKeeper之本地运行Bookies测试(二)' 
date: 2021-10-05
author: 李新
tags:  BookKeeper
---

### (1). 概述
前面对BookKeeper的概念有了一个了解,在这里,按照官网的提示,运行一批:Bookies进行读写.                
["参考官网 Run bookies locally"](https://bookkeeper.apache.org/docs/latest/getting-started/run-locally/)    

### (2). 测试本地运行6个Bookies
```
# 1. 进入源码目录
lixin-macbook:~ lixin$ cd /Users/lixin/GitRepository/bookkeeper
# 2. 在本地运行6个Bookies(会在控制台一直运行)
lixin-macbook:bookkeeper lixin$ ./bin/bookkeeper localbookie 6
```
### (3). 查看Bookies产生的数据目录
```
lixin-macbook:bk-data lixin$ tree  /tmp/bk-data/
/tmp/bk-data/
├── bookie0
│   └── current
│       ├── VERSION
│       └── lastMark
├── bookie1
│   └── current
│       ├── VERSION
│       └── lastMark
├── bookie2
│   └── current
│       ├── VERSION
│       └── lastMark
├── bookie3
│   └── current
│       ├── VERSION
│       └── lastMark
├── bookie4
│   └── current
│       ├── VERSION
│       └── lastMark
└── bookie5
    └── current
        ├── VERSION
        └── lastMark
```
### (4). 总结
为什么要运行这个案例,原因在于:后面要对这个脚本运行的Java源码进行剖析.      