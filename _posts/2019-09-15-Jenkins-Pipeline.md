---
layout: post
title: 'Jenkins + Ansible 自动化部署解决方案(八)'
date: 2019-09-15
author: 李新
tags: Jenkins Ansible 自动化部署 DevOps
---

### (1). 需求
> 1. 不想重复造轮子,因为,我要的就三个参数而已,为了这三个参数增加一个工程不划算.      
> 2. 开发或者运维工程师,可以通过Jenkins浏览(注意是:浏览,而不是手写输入)构建物,从而进行发版.  
> 3. 效果如下:  

!["jenkins params"](/assets/jenkins/imgs/jenkins-params.jpg)

```
# 运维定义的发版构建物目录如下:
# 可以通过git去拿取,也可以sync同步过来.
lixin-macbook:Desktop lixin$ tree ansible_files/
ansible_files/
├── hello-service                                                           
└── test-service                                      # 第一级下拉列表框
    ├── 2019-09-20                                    # 第二级下拉列表框 
    │   └── 18.20                                     # 第三级下拉列表框
    │       ├── conf
    │       │   └── application.properties
    │       └── lib
    │           └── hello-service.jar
    └── 2019-09-21
```

### (2). Jenkins安装插件
> [Active Choices Plug-in](https://github.com/jenkinsci/active-choices-plugin)这是一个构建动态参数的插件.  

### (3). Pipeline定义
> Ansible脚本内容请参考该链接["Jenkins + Ansible 自动化部署解决方案(七)"](/2019/09/20/Ansible-AutoDeploy.html)

```
properties([
    parameters([
        [$class: 'ChoiceParameter', 
            choiceType: 'PT_SINGLE_SELECT', 
            description: '微服务名称', 
            filterLength: 1, 
            filterable: true, 
            name: 'service_name', 
            randomName: 'choice-parameter-5631314456178620', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        '''
                          return [\'Could not get service_name\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        '''
                          def result = ['/bin/bash', '-c', "/Users/lixin/WorkspaceAnsible/shell/list-dir.sh"].execute()
                          return result.text.readLines()
                        '''
                ]
            ]
        ],
        [$class: 'CascadeChoiceParameter', 
            choiceType: 'PT_CHECKBOX', 
            description: '发版机器', 
            filterLength: 1, 
            filterable: true, 
            name: 'hosts', 
            randomName: 'choice-parameter-5631314456178621', 
            referencedParameters: 'service_name', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        '''
                          return [\'Could not get hosts\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        '''
                          def result = ['/bin/bash', '-c', "/Users/lixin/WorkspaceAnsible/shell/list-hosts.sh  '${service_name}'  "].execute()
                          return result.text.readLines()
                        '''
                ]
            ]
        ],
        [$class: 'CascadeChoiceParameter', 
            choiceType: 'PT_SINGLE_SELECT', 
            description: '发版日期', 
            filterLength: 1, 
            filterable: true, 
            name: 'deploy_day', 
            randomName: 'choice-parameter-5631314456178622', 
            referencedParameters: 'service_name', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          return[\'Could not get Environment from deploy_day Param\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          def result = ['/bin/bash', '-c', "/Users/lixin/WorkspaceAnsible/shell/list-dir.sh '${service_name}' "].execute()
                          return result.text.readLines()
                        '''
                ]
            ]
        ],
        [$class: 'CascadeChoiceParameter', 
            choiceType: 'PT_SINGLE_SELECT', 
            description: '发版时间', 
            filterLength: 1, 
            filterable: true, 
            name: 'deploy_time', 
            randomName: 'choice-parameter-5631314456178623', 
            referencedParameters: 'service_name,deploy_day', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          return[\'Could not get Environment from deploy_time Param\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          def result = ['/bin/bash', '-c', "/Users/lixin/WorkspaceAnsible/shell/list-dir.sh '${service_name}'  '${deploy_day}' "].execute()
                          return result.text.readLines()
                        '''
                ]
            ]
        ]
    ])
])

pipeline {
  agent any
  stages {
    stage ("deploy") {
        steps {
            script {
                echo "${params.service_name}"
                echo "${params.deploy_day}"
                echo "${params.deploy_time}"
                echo "${hosts}"

                if( 
                   params.service_name.equals("Could not get service_name") || 
                   params.service_name.equals("") || 
                   params.deploy_day.equals("Could not get Environment from deploy_day Param") || 
                   params.deploy_day.equals("") || 
                   params.deploy_time.equals("Could not get Environment from deploy_time Param") ||
                   params.deploy_time.equals("") ) 
                {
                  echo "构建退出,原因:所有的参数都是必选项"
                  currentBuild.result = 'ABORTED'
                  return
                } else {
                  sh '/usr/local/bin/ansible-playbook /Users/lixin/GitRepository/ansible_roles/deploy/deploy_roles.yml -i /Users/lixin/WorkspaceAnsible/hosts/"${service_name}" -e "hosts=${hosts}"  -e "service_name=${service_name}" -e "second_dir=${deploy_day}" -e "three_dir=${deploy_time}" -t init,deploy'
                }
            }
        }
    }
  }
}
```
### (4). /Users/lixin/WorkspaceAnsible/shell/list-dir.sh
```
# 记得添加可执行权限
lixin-macbook:Desktop lixin$ cat list-dir.sh
#!/bin/bash
ansible_files_path=/Users/lixin/WorkspaceAnsible/files

if [ $# == 0 ];
then
  ansible_files_path=/Users/lixin/WorkspaceAnsible/files
elif [ $# == 1 ];
 then
  ansible_files_path=${ansible_files_path}/$1
else
  ansible_files_path=${ansible_files_path}/$1/$2
fi
ls -lh ${ansible_files_path} | awk 'NR > 1 {print $NF}'
```

### (5). 结合hosts(/Users/lixin/WorkspaceAnsible/shell/list-hosts.sh)
```
#!/bin/bash
hosts_file=/Users/lixin/WorkspaceAnsible/hosts/$1
echo "${hosts_file}" >> /tmp/log.txt
cat $hosts_file | grep -i \\[[^]\\]* | tr -d '[' | tr -d ']'
```

### (6). jenkins定义hosts文件夹
```
# 1. 定义hosts文件夹
lixin-macbook:hosts lixin$ pwd
	/Users/lixin/WorkspaceAnsible/hosts

# 2.每个微服务,定义一个hosts文件
lixin-macbook:hosts lixin$ cat test-service

[erp]
10.211.55.100 ansible_ssh_user=root
10.211.55.101 ansible_ssh_user=root

# a/b
[erp-a]
10.211.55.100 ansible_ssh_user=root

[erp-b]
10.211.55.101 ansible_ssh_user=root

[alibaba]
10.211.55.100 ansible_ssh_user=root

[tencent]
10.211.55.101 ansible_ssh_user=root
```
### (7). jenkins 查看运行后结果
```
Started by user admin
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] properties
[Pipeline] node
Running on Jenkins in /Users/lixin/Developer/jenkins/workspace/workspace/test
[Pipeline] {
[Pipeline] stage
[Pipeline] { (deploy)
[Pipeline] script
[Pipeline] {
	
[Pipeline] echo test-service
[Pipeline] echo 2019-09-20
[Pipeline] echo 18.20
[Pipeline] echo  alibaba,tencent

############################################################################################
# 要执行的shell脚本
# -i 指定hosts文件
############################################################################################
[Pipeline] sh
+ /usr/local/bin/ansible-playbook /Users/lixin/GitRepository/ansible_roles/deploy/deploy_roles.yml -i /Users/lixin/WorkspaceAnsible/hosts/test-service -e hosts=alibaba,tencent -e service_name=test-service -e second_dir=2019-09-20 -e three_dir=18.20 -t init,deploy
[WARNING]: Invalid characters were found in group names but not replaced, use
-vvvv to see details
[WARNING]: Found variable using reserved name: hosts

PLAY [alibaba,tencent] *********************************************************

TASK [jdk : print vars] ********************************************************
ok: [10.211.55.100] => {
    "msg": " unzip jdk-8u271-linux-x64.tar.gz , install to: /usr/local/jdk "
}
ok: [10.211.55.101] => {
    "msg": " unzip jdk-8u271-linux-x64.tar.gz , install to: /usr/local/jdk "
}

TASK [jdk : unzip jdk-8u271-linux-x64.tar.gz -d /usr/local] ********************
changed: [10.211.55.100]
changed: [10.211.55.101]

TASK [jdk : ls -s /usr/local/jdk1.8.0_271  /usr/local/jdk] *********************
changed: [10.211.55.101]
changed: [10.211.55.100]

TASK [jdk : add JAVA_HOME to System Environment Variable] **********************
changed: [10.211.55.101]
changed: [10.211.55.100]

TASK [deploy : define local var local_conf] ************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define local var local_lib] *************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define remote var app_path] *************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define remote var app_bin] **************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define remote var app_conf] *************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define remote var app_lib] **************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : define remote var app_logs] *************************************
ok: [10.211.55.100]
ok: [10.211.55.101]

TASK [deploy : mkdir app dir] **************************************************
changed: [10.211.55.100] => (item=/home/tomcat/test-service/bin)
changed: [10.211.55.101] => (item=/home/tomcat/test-service/bin)
changed: [10.211.55.100] => (item=/home/tomcat/test-service/conf)
changed: [10.211.55.101] => (item=/home/tomcat/test-service/conf)
changed: [10.211.55.100] => (item=/home/tomcat/test-service/lib)
changed: [10.211.55.101] => (item=/home/tomcat/test-service/lib)
changed: [10.211.55.100] => (item=/home/tomcat/test-service/logs)
changed: [10.211.55.101] => (item=/home/tomcat/test-service/logs)

TASK [deploy : create shell app.sh] ********************************************
changed: [10.211.55.100]
changed: [10.211.55.101]

TASK [deploy : copy config] ****************************************************
changed: [10.211.55.100] => (item=/Users/lixin/GitRepository/ansible_roles/deploy/files/test-service/2019-09-20/18.20/conf/application.properties)
changed: [10.211.55.101] => (item=/Users/lixin/GitRepository/ansible_roles/deploy/files/test-service/2019-09-20/18.20/conf/application.properties)

TASK [deploy : copy lib] *******************************************************
changed: [10.211.55.100] => (item=/Users/lixin/GitRepository/ansible_roles/deploy/files/test-service/2019-09-20/18.20/lib/hello-service.jar)
changed: [10.211.55.101] => (item=/Users/lixin/GitRepository/ansible_roles/deploy/files/test-service/2019-09-20/18.20/lib/hello-service.jar)

RUNNING HANDLER [deploy : restart app handler] *********************************
changed: [10.211.55.100]
changed: [10.211.55.101]

PLAY RECAP *********************************************************************
10.211.55.100              : ok=16   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.211.55.101              : ok=16   changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```
### (6). 总结
> 实际上:运维需要的仅仅只是Jenkins的参数以及UI界面而已.主要是不想自己再研发一套UI界面出来,以及权限管理等等.  