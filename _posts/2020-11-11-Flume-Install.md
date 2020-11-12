---
layout: post
title: 'Flume安装'
date: 2020-11-11
author: 李新
tags: Flume
---

### (1). 下载Flume
> 省略,自行去官网下载(http://archive.apache.org/dist/flume)

### (2). Flume目录结构

```
lixin-macbook:apache-flume-1.8.0-bin lixin$ pwd
/Users/lixin/Developer/apache-flume-1.8.0-bin

lixin-macbook:apache-flume-1.8.0-bin lixin$ tree
.
├── bin
│   ├── flume-ng
│   ├── flume-ng.cmd
│   └── flume-ng.ps1
├── conf
│   ├── flume-conf.properties.template        # Flume配置文件
│   ├── flume-env.ps1.template
│   ├── flume-env.sh.template                 # Flume环境配置文件
│   └── log4j.properties
├── lib
│   ├── ...
```

### (3). 重命名环境变量和配置文件
```
lixin-macbook:apache-flume-1.8.0-bin lixin$ mv conf/flume-env.sh.template conf/flume-env.sh
```
### (4). flume-env.sh配置JAVA_HOME
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```