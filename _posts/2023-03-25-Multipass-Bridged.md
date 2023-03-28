---
layout: post
title: 'Multipass网桥配置' 
date: 2023-03-25
author: 李新
tags:  Multipass
---

### (1). 概述
这段时间在勤俭节约,所以,安装一些软件时,会用Multipass来帮忙管理,由于Multipass默认的网络配置为NAT,所以,想改变策略为网桥模式,在此记录下.

### (2). 查看网格信息
```
lixin-macbook:~ lixin$ sudo multipass networks
Name     Type         Description
bridge0  bridge       Network bridge with en1, en2
en0      wifi         Wi-Fi
en1      thunderbolt  Thunderbolt 1
en2      thunderbolt  Thunderbolt 2
```
### (3). 配置passphrase
```
lixin-macbook:~ lixin$ multipass set local.passphrase
Please enter passphrase:
Please re-enter passphrase:
```
### (4). multipass认证
```
lixin-macbook:~ lixin$ sudo multipass authenticate
Please enter passphrase:
```
### (5). 网桥配置
```
# 此处的en0是我的mac网络名称
lixin-macbook:~ lixin$ sudo multipass set local.bridged-network=en0

lixin-macbook:~ lixin$ sudo multipass get local.bridged-network
en0
```
### (6). 指定网桥
```
multipass launch --name hello --cpus 2 --disk 40G --memory 3G --network en0 22.10
```
### (7). 

### (8). 

