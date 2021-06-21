---
layout: post
title: 'Jenkins 安装'
date: 2019-09-15
author: 李新
tags: Jenkins
---

### (1). Jenkins安装和启动
```
# https://mirrors.tuna.tsinghua.edu.cn/jenkins/war-stable/2.289.1/jenkins.war
# 下载后,通过java -jar jenkins.war直接运行
# 默认会在创建:~/.jenkins目录.
# 我想自定义保存目录
```
### (2). 配置Jenkins目录
```
# Jenkins目录
lixin-macbook:jenkins lixin$ tree
.
├── bin
│   └── jenkins.sh
├── lib
│   └── jenkins.war
└── workspace


# 配置启动脚本
lixin-macbook:jenkins lixin$ cat bin/jenkins.sh
#!/bin/bash
# 自定义jenkins_home
export JENKINS_HOME=/Users/lixin/Developer/jenkins/workspace
java -jar /Users/lixin/Developer/jenkins/lib/jenkins.war --httpPort=9999
```
### (3). 启动
```
lixin-macbook:jenkins lixin$ jenkins.sh
Running from: /Users/lixin/Developer/jenkins/lib/jenkins.war
webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
2019-09-15 06:10:15.318+0000 [id=1]	INFO	org.eclipse.jetty.util.log.Log#initialized: Logging initialized @388ms to org.eclipse.jetty.util.log.JavaUtilLog
2019-09-15 06:10:15.457+0000 [id=1]	INFO	winstone.Logger#logInternal: Beginning extraction from war file
2019-09-15 06:10:15.492+0000 [id=1]	WARNING	o.e.j.s.handler.ContextHandler#setContextPath: Empty contextPath
2019-09-15 06:10:15.565+0000 [id=1]	INFO	org.eclipse.jetty.server.Server#doStart: jetty-9.4.39.v20210325; built: 2021-03-25T14:42:11.471Z; git: 9fc7ca5a922f2a37b84ec9dbc26a5168cee7e667; jvm 1.8.0_251-b08
2019-09-15 06:10:15.945+0000 [id=1]	INFO	o.e.j.w.StandardDescriptorProcessor#visitServlet: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
2019-09-15 06:10:16.011+0000 [id=1]	INFO	o.e.j.s.s.DefaultSessionIdManager#doStart: DefaultSessionIdManager workerName=node0
2019-09-15 06:10:16.011+0000 [id=1]	INFO	o.e.j.s.s.DefaultSessionIdManager#doStart: No SessionScavenger set, using defaults
2019-09-15 06:10:16.012+0000 [id=1]	INFO	o.e.j.server.session.HouseKeeper#startScavenging: node0 Scavenging every 660000ms
2019-09-15 06:10:16.555+0000 [id=1]	INFO	hudson.WebAppMain#contextInitialized: Jenkins home directory: /Users/lixin/Developer/jenkins/workspace found at: EnvVars.masterEnvVars.get("JENKINS_HOME")
2019-09-15 06:10:18.716+0000 [id=1]	INFO	o.e.j.s.handler.ContextHandler#doStart: Started w.@2b72cb8a{Jenkins v2.289.1,/,file:///Users/lixin/Developer/jenkins/workspace/war/,AVAILABLE}{/Users/lixin/Developer/jenkins/workspace/war}
2019-09-15 06:10:18.752+0000 [id=1]	INFO	o.e.j.server.AbstractConnector#doStart: Started ServerConnector@6df97b55{HTTP/1.1, (http/1.1)}{0.0.0.0:9999}
2019-09-15 06:10:18.808+0000 [id=1]	INFO	org.eclipse.jetty.server.Server#doStart: Started @3878ms
2019-09-15 06:10:18.809+0000 [id=23]	INFO	winstone.Logger#logInternal: Winstone Servlet Engine running: controlPort=disabled
2019-09-15 06:10:20.099+0000 [id=30]	INFO	jenkins.InitReactorRunner$1#onAttained: Started initialization
2019-09-15 06:10:20.167+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: Listed all plugins
2019-09-15 06:10:21.638+0000 [id=34]	INFO	jenkins.InitReactorRunner$1#onAttained: Prepared all plugins
2019-09-15 06:10:21.647+0000 [id=35]	INFO	jenkins.InitReactorRunner$1#onAttained: Started all plugins
2019-09-15 06:10:21.671+0000 [id=32]	INFO	jenkins.InitReactorRunner$1#onAttained: Augmented all extensions
2019-09-15 06:10:22.147+0000 [id=32]	INFO	jenkins.InitReactorRunner$1#onAttained: System config loaded
2019-09-15 06:10:22.147+0000 [id=28]	INFO	jenkins.InitReactorRunner$1#onAttained: System config adapted
2019-09-15 06:10:22.148+0000 [id=28]	INFO	jenkins.InitReactorRunner$1#onAttained: Loaded all jobs
2019-09-15 06:10:22.148+0000 [id=28]	INFO	jenkins.InitReactorRunner$1#onAttained: Configuration for all jobs updated
2019-09-15 06:10:22.160+0000 [id=48]	INFO	hudson.model.AsyncPeriodicWork#lambda$doRun$0: Started Download metadata
2019-09-15 06:10:22.165+0000 [id=48]	INFO	hudson.model.AsyncPeriodicWork#lambda$doRun$0: Finished Download metadata. 5 ms
2019-09-15 06:10:22.294+0000 [id=33]	INFO	jenkins.install.SetupWizard#init:

*************************************************************
*************************************************************
*************************************************************

# 这是admin的密码,密码保存在:
# /Users/lixin/Developer/jenkins/workspace/secrets/initialAdminPassword
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

f2522f937ab5402691f597b5567486eb

This may also be found at: /Users/lixin/Developer/jenkins/workspace/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************

2019-09-15 06:10:43.309+0000 [id=34]	INFO	jenkins.InitReactorRunner$1#onAttained: Completed initialization
2019-09-15 06:10:43.323+0000 [id=22]	INFO	hudson.WebAppMain$3#run: Jenkins is fully up and running
```
### (4). 通过WEB访问
!["Jenkins 启动"](/assets/jenkins/imgs/jenkin-start.png)  
!["Jenkins 自定义安装插件"](/assets/jenkins/imgs/jenkins_customer-install-plugin.png)
