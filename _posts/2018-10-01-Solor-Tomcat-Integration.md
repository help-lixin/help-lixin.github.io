---
layout: post
title: 'Solor和Tomcat集成'
date: 2018-10-01
author: 李新
tags: Solor
---

### (1). Solr和Tomcat集成

> 省略,请自行下载Tomcat和Solr(solr-7.7.3)

```
## 我的工作目录如下:
lixin-macbook:solr lixin$ pwd
/Users/lixin/Developer/solr
lixin-macbook:solr lixin$ ll
drwxr-xr-x  16 lixin  staff  512 11 14 13:43 apache-tomcat-8.5.54/
drwxr-xr-x@ 10 lixin  staff  320 11 13 23:56 solr-7.7.3/
```

### (2). 集成步骤
```
# 我的工作目录如下:
lixin-macbook:solr lixin$ pwd
/Users/lixin/Developer/solr

# 1.拷贝${SOLR}/server/solr-webapp/webapp 到 
#   ${TOMCAT}/webapps下,并重命名为:solr
lixin-macbook:solr lixin$ cp -rf solr-7.7.3/server/solr-webapp/webapp  apache-tomcat-8.5.54/webapps/solr


# 2.拷贝${SOLR}/server/lib/ext目录下所有的jar包 到
#   ${TOMCAT}/webapps/solr/WEB-INF/lib/ 目录下
lixin-macbook:solr lixin$ cp -rf solr-7.7.3/server/lib/ext/*.jar  apache-tomcat-8.5.54/webapps/solr/WEB-INF/lib/

# 3.拷贝${SOLR}/server/lib目录下以metrics开头的jar 到
#  ${TOMCAT}/webapps/solr/WEB-INF/lib/ 目录下
lixin-macbook:solr lixin$ cp solr-7.7.3/server/lib/metrics-* apache-tomcat-8.5.54/webapps/solr/WEB-INF/lib/


# 4.拷贝${SOLR}/server/lib目录下gmetric4j-1.0.7.jar 到
#  ${TOMCAT}/webapps/solr/WEB-INF/lib/  目录下
lixin-macbook:solr lixin$ cp solr-7.7.3/server/lib/gmetric4j-1.0.7.jar apache-tomcat-8.5.54/webapps/solr/WEB-INF/lib/


# 5.创建目录 ${TOMCAT}/webapps/solr/WEB-INF/classes
lixin-macbook:solr lixin$ mkdir apache-tomcat-8.5.54/webapps/solr/WEB-INF/classes

# 6. 拷贝 ${SOLR}/server/resources/ 目录下的日志配置信息 到
#  ${TOMCAT}/webapps/solr/WEB-INF/classes/  
lixin-macbook:solr lixin$ cp solr-7.7.3/server/resources/log4j2* apache-tomcat-8.5.54/webapps/solr/WEB-INF/classes/

# 7. 创建solr目录
lixin-macbook:solr lixin$ mkdir solr_home

# 8. 拷贝${SOLR}/server/solr 目录下所有文件 到
#    solr_home 目录下
lixin-macbook:solr lixin$ cp -rf solr-7.7.3/server/solr/*  solr_home/

# 9. 拷贝${SOLR}/contrib   到 solr_home 目录下
#    拷贝${SOLR}/dist      到 solr_home 目录下
lixin-macbook:solr lixin$ cp -rf solr-7.7.3/contrib solr_home/
lixin-macbook:solr lixin$ cp -rf solr-7.7.3/dist  solr_home/

# 10. 在solr_home目录下创建:core
lixin-macbook:solr lixin$ mkdir solr_home/core_example


lixin-macbook:solr lixin$ cp -rf solr_home/configsets/sample_techproducts_configs/conf     solr_home/core_example/

```
### (3). 修改配置文件(solr_home/core_example/conf/solrconfig.xml),75行处

!["solr_home/core_example/conf/solrconfig.xml"](/assets/solr/imgs/solr-tomcat.png)

### (4). 修改配置文件(apache-tomcat-8.5.54/webapps/solr/WEB-INF/web.xml)
> 增加XML节点,配置solr_home目录

```
<env-entry>
    <env-entry-name>solr/home</env-entry-name>
    <env-entry-value>/Users/lixin/Developer/solr/solr_home</env-entry-value>
    <env-entry-type>java.lang.String</env-entry-type>
</env-entry>
```
### (5). 修改配置文件(apache-tomcat-8.5.54/webapps/solr/WEB-INF/web.xml)
> 注释如下内容:

```
 <!-- Get rid of error message 
  <security-constraint>
    <web-resource-collection>
      <web-resource-name>Disable TRACE</web-resource-name>
      <url-pattern>/</url-pattern>
      <http-method>TRACE</http-method>
    </web-resource-collection>
    <auth-constraint/>
  </security-constraint>
  <security-constraint>
    <web-resource-collection>
      <web-resource-name>Enable everything but TRACE</web-resource-name>
      <url-pattern>/</url-pattern>
      <http-method-omission>TRACE</http-method-omission>
    </web-resource-collection>
  </security-constraint>
   -->
```

### (7).修改tomcat日志输出文件
> apache-tomcat-8.5.54/webapps/solr/WEB-INF/classes/log4j2.xml   
> 把xml配置中的"${sys:solr.log.dir}"替换成指定的目录   

```
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
  -->

<Configuration>
  <Appenders>

    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout>
        <Pattern>
          %d{yyyy-MM-dd HH:mm:ss.SSS} %-5p (%t) [%X{collection} %X{shard} %X{replica} %X{core}] %c{1.} %m%n
        </Pattern>
      </PatternLayout>
    </Console>

    <RollingFile
        name="RollingFile"
        fileName="/Users/lixin/Developer/solr/apache-tomcat-8.5.54/logs/solr.log"
        filePattern="/Users/lixin/Developer/solr/apache-tomcat-8.5.54/logs/solr.log.%i" >
      <PatternLayout>
        <Pattern>
          %d{yyyy-MM-dd HH:mm:ss.SSS} %-5p (%t) [%X{collection} %X{shard} %X{replica} %X{core}] %c{1.} %m%n
        </Pattern>
      </PatternLayout>
      <Policies>
        <OnStartupTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="32 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="10"/>
    </RollingFile>

    <RollingFile
        name="SlowFile"
        fileName="/Users/lixin/Developer/solr/apache-tomcat-8.5.54/logs/solr_slow_requests.log"
        filePattern="/Users/lixin/Developer/solr/apache-tomcat-8.5.54/logs/solr_slow_requests.log.%i" >
      <PatternLayout>
        <Pattern>
          %d{yyyy-MM-dd HH:mm:ss.SSS} %-5p (%t) [%X{collection} %X{shard} %X{replica} %X{core}] %c{1.} %m%n
        </Pattern>
      </PatternLayout>
      <Policies>
        <OnStartupTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="32 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="10"/>
    </RollingFile>

  </Appenders>
  <Loggers>
    <Logger name="org.apache.hadoop" level="warn"/>
    <Logger name="org.apache.solr.update.LoggingInfoStream" level="off"/>
    <Logger name="org.apache.zookeeper" level="warn"/>
    <Logger name="org.apache.solr.core.SolrCore.SlowRequest" level="info" additivity="false">
      <AppenderRef ref="SlowFile"/>
    </Logger>

    <Root level="info">
      <AppenderRef ref="RollingFile"/>
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```
### (6). 启动Tomcat
```
lixin-macbook:solr lixin$ ./apache-tomcat-8.5.54/bin/startup.sh 
```
### (7). 访问Tomcat
> http://localhost:8080/solr/index.html

!["Solr Dashboard"](/assets/solr/imgs/solr_dashboard.png)
