---
layout: post
title: 'Ansible Roles(五)'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). Roles介绍
> roles就是通过分别将变量、文件、任务、模板及处理器放置于单独的目录中,并可以通过:include简单的使用.  

### (2). Roles目录结构
```
roles:          <--所有的角色必须放在roles目录下,这个目录可以自定义位置,默认的位置在:/etc/ansible/roles
  project:      <---具体的角色项目名称,比如nginx、tomcat、php
    files：     <--用来存放由copy模块或script模块调用的文件。
    templates： <--用来存放jinjia2模板,template模块会自动在此目录中寻找jinjia2模板文件.
    tasks：     <--此目录应当包含一个main.yml文件,用于定义此角色的任务列表,此文件可以使用include包含其它的位于此目录的task文件. 
      main.yml
    handlers：  <--此目录应当包含一个main.yml文件，用于定义此角色中触发条件时执行的动作. --> 
      main.yml
    vars：      <--此目录应当包含一个main.yml文件，用于定义此角色用到的变量 --> 
      main.yml
    defaults：  <--此目录应当包含一个main.yml文件，用于为当前角色设定默认变量 -->
      main.yml
    meta：      <--此目录应当包含一个main.yml文件，用于定义此角色的特殊设定及其依赖关系 -->
      main.yml
```
### (3). Roles案例
```
# 1. 进入到roles目录下
[root@manager ~]# cd /etc/ansible/roles/
# 2. 创建roles需要的目录
[root@manager roles]# mkdir -p nginx/{handlers,tasks,templates,vars}

# 3. 配置变量信息(vars/main.yml)
[root@manager roles]# cat >  nginx/vars/main.yml <<EOF
listen: 81
server_name: _
root: /usr/share/nginx/html
EOF


# 4. 定义模板(templates/nginx.conf.j2)
# 注意:我省略掉了一些没用的信息
[root@manager roles]# cat  nginx/templates/nginx.conf.j2
// ... ... 
server {
        listen       {{listen}};
        listen       [::]:{{listen}};

        server_name  {{server_name}};
        root         {{root}};

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
// ... ... 


# 5. 定义task(install nginx)
[root@manager roles]# cat > nginx/tasks/install.yml <<EOF
 - name: "install nginx"
   yum: name=nginx state=installed
   tags: install
 EOF

# 5. 定义task,定义配置模板(nginx.conf.j2)
[root@manager roles]# cat > nginx/tasks/config.yml <<EOF
 - name: "config nginx.conf"
   template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
   notify: restart nginx
   tags:
     - install
     - upgrade
EOF

# 5. 定义task(start)
[root@manager roles]# cat > nginx/tasks/start.yml <<EOF
 - name: "start nginx"
   service: name=nginx state=started enabled=yes
   tags:
     - install
     - start
EOF

# 5. 定义task(stop)
[root@manager roles]# cat > nginx/tasks/stop.yml <<EOF
 - name: "stop nginx"
   service: name=nginx state=stopped 
   tags:
     - stop
EOF

# 5. 定义task(main)
[root@manager roles]# cat > nginx/tasks/main.yml <<EOF
- include: install.yml
- include: config.yml
- include: stop.yml
- include: start.yml
EOF


# 6. 定义handler(restart)
[root@manager roles]# cat > nginx/handlers/restart.yml <<EOF
 - name: restart nginx
   service: name=nginx state=restarted
   tags:
     - restart
EOF

# 7. 编写主的playbook
[root@manager roles]# cd /etc/ansible/roles/nginx/
[root@manager nginx]# cat > nginx_roles.yml <<EOF
 - hosts: erp
   remote_user: root
   gather_facts: no
   roles:
     - role: nginx
EOF
```
### (4). 查看项目结构
```
[root@manager roles]# pwd
/etc/ansible/roles

[root@manager roles]# tree nginx
nginx
├── handlers
│   └── restart.yml
├── nginx_roles.yml
├── tasks
│   ├── config.yml
│   ├── install.yml
│   ├── main.yml
│   ├── start.yml
│   └── stop.yml
├── templates
│   └── nginx.conf.j2
└── vars
    └── main.yml
```
### (5). 测试
```
# 1. 安装nginx
[root@manager nginx]# ansible-playbook -i /home/lixin/erp/hosts  nginx_roles.yml  -t install
PLAY [erp] ***********************************************************************************************************
TASK [install nginx] *************************************************************************************************
ok: [test]
TASK [config nginx.conf] *********************************************************************************************
ok: [test]
TASK [start nginx] ***************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


# 2. 停止nginx
[root@manager nginx]# ansible-playbook -i /home/lixin/erp/hosts  nginx_roles.yml  -t stop
PLAY [erp] ***********************************************************************************************************
TASK [stop nginx] ****************************************************************************************************
ok: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

# 3. 启动nginx
[root@manager nginx]# ansible-playbook -i /home/lixin/erp/hosts  nginx_roles.yml  -t start
PLAY [erp] ***********************************************************************************************************
TASK [start nginx] ***************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```