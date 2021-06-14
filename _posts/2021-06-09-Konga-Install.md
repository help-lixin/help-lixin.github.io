---
layout: post
title: 'Konga 安装(三)'
date: 2021-06-09
author: 李新
tags:  Kong
---

### (1). 简介 
> kong虽然很强大,但是在管理方式上比较单一只能通过API请求来管理,而konga则是一款优秀的UI界面的管理工具(相信是运维的最爱). 

### (2). 安装依赖nodejs
```
[root@tomcat-1 ~]# yum -y install git nodejs

# 在国内实在不行的话,就买台阿里的ECS在香港,进行以下操作
# 配置阿里云的npm仓库
# [root@tomcat-1 ~]# npm config set registry https://registry.npm.taobao.org
# 获取配置信息
# [root@tomcat-1 ~]# npm config get
	; cli configs
	user-agent = "npm/3.10.10 node/v6.17.1 linux x64"
	; userconfig /root/.npmrc
	registry = "https://registry.npm.taobao.org/"
	; node bin location = /usr/bin/node
	; cwd = /root
	; HOME = /root
	; "npm config ls -l" to show all defaults.
```
### (3). 安装konga
```
# github地址:https://github.com/pantsel/konga
# 我是fork了一份到我的github上
# 1. 下载
[root@tomcat-1 ~]# wget https://github.com/pantsel/konga/archive/refs/tags/0.14.9.tar.gz

# 2. 解压
[root@tomcat-1 ~]# tar -zxvf konga-0.14.9.tar.gz

# 3. 安装依赖(出现如下信息,代表依赖安装成功)
[root@tomcat-1 ~]# cd konga-0.14.9/
[root@tomcat-1 konga-0.14.9]# npm i
npm WARN lifecycle kongadmin@0.14.9~postinstall: cannot run in wd %s %s (wd=%s) kongadmin@0.14.9 bower --allow-root install /root/konga-0.14.9

# 4. 配置.env文件
[root@tomcat-1 konga-0.14.9]# cp ./.env_example  ./.env
[root@tomcat-1 konga-0.14.9]# vi ./.env
PORT=1337
NODE_ENV=production
KONGA_HOOK_TIMEOUT=120000
DB_ADAPTER=postgres
DB_USER=kong
DB_PASSWORD=123456
DB_URI=postgresql://10.211.55.101:5432/konga
KONGA_LOG_LEVEL=warn
TOKEN_SECRET=some_secret_token
```
### (4). 启动konga
```
# 出现以下错误,是因为:pgsql版本过高,切换到:9.6就可以了
# 1. 创建表
[root@tomcat-1 konga-0.14.9]# node ./bin/konga.js  prepare --adapter postgres --uri postgresql://10.211.55.101:5432/konga  --trace-warnings
Preparing database...
(node:15741) Warning: Accessing non-existent property 'padLevels' of module exports inside circular dependency
(Use `node --trace-warnings ...` to show where the warning was created)
error: Failed to prepare database: Error: The hook `orm` is taking too long to load.
Make sure it is triggering its `initialize()` callback, or else set `sails.config.orm._hookTimeout to a higher value (currently 60000)
    at Timeout.tooLong [as _onTimeout] (/root/konga-0.14.9/node_modules/sails/lib/app/private/loadHooks.js:85:21)
    at listOnTimeout (node:internal/timers:557:17)
    at processTimers (node:internal/timers:500:7)

# 2. 启动
[root@tomcat-1 konga-0.14.9]# npm run production
```
### (5). 总结
> 通过NPM安装了一个下午的依赖,都没有把依赖全部下载下来,反正,我对UI界面不感兴趣,直接玩API先.  