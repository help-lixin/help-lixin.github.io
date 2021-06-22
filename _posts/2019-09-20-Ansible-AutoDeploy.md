---
layout: post
title: 'Ansible 自动化部署解决方案(七)'
date: 2019-09-20
author: 李新
tags: Ansible 自动化部署 DevOps
---

### (1). 前言
+ 开发如何交付给测试?
  - 开发和测试各自部署一套Jenkins,开发交付测试时,让测试用Jenkins打包.  
  - 交付的构建物为:jar包/配置文件.
  - 在开发和测试的环境下,Jenkins都会自动打包,并结合Ansible进行应用程序的发版.  
  - 开发交付给测试,相比来说要自由一些,因为,受影不大.
+ 开发/测试如何交付给运维?
  - 运维有一套自己的Jenkins
  - 运维在定义了一个目录,专门用来存放构建物(目录需要按照一定的结构来定义).
  - 开发/测试,将要交付的构建物,放在目录里.  
  - 在Ansible的服务器上,定时获取最新的目录里的内容.  
  - 在Ansible的服务器上,定时清理5天以上的构建物.  
  - 在Jenkin定义变量(微服务名称/发布时间/详细时间),手动(定时)触发Ansible进行生产上的机器发版.  

### (2). 目录定义
```
lixin-macbook:deploy lixin$ tree
.
├── deploy_roles.yml
├── files -> /Users/lixin/WorkspaceAnsible/files    # 注意:这里的files实际是git clone到本地的一个目录,把clone后目录,软引用过来的
├── handlers
│   └── main.yml
├── tasks
│   ├── def-vars.yml
│   ├── deploy.yml
│   ├── init.yml
│   ├── main.yml
│   └── service.yml
├── templates
│   └── app.sh.j2
└── vars
    └── main.yml


# *************************************************************************
# files结构
# 注意:发版时,定位的构建物的方式是: test-service/2019-09-20/18.20,个人觉得,一个微服务,发版精确到分钟应该差不多.
# 所以,三个参数都要传递,从Jenkins传递变量进来.
# *************************************************************************
lixin-macbook:Desktop lixin$ tree /Users/lixin/WorkspaceAnsible/files
/Users/lixin/WorkspaceAnsible/files
└── test-service
    └── 2019-09-20
        └── 18.20                           ## 需要发版的内容.
            ├── conf                        ## 配置目录
            │   └── application.properties
            └── lib                         ## jar包
                └── hello-service.jar
```
### (3). 规定hosts
```
# 微服务名称就是:hosts文件名称
# /etc/hosts/hello-service
[hello-service]
10.211.55.100   ansible_ssh_user=root
10.211.55.101   ansible_ssh_user=root

# 针对A/B发版使用
[hello-service-a]
10.211.55.100   ansible_ssh_user=root

# 针对A/B发版使用
[hello-service-b]
10.211.55.101   ansible_ssh_user=root
```

### (4). Playbook定义
> <font color='re'>由于jekyll语法和代码有冲突,建议直接去github下载,参照学习.</font>  

```
# 1. 创建部署目录
lixin-macbook:~ lixin$ cd ~/GitRepository/
lixin-macbook:GitRepository lixin$ mkdir ansible_roles/
lixin-macbook:WorkspaceAnsible lixin$ mkdir -p deploy/{files,handlers,tasks,templates,vars}
lixin-macbook:WorkspaceAnsible lixin$ cd deploy


# 2. 定义变量(在Ansible中,变量不能包含中划线)
# service_name,通过jenkins读取变量,传递进来.
lixin-macbook:deploy lixin$ cat > vars/main.yml <<EOF
app:
  dir: /home/tomcat
  name: "{{ service_name }}"
  bin:  bin
  conf: conf
  lib:  lib
  logs: logs
  shell: app.sh
EOF
  
# 3. 定义task(定义变量)
lixin-macbook:deploy lixin$ cat > tasks/def-vars.yml <<EOF
- name: define local var local_conf
  set_fact: local_conf="{{ app.name }}/{{ second_dir }}/{{ three_dir }}/conf/*"
  tags :
    - deploy

- name: define local var local_lib
  set_fact: local_lib="{{ app.name }}/{{ second_dir }}/{{ three_dir }}/lib/*"
  tags :
    - deploy

- name: define remote var app_path
  set_fact: app_path="{{app.dir}}/{{app.name}}"
  tags :
    - init

- name: define remote var app_bin
  set_fact: app_bin="{{app.dir}}/{{app.name}}/{{app.bin}}"
  tags :
    - stop
    - start
    - restart
    - init
    - deploy

- name: define remote var app_conf
  set_fact: app_conf="{{app.dir}}/{{app.name}}/{{app.conf}}"
  tags :
    - deploy
    - init

- name: define remote var app_lib
  set_fact: app_lib="{{app.dir}}/{{app.name}}/{{app.lib}}"
  tags :
    - deploy
    - init

- name: define remote var app_logs
  set_fact: app_logs="{{app.dir}}/{{app.name}}/{{app.logs}}"
  tags :
    - init

# 日志输出
#- name: print vars
#  ansible.builtin.debug:
#    msg: "{{ app_bin  }} , {{ app_conf }} , {{ app_lib }} , {{ app_logs  }} , {{local_conf}} , {{local_lib}} "

EOF

# 3. 定义task(deploy.yml)
#   注意:这里只有配置文件或者JAR包有变化的情况下,才会触发:restarts,切记
lixin-macbook:deploy lixin$ cat > tasks/deploy.yml <<EOF
- name: copy config
  copy: src="{{ item }}" dest="{{ app_conf }}/" backup=yes
  notify: restart app handler
  with_fileglob:
    - "{{ local_conf }}"

- name: copy lib
  copy: src="{{ item }}" dest="{{ app_lib }}/" backup=yes
  notify: restart app handler
  with_fileglob:
    - "{{ local_lib }}"

EOF

# 3. 定义task(init.yml)
lixin-macbook:deploy lixin$ cat > tasks/init.yml <<EOF
- name: mkdir app dir
  file: state=directory recurse=yes path={{item}}
  with_items:
    - "{{ app_bin }}"
    - "{{ app_conf }}"
    - "{{ app_lib }}"
    - "{{ app_logs }}"
- name: create shell app.sh
  template: src=app.sh.j2 dest="{{ app_bin }}/app.sh" owner=root group=root mode=755 force=yes

EOF

# 3. 定义task(停止/启动微服务)
lixin-macbook:deploy lixin$ cat >  tasks/service.yml  <<EOF
- name: stop app
  shell: "{{ app_bin }}/{{ app.shell }} stop"
  tags :
    - stop
- name: start app
  shell: "{{ app_bin }}/{{ app.shell }} start"
  tags :
    - start
- name: restart app task
  shell: "{{ app_bin }}/{{ app.shell }} restart"
  tags :
    - restart

EOF


# 3. 定义task(man.yml)
lixin-macbook:deploy lixin$ cat > tasks/main.yml <<EOF
# init                   :  主要用于创建项目脚手架(mkdir -p test-service/{bin/conf/lib/logs}),以及创建可执行文件app.sh
# deploy                 :  主要用于拷贝conf/lib到被管控机器上
# stop/start/restart     :  主要用于服务的启动/关闭/重启(app.sh start/stop/restart)

- include: def-vars.yml
- include: init.yml
  tags :
    - init

- include: deploy.yml
  tags :
    - deploy

- include: service.yml

EOF

# 4. 定义handler(main.yml)
lixin-macbook:deploy lixin$ cat >  handlers/main.yml  <<EOF
- name: restart app handler
  shell: "{{ app_bin }}/{{ app.shell }} restart"

EOF

# 5. 定义roles
lixin-macbook:deploy lixin$ cat >  handlers/deploy_roles.yml  <<EOF
 - hosts: erp
   remote_user: root
   gather_facts: no
   roles:
     - role: jdk            # 会先安装JDK
     - role: deploy         # 再进行部署

EOF
```
### (5). 定义模板app.sh.j2
```
#!/bin/bash
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

APP_HOME={{ app_path }}
PID_FILE=${APP_HOME}/bin/pid
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m"

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
        nohup $JAVA_HOME/bin/java $JAVA_OPT -jar ${APP_HOME}/lib/* > /dev/null 2>&1 &
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
### (6). 调用Ansible
```
# -t init要先执行,相当于运维先打一个脚手架出来,也可以:init,deploy一起执行.
# 创建骨架以及停止,启动,重启之所以需要servie_name,是因为:要定位到该目录下执行:app.sh stop/start/restart
# -t init: 创建项目骨架
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service"  -t init

# -t stop/start/restart 服务管理
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service"  -t stop
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service"  -t start
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service"  -t restart

# -t deploy: 部署项目
# Ansible会把以下三个变量拼接出来,来定位要发版的构建物.  
# -e "service_name=test-service"  : 微服务的名称
# -e "second_dir=2019-09-20"      : 二级目录
#  -e "three_dir=18.20"           : 三级目录
ansible-playbook deploy/deploy_roles.yml -e "service_name=test-service" -e "second_dir=2019-09-20" -e "three_dir=18.20" -t init,deploy
```

### (7). 下载地址
["ansible roles"](https://github.com/help-lixin/ansible_roles.git)