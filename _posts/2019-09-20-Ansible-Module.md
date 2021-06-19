---
layout: post
title: 'Ansible 常用模块(三)'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). command模块
功能:在远程主机上执行命令,这个模块为默认模块,用ansible运行时,可以不用指定 -m            
<font color='red'>注意:command不支持管道和重定向符号('<','>','|',';',&','*')</font>             

```
# cd /home/lixin/Desktop && ls -lha
[lixin@manager ~]$ ansible test -i erp/hosts -m command -a "chdir=/home/lixin/Desktop ls -lha"
```
### (2). shell模块
功能: shell模块可以帮助我们在远程主机上执行命令,<font color='red'>支持管道和重定向符号</font>,与command模块不同的是,shell模块在远程主机中执行命令时,会经过远程主机上的/bin/sh程序处理.  

```
# 支持管道符
# ls -lha|grep .ansible
[lixin@manager ~]$ ansible all -i erp/hosts -m shell -a "ls -lha|grep .ansible"
```
### (3). script模块
功能:script模块可以帮助我们在远程主机上执行ansible主机上的脚本. 

```
# 1. 创建脚本
[root@manager ~]$ cat tomcat.sh <<EOF
 > #!/bin/bash
 > yum -y install tomcat
 > EOF

# 2. 配置可执行权限 
[root@manager ~]$ chmod 755 tomcat.sh

# 3. 在远程主机上,执行tomcat.sh脚本
[root@manager lixin]# ansible all -i erp/hosts -m script -a "/home/lixin/tomcat.sh"

# 4. 验证是否安装了tomcat
[root@test ~]# yum list tomcat
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * epel: epel.01link.hk
 * extras: mirror.lzu.edu.cn
 * updates: mirror.lzu.edu.cn
## **************************************************************************************************************
Installed Packages   代表tomcat安装成功
## **************************************************************************************************************
tomcat.noarch                                         7.0.76-16.el7_9                                         @updates
[root@test ~]#
```

### (4). copy模块
功能:copy模块的作用就是拷贝ansible主控节点上的文件到被控节点上.

```
# src/content   : 源文件
# dest          : 目标文件
# mode          : 文件拷贝到远程主机后的权限
# backup        : 当远程主机的目标路径中已经存在同名文件,并且与ansible主机中的文件内容不同时,是否对远程主机的文件进行备份
# owner         : 所属用户
# group         : 所属组
[root@manager lixin]# ansible erp -i erp/hosts -m copy -a 'src=/home/lixin/tomcat.sh dest=/root/ mode=0755 backup=yes owner=lixin group=lixin'
```
### (5). fetch模块
功能:从远程主机(被控机器)提取文件(不支持目录)到ansbible的主控节点.

```
# 可以提取远程主机的日志回到本地
# src   : 远程主机文件
# dest  : 主控节点目录(只能是目录,因为src会是多个,会在这个目录下生成主机名)
[root@manager lixin]# ansible erp -i erp/hosts -m fetch -a 'src=/root/tomcat.sh dest=/root/'
# 验证目录
[root@manager /]# tree /root/
/root/
└── test               # 远程主机的名称
    └── root           
        └── tomcat.sh
```
### (6). file模块
功能:可以帮助我们完成一些对文件操作,创建文件或目录、删除文件或目录、修改文件权限.

```
# path     : 路径
# state    : directory(目录)/touch(文件)/link(软链接)/hard(硬链接)/absent(删除)
# recurse  : 递归
# force    : 强制

[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/root/hello.txt state=touch' 
# 递归创建目录,并且指定所属组,所属用户,执行权限
[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/root/a/b state=directory recurse=yes owner=lixin group=lixin mode=0755'
# 在目录下再创建文件
[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/root/a/b/hello.txt state=touch'
# 删除文件
[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/root/a/b/hello.txt state=absent'
# 删除目录
[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/root/a state=absent  force=yes'
# 创建软链接
[root@manager lixin]# ansible erp -i erp/hosts -m file -a 'path=/opt/mysql-5.7.34-linux-glibc2.12-x86_64  state=link  src=/opt/mysql'
```
### (7). unarchive模块
功能:可以帮且我们进行解压缩.

```
# copy       :  yes(拷贝文件是从主控机器到远程机器)/no(从远程机器到拷贝文件到主控机器)
# remote_src :  和copy互斥,yes:表示文件在远程主机上,不在ansible主控节点上,no:表示文件在ansible主控节点上.
# src        : 源路径
# dest       : 解压缩到目标主机上的目标路径

# copy如果是yes,代表拷贝文件到目标主机并解压.
# copy如果是no,代表不需要拷贝文件,直接在目标主机上解压.

[root@manager lixin]# wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz
# 拷贝本机文件(/home/lixin/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz)到目标主机(test)的/opt目录下,并在目标主机的/opt下解压.
[root@manager lixin]# ansible erp -i erp/hosts -m unarchive -a 'src=/home/lixin/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz dest=/opt copy=yes'

# 拷贝文件到远程主机上
[root@manager lixin]# ansible erp -i erp/hosts -m copy -a 'src=/home/lixin/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz dest=/opt'
# 不需要拷贝文件(copy=no),直接解压远程主机目录:/opt/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz)到/opt目录下.
[root@manager lixin]# ansible erp -i erp/hosts -m unarchive -a 'src=/opt/mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz dest=/opt copy=no'
```
### (8). archive模块
功能:对ansible远程主机上的文件进行压缩.

```
#  format : (Choices: bz2, gz, tar, xz, zip)[Default: gz]
#  path   : 过程主机的目录
#  dest   : 目标路径

# 压缩远程主机上的目录(/home/lixin/a),生成压缩文件到:/home/lixin/a.zip,压缩格式为:zip
[root@manager lixin]# ansible erp -i erp/hosts -m archive  -a 'path=/home/lixin/a dest=/home/lixin/a.zip format=zip'
```
### (9). hostname模块
功能:管理主机名

```
# 更改主机清单中某个机器的名称
# 修改远程主机的名称
# echo "test2" > /etc/hostname
[root@manager lixin]# ansible erp  -i erp/hosts  -m hostname  -a 'name=test2'
```
### (10). cron模块
功能: 定时任务
支持时间: minute/hour/day/month/weekday
 
```
# disabled : 禁用定时任务
# name     : 给定时任务定义一个名称,用于查找定时任务.
# state    : absent删除定时任务

[root@manager lixin]# cat > test.sh <<EOF
echo `date "+%Y-%m-%d %H:%M:%S"` >> /tmp/log.txt
EOF

# 1. 拷贝脚本到远程主机
[root@manager lixin]# ansible erp -i erp/hosts -m copy -a 'src=/home/lixin/test.sh dest=/opt/test.sh mode=0755'

# 2. 创建定时任务(周一到周五,0点过一分执行一次)
[root@manager lixin]# ansible erp -i erp/hosts -m cron -a 'weekday=1-5 hour=0 minute=1 name="mysql-dump" job="/mysql-dump.sh &> /dev/null " '

# 3. 创建定时任务(每隔1分钟执行一次)
[root@manager lixin]# ansible erp -i erp/hosts -m cron -a 'minute=*/1 name=log job="/opt/test.sh" disabled=no'
# 去test机器查询有哪些执行任务:
[root@test opt]# crontab -l
#Ansible: log
*/1 * * * * /opt/test.sh
``` 
### (11). yum模块
功能:管理软件包

```
# name   : 包名称
# state  : installed/removed 

# 安装tomcat
[root@manager lixin]# ansible erp -i erp/hosts -m yum -a 'name=tomcat state=installed'
# 列出可安装的tomcat包(yum list tomcat)
[root@manager lixin]# ansible erp -i erp/hosts -m yum -a 'list=tomcat'
# 删除tomcat
[root@manager lixin]# ansible erp -i erp/hosts -m yum -a 'name=tomcat state=removed'
```
### (12). service模块
功能: 启动服务

```
# state  :  started(启动)/stopped(停止)/reloaded(重新加载)/restarted(重新启动)

# 启动服务,并设置为开机就启动
[root@manager lixin]# ansible erp -i erp/hosts -m service -a 'name=tomcat state=started enabled=yes'

# 停止服务
[root@manager lixin]# ansible erp -i erp/hosts -m service -a 'name=tomcat state=stopped'

# 检查下端口是否打开
[root@manager lixin]# ansible erp -i erp/hosts -m shell  -a 'ss -tlnp|grep 8080'
```
### (13). group模块
功能:对用户组进行管理

```
# 创建group
[root@manager lixin]# ansible erp -i erp/hosts -m group -a 'name=mysql'
```
### (14). user模块
功能:对用户进行管理

```
# remove : yes(是否删除目录)

# 创建用户
[root@manager lixin]# ansible erp -i erp/hosts -m user -a 'name=mysql group=mysql groups="root,mysql" home="/home/mysql" shell="/sbin/nologin" system=yes create_home=yes comment="mysql user" '

# 删除用户以及目录
[root@manager lixin]# ansible erp -i erp/hosts -m user -a 'name=mysql state=absent remove=yes'
```
### (15). lineinfile模块
功能:基于正则,替换某一行的内容

```
# 有点类似于 sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# path   : 路径
# regexp : 使用正则表达式匹配对应的行,如果有多行文本都能被匹配,则只有最后面被匹配到的那行文本才会被替换.
# line   : 使用此参数指定文本内容
# backup : True/False(操作之前是否先备份)

# 整行替换
[root@manager lixin]# ansible erp -i erp/hosts -m lineinfile -a " path='/etc/selinux/config'  regexp='^SELINUX=enforcing' line='SELINUX=disabled' "

```
### (16). replace模块
功能:基于正则,替换所有内容

```
# 有点类似于 sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
# regexp : 使用正则表达式匹配对应的行,如果有多行文本都能被匹配,则所有被匹配到的文本都会被替换(与lineinfile).

[root@manager lixin]# ansible erp -i erp/hosts -m replace -a " path='/root/fstab'  regexp='^(UUID.*)' replace='#\1' "
```
### (17). setup模块
功能:用来收集远程主机的系统信息,因为收集信息,影响速度,可以配置:gather_facts:no来禁止Ansible收集facts信息.

```
[root@manager lixin]# ansible erp -i erp/hosts -m  setup
# 主机名称
[root@manager lixin]# ansible erp -i erp/hosts -m  setup -a "filter=ansible_hostname"
# 内存信息
[root@manager lixin]# ansible erp -i erp/hosts -m  setup -a "filter=ansible_memory_mb"
# 系统版本
[root@manager lixin]# ansible erp -i erp/hosts -m setup -a "filter=ansible_distribution_major_version"
```