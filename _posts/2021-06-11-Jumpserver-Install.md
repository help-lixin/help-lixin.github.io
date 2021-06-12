---
layout: post
title: 'JumpServer 安装与使用'
date: 2021-06-11
author: 李新
tags:  Jumpserver
---

### (1). 前言
> 有这样的场景: 
> 1. 企业拥有100多台机器,账号应该如何管理?    
> 2. 企业IT员工有100多人,人员离职如何进行账户回收?    

### (2). Jumpserver介绍
> JumpServer是全球首款开源的堡垒机,使用GNU GPL v2.0开源协议,是符合4A规范的运维安全审计系统.   
> JumpServer使用 Python/Django为主进行开发,遵循Web 2.0规范,配备了业界领先的Web Terminal方案,交互界面美观、用户体验好.   
> JumpServer采纳分布式架构,支持多机房跨区域部署,支持横向扩展,无资产数量及并发限制.

### (3). Jumpserver机器准备

|  机器名称   |    IP         |    角色            |
|  ----      | ----          | ----              | 
|  app-1     | 10.211.55.100 | jumpserver        |
|  app-2     | 10.211.55.101 | web server(资产)   |

### (4). 关闭防火墙以及selinux
```
[root@app-1 ~]# sed -i 's/enforcing/disabled/g' /etc/sysconfig/selinux
[root@app-1 ~]# systemctl disable firewalld && reboot
```
### (5). 修改字符集
```
[root@app-1 ~]# localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8
[root@app-1 ~]# export LC_ALL=zh_CN.UTF-8
[root@app-1 ~]# echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf
```
### (6). 准备python3和python虚拟环境
```
[root@app-1 ~]# yum -y install wget sqlite-devel xz gcc automake zlib-devel openssl-devel epel-release git
# 下载python3.6.1(因为centos7是python 2.7.5)
[root@app-1 ~]# wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tar.xz
# 解压
[root@app-1 ~]# mv Python-3.6.1.tar.xz /usr/src && cd /usr/src/ && tar xvf Python-3.6.1.tar.xz && cd Python-3.6.1
[root@app-1 Python-3.6.1]# ./configure && make && make install
```
### (7). 建立Python 虚拟环境
```
[root@app-1 Python-3.6.1]# cd /opt/
[root@app-1 opt]# python3.6 -m venv py3

[root@app-1 opt]# source /opt/py3/bin/activate
# 能看到这个信息代表成功了,以后所有的命令,都是基于该基础运行.
(py3) [root@app-1 opt]#

# deactivate 退出
(py3) [root@app-1 opt]# deactivate
```
### (8). 自动载入虚拟环境
> 每次运行命令都要先运行(source /opt/py3/bin/activate),可通过以下操作,自动执行.
```
(py3) [root@app-1 opt]# git clone https://github.com/kennethreitz/autoenv.git
(py3) [root@app-1 opt]# echo "source /opt/autoenv/activate.sh" >> ~/.bashrc
(py3) [root@app-1 opt]# source ~/.bashrc
```
### (9). 安装jumpserver
```
(py3) [root@app-1 opt]# wget https://github.com/jumpserver/jumpserver/releases/download/v2.10.4/jumpserver-v2.10.4.tar.gz
(py3) [root@app-1 opt]# tar -zxvf jumpserver-v2.10.4.tar.gz
(py3) [root@app-1 opt]# cd jumpserver-v2.10.4/
(py3) [root@app-1 jumpserver-v2.10.4]# echo "source /opt/py3/bin/activate" > /opt/jumpserver-v2.10.4/.env
```
### (10). 安装依赖
```
# 首次进入,会有提示,输入Y即可
[root@app-1 jumpserver-v2.10.4]# cd requirements/
autoenv:
autoenv: WARNING:
autoenv: This is the first time you are about to source /opt/jumpserver-v2.10.4/.env:
autoenv:
autoenv:   --- (begin contents) ---------------------------------------
autoenv:     source /opt/py3/bin/activate$
autoenv:
autoenv:   --- (end contents) -----------------------------------------
autoenv:
autoenv: Are you sure you want to allow this? (y/N) y

# 安装依赖
(py3) [root@app-1 requirements]# yum -y install $(cat rpm_requirements.txt)
```
### (11). 安装Python库依赖
```
(py3) [root@app-1 requirements]# pip install --upgrade pip setuptools
(py3) [root@app-1 requirements]# pip install -r requirements.txt
```
### (12). 安装redis,Jumpserver使用Redis做cache和celery broke(python分布式调度模块)
```
(py3) [root@app-1 ~]# yum -y install redis
(py3) [root@app-1 ~]# systemctl enable  redis
(py3) [root@app-1 ~]# systemctl start redis
```
### (13). 安装MySQL
```
# 登录mysql
# 创建库和角色
mysql> CREATE DATABASE jumpserver DEFAULT CHARSET 'UTF8';
mysql> GRANT ALL ON jumpserver.* TO jumpserver@'127.0.0.1' identified by '1q2w3e4r5t@2';
mysql> FLUSH PRIVILEGES;
```
### (14). 修改Jumpserver配置文件
```
(py3) [root@app-1 opt]# cd /opt/jumpserver-v2.10.4/
(py3) [root@app-1 jumpserver-v2.10.4]# cp config_example.yml config.yml
# 生成随机SECRET_KEY
(py3) [root@app-1 jumpserver-v2.10.4]# SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`
# 生成随机BOOTSTRAP_TOKEN
(py3) [root@app-1 jumpserver-v2.10.4]# BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`
# 配置DB_PASSWORD
(py3) [root@app-1 jumpserver-v2.10.4]# DB_PASSWORD=1q2w3e4r5t@2

(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver-v2.10.4/config.yml
(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver-v2.10.4/config.yml
(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/# DEBUG: true/DEBUG: false/g" /opt/jumpserver-v2.10.4/config.yml
(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: ERROR/g" /opt/jumpserver-v2.10.4/config.yml
(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: true/g" /opt/jumpserver-v2.10.4/config.yml
(py3) [root@app-1 jumpserver-v2.10.4]# sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver-v2.10.4/config.yml
```
### (15). config.yml模板
```
# SECURITY WARNING: keep the secret key used in production secret!
# 加密秘钥 生产环境中请修改为随机字符串，请勿外泄, 可使用命令生成
# $ cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 49;echo
SECRET_KEY: cIoGmAWuKgu2aswEaCzaeF8TiGX4FVCS9SOsSWwRqrjDFizq8u

# SECURITY WARNING: keep the bootstrap token used in production secret!
# 预共享Token coco和guacamole用来注册服务账号，不在使用原来的注册接受机制
BOOTSTRAP_TOKEN: 0z6gUWF9c0Q9MUDp

# Development env open this, when error occur display the full process track, Production disable it
# DEBUG 模式 开启DEBUG后遇到错误时可以看到更多日志
DEBUG: false

# DEBUG, INFO, WARNING, ERROR, CRITICAL can set. See https://docs.djangoproject.com/en/1.10/topics/logging/
# 日志级别
LOG_LEVEL: ERROR
# LOG_DIR:

# Session expiration setting, Default 24 hour, Also set expired on on browser close
# 浏览器Session过期时间，默认24小时, 也可以设置浏览器关闭则过期
# SESSION_COOKIE_AGE: 86400
SESSION_EXPIRE_AT_BROWSER_CLOSE: true

# Database setting, Support sqlite3, mysql, postgres ....
# 数据库设置
# See https://docs.djangoproject.com/en/1.10/ref/settings/#databases

# SQLite setting:
# 使用单文件sqlite数据库
# DB_ENGINE: sqlite3
# DB_NAME:
# MySQL or postgres setting like:
# 使用Mysql作为数据库
DB_ENGINE: mysql
DB_HOST: 127.0.0.1
DB_PORT: 3306
DB_USER: jumpserver
DB_PASSWORD: 1q2w3e4r5t@2
DB_NAME: jumpserver

# When Django start it will bind this host and port
# ./manage.py runserver 127.0.0.1:8080
# 运行时绑定端口
HTTP_BIND_HOST: 0.0.0.0
HTTP_LISTEN_PORT: 8080
WS_LISTEN_PORT: 8070

# Use Redis as broker for celery and web socket
# Redis配置
REDIS_HOST: 127.0.0.1
REDIS_PORT: 6379
# REDIS_PASSWORD:
# REDIS_DB_CELERY: 3
# REDIS_DB_CACHE: 4

# Use OpenID Authorization
# 使用 OpenID 进行认证设置
# AUTH_OPENID: False # True or False
# BASE_SITE_URL: None
# AUTH_OPENID_CLIENT_ID: client-id
# AUTH_OPENID_CLIENT_SECRET: client-secret
# AUTH_OPENID_PROVIDER_ENDPOINT: https://op-example.com/
# AUTH_OPENID_PROVIDER_AUTHORIZATION_ENDPOINT: https://op-example.com/authorize
# AUTH_OPENID_PROVIDER_TOKEN_ENDPOINT: https://op-example.com/token
# AUTH_OPENID_PROVIDER_JWKS_ENDPOINT: https://op-example.com/jwks
# AUTH_OPENID_PROVIDER_USERINFO_ENDPOINT: https://op-example.com/userinfo
# AUTH_OPENID_PROVIDER_END_SESSION_ENDPOINT: https://op-example.com/logout
# AUTH_OPENID_PROVIDER_SIGNATURE_ALG: HS256
# AUTH_OPENID_PROVIDER_SIGNATURE_KEY: None
# AUTH_OPENID_SCOPES: "openid profile email"
# AUTH_OPENID_ID_TOKEN_MAX_AGE: 60
# AUTH_OPENID_ID_TOKEN_INCLUDE_CLAIMS: True
# AUTH_OPENID_USE_STATE: True
# AUTH_OPENID_USE_NONCE: True
# AUTH_OPENID_SHARE_SESSION: True
# AUTH_OPENID_IGNORE_SSL_VERIFICATION: True
# AUTH_OPENID_ALWAYS_UPDATE_USER: True

# Use Radius authorization
# 使用Radius来认证
# AUTH_RADIUS: false
# RADIUS_SERVER: localhost
# RADIUS_PORT: 1812
# RADIUS_SECRET:

# CAS 配置
# AUTH_CAS': False,
# CAS_SERVER_URL': "http://host/cas/",
# CAS_ROOT_PROXIED_AS': 'http://jumpserver-host:port',
# CAS_LOGOUT_COMPLETELY': True,
# CAS_VERSION': 3,

# LDAP/AD settings
# LDAP 搜索分页数量
# AUTH_LDAP_SEARCH_PAGED_SIZE: 1000
#
# 定时同步用户
# 启用 / 禁用
# AUTH_LDAP_SYNC_IS_PERIODIC: True
# 同步间隔 (单位: 时) (优先）
# AUTH_LDAP_SYNC_INTERVAL: 12
# Crontab 表达式
# AUTH_LDAP_SYNC_CRONTAB: * 6 * * *
#
# LDAP 用户登录时仅允许在用户列表中的用户执行 LDAP Server 认证
# AUTH_LDAP_USER_LOGIN_ONLY_IN_USERS: False
#
# LDAP 认证时如果日志中出现以下信息将参数设置为 0 (详情参见：https://www.python-ldap.org/en/latest/faq.html)
# In order to perform this operation a successful bind must be completed on the connection
# AUTH_LDAP_OPTIONS_OPT_REFERRALS: -1

# OTP settings
# OTP/MFA 配置
# OTP_VALID_WINDOW: 0
# OTP_ISSUER_NAME: Jumpserver

# Perm show single asset to ungrouped node
# 是否把未授权节点资产放入到 未分组 节点中
# PERM_SINGLE_ASSET_TO_UNGROUP_NODE: False
#
# 同一账号仅允许在一台设备登录
# USER_LOGIN_SINGLE_MACHINE_ENABLED: False
#
# 启用定时任务
# PERIOD_TASK_ENABLED: True
#
# 启用二次复合认证配置
# LOGIN_CONFIRM_ENABLE: False
#
# Windows 登录跳过手动输入密码
# WINDOWS_SKIP_ALL_MANUAL_PASSWORD: False
```
### (16). 生成数据库表结构和初始化数据文件
```
(py3) [root@app-1 jumpserver-v2.10.4]# cd /opt/jumpserver-v2.10.4/utils/
(py3) [root@app-1 utils]# bash make_migrations.sh
Migrations for 'users':
  /opt/jumpserver-v2.10.4/apps/users/migrations/0035_auto_20210612_1149.py
    - Alter field need_update_password on user
Operations to perform:
  Apply all migrations: acls, admin, applications, assets, audits, auth, authentication, captcha, common, contenttypes, django_cas_ng, django_celery_beat, jms_oidc_rp, ops, orgs, perms, sessions, settings, terminal, tickets, users
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0001_initial... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  ... ...
```
### (17). 运行Jumpserver 
```
(py3) [root@app-1 utils]# cd /opt/jumpserver-v2.10.4/

# ./jms start|stop|status|restart all -d
# -d : 后台运行
(py3) [root@app-1 jumpserver-v2.10.4]# ./jms start all -d
```
### (18). 下载koko
```
[root@app-1 ~]# source /opt/py3/bin/activate
(py3) [root@app-1 ~]# cd /opt/
(py3) [root@app-1 opt]# wget https://github.com/jumpserver/koko/releases/download/v2.10.4/koko-v2.10.4-linux-amd64.tar.gz
(py3) [root@app-1 opt]# tar xf koko-v2.10.4-linux-amd64.tar.gz
(py3) [root@app-1 opt]# mv koko-v2.10.4-linux-amd64 koko
```
### (19). 配置koko并启动
```
(py3) [root@app-1 opt]# cd koko/
(py3) [root@app-1 koko]# cp config_example.yml  config.yml

# 这里的:0z6gUWF9c0Q9MUDp,就是上面jumpServer安装时配置的:BOOTSTRAP_TOKEN
(py3) [root@app-1 koko]# sed -i "s/<PleasgeChangeSameWithJumpserver>/0z6gUWF9c0Q9MUDp/g" /opt/koko/config.yml
(py3) [root@app-1 koko]# sed -i "s/# REDIS_HOST/REDIS_HOST/" /opt/koko/config.yml
(py3) [root@app-1 koko]# sed -i "s/# REDIS_PORT/REDIS_PORT/" /opt/koko/config.yml

# 启动koko
(py3) [root@app-1 koko]# ./koko -d
```
### (20). 下载Lina和Luna
```
# lina是一个前端UI工具
[root@app-1 ~]# source /opt/py3/bin/activate
(py3) [root@app-1 ~]# cd /opt/
(py3) [root@app-1 opt]# wget https://github.com/jumpserver/lina/releases/download/v2.10.4/lina-v2.10.4.tar.gz
(py3) [root@app-1 opt]# tar xf lina-v2.10.4.tar.gz
(py3) [root@app-1 opt]# mv lina-v2.10.4 lina

# lua是web terminal
(py3) [root@app-1 opt]# wget https://github.com/jumpserver/luna/releases/download/v2.10.4/luna-v2.10.4.tar.gz
(py3) [root@app-1 opt]# tar xf luna-v2.10.4.tar.gz
(py3) [root@app-1 opt]# mv luna-v2.10.4 luna
```
### (21). 安装Nginx
```
(py3) [root@app-1 opt]# yum -y install nginx
(py3) [root@app-1 opt]# vi /etc/nginx/conf.d/jumpserver.conf
```
### (22). 配置nginx(jumpserver.conf)
```
server {
    listen 80;

    client_max_body_size 100m;

    location /ui/ {
        try_files $uri / /index.html;
        alias /opt/lina/;
    }

    location /luna/ {
        try_files $uri / /index.html;
        alias /opt/luna/;
    }

    location /media/ {
        add_header Content-Encoding gzip;
        root /opt/jumpserver-v2.10.4/data/;
    }

    location /static/ {
        root /opt/jumpserver-v2.10.4/data/;
    }

    location /koko/ {
        proxy_pass       http://localhost:5000;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /guacamole/ {
        proxy_pass       http://localhost:8081/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /ws/ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8070;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /core/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        rewrite ^/(.*)$ /ui/$1 last;
    }
}
```
### (23). 启动nginx
```
# 测试配置是否正确
(py3) [root@app-1 opt]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# 启动nginx
(py3) [root@app-1 opt]# nginx

# 检查启动是否成功
(py3) [root@app-1 opt]# ps -ef|grep nginx
root      7279     1  0 13:28 ?        00:00:00 nginx: master process nginx
nginx     7280  7279  0 13:28 ?        00:00:00 nginx: worker process
nginx     7281  7279  0 13:28 ?        00:00:00 nginx: worker process
```
### (24). 访问nginx(http://10.211.55.100/)
> jumpserver默认的账号和密码是:admin/admin,首次进入会要求改密码,只有改密码之后,才能通过终端:koko登录.

!["jumpserver"](/assets/jumpserver/img/jumpserver-home.png)

### (25). 测试koko
```
lixin-macbook:~ lixin$ ssh -p 2222 admin@10.211.55.100
admin@10.211.55.100's password:
		Administrator,  欢迎使用JumpServer开源堡垒机系统

	1) 输入 部分IP，主机名，备注 进行搜索登录(如果唯一).
	2) 输入 / + IP，主机名，备注 进行搜索，如：/192.168.
	3) 输入 p 进行显示您有权限的主机.
	4) 输入 g 进行显示您有权限的节点.
	5) 输入 d 进行显示您有权限的数据库.
	6) 输入 k 进行显示您有权限的Kubernetes.
	7) 输入 r 进行刷新最新的机器和节点信息.
	8) 输入 h 进行显示帮助.
	9) 输入 q 进行退出.
```
### (26). JumpServer用户管理
> 用户管理:在这里添加的账号可以在JumpServer UI界面以及koko登录. 

!["jumpserver-account"](/assets/jumpserver/img/jumpserver-account.png)
!["jumpserver-account-group-add"](/assets/jumpserver/img/jumpserve-account-group-add.jpg)
!["jumpserver-account-add"](/assets/jumpserver/img/jumpserve-account-add.png)

### (27). JumpServer资产管理
> 在添加"系统用户"时,记得打开"自动推送".  

!["添加被管控的机器root账号信息"](/assets/jumpserver/img/jumpserver-admin-users.jpg)
!["资产管理"](/assets/jumpserver/img/jumpserver-assets-manager.png)
!["系统用户添加"](/assets/jumpserver/img/jumpserver-system-users.png)

### (28). JumpServer权限管理
!["给资产与用户进行绑定"](/assets/jumpserver/img/jumpserver-user-role-bind.png)

### (29). 测试zhangsan
```
lixin-macbook:~ lixin$ ssh -p 2222 zhangsan@10.211.55.100
zhangsan@10.211.55.100's password:
		张三,  欢迎使用JumpServer开源堡垒机系统

	1) 输入 部分IP，主机名，备注 进行搜索登录(如果唯一).
	2) 输入 / + IP，主机名，备注 进行搜索，如：/192.168.
	3) 输入 p 进行显示您有权限的主机.
	4) 输入 g 进行显示您有权限的节点.
	5) 输入 d 进行显示您有权限的数据库.
	6) 输入 k 进行显示您有权限的Kubernetes.
	7) 输入 r 进行刷新最新的机器和节点信息.
	8) 输入 h 进行显示帮助.
	9) 输入 q 进行退出.

Opt> p
  ID    | 主机名                                                 | IP                            | 备注
+-------+--------------------------------------------------------+-------------------------------+---------------------+
  1     | app-2                                                  | 10.211.55.101                 |
页码：1，每页行数：17，总页数：1，总数量：1
提示：输入资产ID直接登录，二级搜索使用 // + 字段，如：//192 上一页：b 下一页：n
搜索：	

# 通过jumpserver登录到:10.211.55.101
[Host]> 1
开始连接到 zhangsan@10.211.55.101  0.3
Last login: Sat Jun 12 15:23:16 2021 from 10.211.55.100

# 登录成功
[zhangsan@app-2 ~]$

# 额,在jumpserver创建的每个用户,实际将会是对应Linux下的用户.
[zhangsan@app-2 ~]$ cat /etc/passwd|grep zhangsan
zhangsan:x:1002:1002:tomcat[张三(zhangsan)]:/home/zhangsan:/bin/bash

```
### (30). 总结
> 说实话,JumpServer的UI界面不是很人性化,我反正是摸索了半天才理解(当然,也有可能是我自己的问题).  