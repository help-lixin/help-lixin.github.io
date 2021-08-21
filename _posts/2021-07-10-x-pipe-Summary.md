---
layout: post
title: 'x-pipe 介绍(一)' 
date: 2021-07-10
author: 李新
tags:  x-pipe 
---

### (1). x-pipe 是什么
X-Pipe是由携程框架部门研发的Redis多数据中心复制管理系统.基于Redis的Master-Slave复制协议,实现低延时、高可用的Redis多数据中心、跨公网数据复制,并且提供一键机房切换,复制监控、异常报警等功能.开源版本和携程内部生产环境版本一致. 

### (2). 源码下载与编译
```
# 1. jdk
# 2. maven
# 3. 清空本地 maven 仓库的 com 和 org 目录

# 4. clone x-pipe的分支:mvn_repo
[root@erp-100 ~]# git clone -b mvn_repo git@github.com:help-lixin/x-pipe.git x-pipe-mvn-repo
[root@erp-100 ~]# cd x-pipe-mvn-repo/
[root@erp-100 x-pipe-mvn-repo]# sh install.sh

# 5. x-pip依赖cat,所以,在打包前,先本地安装cat
[root@erp-100 ~]# git clone git@github.com:help-lixin/cat.git
[root@erp-100 ~]# cd cat/


# 6. clone x-pipe
# 编译了一个下午,还是没有编译通过,想放弃的心都有了.
#  Could not resolve dependencies for project com.ctrip.framework.xpipe:xpipe-parent:pom:1.2.4: com.dianping.cat:cat-client:jar:3.0.0
[root@erp-100 ~]# git clone git@github.com:help-lixin/x-pipe.git
[root@erp-100 ~]# cd x-pipe
[root@erp-100 x-pipe]# mvn clean install -Plocal,package -DskipTests

```

### (3). 总结
