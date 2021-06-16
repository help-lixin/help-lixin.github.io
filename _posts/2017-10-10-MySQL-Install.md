---
layout: post
title: 'CentOS 免安装MySQL'
date: 2017-10-10
author: 李新
tags: MySQL
---

### (1).  MySQL安装准备
```
# 1. 卸载系统自带的 Mariadb
[root@app-1 opt]# rpm -qa|grep mariadb
mariadb-libs-5.5.68-1.el7.x86_64
[root@app-1 opt]# rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64

# 2. 检查mysql是否存在
[root@app-1 opt]# rpm -qa | grep mysql

# 3. 创建用户和组,并修改密码
[root@app-1 opt]# groupadd mysql
[root@app-1 opt]# useradd -g mysql mysql
[root@app-1 opt]# passwd mysql
Changing password for user mysql.
New password:
BAD PASSWORD: The password is a palindrome
Retype new password:
passwd: all authentication tokens updated successfully.

# 4. 配置软连接数量
[root@app-1 opt]# echo "*         hard    nofile      524288" >> /etc/security/limits.conf
[root@app-1 opt]# echo "*         soft    nofile      524288" >> /etc/security/limits.conf
```
### (2). MySQL下载与解压
```
# ************************************************
# 注意:我已切换到mysql账号下了
# ************************************************
# 1. 进入下载目录
[mysql@app-1 ~]$ pwd
/home/mysql

# 2. 下载(MySQL Community Server)
[mysql@app-1 ~]$ wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz

# 3. 创建应用程序目录
[mysql@app-1 ~]$ mkdir application

# 4. 解压,并创建mysql数据目录
[mysql@app-1 ~]$ cd application/
[mysql@app-1 application]$ mv mysql-5.7.34-linux-glibc2.12-x86_64 mysql-5.7.34
[mysql@app-1 application]$ mkdir mysql-5.7.34/data
```

### (3). 配置my.cnf
> my.cnf查找顺序:/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf  ~/.my.cnf   

```
[mysql@app-1 ~]$ vi  ~/.my.cnf

[mysql]
default-character-set=utf8

[mysqld]
lower_case_table_names=1
basedir=/home/mysql/application/mysql-5.7.34
datadir=/home/mysql/application/mysql-5.7.34/data
port=3306
character-set-server=utf8
max_connections=2000
innodb_buffer_pool_size=128M
default-storage-engine=INNODB
log-error=/home/mysql/application/mysql-5.7.34/data/error.log
pid-file=/home/mysql/application/mysql-5.7.34/data/mysql.pid
socket=/tmp/mysql.sock
```
### (4). MySQL初始化
```
[mysql@app-1 application]$ ./mysql-5.7.34/bin/mysql_install_db --user=mysql --basedir=/home/mysql/application/mysql-5.7.34 --datadir=/home/mysql/application/mysql-5.7.34/data/
2021-06-11 16:45:06 [WARNING] mysql_install_db is deprecated. Please consider switching to mysqld --initialize
2021-06-11 16:45:11 [WARNING] The bootstrap log isn't empty:
2021-06-11 16:45:11 [WARNING] 2021-06-11T08:45:06.129387Z 0 [Warning] --bootstrap is deprecated. Please consider using --initialize instead
```
### (5). MySQL配置开机启动(/etc/systemd/system/mysql.service)
```
# ************************************************
# 注意:我已切换到root账号下了
# ************************************************
[root@app-1 ~]# vi /etc/systemd/system/mysql.service

[Unit]
Description=mysql service

[Service]
ExecStart=/home/mysql/application/mysql-5.7.34/bin/mysqld --defaults-file=/home/mysql/.my.cnf  --user=mysql
User=mysql

[Install]
WantedBy=multi-user.target
```
### (6). 启动mysql
```
# ************************************************
# 注意:我已切换到root账号下了
# ************************************************
# 启动mysql
[root@app-1 ~]# systemctl enable mysql.service
[root@app-1 ~]# systemctl daemon-reload
[root@app-1 ~]# systemctl start mysql.service
[root@app-1 ~]# systemctl status mysql.service
● mysql.service - mysql service
   Loaded: loaded (/etc/systemd/system/mysql.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2021-06-11 16:56:03 CST; 13s ago
 Main PID: 4398 (mysqld)
   CGroup: /system.slice/mysql.service
           └─4398 /home/mysql/application/mysql-5.7.34/bin/mysqld --defaults-file=/home/mysql/.my.cnf --user=mysql

Jun 11 16:56:03 app-1 systemd[1]: Started mysql service.
```

### (7). 添加mysql至path
```
# 1. 添加mysql到path下
[mysql@app-1 ~]$ vi ~/.bash_profile

MYSQL_HOME=/home/mysql/application/mysql-5.7.34
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$MYSQL_HOME/bin:
export MYSQL_HOME
export PATH

# 2. 环境变量生效
[mysql@app-1 ~]$ source ~/.bash_profile
```
### (8). 修改root密码
```
# 1. 查看root密码
[mysql@app-1 ~]$ cat ~/.mysql_secret
# Password set for user 'root@localhost' at 2017-06-10 16:45:06
ZVU50NAK5Atb

# 2. 修改root密码
[mysql@app-1 ~]$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.34
Copyright (c) 2000, 2021, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 修改密码为:111111
mysql> SET PASSWORD=PASSWORD('111111');
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```