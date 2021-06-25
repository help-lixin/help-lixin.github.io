---
layout: post
title: 'LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443'
date: 2021-06-25
author: 李新
tags:  git
---

### (1). 前言
> 这几天,突然,github提交代码时,总是报错:LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443.  
> 解决方法有两种:  
> 1. git配置代理(翻墙).  
> 2. 通过SSH进行提交.  
### (2). git配置代理
```
lixin-macbook:~ lixin$ vi ~/.gitconfig
[user]
	// ... ...
[http]
	// 注意:这里的52822端口,是你的翻墙软件,开启的端口,Mac在"网络偏好设置"--"代理"-"网页代理/安全全网页代理"查看.
	proxy = http://127.0.0.1:52822
[https]
	proxy = https://127.0.0.1:52822
```
### (3). 通过SSH提交
```
# 1. 在你本机创建一对公私钥(略).
lixin-macbook:~ lixin$ ll ~/.ssh/
-rw-------    1 lixin  staff  1675  4 15  2020 id_rsa
# id_rsa.pub是公钥
-rw-------    1 lixin  staff   401  4 15  2020 id_rsa.pub


# 2. 拷贝公钥,到github里(略).


# 3. 配置你clone下来的项目
# help-lixin.github.io就是我通过htts clone到本地的项目
lixin-macbook:help-lixin.github.io lixin$ pwd
/Users/lixin/GitRepository/help-lixin.github.io


lixin-macbook:help-lixin.github.io lixin$ vi ./.git/config
# **************************************************************************************
# 修改前的内容,仅仅是改了url部份
# **************************************************************************************

[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = true
[remote "origin"]
        url = https://github.com/help-lixin/help-lixin.github.io.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main

# **************************************************************************************
# 修改后的内容.
# **************************************************************************************

[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = true
[remote "origin"]
        url = git@github.com:help-lixin/help-lixin.github.io.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
```
### (4). 测试
```
lixin-macbook:ansible_roles lixin$ git add . && git commit -m "init ansible" && git push
[main 6fc6267] init ansible
 1 file changed, 1 insertion(+), 1 deletion(-)
枚举对象中: 5, 完成.
对象计数中: 100% (5/5), 完成.
使用 4 个线程进行压缩
压缩对象中: 100% (3/3), 完成.
写入对象中: 100% (3/3), 284 字节 | 284.00 KiB/s, 完成.
总共 3（差异 2），复用 0（差异 0），包复用 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To github.com:help-lixin/ansible_roles.git
   1ca51b1..6fc6267  main -> main
```