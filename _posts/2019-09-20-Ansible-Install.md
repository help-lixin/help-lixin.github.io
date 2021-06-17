---
layout: post
title: 'Ansible 安装(二)'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). 机器准备

|  机器名称   |  ip           |  角色            |
|  ----      | ----          | ----            |
| manager    | 10.211.55.100 | 控制节点         |
| test       | 10.211.55.101 | 被控节点         |

### (2). 配置hosts
```
[lixin@manager ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.211.55.100   manager manager
10.211.55.101   test    test
```
### (3). 配置密钥
```
# 创建密钥对
[lixin@manager ~]$ ssh-keygen
# 拷贝公钥到远程机器上
[lixin@manager ~]$ for node in test
> do
> ssh-copy-id $node
> done

# 配置本机连接远程SSH时不提示是否要保存密钥
[lixin@manager ~]$ ssh-keyscan test >> ~/.ssh/known_hosts

# 测试是否能连接到被控机器
[lixin@manager ~]$ ssh test
Last login: Thu Jun 17 14:38:00 2021 from manager
[lixin@test ~]$
```
### (4). 安装Ansible
```
# 只需要在控制节点,安装Ansible,被控节点只需要开启ssh即可.  

# 切换到root账户下
[lixin@manager ~]$ su - root
# 安装ansible
[root@manager ~]# yum install -y  epel-release
[root@manager ~]# yum install -y  ansible
# 添加普通用户到root组
[root@manager ~]# gpasswd -a lixin root
```
### (5). Ansible相关文件
+ /etc/ansible/ansible.cfg
  - 主机配置文件,配置ansible工作特性
+ /etc/ansible/hosts
  - 主机清单文件(被管理机器的详细信息)
+ /etc/ansible/roles
  - 存放角色目录

### (6). ansible.cfg
```
[defaults]

# 主机清单文件
#inventory      = /etc/ansible/hosts
#library        = /usr/share/my_modules/
#module_utils   = /usr/share/my_module_utils/
# 临时存放py脚本目录
#remote_tmp     = ~/.ansible/tmp
#local_tmp      = ~/.ansible/tmp
# 插件文件
#plugin_filters_cfg = /etc/ansible/plugin_filters.yml
# 并发管理ansible机器
#forks          = 5
#poll_interval  = 15
# 通过sudo变成root
#sudo_user      = root
#ask_sudo_pass = True
#ask_pass      = True

# 检查对应目标服务器的host_key(建议打开,免去每次连接被控机器,都要通过键盘敲:yes)
host_key_checking = False
# 指定角色目录
#roles_path    = /etc/ansible/roles

# 日志存放路径(建议打开)
log_path = /var/log/ansible.log
```

### (7). /etc/ansible/hosts
>  主要是对被控机器进行分组管理

```
# 1. web服务器
[webservers]
alpha.example.org
beta.example.org
192.168.1.100:222
192.168.1.110

# 2. 数据库服务器
[dbservers]
db01.intranet.mydomain.net
db02.intranet.mydomain.net
10.25.1.56
10.25.1.57
```

### (8). Ansible常用命令
```
# 列出所有ansible的所有模块
[lixin@manager ~]$ ansible-doc --list

# 查看模块:ping文档
# -s  : 只看文档的简要信息,无须看所有内容
[lixin@manager ~]$ ansible-doc -s ping
```
### (9). 测试
```
[lixin@manager ~]$ pwd
/home/lixin

# 1. 创建目录(项目)
[lixin@manager ~]$ mkdir -p erp

# 2. 创建主机清单文件
# [erp]  : 标签名称(对主机进行分类)
# test   : 机器名称(/etc/hosts)
[lixin@manager ~]$ cat <<EOF > erp/hosts
> [erp]
> test
> EOF

# 3. 测试ping 
#  格式: ansible  <host-pattern> [-m module_name] [-a args]
#  all : 所有的主机(你可以指定主机)
# -i : 指定主机清单
# -m : 模块名称 
# -u : 指定用户名
# -k : ansible默认是通过SSH KEY登录,也可以,通过密码登录(但是,不太建议这样做)  
# -vvv  : 查看详细的日志内容
# 
# [lixin@manager ~]$ ansible erp -i ./erp/hosts -m ping -vvv
# [lixin@manager ~]$ ansible all -i erp/hosts  -u root -k -m ping -vvv
[lixin@manager ~]$ ansible all -i ./erp/hosts -m ping -vvv
test | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
### (10). Ansible执行顺序
+ 加载配置文件(/etc/ansible/ansible.cfg).  
+ 加载模块文件,如:ping.
+ 通过Ansible将模块或命令生成对应的临时文件(.py),保存在$HOME/.ansible/tmp/,并将该文件传输到远程服务器对应执行用户目录下($HOME/.ansible/tmp)  
+ 给.py添加可执行权限.
+ 执行并返回结果.
+ 删除临时.py文件

### (11). Ansible Galaxy
> 可以到[Galaxy官网](https://galaxy.ansible.com/)下载roles,学习.

```
# 比如:
[lixin@manager ~]$ ansible-galaxy collection install community.mysql
Process install dependency map
Starting collection install process
# 下载解后目录:/home/lixin/.ansible/collections/ansible_collections/community/mysql
Installing 'community.mysql:2.1.0' to '/home/lixin/.ansible/collections/ansible_collections/community/mysql'
```