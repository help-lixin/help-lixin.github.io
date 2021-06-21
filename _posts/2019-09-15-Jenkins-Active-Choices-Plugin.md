---
layout: post
title: 'Jenkins Active Choices Plug-in结合Pipeline'
date: 2019-09-15
author: 李新
tags: Jenkins
---

### (1). 需求
> 1. 不想自己再造轮子,开发一套运维平台.    
> 2. 开发/运维,可以通过Jenkins浏览(注意是浏览,不是手写输入)构建物,自由选择发版.  
> 3. 期望效果如下:  

!["jenkins params"](/assets/jenkins/imgs/jenkins-params.jpg)

```
# 目录结构如下
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
### (2). 安装插件
> [Active Choices Plug-in](https://github.com/jenkinsci/active-choices-plugin)这是一个构建动态参数的插件.  

### (3). pipeline
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
                          return [\'Could not get Env\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        '''
                          def result = ['/bin/bash', '-c', "/Users/lixin/Desktop/list-dir.sh"].execute()
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
            randomName: 'choice-parameter-5631314456178621', 
            referencedParameters: 'service_name', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          return[\'Could not get Environment from Env Param\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          def result = ['/bin/bash', '-c', "/Users/lixin/Desktop/list-dir.sh '${service_name}' "].execute()
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
            randomName: 'choice-parameter-5631314456178622', 
            referencedParameters: 'service_name,deploy_day', 
            script: [
                $class: 'GroovyScript', 
                fallbackScript: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          return[\'Could not get Environment from Env Param\']
                        '''
                ], 
                script: [
                    classpath: [], 
                    sandbox: false, 
                    script: 
                        ''' 
                          def result = ['/bin/bash', '-c', "/Users/lixin/Desktop/list-dir.sh '${service_name}'  '${deploy_day}' "].execute()
                          return result.text.readLines()
                        '''
                ]
            ]
        ]
    ])
])

pipeline {
  environment {
	ansible_files = "/Users/lixin/Desktop/ansible_files"
  }
  agent any
  stages {
      stage ("Example") {
            steps {
                    script{
                    echo 'Hello'
                    echo "${params.service_name}"
                    echo "${params.deploy_day}"
                    echo "${params.deploy_time}"
                }
            }
      }
    }
}
```
