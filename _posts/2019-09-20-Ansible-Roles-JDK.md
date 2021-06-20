---
layout: post
title: 'Ansible Roles定义JDK(六)'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). 创建Roles目录
```
lixin-macbook:~ root# cd /etc/ansible/roles/
lixin-macbook:ansible-roles root$ mkdir -p jdk/{files,handlers,tasks,templates,vars}
lixin-macbook:ansible-roles root$ sudo ln -s ./jdk /etc/ansible/roles/
lixin-macbook:ansible-roles root$ ll /etc/ansible/roles/
lrwxr-xr-x  1 root  wheel    5  6 20 11:49 jdk
```
### (2). 配置task
```
# 1. 我已jdk1.8压缩包,直接放到files目录下
lixin-macbook:ansible-roles lixin$ cp ~/Downloads/jdk-8u271-linux-x64.tar.gz  ./jdk/files/


# 2. 定义模板(/etc/profile.d/jdk.sh)
#    注意:EOF时,变量居然会自动解析,所以,在变量前面加上了一个反斜杆.
lixin-macbook:ansible-roles lixin$ cat > ./jdk/templates/jdk.sh.j2 <<EOF
#!/bin/bash
export JAVA_HOME={{remote.install.destination}} 
export PATH=\$PATH:\$JAVA_HOME/bin:
export CLASSPATH=.:\$JAVA_HOME/jre/lib/rt.jar:\$JAVA_HOME/lib/tools.jar:\$JAVA_HOME/lib/dt.jar:
EOF

# 3. 定义变量(定义变量时,要注意:不能这样写:java-install-pkg,否则,在运行时会报错)
#     还有,不需要把:jdk-8u271-linux-x64.tar.gz路径写全,因为:ansible会默认就到:files下去找文件
lixin-macbook:ansible-roles lixin$ cat > ./jdk/vars/main.yml <<EOF
java:
  install:
    pkg: jdk-8u271-linux-x64.tar.gz

remote:
  install:
    dir: /usr/local
    path: /jdk1.8.0_271
    destination: /usr/local/jdk
EOF


# 4. 定义task(打印变量信息)
lixin-macbook:ansible-roles lixin$ cat > ./jdk/tasks/ready-install.yml <<EOF
- name: print vars
  ansible.builtin.debug:
    msg: " unzip {{ java.install.pkg }} , install to: {{ remote.install.destination}} "
EOF

# 4. 定义task(解压,并创建软链接)
lixin-macbook:ansible-roles lixin$ cat > ./jdk/tasks/install.yml <<EOF
- name: "unzip {{ java.install.pkg }} -d {{ remote.install.dir }}"
  unarchive: copy=yes src={{ java.install.pkg }} dest={{ remote.install.dir }}
- name: "ls -s {{ remote.install.dir }}{{ remote.install.path }}  {{remote.install.destination}}"
  file: src={{ remote.install.dir }}{{ remote.install.path }} state=link  dest={{remote.install.destination}} owner=root group=root force=yes
EOF

# 4. 定义task(配置环境变量)
lixin-macbook:ansible-roles lixin$ cat > ./jdk/tasks/config-env-var.yml <<EOF
- name: add JAVA_HOME to System Environment Variable
  template: src=jdk.sh.j2 dest=/etc/profile.d/jdk.sh
EOF

# 4. 定义task:main
lixin-macbook:ansible-roles lixin$ cat > ./jdk/tasks/main.yml <<EOF
- include: ready-install.yml
- include: install.yml
- include: config-env-var.yml
EOF
```
### (3). 配置role
```
# 7. 编写主的playbook
lixin-macbook:ansible-roles lixin$  cat > ./jdk/jdk_roles.yml <<EOF
 - hosts: erp
   remote_user: root
   gather_facts: no
   roles:
     - role: jdk
EOF
```
### (4). 目录结构如下
```
lixin-macbook:roles lixin$ tree
.
└── jdk
    ├── files
    │   └── jdk-8u271-linux-x64.tar.gz
    ├── handlers
    ├── jdk_roles.yml
    ├── tasks
    │   ├── config-env-var.yml
    │   ├── install.yml
    │   ├── main.yml
    │   └── ready-install.yml
    ├── templates
    │   └── jdk.sh.j2
    └── vars
        └── main.yml
```
### (5). 测试验证
```
lixin-macbook:jdk lixin$ ansible-playbook jdk_roles.yml
PLAY [erp] *******************************************************************************************************************

TASK [jdk : print vars] ******************************************************************************************************
ok: [10.211.55.100] => {
    "msg": " unzip jdk-8u271-linux-x64.tar.gz , install to: /usr/local/jdk "
}

TASK [jdk : unzip jdk-8u271-linux-x64.tar.gz -d /usr/local] ******************************************************************
ok: [10.211.55.100]

TASK [jdk : ls -s /usr/local/jdk1.8.0_271  /usr/local/jdk] *******************************************************************
ok: [10.211.55.100]

TASK [jdk : add JAVA_HOME to System Environment Variable] ********************************************************************
changed: [10.211.55.100]

PLAY RECAP *******************************************************************************************************************
10.211.55.100              : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```