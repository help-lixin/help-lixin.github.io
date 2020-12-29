---
layout: post
title: '自定义Java应用启动的Shell脚本'
date: 2017-06-04
author: 李新
tags: Linux
---

### (1). 目录结构
```
lixin-macbook:spring-example-1.1 lixin$ tree
.
├── bin
│   ├── app.sh
│   └── pid
├── conf
│   ├── application.properties
│   └── logback-spring.xml
├── lib
│   ├── accessors-smart-1.2.jar
│   ├── android-json-0.0.20131108.vaadin1.jar
│   ├── asm-5.0.4.jar
│   ├── assertj-core-3.11.1.jar
│   ├── byte-buddy-1.9.3.jar
│   ├── byte-buddy-agent-1.9.3.jar
│   ├── classmate-1.4.0.jar
│   ├── hamcrest-core-1.3.jar
│   ├── hamcrest-library-1.3.jar
│   ├── hibernate-validator-6.0.13.Final.jar
│   ├── jackson-annotations-2.9.0.jar
│   ├── jackson-core-2.9.7.jar
│   ├── jackson-databind-2.9.7.jar
│   ├── jackson-datatype-jdk8-2.9.7.jar
│   ├── jackson-datatype-jsr310-2.9.7.jar
│   ├── jackson-module-parameter-names-2.9.7.jar
│   ├── javax.annotation-api-1.3.2.jar
│   ├── jboss-logging-3.3.2.Final.jar
│   ├── json-path-2.4.0.jar
│   ├── json-smart-2.3.jar
│   ├── jsonassert-1.5.0.jar
│   ├── jul-to-slf4j-1.7.25.jar
│   ├── junit-4.12.jar
│   ├── log4j-api-2.11.1.jar
│   ├── log4j-to-slf4j-2.11.1.jar
│   ├── logback-classic-1.2.3.jar
│   ├── logback-core-1.2.3.jar
│   ├── mockito-core-2.23.0.jar
│   ├── objenesis-2.6.jar
│   ├── slf4j-api-1.7.25.jar
│   ├── snakeyaml-1.23.jar
│   ├── spring-aop-5.1.2.RELEASE.jar
│   ├── spring-beans-5.1.2.RELEASE.jar
│   ├── spring-boot-2.1.0.RELEASE.jar
│   ├── spring-boot-autoconfigure-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-json-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-logging-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-test-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-tomcat-2.1.0.RELEASE.jar
│   ├── spring-boot-starter-web-2.1.0.RELEASE.jar
│   ├── spring-boot-test-2.1.0.RELEASE.jar
│   ├── spring-boot-test-autoconfigure-2.1.0.RELEASE.jar
│   ├── spring-context-5.1.2.RELEASE.jar
│   ├── spring-core-5.1.2.RELEASE.jar
│   ├── spring-example-1.0.0-SNAPSHOT.jar
│   ├── spring-expression-5.1.2.RELEASE.jar
│   ├── spring-jcl-5.1.2.RELEASE.jar
│   ├── spring-test-5.1.2.RELEASE.jar
│   ├── spring-web-5.1.2.RELEASE.jar
│   ├── spring-webmvc-5.1.2.RELEASE.jar
│   ├── tomcat-embed-core-9.0.12.jar
│   ├── tomcat-embed-el-9.0.12.jar
│   ├── tomcat-embed-websocket-9.0.12.jar
│   ├── validation-api-2.0.1.Final.jar
│   └── xmlunit-core-2.6.2.jar
└── logs
    └── level-logs
        ├── log_debug.log
        ├── log_error.log
        ├── log_info.log
        └── log_warn.log
```
### (2). app.sh脚本
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
# 可以能过Ant进行替换
MAIN_CLASS=help.lixin.spring.App
# 应用程序目录
export APP_PATH=`dirname $(pwd)`
PID_FILE=${BIN_DIR}/pid
CLASSPATH=.:${APP_PATH}/conf:${APP_PATH}/lib/*:
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

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
	nohup $JAVA_HOME/bin/java $JAVA_OPT $MAIN_CLASS "$@" > /dev/null 2>&1 &
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
### (3). 运行后查看进程信息
```
 lixin-macbook:bin lixin$ ps -ef|grep java
  501  3085     1   0  3:16下午 ttys001    0:10.72 /Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/bin/java -server -Xms1g -Xmx1g -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m -cp .:/Users/lixin/Workspace/spring-example/target/spring-example-1.1/conf:/Users/lixin/Workspace/spring-example/target/spring-example-1.1/lib/*: help.lixin.spring.App
```
### (4). 项目下载
!["Maven结合Ant打包zip项目"](/assets/ant/spring-example.zip)
