---
layout: post
title: 'Redis Bitmaps操作'
date: 2021-07-05
author: 李新
tags:  Redis 
---

### (1). 前言


### (2). SETBIT设置
```
# unique:users:2021-07-21 :  key
# 0/5/10/15               : 下标,offset(user_id)
# 1                       : value(只能是0/1)
# 定义了两个key: unique:users:2021-07-20/unique:users:2021-07-21

127.0.0.1:6380> SETBIT unique:users:{2021-07}-21 0 1
127.0.0.1:6380> SETBIT unique:users:{2021-07}-21 5 1
127.0.0.1:6380> SETBIT unique:users:{2021-07}-21 10 1
127.0.0.1:6380> SETBIT unique:users:{2021-07}-21 15 1

127.0.0.1:6380> SETBIT unique:users:{2021-07}-03 0 1
127.0.0.1:6380> SETBIT unique:users:{2021-07}-03 5 1
127.0.0.1:6380> SETBIT unique:users:{2021-07}-03 10 1
```
### (3). BITCOUNT统计
```
127.0.0.1:6380> BITCOUNT unique:users:{2021-07}-21
(integer) 4

127.0.0.1:6380> BITCOUNT unique:users:{2021-07}-03
(integer) 3
```
### (4). BITOP统计
```
# "tmp_:{2021-07}_result"                                   : AND之后保存结果
# "unique:users:{2021-07}-03" / "unique:users:{2021-07}-21" : key
# unique:users:{2021-07}-21  1 0 0 0 1   0 0 0 0 1  0 0 0 0 1  
# unique:users:{2021-07}-03  1 0 0 0 1   0 0 0 0 1  0 0 0 0 0  
127.0.0.1:6381> BITOP AND "tmp_:{2021-07}_result"  "unique:users:{2021-07}-03" "unique:users:{2021-07}-21"
(integer) 2
127.0.0.1:6381> BITCOUNT "tmp_:{2021-07}_result"
(integer) 3

127.0.0.1:6381> BITOP OR "tmp_:{2021-07}_result"  "unique:users:{2021-07}-03" "unique:users:{2021-07}-21"
-> Redirected to slot [868] located at 127.0.0.1:6381
(integer) 2
127.0.0.1:6381> BITCOUNT "tmp_:{2021-07}_result"
(integer) 3
```
