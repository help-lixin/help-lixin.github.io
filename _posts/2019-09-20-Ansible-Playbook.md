---
layout: post
title: 'Ansible Playbook(四)'
date: 2019-09-20
author: 李新
tags: Ansible
---

### (1). Playbook是什么
> Playbook由多个Task组成,而这些Task实际就是ansible module.通过组装多个Task,形成Playbook.

### (2). Plabook核心元素
+ Hosts      执行的远程主机列表.  
+ Tasks      任务集
+ Variables  变量
+ Templates  模板
+ Handlers   和notify结合使用,由特定条件触发的操作,满足条件才会执行,否则,不执行.
+ Tags       标签(可以带上标签,做条件选择)

### (3). Playbook安装Nginx
```
[lixin@manager nginx]$ cat nginx-install-playbook.yml
---
- hosts: erp
  remote_user: root
  gather_facts: no
  tasks:
      - name: "install nginx"
        yum: name=nginx state=installed
      - name: "copy nginx.conf"
        copy: src=./nginx.conf dest=/etc/nginx
        notify: restart nginx
      - name: "start nginx"
        service: name=nginx state=started enabled=yes
  handlers:
      - name: restart nginx
        service: name=nginx state=restarted
```

```
[root@manager nginx]# cat nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       81;
        listen       [::]:81;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }

}
```

```
# ansible-playbook options: 
# -C --check     : 只检测可能会发生的变化,不会真正执行操作
# --list-hosts   : 列出运行任务的主机
# --list-tags    : 列出tag
# --list-tasks   : 列出task
# -vvv           : 显示详细
# -e             : 指定变量

# [lixin@manager nginx]$ ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml
[lixin@manager ~]$ ansible-playbook ./tools/nginx/nginx-install-playbook.yml  -i erp/hosts  --become  --become-method=sudo --become-user=root -K
BECOME password[defaults to SSH password]:  # root对应的密码.
```
### (4). Playbook 变量的使用
Variables的配置方式有两种:
+ 运行ansible-playbook时,通过参数传入变量.    
+ hosts文件配置变量.   
+ playbook文件中定义变量.   
+ 通过setup获得变量. 
+ 通过独立的YAML定义变量,并引入.  

```
#  演示通过参数传入变量.    
# 1. 定义yml(创建目录)
[root@manager lixin]# cat var.yml
---
- hosts: erp
  remote_user: root
  tasks:
    - name: "create direct"
      shell: mkdir -p {{path}}

# 2. 通过-e指定传递变量
[root@manager lixin]# ansible-playbook -i erp/hosts var.yml -e "path=/tmp/test2"
```


```
# 演示通过hosts文件配置变量
# 1. hosts定义变量
[root@manager lixin]# cat erp/hosts
[erp]
  test   path=/tmp/erp-1   # 2. 通过参数(-e)传入变量,优于单主机定义的变量,而,单主机定义的变量又优先于组定义变量

[erp:vars]                 
  path=/tmp/erp-2          # 3. 定义整个组的变量
  
  
# 2. 直接定义hosts
[root@manager lixin]# ansible-playbook -i erp/hosts var.yml
```

```
# 演示:playbook文件中定义变量.   
# 1. 通过playbook中定义变量
[root@manager lixin]# cat var.yml
---
- hosts: erp
  remote_user: root
  vars:                     # 通过定义vars定义变量
    path: /tmp/erp-3

  tasks:
    - name: "create direct"
      shell: mkdir -p {{path}}

# 2. 运行
[root@manager lixin]# ansible-playbook -i erp/hosts var.yml
```

```
# 演示通过setup获得变量. 
# 1. 获取setup中的变量
[root@manager lixin]# cat var.yml
---
- hosts: erp
  remote_user: root

  tasks:
    - name: "create direct"
      shell: mkdir -p /tmp/{{ansible_fqdn}}

# 2. 运行
[root@manager lixin]# ansible-playbook -i erp/hosts var.yml
```

```

# 1. 定义变量文件
[root@manager lixin]# cat tmp.yml
path: /tmp/hello

# 2. 定义playbook
[root@manager lixin]# cat var.yml
---
- hosts: erp
  remote_user: root
  vars_files:          # 引用变量文件
    - ./tmp.yml
  tasks:
    - name: "create direct"        # 使用变量
      shell: mkdir -p {{path}}

# 3. 运行
[root@manager lixin]# ansible-playbook -i erp/hosts var.yml
```
### (5). Playbook Tags使用
一个playbook文件中,执行时如果想执行某一个任务,那么可以给每个任务集进行打标签,这样在执行的时候可以通过-t选择指定标签执行,还可以通过--skip-tags选择除了某个标签外全部执行等.  

```
# 演示tags
# 1. 定义playbook文件
[root@manager nginx]# cat nginx-install-playbook.yml
---
- hosts: erp
  remote_user: root
  gather_facts: no
  tasks:
      - name: "install nginx"
        yum: name=nginx state=installed
        tags: install
      - name: "copy nginx.conf"
        copy: src=./nginx.conf dest=/etc/nginx
        tags:
          - install
          - upgrade
        notify: restart nginx
      - name: "start nginx"
        service: name=nginx state=started enabled=yes
        tags:
          - install
          - start
      - name: "stop nginx"
        service: name=nginx state=stopped
        tags:
          - stop
      - name: "restart nginx"
        service: name=nginx state=restarted
        tags:
          - restart
          - upgrade
  handlers:
      - name: restart nginx
        service: name=nginx state=restarted


# 2. 仅运行拥有标签:install的task
[root@manager nginx]# ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml -t install
PLAY [erp] ***********************************************************************************************************
TASK [install nginx] *************************************************************************************************
changed: [test]
TASK [copy nginx.conf] ***********************************************************************************************
changed: [test]
TASK [start nginx] ***************************************************************************************************
changed: [test]
RUNNING HANDLER [restart nginx] **************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=4    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

# 3. 仅运行拥有标签:restart的task(可,查看test机器上,nginx进程ID,就知道是否生效)
# systemctl  restart nginx
[root@manager nginx]# ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml -t restart
PLAY [erp] ***********************************************************************************************************
TASK [restart nginx] *************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


# 4. 仅运行拥有标签:stop的task
# # systemctl stop nginx
[root@manager nginx]# ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml -t stop
PLAY [erp] ***********************************************************************************************************
TASK [stop nginx] ****************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

# 5. 仅运行拥有标签:start的task
[root@manager nginx]# ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml -t start
# systemctl  start nginx
PLAY [erp] ***********************************************************************************************************
TASK [start nginx] ***************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


# 6. 排除某些标签:--skip-tags install,start,stop,在这里仅只有:restart nginx是运行的
[root@manager nginx]# ansible-playbook -i ../../erp/hosts  nginx-install-playbook.yml --skip-tags install,start,stop
PLAY [erp] ***********************************************************************************************************
TASK [restart nginx] *************************************************************************************************
changed: [test]
PLAY RECAP ***********************************************************************************************************
test                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
### (6). Playbook Templates
template模板提供了动态配置服务,使用jinja2语言,里面支持多种条件判断、循环、逻辑运算、比较操作等.和之前配置文件使用copy一样,只是使用copy,不能根据服务器配置不一样进行不同动态的配置,这样就不利于管理. 
注意:   
1、多数情况下都将template文件放在和playbook文件同级的templates目录下,这样playbook文件中可以直接引用,会自动去找这个文件.如果放在别的地方,也可以通过绝对路径去指定.    
2、模板文件后缀名为.j2.  


```
# 1. 目录结构如下,一般templates与我们的playbook是在同一级
[root@manager nginx]# tree
.
├── nginx-playbook.yml
└── templates               # 模板目录
    └── nginx.conf.j2

# 2. 定义playbook(nginx-playbook.yml)
[root@manager nginx]# cat nginx-playbook.yml
---
- hosts: erp
  remote_user: root
  gather_facts: no
  vars:                          # 1. 注意:此处声明了变量,在Templates中使用了变量
    listen_port: 80
  tasks:
      - name: "install nginx"
        yum: name=nginx state=installed
        tags: install
      - name: "config nginx.conf"
        template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
        tags:
          - install
          - upgrade
        notify: restart nginx
      - name: "start nginx"
        service: name=nginx state=started enabled=yes
        tags:
          - install
          - start
      - name: "stop nginx"
        service: name=nginx state=stopped
        tags:
          - stop
      - name: "restart nginx"
        service: name=nginx state=restarted
        tags:
          - restart
          - upgrade
  handlers:
      - name: restart nginx
        service: name=nginx state=restarted

# 3. 定义模板
# 模板中引用变量:{{listen_port}}
[root@manager nginx]# cat templates/nginx.conf.j2|grep listen
        listen       {{listen_port}};
        listen       [::]:{{listen_port}};
```

### (7). Playbook Templates复杂用法


```
# when用法
# 把字符串转换成数字
tasks:
  - shell: echo "only on Red Hat 6, derivatives, and later"
    when: ansible_os_family == "RedHat" and ansible_lsb.major_release|int >= 6
	
# when用法
# is defined(变量是否存在) / is not defined(变量是否不存在)
tasks:
    - shell: echo "I've got '{{ foo }}' and am not afraid to use it!"
      when: foo is defined

    - fail: msg="Bailing out. this play requires 'bar'"
      when: bar is not defined	


# 通过with_items安装多个不同软件
# 对迭代项的引用,固定变量名为"item"
[root@manager lixin]# cat with_items.yml
---
- hosts: erp
  remote_user: root

  tasks:
    - name: "Install Package"
      yum: name={{item}} state=installed
      with_items:
        - nginx
        - ntpdate


# 通过with_items创建子目录
# mkdir -p /tmp/test-service/{bin,conf,lib,logs}
[root@manager lixin]# cat test-service.yml
---
- hosts: erp
  remote_user: root
  tasks:
    - name: "mkdir workd dir"
      file: state=directory recurse=yes path={{item.dir}}{{item.child}}
      with_items:
        - { dir: "/tmp/test-service" , child: "/bin" }
        - { dir: "/tmp/test-service" , child: "/conf" }
        - { dir: "/tmp/test-service" , child: "/lib" }
        - { dir: "/tmp/test-service" , child: "/logs" }



# 通过for,创建多个server
1. 查看目录结构
[root@manager nginx]# tree
.
├── nginx-playbook.yml
└── templates
    └── nginx.conf.j2
	
	
# 2. playbook定义	
[root@manager nginx]# cat nginx-playbook.yml
---
- hosts: erp
  remote_user: root
  gather_facts: no
  vars:
    nginx_vhosts:                # 定义变量
      - web1:
        listen: 8081
        server_name: "web1.example.com"
        root: "/usr/share/nginx/html"
      - web2:
        listen: 8082
        server_name: "web2.example.com"
        root: "/usr/share/nginx/html"
  tasks:
      - name: "install nginx"
        yum: name=nginx state=installed
        tags: install
      - name: "config nginx.conf"
        template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
        tags:
          - install
          - upgrade
        notify: restart nginx
      - name: "start nginx"
        service: name=nginx state=started enabled=yes
        tags:
          - install
          - start
      - name: "stop nginx"
        service: name=nginx state=stopped
        tags:
          - stop
      - name: "restart nginx"
        service: name=nginx state=restarted
        tags:
          - restart
          - upgrade
  handlers:
      - name: restart nginx
        service: name=nginx state=restarted	
	
# 3. 定义模板,使用for循环
[root@manager nginx]# cat templates/nginx.conf.j2
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    # for循环
    {% for vhost in nginx_vhosts %}
    server {
		# if-else-endif 判断
        {% if vhost.listen is defined %}
          listen       {{vhost.listen}};
          listen       [::]:{{vhost.listen}};
        {% else %}
          listen       80;
          listen       [::]:80;
        {% endif %}


        server_name  {{vhost.server_name}};
        root         {{vhost.root}};

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    {% endfor %}
}
```
