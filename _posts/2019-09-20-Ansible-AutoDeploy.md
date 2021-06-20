---
layout: post
title: ' GitLab + Jenkins +  Ansible 自动化部署(七)'
date: 2019-09-20
author: 李新
tags: Ansible 解决方案 DevOps
---

### (1). 前言
+ 开发如何交付给测试?
  - 开发和测试各自部署一套Jenkins,开发交付测试时,让测试用Jenkins打包.  
  - 交付的组件为:jar包/配置文件.
  - 在开发和测试的环境下,Jenkins都会自动打包,并结合Ansible进行应用程序的发版.  
+ 开发/测试如何交付给运维?
  - 运维有一套自己的Jenkins
  - 运维在git里定义了一个目录.后面详解目录结构.
  - 开发/测试,将要交付的内容,放在git目录里.  
  - 通过Jenkins定义变量(微服务名称),读取git目录里的构件,调用Ansible进行发版.

### (2). 目录定义
```
# 注意:发版时,定位的构件物的方式是: test-service/2019-09-20/18.20
# 所以,三个参数都要传递,可以,从Jenkins传递变量进来.
lixin-macbook:roles lixin$ tree deploy/
deploy/
├── deploy_roles.yml
├── files                                       ## 这个目录下的内容,通过git拉取,然后软链接进来即可.
│   └── test-service                            ## 微服务名称
│       └── 2019-09-20                          ## 发版时间
│           └── 18.20                           ## 发版详细时间
│               ├── conf                        ## conf(实际微服务之后,压根就不会在本地有配置文件了)
│               │   └── application.yml
│               └── lib                         ## lib
│                   └── hello-service.jar
├── handlers
│   ├── handlers.yml
│   └── main.yml
├── tasks
│   ├── copy.yml
│   ├── def-vars.yml
│   ├── main.yml
│   └── service.yml
├── templates
└── vars
    └── main.yml
```
### (3). 规定hosts
```
# 微服务名称就是:hosts文件名称
# /etc/hosts/hello-service
[hello-service]
10.211.55.100
10.211.55.101
10.211.55.102
10.211.55.103
```

### (4). Playbook定义

```
# 1. 创建部署目录
lixin-macbook:roles root# mkdir -p deploy/{files,handlers,tasks,templates,vars}
lixin-macbook:roles root# cd deploy
lixin-macbook:deploy root# 


# 2. 定义变量(在Ansible中,变量不能包含中划线)
# service_name,通过jenkins读取变量,传递进来.
lixin-macbook:deploy root# cat > vars/main.yml <<EOF
app:
  dir: /home/tomcat/
  name: "{{ service_name }}"
  bin:  bin
  conf: conf
  lib:  lib
  logs: logs
  shell: app.sh
EOF
  
# 3. 定义task(定义变量)
lixin-macbook:deploy root# cat > tasks/def-vars.yml <<EOF
- name: define local var local_conf
  set_fact: local_conf="{{ app.name }}/{{ second_dir }}/{{ three_dir }}/conf/*"
- name: define local var local_lib
  set_fact: local_lib="{{ app.name }}/{{ second_dir }}/{{ three_dir }}/lib/*"

- name: define remote var app_bin
  set_fact: app_bin="{{app.dir}}/{{app.name}}/{{app.bin}}"
- name: define remote var app_conf
  set_fact: app_conf="{{app.dir}}/{{app.name}}/{{app.conf}}"
- name: define remote var app_bin
  set_fact: app_lib="{{app.dir}}/{{app.name}}/{{app.lib}}"
- name: define remote var app_bin
  set_fact: app_logs="{{app.dir}}/{{app.name}}/{{app.logs}}"
- name: print vars
  ansible.builtin.debug:
    msg: "{{ app_bin  }} , {{ app_conf }} , {{ app_lib }} , {{ app_logs  }} , {{local_conf}} , {{local_lib}} "
EOF

# 3. 定义task(copy file)
#   注意:这里只有配置文件或者JAR包有变化的情况下,才会触发:restarts,切记
lixin-macbook:deploy root# cat > tasks/copy.yml <<EOF
- name: copy config
  copy: src="{{ item }}" dest="{{ app_conf }}/" backup=yes
  notify: restart app
  with_fileglob:
    - "{{ local_conf }}"
- name: copy lib
  copy: src="{{ item }}" dest="{{ app_lib }}/" backup=yes
  notify: restart app
  with_fileglob:
    - "{{ local_lib }}"
EOF

# 3. 定义task(停止/启动微服务)
lixin-macbook:deploy root# cat >  tasks/service.yml  <<EOF
- name: stop
  shell: "{{ app_bin }}/{{ app.shell }} stop"  
- name: start
  shell: "{{ app_bin }}/{{ app.shell }} start"
EOF


# 3. 定义task(man.yml)
lixin-macbook:deploy root# cat > tasks/main.yml <<EOF
- include: def-vars.yml
- include: copy.yml
- include: service.yml
EOF

# 4. 定义handler
lixin-macbook:deploy root# cat > handlers/restart.yml <<EOF
- name: restart app
  shell: "{{ app_bin }}/{{ app.shell }} restart"
EOF

# 5. 定义handler(main.yml)
lixin-macbook:deploy root# cat >  handlers/main.yml  <<EOF
- include: handlers.yml
EOF
```
### (5). app.sh
```
#! /bin/bash

error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"


# shell执行目录
BIN_DIR=`pwd`
# 应用程序目录
export APP_PATH=`dirname $(pwd)`
PID_FILE=${BIN_DIR}/pid
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m"


# 如果参数小于1个,则提示,并退出.
if [ $# -ne 1 ]; then
   echo "Usage: $0 {start|stop|restart}"
   exit 1;
fi


check(){
   # 如果pid文件存在,则不允许重新启动
   if [ -f ${PID_FILE} ]; then
       echo "应用程序已经启动,不允许重复启动!";
       exit 1;
   fi
}

start(){
        # 提示启动中
        echo "start..."
        echo $JAVA_HOME/bin/java $JAVA_OPT -jar /home/tomcat/test-service/lib/* > /tmp/app.log
        nohup $JAVA_HOME/bin/java $JAVA_OPT -jar /home/tomcat/test-service/lib/* > /dev/null 2>&1 &
        # 生成pid文件
    echo $! > ${PID_FILE}
}

stop(){
        # 文件存在的情况下,才能根据进程pid kill
        if [ -f ${PID_FILE} ]; then
            echo "kill pid:`cat ${PID_FILE}`"
            echo "stop..."
                kill -SIGTERM `cat ${PID_FILE}`
                rm -rf ${PID_FILE}
        fi
}

case "$1" in
  "start")
    check
    start
  ;;

  "stop")
    stop
  ;;

  "restart")
    stop
    start
  ;;
esac
```
### (6). Jenkins调用Ansible
```
# Ansible会把以下三个变量添加拼接出来,来定位要发版的构建物.  
# -e "service_name=test-service"  : 微服务的名称
# -e "second_dir=2019-09-20"      : 二级目录
#  -e "three_dir=18.20"           : 三级目录
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service" -e "second_dir=2019-09-20" -e "three_dir=18.20"
```
 